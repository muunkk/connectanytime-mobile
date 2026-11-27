# Releasing ConnectAnytime Mobile

Use this runbook after the cutover is approved. See the [migration plan](monorepo-plan.md) for the first-release and real-device checks.

## Tag & GitHub Release conventions

| App | Git tag | GitHub Release? | Distribution |
|---|---|---|---|
| **iOS** | `ios-vX.Y.Z` | **No** (git tags + TestFlight notes only) | Internal TestFlight (app ID `6825901619`) |
| **Android** | `android-vX.Y.Z` | **Yes** (`--prerelease`, asset `ConnectAnytime-<versionName>-<versionCode>.apk`) | In-app updater |

> Keep GitHub Releases **Android-only**. Git tags never appear in GitHub's `/releases` API feed, so `Updates.kt` keeps its single-page, single-ETag fetcher without pagination.

---

## iOS (Internal TestFlight)

Enter `ios/` from the repository root and stay there through the release. A signed archive requires `Config/Local.xcconfig` with the existing Apple developer team.

### 1. Version bump & tag

1. Check App Store Connect for existing builds on the target marketing version, including in-flight builds. Increment `MARKETING_VERSION` and `CURRENT_PROJECT_VERSION` in `ios/project.yml`. Use a fresh version/build pair.
2. With a clean checkout, merge to `main`, tag that commit locally, and build from the tag. iOS tags are Git-only.

```sh
git fetch origin main
git tag -a ios-vX.Y.Z origin/main -m "iOS X.Y.Z (build N)"
git switch --detach ios-vX.Y.Z
```

### 2. Build dependencies & verify

Verify the native libraries and rebuild any stale outputs.

```sh
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
  -destination 'platform=iOS Simulator,name=iPhone 17'
scripts/test-strap-cookie.sh
```

If that simulator is missing, pick any iPhone from `xcrun simctl list devices available`.

### 3. Signed Release archive & export

Put `/usr/bin` ahead of Homebrew in `PATH` so Xcode's `rsync` subprocess uses Apple's `/usr/bin/rsync`. Write the archive and IPA to a temporary directory outside the repo:

```sh
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

Expand the exported IPA and verify the app, tunnel, and widget signatures before uploading:

```sh
ditto -x -k "$OUT_DIR/export/ConnectAnytime.ipa" "$OUT_DIR/ipa"
APP="$OUT_DIR/ipa/Payload/ConnectAnytime.app"
(
  set -e
  for TARGET in "$APP" "$APP/PlugIns/ConnectAnytimeTunnel.appex" "$APP/PlugIns/ConnectAnytimeWidgets.appex"; do
    codesign --verify --strict "$TARGET"
    codesign -d --entitlements :- "$TARGET"
    /usr/libexec/PlistBuddy -c 'Print CFBundleShortVersionString' "$TARGET/Info.plist"
    /usr/libexec/PlistBuddy -c 'Print CFBundleVersion' "$TARGET/Info.plist"
  done
)
```

Require all three signature checks to pass. Confirm the app, tunnel, and widget IDs (`com.muunkk.connectanytime`, `com.muunkk.connectanytime.tunnel`, `com.muunkk.connectanytime.widgets`), the existing team, the `packet-tunnel-provider` entitlement, and the shared keychain group (`$(AppIdentifierPrefix)com.muunkk.connectanytime.shared`). Verify the tagged marketing version and build number on all three targets.

### 4. Upload to Internal TestFlight

Push the verified tag, then upload.

```sh
git push origin ios-vX.Y.Z &&
asc builds upload \
  --app "6825901619" \
  --ipa "$OUT_DIR/export/ConnectAnytime.ipa" \
  --wait \
  --locale "en-US" \
  --test-notes "X.Y.Z (N) — <summary of fixes and requested checks>"
```

### 5. Confirm export compliance and internal availability

Check the exact version and build in App Store Connect. If it reports Missing Compliance, provide the app's confirmed encryption answer before handing the build to testers. If the release owner confirms the existing `usesNonExemptEncryption=false` classification still applies, update that exact build:

```sh
asc builds update --app "6825901619" --platform IOS \
  --version "X.Y.Z" --build-number "N" --uses-non-exempt-encryption=false
