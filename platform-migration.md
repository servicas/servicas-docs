# Servicas platform migration

Servicas ships one React 19 + Vite codebase across **three shells** —
web (PWA), mobile (Capacitor → iOS + Android), desktop (Tauri → macOS,
Windows, Linux). Same `src/`, three thin shells under `apps/`, a
single `packages/shared-platform` that fans out to the right native
API at runtime.

## Status

| Shell    | Status                    | Runnable today?                                   |
| -------- | ------------------------- | ------------------------------------------------- |
| Web      | Production                | Yes — `npm run dev`, deploys to Cloud Run via CI. |
| Mobile   | Code-complete             | Needs Xcode + CocoaPods + Android Studio + SDK.   |
| Desktop  | Code-complete             | Needs Rust 1.77+ via rustup.                      |

Both native shells are **structurally production-ready** — every config,
script, plugin wrapper, and shared-platform abstraction is wired up.
What's missing on this machine are the native toolchains (large, slow,
license-gated installs that have to be done out-of-band).

## What's in `apps/`

```
apps/
  web/      — Vite + React PWA. Production. Cloud Run target.
  mobile/   — Capacitor wrapper around the same <App />. iOS + Android.
  desktop/  — Tauri 2 wrapper around the same <App />. macOS, Windows, Linux.
```

Each shell renders the **exact same** `<App />` from `src/app/App.tsx`.
Platform-specific behavior is feature-detected via
`@servicas/shared-platform`, not duplicated.

## What's in `packages/`

```
packages/
  shared-auth/      — Auth client + session helpers.
  shared-config/    — Runtime config loader.
  shared-domain/    — Cross-portal domain logic.
  shared-platform/  — Native-aware wrappers:
    runtime         — isNative(), getPlatform()
    secureStore     — Keychain / SharedPreferences on native; localStorage on web.
    network         — @capacitor/network on native; navigator.onLine on web.
    haptics, share, camera
  shared-ui/        — Reusable UI primitives.
```

## Mobile (Capacitor)

What's in place:

- `@capacitor/core`, `@capacitor/ios`, `@capacitor/android`, plus 10
  plugins: app, preferences, network, camera, status-bar,
  splash-screen, haptics, share, keyboard, filesystem.
- [`capacitor.config.ts`](../capacitor.config.ts) — full prod config
  (splash, status bar, keyboard, schemes).
- [`apps/mobile/`](../apps/mobile/) — `index.html`, `main.tsx`,
  `renderMobileShell.tsx`, `MobileShell.tsx`. Native polish (splash hide,
  status bar style, keyboard mode) runs only when `isNative()`; web
  build is unaffected.
- [`packages/shared-platform/`](../packages/shared-platform/) — workspace
  package wrapping every native API.
- `BookingMediaPanel` surfaces native camera buttons when `isNative()`
  is true. Web continues to use `<input type="file">`.
- npm scripts: `dev:mobile`, `build:mobile`, `cap:sync`,
  `cap:open:ios`, `cap:open:android`, `cap:add:ios`, `cap:add:android`.

What's needed to actually run on a device:

1. Install Xcode (App Store) + CocoaPods (`brew install cocoapods`) +
   Android Studio + Android SDK. Details in
   [`INSTALL_MOBILE.md`](./INSTALL_MOBILE.md).
2. `npm install && npm run build && npm run cap:add:ios && npm run cap:add:android`.
3. `npm run cap:open:ios` or `cap:open:android`.

## Desktop (Tauri 2)

What's in place:

- `@tauri-apps/cli`, `@tauri-apps/api`, plus 5 plugins:
  shell, dialog, fs, os, notification.
- [`apps/desktop/`](../apps/desktop/) — React shell entry
  (`DesktopShell`, `renderDesktopShell`, `tauriBridge` with dynamic
  `@tauri-apps/*` imports so the web bundle stays clean).
- [`src-tauri/`](../src-tauri/) — Rust binary registering the 5 plugins.
  `Cargo.toml`, `build.rs`, `src/main.rs`, `src/lib.rs`,
  `capabilities/default.json` all written by hand against the Tauri 2
  API (not yet compiled — Rust toolchain absent on this machine).
- [`src-tauri/tauri.conf.json`](../src-tauri/tauri.conf.json) — full
  Tauri 2 config including window metadata, CSP, bundle targets, and
  identifier `com.servicas.desktop`.
- npm scripts: `tauri`, `dev:desktop`, `build:desktop`, `tauri:icon`.

What's needed to actually run a desktop binary:

1. Install Rust 1.77+ via rustup. Details in
   [`INSTALL_DESKTOP.md`](./INSTALL_DESKTOP.md).
2. `npm install && npm run dev:desktop` (opens a dev window with HMR).
3. `npm run build:desktop` produces `.dmg` / `.msi` / `.deb` /
   `.AppImage` under `src-tauri/target/release/bundle/`.
4. Replace placeholder icons in `src-tauri/icons/` before a real release.
5. Code-signing: macOS notarization + Windows Authenticode (see
   `INSTALL_DESKTOP.md` → Code-signing TODO).

## Next slices (optional polish)

- **Push notifications** — `@capacitor/push-notifications` wired into
  the existing notification-service backend. Needs APNs + FCM.
- **Deep links** — `@capacitor/app` URL handlers for booking links.
- **CI for native artifacts** — GitHub Actions workflows to produce
  signed `.ipa` / `.aab` / `.dmg` on `workflow_dispatch`. Separate from
  the existing web/cloud-run pipeline.
- **Async secureStore migration** — `authService` currently uses
  sessionStorage. On native that's the WKWebView's sandbox (acceptable
  for pilot, not Keychain-grade). Switching to `secureStore` is a
  multi-caller async migration that's deferred until past the pilot.
