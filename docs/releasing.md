# Releasing ConnectAnytime Mobile

## Tag & GitHub Release conventions

| App | Git tag | GitHub Release? | Distribution |
|---|---|---|---|
| **iOS** | `ios-vX.Y.Z` | **No** (git tags + TestFlight notes only) | Internal TestFlight (app ID `6825901619`) |
| **Android** | `android-vX.Y.Z` | **Yes** (`--prerelease`, asset `ConnectAnytime-<versionName>-<versionCode>.apk`) | In-app updater |

> Keep GitHub Releases **Android-only**. Git tags never appear in GitHub's `/releases` API feed, so `Updates.kt` keeps its single-page, single-ETag fetcher without pagination.

---

## iOS (Internal TestFlight)

Run all commands from `ios/` unless noted.

### 1. Version bump & tag

1. Increment `MARKETING_VERSION` and `CURRENT_PROJECT_VERSION` in `ios/project.yml` (never reuse a version/build pair).
2. Merge to `main`, then create and push an immutable git tag (no GitHub Release):

```sh
git tag -a ios-vX.Y.Z -m "iOS X.Y.Z (build N)"
git push origin ios-vX.Y.Z
```

### 2. Build dependencies & verify

If `ios/Vendor/patches/` changed or this is a fresh checkout:

```sh
cd ios
Vendor/build-deps.sh
```

Generate the Xcode project and run local checks:

```sh
xcodegen generate
xcodebuild build \
  -project ConnectAnytime.xcodeproj \
  -scheme ConnectAnytime \
  -destination 'generic/platform=iOS' \
  CODE_SIGNING_ALLOWED=NO
xcodebuild test \
  -project ConnectAnytime.xcodeproj \
  -scheme ConnectAnytimeTests \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro'
scripts/test-strap-cookie.sh
```

### 3. Signed Release archive & export

Put `/usr/bin` ahead of Homebrew in `PATH` so Xcode's `rsync` subprocess uses Apple's `/usr/bin/rsync`. Write the archive and IPA to a temporary directory outside the repo:

```sh
cd ios
OUT_DIR=$(mktemp -d /tmp/connectanytime-ios-release.XXXXXX)

PATH="/usr/bin:$PATH" xcodebuild archive \
  -project ConnectAnytime.xcodeproj \
  -scheme ConnectAnytime \
  -configuration Release \
  -destination 'generic/platform=iOS' \
  -archivePath "$OUT_DIR/ConnectAnytime.xcarchive"

PATH="/usr/bin:$PATH" xcodebuild -exportArchive \
  -archivePath "$OUT_DIR/ConnectAnytime.xcarchive" \
  -exportOptionsPlist Config/ExportOptions-TestFlight.plist \
  -exportPath "$OUT_DIR/export"
```

Verify entitlements on the archived app, tunnel, and widget extensions before uploading:

```sh
APP="$OUT_DIR/ConnectAnytime.xcarchive/Products/Applications/ConnectAnytime.app"
codesign -dv --entitlements - "$APP"
codesign -dv --entitlements - "$APP/PlugIns/ConnectAnytimeTunnel.appex"
codesign -dv --entitlements - "$APP/PlugIns/ConnectAnytimeWidgets.appex"
```

Confirm bundle IDs (`com.muunkk.connectanytime`, `com.muunkk.connectanytime.tunnel`, `com.muunkk.connectanytime.widgets`), signing team, and `keychain-access-groups` (`$(AppIdentifierPrefix)com.muunkk.connectanytime.shared`).

### 4. Upload to Internal TestFlight

```sh
asc builds upload \
  --app "6825901619" \
  --ipa "$OUT_DIR/export/ConnectAnytime.ipa" \
  --wait \
  --locale "en-US" \
  --test-notes "X.Y.Z (N) — <summary of fixes and requested checks>"
```

Verify in App Store Connect that Apple processed the build and made it available to the Internal TestFlight group, then clean up `$OUT_DIR`.

---

## Android (GitHub Prerelease + In-App Updater)

Run all commands from `android/` unless noted.

### 1. Version bump & tag

1. Increment `versionCode` (strictly above the highest `versionCode` ever exposed on any repo, including withdrawn releases) and `versionName` in `android/app/build.gradle.kts`.
2. Merge to `main`, tag the merge commit, push the tag, and check out the exact tag before building:

```sh
git tag -a android-vX.Y.Z -m "Android X.Y.Z (build N)"
git push origin android-vX.Y.Z
git checkout android-vX.Y.Z
```

### 2. Build & verify

Ensure `android/keystore.properties` and `android/local.properties` exist (both gitignored). Rebuild native libraries if needed:

```sh
cd android
Vendor/build-deps.sh
```

Run tests, lint, debug/release builds, and verification scripts:

```sh
./gradlew testDebugUnitTest lintDebug lintRelease assembleDebug assembleRelease
python3 scripts/verify-release.py
scripts/verify-native.sh
scripts/test-strap-cookie.sh
```

Explicitly assert the release signing certificate SHA-256 fingerprint:

```sh
ANDROID_SDK="${ANDROID_HOME:-$HOME/Library/Android/sdk}"
"$ANDROID_SDK/build-tools/36.0.0/apksigner" verify --print-certs \
  app/build/outputs/apk/release/app-release.apk \
  | grep -q b5e81629e55045309f942e9c954eb7cf8cea7942095ffbabb9c558b63080035e
```

### 3. Stage APK & publish draft-then-verify GitHub Prerelease

Copy the release APK to the versioned asset filename expected by `Updates.kt` (stays gitignored under `app/build/`):

```sh
VERSION_NAME="X.Y.Z"
VERSION_CODE="N"
ASSET="app/build/outputs/apk/release/ConnectAnytime-${VERSION_NAME}-${VERSION_CODE}.apk"
cp app/build/outputs/apk/release/app-release.apk "$ASSET"
LOCAL_SHA256=$(shasum -a 256 "$ASSET" | awk '{print $1}')
echo "Local SHA-256: $LOCAL_SHA256"
```

Create a **draft** prerelease first so the uploaded asset's `sha256:` digest can be verified before the public `/releases` feed exposes it:

```sh
gh release create "android-v${VERSION_NAME}" "$ASSET" \
  --repo muunkk/connectanytime-mobile \
  --verify-tag \
  --prerelease \
  --draft \
  --title "ConnectAnytime Android ${VERSION_NAME} (build ${VERSION_CODE})" \
  --notes "<release notes>"
```

Verify the uploaded asset state and digest via `gh api`:

```sh
gh api "repos/muunkk/connectanytime-mobile/releases" \
  --jq ".[] | select(.tag_name==\"android-v${VERSION_NAME}\") | .assets[] | {name, state, digest}"
```

Confirm `state` is `"uploaded"` and `digest` is `"sha256:${LOCAL_SHA256}"`, then publish:

```sh
gh release edit "android-v${VERSION_NAME}" \
  --repo muunkk/connectanytime-mobile \
  --draft=false
```

Confirm it appears in the public `/releases` feed (`https://api.github.com/repos/muunkk/connectanytime-mobile/releases`).
