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

## Optional / later

- **GPU acceleration:** current GL is `llvmpipe` (CPU software rendering) — fine for 2D, slow for the 3D viewer on complex boards. To use the WSL GPU, pass `--device /dev/dxg`, `-v /usr/lib/wsl:/usr/lib/wsl`, and `-e LD_LIBRARY_PATH=/usr/lib/wsl/lib` into the container at run time. Only do this if the 3D viewer becomes annoying.
