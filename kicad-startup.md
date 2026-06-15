# KiCad startup

This project is going to involve some pcb creation.  I've used KiCad in the past, but don't remember it very well.  Lately I've been using easyeda, and I've been very impressed.  I especially like the tight integration with JLCPCB, since I now use JLCPCB's board and assembly services almost exclusively.

But I'm getting more and more concerned with my projects being stored in the cloud, and with easyeda switching versions in incompatible ways (std to pro).  It's been on my mind lately that perhaps I should be moving to something more under my control, like KiCad.  It'll let me store my files in version control, organize them as I want, etc.

But I still want that easy integration with jlcpcb - all the parts available with schematics and footprints, easy 3d views, easy pcba ordering.  I suspect there are kicad libraries or plugins or something that help with all that.  I'd like to learn how to use those effectively.

I also anticipate multiple projects using kicad, across multiple repos.  I'm not sure if there are local artifacts (common libraries or whatnot) that I should store in some kind of central repo.  Curious what techniques people use.

And would welcome some thoughts about getting this into my own setup.  My dev env is a docker container running in wsl, using docker volumes for storage.  That's where all my repos are kept, and really anything I want to keep around permanently.  I *think* those can be mounted directly from windows, which is where KiCad would be running.  In fact, if kicad keeps any other kinds of files around, like preferences, etc., I'd like to put those into the docker volumes rather than windows.

---

# Setup plan & findings

_Written 2026-06-15. Current KiCad is 9.0 (released Feb 2025)._

## TL;DR recommendation

- **Run KiCad *inside* the Linux dev container via WSLg**, not as a Windows app. This keeps every file (projects, libraries, preferences) in the container/volume world you already trust, with no 9P network-mount fragility. **This was tested in this container and works** (see "Verified" below).
- **JLCPCB workflow** = `easyeda2kicad` (pull per-part symbol/footprint/3D model from any LCSC part number) + the **`kicad-jlcpcb-tools`** plugin (assign LCSC part numbers, search the JLCPCB catalog, generate Gerber/BOM/CPL for PCBA ordering).
- **Shared parts across repos** = one dedicated git repo of symbols/footprints/3D models, referenced from KiCad via an **environment variable** so library tables stay portable.
- **Bake KiCad into the devenv Dockerfile** (snippet below) so the toolchain is reproducible.

## Why run KiCad in the container instead of on Windows (the 9P gotcha)

- **Docker *named volumes*** live inside the WSL2 VM's virtual disk. Windows *can* reach them at `\\wsl$\docker-desktop-data\...`, but Docker Desktop discourages this and it breaks across updates. **Bind mounts** into a WSL distro folder are reachable reliably at `\\wsl.localhost\<distro>\...`.
- Running a **Windows GUI KiCad against files on a `\\wsl$` 9P path** works but is slow on project load, and the lock/temp files KiCad writes over 9P have caused corruption reports. Not worth the risk for active project files.
- Running KiCad **natively in Linux (the container)** sidesteps all of this — KiCad sits in the same filesystem as the repos. WSLg forwards the GUI to the Windows desktop automatically.

## Verified in *this* container (2026-06-15)

Probed the running devenv container (Ubuntu 24.04, root):

- ✅ `DISPLAY=:0.0` set; WSLg X11 socket present at `/tmp/.X11-unix/X0`. `xset q` reaches the server; an `xeyes` window displayed on the Windows desktop. So **the container → WSLg → Windows display path already works** with no extra config.
- ✅ OpenGL available: `glxinfo` reports GL 4.5, direct rendering. KiCad's editing canvas (GAL) and 3D viewer will run.
- ⚠️ **Renderer is `llvmpipe` (Mesa software rasterizer), not the GPU.** The container has the WSLg *X socket* but not GPU passthrough. 2D editing is fine; the **3D viewer will be CPU-rendered and slow on complex boards.** Fix when needed — see "GPU acceleration" below.
- Note: only the X11 socket is passed in, not the full WSLg Wayland mount (`/mnt/wslg`, `WAYLAND_DISPLAY`, `PULSE_SERVER` are unset). KiCad runs fine over X11/Xwayland, so this is OK.

## Bake KiCad into the devenv Dockerfile