```

Use [Apple's TestFlight compliance workflow](https://developer.apple.com/help/app-store-connect/test-a-beta-version/provide-export-compliance-information-for-beta-builds) to review the classification or attach required documentation. Confirm the build is processed and available to the Internal TestFlight group, then clean up `$OUT_DIR` and return to `main`.

---

## Android (GitHub Prerelease + In-App Updater)

Enter `android/` from the repository root and stay there through the release.

### 1. Version bump & tag

1. Check both GitHub release histories and the release operator’s record for any withdrawn builds. Increment `versionName` and set `versionCode` strictly above every code exposed on either repo in `android/app/build.gradle.kts`.
2. Merge to `main`, tag the merge commit locally, and check out the exact tag before building:

```sh
git fetch origin main
git tag -a android-vX.Y.Z origin/main -m "Android X.Y.Z (build N)"
git switch --detach android-vX.Y.Z
```

### 2. Build & verify

Ensure `android/keystore.properties` and `android/local.properties` exist (both gitignored). Verify the native libraries and rebuild any stale outputs.

```sh
Vendor/build-deps.sh
```

Run tests, lint, debug/release builds, and verification scripts:

```sh
./gradlew testDebugUnitTest lintDebug lintRelease assembleDebug assembleRelease
python3 scripts/verify-release.py
scripts/verify-native.sh
scripts/test-strap-cookie.sh
```

Require the legacy release signing certificate. Both the `apksigner` verification and fingerprint comparison must succeed.

```sh
ANDROID_SDK="${ANDROID_HOME:-$HOME/Library/Android/sdk}"
CERTS=$("$ANDROID_SDK/build-tools/36.0.0/apksigner" verify --print-certs \
  app/build/outputs/apk/release/app-release.apk) && \
  printf '%s\n' "$CERTS" | grep -Fx \
    'Signer #1 certificate SHA-256 digest: b5e81629e55045309f942e9c954eb7cf8cea7942095ffbabb9c558b63080035e'
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

Push the verified tag. Create a **draft** prerelease so you can check the uploaded APK before it appears in the public feed:

```sh
git push origin "android-v${VERSION_NAME}" &&
gh release create "android-v${VERSION_NAME}" "$ASSET" \
  --repo muunkk/connectanytime-mobile \
  --verify-tag \
  --prerelease \
  --draft \
  --title "ConnectAnytime Android ${VERSION_NAME} (build ${VERSION_CODE})" \
  --notes "<release notes>"
```

Use a GitHub login with push access to see the draft. Check the exact asset name, upload state, and digest across all pages. Continue only if this command prints a release ID and exits successfully.

```sh
DRAFT_ID=$(gh api --paginate --slurp \
  "repos/muunkk/connectanytime-mobile/releases?per_page=100" |
  jq -er --arg tag "android-v${VERSION_NAME}" --arg name "${ASSET##*/}" \
    --arg digest "sha256:${LOCAL_SHA256}" '
    [.[][] | select(.tag_name == $tag and .draft and .prerelease)]
    | select(length == 1)
    | .[0]
    | select([.assets[] | select(.name == $name and .state == "uploaded" and .digest == $digest)] | length == 1)
    | .id') &&
printf '%s\n' "$DRAFT_ID"
```

Publish the verified draft by ID and confirm that the asset appears in the same public feed the app reads.

```sh
test -n "$DRAFT_ID" && gh api --method PATCH \
  "repos/muunkk/connectanytime-mobile/releases/${DRAFT_ID}" \
  -F draft=false -F prerelease=true
```

After publishing succeeds, check the public feed.

```sh
curl -fsSL 'https://api.github.com/repos/muunkk/connectanytime-mobile/releases?per_page=20' |
  jq -e --arg tag "android-v${VERSION_NAME}" --arg name "${ASSET##*/}" \
    --arg digest "sha256:${LOCAL_SHA256}" '
    any(.[]; .tag_name == $tag and (.draft | not) and .prerelease and
      any(.assets[]; .name == $name and .state == "uploaded" and .digest == $digest))'
```

Return to `main` after verification.
