# Plans / next steps

_Last updated 2026-06-15. Context: setting up KiCad for PCB work. Full rationale and findings are in [kicad-startup.md](kicad-startup.md)._

## Decision made

Run KiCad **inside the Linux devenv container via WSLg** (not as a Windows app), so all files (projects, libraries, preferences) stay in the container/volume world. Verified working in the live container today.

## Status

- ✅ Display path verified: WSLg X11 socket + `DISPLAY=:0.0` are already passed into the container. `xeyes` and the KiCad GUI both showed on the Windows desktop.
- ✅ KiCad **9.0.9** (official PPA) and **easyeda2kicad 1.0.1** (via pipx) installed and confirmed — **but ad-hoc in the running container, so they vanish on rebuild.** Must be baked into the Dockerfile (below).
- ✅ `install-kicad` script written (this repo), follows the existing `COPY script + RUN script` convention. Assumes `pipx` is provided by `install-base-utils`.

## Next steps

1. ✅ **Add `pipx` to `install-base-utils`** — done (package is `pipx`, not `python3-pipx`).
2. ✅ **Wire `install-kicad` into the devenv Dockerfile** (`COPY install-kicad` + `RUN ./install-kicad`) — done.
3. ✅ **Rebuild the devenv image** — done; container now builds from the Dockerfile with KiCad included (2026-06-16).
4. **Install the JLCPCB plugins** from inside KiCad (Plugins → Plugin and Content Manager): "JLCPCB tools" (Bouni) and/or "Fabrication Toolkit". These are per-user-config/GUI, intentionally NOT in the install script.
5. **Set up the `kicad-shared` library repo** (hybrid path scheme: one env var → absolute path, everything references `${KICAD_USER_LIB}`). Checklist:

   - [ ] **Create the repo + initial layout.** Match `easyeda2kicad`'s output convention: a shared base name with three siblings (symbol file, footprint dir, 3D-model dir). Start with one base name `jlcpcb` for tool-pulled parts:
     ```
     kicad-shared/
     ├── README.md            # purpose, layout, the ${KICAD_USER_LIB} convention, lib-table entries to add
     ├── .gitignore           # fp-info-cache, *-bak, *.bak, *~, _autosave-*, .DS_Store
     ├── jlcpcb.kicad_sym     # symbol library (start empty / created on first pull)
     ├── jlcpcb.pretty/       # footprint library (dir of .kicad_mod); add .gitkeep
     └── jlcpcb.3dshapes/     # 3D models (.wrl/.step); add .gitkeep
     ```
     (Add a second base name later for hand-curated parts, e.g. `custom.kicad_sym` / `custom.pretty/` / `custom.3dshapes/`.) Clone it to a stable absolute path in the container, e.g. `/claude-repos/kicad-shared`.
   - [ ] **Define `KICAD_USER_LIB` as a container OS env var** in the devenv compose (NOT via Configure Paths) → the absolute clone path. This is the single point of truth; KiCad reads OS env vars and they take precedence, so no config syncing is needed across machines.
   - [ ] **Add global library-table entries** (Preferences → Manage Symbol/Footprint Libraries, or edit `~/.config/kicad/9.0/{sym,fp}-lib-table`):
     - symbol: nickname `jlcpcb`, uri `${KICAD_USER_LIB}/jlcpcb.kicad_sym`
     - footprint: nickname `jlcpcb`, uri `${KICAD_USER_LIB}/jlcpcb.pretty`
   - [ ] **Verify 3D-model paths resolve via the var.** After the first `easyeda2kicad` pull, open a footprint and check the 3D model path it wrote — make sure it's `${KICAD_USER_LIB}/jlcpcb.3dshapes/...` and not a baked-in absolute path. Fix the tool invocation / footprint if it baked an absolute path (this is the one thing the hybrid scheme is protecting against).
   - [ ] **(doc)** Once it's working, write a "library layout + config locations + path setup" section into [kicad-startup.md](kicad-startup.md): the three XDG dirs (`~/.config/kicad/9.0`, `~/.local/share/kicad/9.0`, `~/.cache/kicad`), what's shareable vs the cache (never share), and the `KICAD_USER_LIB` convention.

   Start with a single central clone; move to git submodules per-project only if version pinning is later needed.
