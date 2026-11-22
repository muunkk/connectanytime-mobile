# Plan: Combine ConnectAnytime iOS & Android into One Repo (Lean Revision)

**Status:** Phases 1 and 2 are complete. Phase 3 awaits approval. The cutover covers Internal TestFlight on iOS and the GitHub Releases updater on Android.
**Target repo:** `muunkk/connectanytime-mobile` (public; name is final before Android's switch release).
**Out of scope:** The Mac app (`muunkk/connectanytime`), which drives Cisco Secure Client and shares no OpenConnect code or release flow.

---

## Why do this

1. **Stop OpenConnect patch drift.** Both apps compile `openconnect-9.21` + `openssl-3.5.9` with local patches, and the copies have already diverged (`free-cert-chain` and `unset-cmd-pipe` are iOS-only; the newer 14.5 KB `strap-cookie` patch is iOS-only; an extra `http.c` hunk in `no-post-replay` is Android-only).
2. **Simpler workflow in T3 Code.** One project checkout and one worktree workflow for both apps, making cross-platform parity checks and shared fixes straightforward.
3. **Low blast radius.** This is a personal/small-circle app (Internal TestFlight on iOS; one Android phone via GitHub Releases), so we keep real safety guardrails and skip enterprise migration ceremony.

---

## Layout & release conventions

```text
connectanytime-mobile/
  README.md                  # overview + links to ios/ and android/ READMEs
  .gitignore                 # OS noise only (.DS_Store); each app keeps its own
  docs/
    releasing.md             # exact iOS + Android build & release commands
    history/
      ios-commit-map         # legacy-to-imported iOS commit IDs
      android-commit-map     # legacy-to-imported Android commit IDs
  ios/                       # iOS app imported from the shipped v0.1.3 baseline
  android/                   # Android app imported from the shipped v0.1.4 baseline
  engine/                    # Phase 4 only: shared OpenConnect patches & host tests
```

Why each app builds unchanged inside `ios/` and `android/`:
- **iOS:** Run `xcodegen generate` from `ios/`. `$(SRCROOT)` resolves to `ios/`, so `Vendor/out*`, `Config/*.xcconfig`, and the tunnel's pre-build patch check work as-is. `Vendor/build-deps.sh` uses `cd "${0:A:h}"`.
- **Android:** Gradle root is `android/`. `keystore.properties`, CMake's relative `Vendor/out/${ANDROID_ABI}` path, and `scripts/verify-release.py` (`Path(__file__).resolve().parent.parent`) all stay relative to `android/`. (AGP 8.13 won't see `.git` inside `android/` and will omit the commit hash from `META-INF/version-control-info.textproto`; the git tag records the commit instead.)

### Tags & releases (keep GitHub Releases Android-only)

| App | Git tag | GitHub Release? | Distribution |
|---|---|---|---|
| **iOS** | `ios-vX.Y.Z` | **No** (git tags + TestFlight notes only, same as today) | Internal TestFlight |
| **Android** | `android-vX.Y.Z` | **Yes** (`--prerelease`, asset `ConnectAnytime-<versionName>-<versionCode>.apk`) | In-app updater |

**Why no GitHub Releases for iOS:** Git tags never appear in GitHub's `/releases` API feed. Keeping Releases Android-only means `Updates.kt` keeps its single-page, single-ETag fetcher—no pagination or multi-page cache needed.

---

## Phases

### Import and publication record

