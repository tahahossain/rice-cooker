# AGENTS.md

## Cursor Cloud specific instructions

Rice Cooker is a single Linux desktop app (not a monorepo): an Electron 41 + React 19 + Vite renderer whose main process spawns a Rust CLI backend (`rice-cooker-backend`) and reads NDJSON events from it. There is no web server, database, docker, or network port. Standard commands live in `package.json` scripts and `backend/Cargo.toml`; prefer those over duplicating them here.

### Toolchain gotcha (important)
- The Rust backend uses `edition = "2024"` (`backend/Cargo.toml`), which requires Rust >= 1.85. The base image's default `rustc` may be older (1.83), which fails with `feature edition2024 is required`. The update script runs `rustup default stable` to fix this; if you ever hit that error, run `rustup default stable` (or `rustup toolchain install stable`) yourself.

### Running / services
- Dev: `npm run dev` builds the backend (`cargo build --manifest-path backend/Cargo.toml`) then launches `electron-vite dev` (Vite renderer + Electron). It is a GUI app — it needs a display. An X display is available at `DISPLAY=:1`.
- Do NOT set `XDG_SESSION_TYPE=wayland` when launching: the app forces Electron's Wayland ozone backend when that var is `wayland`, and there is no Wayland display here, so the window won't render. Leaving it unset makes Electron use X11 (`:1`), which works. The `dbus` connection errors in the Electron log are harmless in this container.

### Environment gate & how to reach the UI without Arch/Hyprland
- On startup the app calls `environment:check`, which only reports "supported" on Arch Linux + Wayland + Hyprland + `quickshell` in PATH. On this VM it will show the boot/compatibility screen ("oh no, this rice won't cook").
- To reach the real catalog UI, use the built-in force-boot: focus the window, press `w` to select the **ENTER** boot option, then hold the `e` (or Enter) key for ~4 seconds. This bypasses the gate and shows the rice catalog, which is populated live by the Rust backend's `list` command over IPC. Navigate rices with `w`/`s` (up/down).
- Actually previewing/installing a rice (Enter on a rice) shells out to `git`, `quickshell`, `pacman`/AUR helper, and polkit — none of which exist here — so those flows will hit the failure overlay. Browsing the catalog is the extent of end-to-end functionality without a real Arch+Hyprland+Quickshell host.

### Lint / test / build / typecheck
- Typecheck: `npm run typecheck` (runs `typecheck:node` + `typecheck:web`). There is no separate ESLint config in this repo.
- Rust tests (unit + CLI snapshot via `assert_cmd`/`insta`): `cargo test --manifest-path backend/Cargo.toml`.
- Backend CLI can be exercised directly, e.g. `./backend/target/debug/rice-cooker-backend --catalog backend/catalog.toml list`.
- Production bundle: `npm run build` (`electron-vite build` → `out/`). Not needed for dev.

### Useful env vars (from `electron/main/index.ts`)
- `RICE_COOKER_BACKEND` — path to the backend binary; `RICE_COOKER_CATALOG` — path to a `catalog.toml`; `RICE_SCALE` — window scale; `RICE_CAPTURE_OUT` / `RICE_CAPTURE_*` — headless screenshot-capture hooks used by the renderer.
