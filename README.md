# ConnectAnytime Mobile

One-tap Connect/Disconnect for the NYU Shanghai VPN on **iOS** and **Android**, without Cisco Secure Client.

This monorepo combines both mobile apps while preserving their full git histories:

- **[ConnectAnytime for iOS](ios/README.md)** (`ios/`) — Swift / Network Extension packet tunnel (`NEPacketTunnelProvider`) with Control Center widget, Siri/Shortcuts, and Stay Connected. Distributed via Internal TestFlight (`ios-vX.Y.Z` git tags only).
- **[ConnectAnytime for Android](android/README.md)** (`android/`) — Kotlin / Jetpack Compose `VpnService` app with Glance home screen widget, USB/hotspot VPN sharing proxy, and verified in-app GitHub Releases updater (`android-vX.Y.Z` tags + prereleases).

Sister project: [ConnectAnytime for macOS](https://github.com/muunkk/connectanytime) (separate repository; drives Cisco Secure Client and shares no OpenConnect code or release flow).

## Repository layout

```text
connectanytime-mobile/
  README.md                 # This overview
  .gitignore                # OS noise only; each app keeps its own .gitignore
  docs/
    monorepo-plan.md        # Approved monorepo consolidation & cutover plan
    releasing.md            # Verified iOS TestFlight and Android GitHub Release commands
    history/
      ios-commit-map        # Old-to-new SHA mapping from muunkk/connectanytime-ios
      android-commit-map    # Old-to-new SHA mapping from muunkk/connectanytime-android
  ios/                      # ConnectAnytime for iOS
  android/                  # ConnectAnytime for Android
```

## Quick start

Each platform builds from its own subdirectory:

- **iOS (`cd ios`):** run `Vendor/build-deps.sh`, configure `Config/Local.xcconfig`, then `xcodegen generate`. See [ios/README.md](ios/README.md).
- **Android (`cd android`):** run `Vendor/build-deps.sh`, configure `local.properties` / `keystore.properties`, then `./gradlew assembleDebug testDebugUnitTest`. See [android/README.md](android/README.md).

See [docs/releasing.md](docs/releasing.md) for tag naming, build verification, and release commands.

## Licenses

Each app's own code is MIT (see [ios/LICENSE](ios/LICENSE) and [android/LICENSE](android/LICENSE)). Both apps statically link OpenConnect (LGPL-2.1) with local patches and OpenSSL (Apache-2.0); Android additionally links libxml2 (MIT).

Not affiliated with Cisco or NYU.
