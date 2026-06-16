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

## In progress: GPU acceleration troubleshooting (paused 2026-06-16)

**Goal:** get hardware OpenGL (D3D12 renderer) instead of `llvmpipe` (CPU software) for a smooth KiCad 3D viewer. Optional polish — `llvmpipe` is fine for schematic + 2D layout.

**Where we are: the problem is WSL/Windows-side, NOT the container.** The container is wired correctly and faithfully inherits a host that can't do GPU GL.

Confirmed:
- Container config is correct and in place: compose mounts `/dev:/dev` (USB + brings `/dev/dxg`) and `/usr/lib/wsl:/usr/lib/wsl:ro`, plus `LD_LIBRARY_PATH=/usr/lib/wsl/lib`. The `d3d12_dri.so` driver ships in `libgl1-mesa-dri`. Container checklist steps 1–6 all passed.
- **Host (WSL) itself has no GPU GL** — this is the root cause:
  - `glxinfo -B` on the host → `MESA: error: ZINK: failed to choose pdev` / `glx: failed to create drisw screen` / `Dropped Escape call ... 0x03007703`.
  - `vulkaninfo --summary` on the host → only `llvmpipe` (lavapipe = software Vulkan). So **no hardware GPU is reaching WSL at all**, neither GL (d3d12) nor Vulkan (dzn). Everything falls back to software.

**Next steps (host/Windows side) — pick up here after the WSL restart:**
1. **Windows PowerShell:** `wsl --version`, then `wsl --update`, then `wsl --shutdown`. Reopen WSL, re-test host: `glxinfo -B | grep -i renderer`.
2. If still software → **update the Windows vendor GPU driver** (NVIDIA/AMD/Intel — the WSL D3D12/dzn drivers ship inside it). Reboot Windows, `wsl --shutdown`, re-test.
3. If still software after both → GPU/Windows likely too old for GPU-PV (needs WDDM 2.9+, Win10 21H2+/Win11); software rendering is the ceiling on this machine — stop here and live with `llvmpipe`.
4. **Once the host `glxinfo` shows `D3D12 (…)`**, the container inherits it automatically (config already correct) — just verify inside the container: `glxinfo -B | grep -i renderer`.

**Info still to gather (helps decide if worth pushing):** GPU model, Windows version, `wsl --version` output, `ls -l /dev/dxg` and `uname -r` on the host.

**Resume note:** restarting all of WSL (`wsl --shutdown`) will also take down the Claude sandbox container. Everything persists on the `claude-repos` + `claude-home` docker volumes; reconnect with `claude -c` from `/claude-repos/ultranetMonitor`. Full GPU details/decode are in [kicad-startup.md](kicad-startup.md) "GPU acceleration".
