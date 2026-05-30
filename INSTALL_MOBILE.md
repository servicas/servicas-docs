# Servicas mobile (Capacitor) — install & build

The Servicas mobile app is the **same React/Vite codebase** as the web app,
wrapped in a native iOS/Android WebView by [Capacitor](https://capacitorjs.com).
No React Native. No Swift/Kotlin UI. Just a thin native shell around `src/`.

## 1. Required toolchain (one-time, per developer)

Pure JavaScript dependencies are already covered by `npm install`. The
toolchains below are only needed when you want to **build the native
project** (i.e. produce an `.ipa`/`.apk` or open Xcode/Android Studio).

### iOS

| Tool                    | Why                                              | Install                                                       |
| ----------------------- | ------------------------------------------------ | ------------------------------------------------------------- |
| Xcode (full IDE)        | iOS SDK, simulator, code signing                 | App Store → "Xcode"                                           |
| Xcode command-line tools | `xcodebuild`, `simctl`                          | `xcode-select --install`                                      |
| CocoaPods               | Capacitor uses CocoaPods to vendor native pods   | `brew install cocoapods` (or `sudo gem install cocoapods`)    |
| Ruby ≥ 2.7              | Required by CocoaPods                            | Usually shipped with macOS                                    |

### Android

| Tool                | Why                              | Install                                              |
| ------------------- | -------------------------------- | ---------------------------------------------------- |
| Android Studio      | SDK manager, emulator, Gradle    | https://developer.android.com/studio                 |
| Java JDK 17         | Required by Gradle 8             | `brew install --cask zulu@17` (or Adoptium)          |
| Android SDK         | Build-tools, platform-tools      | Installed automatically by Android Studio            |
| `ANDROID_HOME`      | Lets Gradle find the SDK         | Add `export ANDROID_HOME=$HOME/Library/Android/sdk` to your shell rc |

## 2. First-time native scaffolding

After the toolchain is installed, generate the native projects:

```bash
npm install
npm run build                # produces dist/ that Capacitor will copy
npm run cap:add:ios          # creates ./ios/, runs `pod install`
npm run cap:add:android      # creates ./android/, configures Gradle
```

The `ios/` and `android/` folders are **not** checked into git. Each
developer regenerates them with `cap add`.

## 3. Day-to-day dev workflow

There are two practical paths.

### Path A — debug the web app on a real device

Fast feedback loop, no native build required:

```bash
npm run dev:mobile     # vite serves on 0.0.0.0:5173
# Find your laptop IP (e.g. ifconfig | grep 'inet ') and open
# http://<your-ip>:5173 on the phone's browser.
```

### Path B — full native build

Required to test native plugins (camera, status bar, share sheet, etc.):

```bash
npm run build:mobile        # vite build + cap sync (copies dist/ into native projects)
npm run cap:open:ios        # opens Xcode → Run on simulator/device
# or
npm run cap:open:android    # opens Android Studio → Run
```

After the first `cap:add:*`, subsequent iterations only need
`npm run build:mobile` followed by Run from the IDE — no re-add.

## 4. Signing & publishing — **TODO**

Not yet wired up. When we are ready for store submission we need:

- **iOS**: Apple Developer Program membership ($99/year), signing
  certificates + provisioning profiles managed in Xcode, App Store
  Connect listing, TestFlight for beta.
- **Android**: Google Play Console (one-time $25), a keystore (.jks)
  that we store securely and never lose, Play Console listing, internal
  testing track for beta.
- **CI**: build artifacts (`.ipa`, `.aab`) from GitHub Actions runners
  with secrets for signing — Cloud Run not relevant for mobile.

## 5. Platform abstraction

All native APIs are reached through `@servicas/shared-platform`
(`packages/shared-platform/`). Features under `src/` should NOT import
`@capacitor/*` directly — go through the abstraction so the same code
path works in the browser, in Capacitor, and (later) in Tauri.

| Capability       | Module                            | Native impl                | Web impl                   |
| ---------------- | --------------------------------- | -------------------------- | -------------------------- |
| Runtime probe    | `runtime.isNative()` etc.         | `Capacitor.getPlatform()`  | feature detection          |
| Secure storage   | `secureStore`                     | `@capacitor/preferences`   | `localStorage`             |
| Network status   | `network.getNetworkStatus()`      | `@capacitor/network`       | `navigator.onLine`         |
| Haptics          | `haptics.success()` etc.          | `@capacitor/haptics`       | no-op                      |
| Share sheet      | `share.share(...)`                | `@capacitor/share`         | `navigator.share` / clipboard |
| Camera (photo)   | `camera.pickPhoto()`              | `@capacitor/camera`        | `<input type="file">`      |
| Camera (video)   | `camera.pickVideo()`              | Android plugin; iOS falls back to `<input>` | `<input type="file">` |