- **iOS:** The imported source is [`overnight-diagnostics` at `v0.1.3`](https://github.com/muunkk/connectanytime-ios/tree/v0.1.3) (`669fc96`). It includes PRs #15 and #16. Legacy `main` remains at `v0.1.1`. Record the seven unique commits on `hotspot-investigation` for follow-up.
- **Android:** The imported source is [`main` at `v0.1.4`](https://github.com/muunkk/connectanytime-android/tree/v0.1.4) (`f9c571b`). Defer [#7 audit fixes](https://github.com/muunkk/connectanytime-android/pull/7), [#9 battery measurement](https://github.com/muunkk/connectanytime-android/pull/9), and [#10 widget hotspot state](https://github.com/muunkk/connectanytime-android/pull/10) until the monorepo updater passes its phone test. The #7 branch was based on `0.1.3 (4)` and changes the updater and storage. Preserve the monorepo release version and feed changes when porting it. #9 and #10 are separate branches based on `0.1.4 (5)`.
- **History:** The platform directories at import commit `e363259` match those legacy trees exactly. The eight prefixed Git tags also match their original app trees. Commit maps cover the selected branch and tag ancestry. Deferred branch and GitHub-generated pull request commits are outside that history. Phase 2 updated the platform READMEs and Android verification notes.
- **Validation:** Phase 1 built both native dependency trees at the canonical checkout, ran the iOS tests and signed export, and checked Android JVM tests, lint, native outputs, release APK alignment and signing. Keep using the existing iOS team and Android signing key. Store local config and generated outputs outside published history.
- **GitHub:** `muunkk/connectanytime-mobile` was published with `main`, the eight prefixed tags and the platform labels. GitHub Releases are reserved for Android from Phase 3B onward. Keep legacy issues, repositories and releases in place until each platform passes its device checks.

### Phase 3 — First real release of each app from the monorepo

Begin only after approval. Refresh the GitHub and TestFlight inventories, then follow the [release runbook](releasing.md). Preserve the legacy branch and PR links before archiving either repo.

#### 3A. iOS (TestFlight)
1. Confirm the current internal build. The audited baseline is `0.1.3 (4)`. Use `0.1.4 (5)` if that pair is still unused. Bump `MARKETING_VERSION` / `CURRENT_PROJECT_VERSION` in `ios/project.yml`, merge, then tag the exact commit locally as `ios-vX.Y.Z`.
2. From `ios/`: `xcodegen generate`, archive and export per `docs/releasing.md` / `ios/README.md` (`Config/ExportOptions-TestFlight.plist`, `/usr/bin` ahead of Homebrew in `PATH`), then push the verified tag and upload the IPA to Internal TestFlight (app ID `6825901619`). Confirm export compliance and internal group availability.
3. Smoke-test on your iPhone installing over the previous TestFlight build: Keychain password preserved, VPN connects, Control Center widget works, and Stay Connected rejoins automatically after a network switch and after a reboot **without tapping Disconnect** (which intentionally clears the saved session).
4. Once confirmed: transfer open iOS issues (`gh issue transfer`), add a "Moved" notice to `connectanytime-ios`'s README, and archive `muunkk/connectanytime-ios`.

*iOS Rollback:* Expire the faulty build in App Store Connect to stop new installs. On the phone, install the previous build from TestFlight's **Previous Builds** list, or rebuild a known-good tag (or the still-unarchived old repo) under a higher build number and upload.

#### 3B. Android (minimal updater switch + end-to-end proof)

Confirm the installed Android baseline reports `0.1.4 (5)` and can connect before installing the switch build. Verify the highest version code ever exposed, including withdrawn builds.

**Step 1 — Tiny updater PR (`android/app/src/main/java/com/muunkk/connectanytime/Updates.kt`):**
- Change `REPO = "muunkk/connectanytime-mobile"` (`API`, `PAGES`, and `DOWNLOADS` all derive from it; `ReleaseFeed.newest` already ignores any release lacking a valid `ConnectAnytime-<ver>-<code>.apk` asset with a `sha256:` digest, so no `tag_name` filter or pagination is needed).
- Rename the cached feed file from `"release-feed.json"` to `"release-feed-v2.json"` (one-line cache reset so the first check against the new repo never sends the old ETag, with zero changes to `Storage.kt`).
- Update `UpdatesTest`, `UpdaterTest`, and `app/src/test/resources/github-releases.json` for `muunkk/connectanytime-mobile` (keep tests for old-repo URL rejection and a first request without the legacy ETag).
- Update updater repo references in `android/README.md` and `android/VERIFICATION.md`.

**Step 2 — Ship the switch release:**
1. Bump `versionCode` (strictly above the highest ever exposed on either repo, including any withdrawn release) and `versionName` in `android/app/build.gradle.kts`, merge, tag the merge commit locally as `android-vX.Y.Z`, and check out that tag so the APK is built from the exact tagged commit.
2. From `android/`: run `Vendor/build-deps.sh`, `./gradlew testDebugUnitTest lintDebug lintRelease assembleDebug assembleRelease`, `python3 scripts/verify-release.py`, `scripts/verify-native.sh`, `scripts/test-strap-cookie.sh`, and the explicit `apksigner ... | grep -q b5e81629...080035e` check.
3. Copy `app-release.apk` to `app/build/outputs/apk/release/ConnectAnytime-<versionName>-<versionCode>.apk` (stays gitignored).
4. Push the verified tag and create a **draft** prerelease on the monorepo:
   `gh release create android-vX.Y.Z --repo muunkk/connectanytime-mobile --verify-tag --prerelease --draft ...`
   Verify via `gh api` that the asset has `state: "uploaded"` and a `sha256:` digest matching `shasum -a 256` locally, then publish by the verified release ID and confirm it appears in the public `/releases` feed.
5. **Install the switch build on your phone (choose one):**
   - **Option A (`adb install -r` — simplest for one phone):** Run `adb install -r ConnectAnytime-<versionName>-<versionCode>.apk`. Preserves all settings, hotspot config, and saved session with zero extra release work on the old repo. *(Android may show an install confirmation on the next update. Complete it if shown and verify the new version.)*
   - **Option B (bridge prerelease on old repo, only with explicit approval):** Publish that exact same APK once via draft → verify digest → publish on `muunkk/connectanytime-android` (`vX.Y.Z`, targeting the old repo's `main` since the monorepo commit doesn't exist there—so omit `--verify-tag` and note the monorepo commit SHA in the release body). Tap **Check for updates** → **Install** in the app (keeps the app as installer of record).
6. Smoke-test on your Android phone: settings, auto-connect list, hotspot sharing, widget, and saved session intact; VPN connects.

**Step 3 — Prove the new updater end-to-end:**
- Ship the next small Android bump *only* to `muunkk/connectanytime-mobile`. Tap **Check for updates** in the app (bypasses the 6-hour auto-check throttle) and confirm the app finds, verifies, and installs it from the monorepo feed.
- Once confirmed: transfer open Android issues (`gh issue transfer`), add a "Moved" notice to `connectanytime-android`'s README, and archive `muunkk/connectanytime-android` (never delete its old releases).

*Android Rollback (roll forward only):* Re-drafting a release stops future downloads (if Option B was used, re-draft on both repos), but it cannot recall an APK already in `Ready` or installed, and Android refuses `versionCode` downgrades. Always reserve every exposed `versionCode` (including withdrawn ones) and roll forward: build known-good code + `muunkk/connectanytime-mobile` with a higher `versionCode`, and either publish it on the monorepo (if the updater works) or install it with `adb install -r`.

---

### Phase 4 — Shared `engine/` (separate follow-up after Phase 3)

Only starts after both apps have shipped cleanly from the monorepo.

```text
engine/
  openconnect/
    patches/          # shared patches + temporary *-ios / *-android splits
    series-ios        # ordered patch list (today's LC_ALL=C order)
    series-android    # ordered patch list (today's LC_ALL=C order)
  tests/              # strap-cookie-test.c, bridge-session-test.c
  scripts/
    test-engine.sh    # host build + tests + negative controls per series
```

Build flags stay in `ios/Vendor/build-deps.sh` and `android/Vendor/build-deps.sh` (SDK libxml2 vs built libxml2, `-fPIC`, and targets differ intentionally).

1. **4a — Move patches with zero content change:** Point both `build-deps.sh` scripts, iOS's `project.yml` pre-build stamp check, and both `test-strap-cookie.sh` scripts at `engine/openconnect/series-{ios,android}` (preserving iOS's negative controls). Keep drifted patches split (`*-ios.patch` / `*-android.patch`) for this PR. Update Android's `build.sha256` to hash `build-deps.sh` + `series-android` + every listed patch.
   - *Important for iOS:* Because moving identical patch bytes leaves `openconnect-patches.sha256` unchanged, **force a clean rebuild** of `ios/Vendor/out` (and `out-sim` if used) during verification—and again if reverting—so the new script actually runs. Verify both apps' stamp-staleness checks and host tests.
2. **4b — Reconcile patch drift (one PR per fix, tested on the target platform):**
   - Port `openconnect-free-cert-chain.patch` and `openconnect-unset-cmd-pipe.patch` to Android.
   - Port the extra `http.c` hunk of `openconnect-no-post-replay.patch` to iOS.
   - Unify `openconnect-strap-cookie.patch` (with the matching Android bridge update) and unify `engine/tests/`.
   - Keep `openconnect-switch-tun.patch` Android-only.
3. **4c — `ocbridge.c` (optional):** Only merge if a small hooks struct actually simplifies it; iOS (`NEPacketTunnelProvider`) and Android (`VpnService` + socket protect + tun swap) have genuinely different lifecycle models.

*(Optional later: add two lightweight path-filtered `macos-latest` GitHub Actions workflows for `ConnectAnytimeTests` and `testDebugUnitTest lintDebug assembleDebug`. Not needed for the move.)*

---

## Summary checklist
- [x] **Phase 1:** `git filter-repo` (HTTPS) into `/Users/mini/Master/code/connectanytime-mobile`; clean `build-deps.sh` runs; verify trees/tags, iOS signed entitlements + tests, and Android build + `verify-release.py` + cert fingerprint.
- [x] **Phase 2:** Push `main` + `ios-v*`/`android-v*` tags to `muunkk/connectanytime-mobile`, create labels, add T3 project, update docs.
- [ ] **Phase 3:** Ship iOS TestFlight from `ios/` → verify on iPhone → transfer iOS issues & archive old iOS repo. Ship Android `REPO` switch (`adb install -r` or bridge prerelease) → prove next monorepo-only in-app update → transfer Android issues & archive old Android repo.
- [ ] **Phase 4:** Extract `engine/openconnect/patches/` (4a, with forced clean iOS rebuild), then reconcile patch drift one PR at a time (4b).
