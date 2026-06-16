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
5. **Create the shared-library repo** — a dedicated git repo with `symbols/` `footprints/` `3dmodels/`. Reference it from KiCad via an env var (e.g. `MYLIB`) set in Preferences → Configure Paths, so library tables stay portable. Start with a single central clone; move to git submodules per-project only if version pinning is needed.
6. **Test the full JLCPCB flow** — pull a part with `easyeda2kicad --full --lcsc_id <Cxxxx>` into the shared library, place it, and generate JLCPCB fab files via the plugin.

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
