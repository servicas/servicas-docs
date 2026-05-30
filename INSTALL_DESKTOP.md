# Servicas Desktop (Tauri 2) — Install & Build

The Servicas desktop app is a [Tauri 2](https://v2.tauri.app/) shell that
renders the same React/Vite web UI as the browser app. The JS side is
already wired (see `apps/desktop/`, `src-tauri/tauri.conf.json`,
`src-tauri/Cargo.toml`). The only missing piece on a fresh dev machine is
the Rust toolchain.

## Prerequisites

### All platforms

- **Node.js 20+** and **npm 10+** (already required by the web app).
- **Rust 1.77+** via rustup:

  ```sh
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
  ```

  After install: `rustup default stable && rustc --version`.

### macOS

- Xcode Command Line Tools — already required for git on macOS. Verify
  with `xcode-select -p`. Install if missing: `xcode-select --install`.

### Windows

- [Microsoft Visual Studio C++ Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/)
  with the **Desktop development with C++** workload.
- [WebView2 runtime](https://developer.microsoft.com/microsoft-edge/webview2/)
  — preinstalled on Windows 11; install the Evergreen Bootstrapper on
  Windows 10.

### Linux (Debian / Ubuntu)

```sh
sudo apt update
sudo apt install -y \
  libwebkit2gtk-4.1-dev \
  libssl-dev \
  librsvg2-dev \
  pkg-config \
  build-essential \
  curl \
  wget \
  file
```

For other distros see the [Tauri 2 Linux prerequisites](https://v2.tauri.app/start/prerequisites/#linux).

## Dev workflow

From the repo root:

```sh
npm install
npm run dev:desktop
```

`npm run dev:desktop` calls `tauri dev`, which:

1. Runs `npm run dev -- --port 1420 --strictPort --host localhost` (Vite
   dev server on port 1420).
2. Compiles `src-tauri/` once via Cargo.
3. Opens a native window pointing at `http://localhost:1420`.

Hot module reload from Vite works inside the Tauri window the same way it
does in a browser.

## Production build

```sh
npm run build:desktop
```

This runs `npm run build` first (producing `dist/`), then `cargo build
--release` on `src-tauri/`. Artifacts land in
`src-tauri/target/release/bundle/`:

| Host | Output |
| --- | --- |
| macOS | `bundle/macos/Servicas.app`, `bundle/dmg/Servicas_<version>_<arch>.dmg` |
| Windows | `bundle/msi/Servicas_<version>_<arch>_en-US.msi`, `bundle/nsis/Servicas_<version>_<arch>-setup.exe` |
| Linux | `bundle/deb/servicas-desktop_<version>_<arch>.deb`, `bundle/appimage/servicas-desktop_<version>_<arch>.AppImage`, `bundle/rpm/...` |

## Icons

Real icon artwork must replace the placeholders described in
[`../src-tauri/icons/README.md`](../src-tauri/icons/README.md) before
shipping. Regenerate the full icon set from a 1024×1024 PNG:

```sh
npm run tauri:icon -- ./docs/brand/servicas-icon-1024.png
```

## Code-signing TODO

These are required before publishing user-facing builds and are NOT yet
configured in this repo:

- **macOS**: Apple Developer ID + notarization (`xcrun notarytool submit`),
  hardened runtime entitlements, and stapling. Set
  `APPLE_CERTIFICATE`, `APPLE_CERTIFICATE_PASSWORD`, `APPLE_ID`,
  `APPLE_PASSWORD`, and `APPLE_TEAM_ID` in CI and reference them from
  `tauri.conf.json -> bundle.macOS`.
- **Windows**: Authenticode certificate (EV recommended for SmartScreen
  reputation), signed via `signtool` or `tauri.conf.json -> bundle.windows
  -> certificateThumbprint` / `digestAlgorithm`.
- **Linux**: AppImages are typically unsigned; `.deb` / `.rpm` can be
  signed with GPG via `dpkg-sig` / `rpmsign`.

## Where things live

| Concern | Path |
| --- | --- |
| Tauri config (window, CSP, bundle) | `src-tauri/tauri.conf.json` |
| Rust app entry | `src-tauri/src/main.rs`, `src-tauri/src/lib.rs` |
| Cargo manifest | `src-tauri/Cargo.toml` |
| Capability / permissions | `src-tauri/capabilities/default.json` |
| Icons | `src-tauri/icons/` |
| Web ↔ native bridge | `apps/desktop/src/tauriBridge.ts` |
| Desktop shell entry | `apps/desktop/src/DesktopShell.tsx`, `apps/desktop/main.tsx` |