6. **Test the full JLCPCB flow** — pull a part into the shared library with `easyeda2kicad --full --lcsc_id <Cxxxx> --output ${KICAD_USER_LIB}/jlcpcb`, place it from the `jlcpcb` libs, confirm the 3D model shows in the viewer, and generate JLCPCB fab files via the plugin.

## Pinned: GPU acceleration — works, but deferring the decision (2026-06-16)

**Goal:** hardware OpenGL (D3D12 renderer) instead of `llvmpipe` (CPU software) for a smooth KiCad 3D viewer. Optional polish — `llvmpipe` is fine for schematic + 2D layout.

**Status: hardware GL is now reachable in the container. Decision to enable it is parked until there's a real board to judge it against.**

What got it working:
- The original root cause (host WSL had no GPU GL) was fixed by **`wsl --update` to 2.7.8.0** alone — no Windows vendor driver update needed. Host `glxinfo -B` now shows `OpenGL renderer string: D3D12 (AMD Radeon(TM) 860M Graphics)`, Mesa 24.0.9. (The `ZINK: failed to choose pdev` / `failed to create drisw screen` lines are harmless fallback noise; the final D3D12 line is what counts.)
- Container wiring confirmed all present: `/dev/dxg` (crw-rw-rw-), `LD_LIBRARY_PATH=/usr/lib/wsl/lib`, WSL libs mounted (`libd3d12.so`, `libd3d12core.so`, `libdxcore.so`), and `d3d12_dri.so` at `/usr/lib/x86_64-linux-gnu/dri/d3d12_dri.so`. Container Mesa is **25.2.8** (newer than host's 24.0.9).
- **But the container defaulted to `llvmpipe` anyway.** Neither host nor container has `/dev/dri`, and WSL's hardware-GL path doesn't use DRI3 — so the `"screen 0 does not appear to be DRI3 capable"` message is a red herring. The real issue: stock Mesa 25.2.8 picks `llvmpipe` for the no-DRI3 present path, while host's WSL-patched Mesa prefers d3d12.
- **Fix: `GALLIUM_DRIVER=d3d12`.** Setting it flips the container to `D3D12 (AMD Radeon 860M)`. (If ever needed, back it up with `MESA_D3D12_DEFAULT_ADAPTER_NAME=AMD` and `MESA_LOADER_DRIVER_OVERRIDE=d3d12`.)

**Why we're not enabling it yet:** with `GALLIUM_DRIVER=d3d12`, **glxgears FPS actually drops** vs llvmpipe. Expected — with no DRI3, the GPU frame must be read back to system memory and blitted to the WSLg X server every frame; for a trivial scene that fixed present/readback cost dominates and software (which renders straight into the shared buffer) wins. Hardware is expected to pull ahead only on a real, heavy 3D scene (a populated board). With no KiCad project yet, there's nothing meaningful to compare, so the perf trade-off can't be judged.

**Resume here once there's a real board to view:**
1. Open a populated board's 3D viewer two ways and compare feel: `GALLIUM_DRIVER=d3d12 kicad` vs plain `kicad` (llvmpipe). Optionally benchmark with `glmark2` (`GALLIUM_DRIVER=d3d12 glmark2` vs `LIBGL_ALWAYS_SOFTWARE=1 glmark2`) — a heavy scene, unlike glxgears.
2. If d3d12 is clearly smoother → bake `GALLIUM_DRIVER=d3d12` into the devenv compose env (and note the glxgears caveat in [kicad-startup.md](kicad-startup.md)). If not → leave it on llvmpipe; this machine's present-path overhead isn't worth it.

**Resume note:** restarting all of WSL (`wsl --shutdown`) also takes down the Claude sandbox container. State persists on the `claude-repos` + `claude-home` docker volumes; reconnect with `claude -c` from `/claude-repos/ultranetMonitor`. Full GPU details/decode are in [kicad-startup.md](kicad-startup.md) "GPU acceleration".