KiCad 9 isn't in Ubuntu 24.04's base repo (that's 8.0) — use the official PPA:

```dockerfile
# KiCad 9 (official PPA; Ubuntu repo only ships 8.0)
RUN apt-get update && apt-get install -y --no-install-recommends software-properties-common \
 && add-apt-repository -y ppa:kicad/kicad-9.0-releases \
 && apt-get update && apt-get install -y --no-install-recommends kicad kicad-libraries \
 && rm -rf /var/lib/apt/lists/*
```

- `kicad-libraries` pulls the official symbol/footprint/3D-model libraries (large; drop it if you only use your own libs).
- KiCad is ~1.5 GB installed — expect a heavier image. Consider a dedicated `kicad` build stage/image if you don't want it in every container.
- The display works because the WSLg X11 socket and `DISPLAY` are passed into the container at *run* time (already the case here). If you ever launch the container yourself, ensure it gets `-e DISPLAY=:0` and `-v /tmp/.X11-unix:/tmp/.X11-unix`.

## GPU acceleration (optional, do later if 3D viewer is too slow)

To replace `llvmpipe` with the WSL d3d12 GPU driver, pass these from the WSL host into the container at run time:

```
--device /dev/dxg
-v /usr/lib/wsl:/usr/lib/wsl
-e LD_LIBRARY_PATH=/usr/lib/wsl/lib
```

Then `glxinfo` should report a `D3D12` renderer instead of `llvmpipe`. Not required to get started.

## JLCPCB / LCSC integration

1. **`easyeda2kicad`** — pull a part (symbol + footprint + 3D `.step`) by LCSC number:
   ```
   pip install easyeda2kicad
   easyeda2kicad --full --lcsc_id C2040
   ```
   Point it at your shared library file; it appends. This covers "all the parts available with schematics and footprints."
2. **`kicad-jlcpcb-tools`** (Bouni) — install from KiCad's Plugin & Content Manager. Local searchable copy of JLCPCB's catalog, assign LCSC part numbers in-layout with stock/price, one-click **Gerber + BOM + CPL** in JLCPCB's assembly-upload format. Closest match to EasyEDA's "order PCBA" button.
3. **Fabrication Toolkit** (bennymeg) — simpler alternative that just generates JLCPCB production files. Use instead of Bouni's if you don't need the integrated parts search.
4. **3D views** are native (KiCad 3D viewer); the `.step` models from easyeda2kicad drop right in.

Start with easyeda2kicad + Bouni's plugin.

## Shared libraries across repos

- One dedicated **git repo** for shared libs: `symbols/`, `footprints/`, `3dmodels/`. easyeda2kicad dumps reusable parts here.
- Reference it via a KiCad **environment variable** (e.g. `MYLIB`) set in *Preferences → Configure Paths*, not absolute paths — keeps the global `sym-lib-table`/`fp-lib-table` portable across machines/containers.
- Bring it into each project as a **git submodule** (per-project version pinning) or a single **central clone** at the `MYLIB` path (simpler, one source of truth). Start with the central clone; move to submodules only if you need pinning.
- **Convention:** one-off parts → project-local library (travels with the board); anything used twice → graduates to the shared repo.

## Preferences in the volume (not Windows)

KiCad honors `KICAD_CONFIG_HOME`. Default is `~/.config/kicad/9.0/` in Linux. Since KiCad runs *in the container*, its config already lands in the container/volume filesystem by default — no Windows footprint, no relocation hack needed. (If you ever did run KiCad on Windows, you'd set `KICAD_CONFIG_HOME` to redirect `%APPDATA%\kicad\9.0\`, but with Option A that's moot.)

## Open items / next steps

- [x] Confirm `xeyes` was visible on the Windows desktop (display path sanity check). **Confirmed 2026-06-15.**
- [x] Finish KiCad install + launch `kicad` to confirm the GUI comes up. **KiCad 9.0.9 installed from PPA; GUI launched and confirmed visible on the Windows desktop 2026-06-15.**
- [ ] Add the Dockerfile snippet to the devenv build so it's reproducible.
- [ ] Create the shared-library repo and wire up the `MYLIB` env var.
- [ ] Install easyeda2kicad + the JLCPCB plugin and do a test part pull.

