# Plan: Combine ConnectAnytime iOS & Android into One Repo (Lean Revision)

**Status:** Plan only — nothing has been moved yet.
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
      ios-commit-map         # from ios.git/filter-repo/commit-map
      android-commit-map     # from android.git/filter-repo/commit-map
  ios/                       # muunkk/connectanytime-ios tree, unchanged
  android/                   # muunkk/connectanytime-android tree, unchanged
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

### Phase 0 — Land & prove pending work first

Bring `main` up to date on both repos before importing so no active stack has to be replayed across the move:

1. **iOS (`muunkk/connectanytime-ios`):** Merge `overnight-diagnostics` (`v0.1.3` / `#15` / `#16`) into `main` (currently at `v0.1.1`).
2. **Android (`muunkk/connectanytime-android`):** Merge or close open branches (`audit-fixes` `#7`, `battery-measure` `#9`, `widget-hotspot-state` `#10`), and decide whether to fix `#12` (Duo spam after cancel) before or after the move.
   - *Watch out on `#7`:* Its branch still declares `0.1.3 (4)` and modifies `Updates.kt` / `Storage.kt`. Make sure the merge leaves `versionCode` ≥ `5`.
   - **Prove merged Android code before changing `REPO`:** Ship merged `main` on the old repo (e.g. `0.1.5 (6)`) and install it on your phone. Because `0.1.4 → 0.1.5` runs `0.1.4`'s updater rather than `#7`'s new installer code, if `#7` is merged before the move, prove `#7`'s updater by letting `0.1.5` self-update to one higher old-repo release (e.g. `0.1.6 (7)`) before changing `REPO`—or keep `#7` for right after Phase 3B. Either way, Phase 3B's switch build carries *only* the `REPO` change on top of a phone-proven updater.
3. **Unmerged branches & worktrees:** Anything left off `main` (such as iOS's 7-commit `hotspot-investigation` branch) stays readable in the old repo even after archiving. Close old T3 worktrees once their branches are settled.
4. **Prep:** Record the highest Android `versionCode` ever exposed (currently `5`) and latest iOS TestFlight version/build. Install `git-filter-repo` (`brew install git-filter-repo`).

*Rollback:* Nothing in the monorepo has changed; any old-repo Android release published in step 2 follows normal roll-forward rules (reserve its `versionCode` and ship a higher one if needed).

---

### Phase 1 — Build & verify the monorepo locally (old repos untouched)

Create the combined history in a scratch folder using HTTPS remotes (SSH host-key verification isn't configured on this Mac), then clone/move to the **final checkout path** *before* building native libraries (since regenerated `.pc`/`.la` metadata embeds the checkout path):

```sh
git clone --mirror https://github.com/muunkk/connectanytime-ios.git ios.git
git clone --mirror https://github.com/muunkk/connectanytime-android.git android.git
git -C ios.git     filter-repo --to-subdirectory-filter ios     --tag-rename '':'ios-'
git -C android.git filter-repo --to-subdirectory-filter android --tag-rename '':'android-'

git init -b main /Users/mini/Master/code/connectanytime-mobile
cd /Users/mini/Master/code/connectanytime-mobile
git commit --allow-empty -m "Initialize ConnectAnytime mobile monorepo"
git pull /path/to/ios.git main     --allow-unrelated-histories --no-rebase
git pull /path/to/android.git main --allow-unrelated-histories --no-rebase
git fetch /path/to/ios.git     'refs/tags/*:refs/tags/*'
git fetch /path/to/android.git 'refs/tags/*:refs/tags/*'
```

Copy `ios.git/filter-repo/commit-map` and `android.git/filter-repo/commit-map` into `docs/history/`, and add root `README.md`, `.gitignore`, and `docs/releasing.md` (with the exact iOS and Android build/release commands). Each app keeps its own `LICENSE` and `.gitignore`.

**Local setup at `/Users/mini/Master/code/connectanytime-mobile`:**
1. Copy gitignored secrets/config:
   - `anyconnectios/Config/Local.xcconfig` → `ios/Config/`
   - `connectanytime-android/keystore.properties` → `android/` (`storeFile` is an absolute path outside the repo)
   - `connectanytime-android/local.properties` → `android/`
2. Copy the downloaded tarballs (`Vendor/src/*.tar.*`) to avoid re-downloading, then run `ios/Vendor/build-deps.sh` (plus `ios/Vendor/build-deps.sh simulator` if building the full app on Simulator) and `android/Vendor/build-deps.sh` cleanly once (expect several minutes, up to ~15 min total).
   - *Why rebuild instead of copying `Vendor/out`:* On iOS, `Vendor/out/lib/pkgconfig/*.pc` embeds the old checkout path and would be read on future patch rebuilds; rebuilding both from the final checkout path guarantees self-contained libraries. Future monorepo worktrees can copy `Vendor/out*` from this canonical checkout, just as today. Keep the old checkouts untouched as fallback.

**Verification before pushing:**
- **Git trees & tags:** Run `git -C <old> fetch`, then verify `git -C <old> rev-parse origin/main^{tree}` equals `git rev-parse HEAD:ios` and `HEAD:android`. Spot-check prefixed tag trees against the old tags (including `ios-v0.1.2`, which lands off `main` if `overnight-diagnostics` is squashed).
- **iOS (from `ios/`):**
  - `xcodegen generate`
  - Build `ConnectAnytime` with `-destination generic/platform=iOS CODE_SIGNING_ALLOWED=NO` (proves the tunnel's pre-build patch check and static linking; `ConnectAnytimeTests` is hostless and skips both).
  - Run `ConnectAnytimeTests` on Simulator and `scripts/test-strap-cookie.sh`.
  - Run one signed Release archive + export (`Config/ExportOptions-TestFlight.plist`, `/usr/bin` ahead of Homebrew in `PATH` for Apple's `rsync`) and inspect `codesign -d --entitlements -` on the app, tunnel, and widget to confirm bundle IDs (`com.muunkk.connectanytime{,.tunnel,.widgets}`), team, and `keychain-access-groups` (`$(AppIdentifierPrefix)com.muunkk.connectanytime.shared`).
- **Android (from `android/`):**
  - Run `./gradlew testDebugUnitTest lintDebug lintRelease assembleDebug assembleRelease`.
  - Run `python3 scripts/verify-release.py`, `scripts/verify-native.sh`, and `scripts/test-strap-cookie.sh`.
  - Explicitly assert the signing cert fingerprint (since `verify-release.py` only prints it):
    `"$ANDROID_HOME/build-tools/36.0.0/apksigner" verify --print-certs app/build/outputs/apk/release/app-release.apk | grep -q b5e81629e55045309f942e9c954eb7cf8cea7942095ffbabb9c558b63080035e`

*Rollback:* Delete `/Users/mini/Master/code/connectanytime-mobile`.

---

### Phase 2 — Publish & switch workspace

1. Create public repo `muunkk/connectanytime-mobile` (verify releases are not set to immutable so drafts/re-drafts work).
2. Push **only** `main` and the `ios-v*` / `android-v*` tags (never `refs/pull/*`).
3. Create labels (`ios`, `android`, `engine`, `bug`, `enhancement`, `priority: low`) so they exist before issues are transferred later.
4. Add `/Users/mini/Master/code/connectanytime-mobile` as a T3 project.
5. Update each app's README (`cd ios` / `cd android`, new tag prefixes, `VERIFICATION.md` checkout path) and copy still-true agent memory notes.
6. **Wait to transfer issues, add "Moved" notices, or archive the old repos until Phase 3 device checks pass**—if anything goes wrong during Phase 3, nothing on the old repos has to be unwound.

*Rollback:* Delete the new repo and keep working in the old repos.

---

### Phase 3 — First real release of each app from the monorepo

#### 3A. iOS (TestFlight)
1. Bump `MARKETING_VERSION` / `CURRENT_PROJECT_VERSION` in `ios/project.yml`, merge, tag and push `ios-vX.Y.Z` (git tag only—no GitHub Release).
2. From `ios/`: `xcodegen generate`, archive and export per `docs/releasing.md` / `ios/README.md` (`Config/ExportOptions-TestFlight.plist`, `/usr/bin` ahead of Homebrew in `PATH`), and upload to Internal TestFlight (app ID `6825901619`) via `asc`.
3. Smoke-test on your iPhone installing over the previous TestFlight build: Keychain password preserved, VPN connects, Control Center widget works, and Stay Connected rejoins automatically after a network switch and after a reboot **without tapping Disconnect** (which intentionally clears the saved session).
4. Once confirmed: transfer open iOS issues (`gh issue transfer`), add a "Moved" notice to `connectanytime-ios`'s README, and archive `muunkk/connectanytime-ios`.

*iOS Rollback:* Expire the faulty build in App Store Connect to stop new installs. On the phone, install the previous build from TestFlight's **Previous Builds** list, or rebuild a known-good tag (or the still-unarchived old repo) under a higher build number and upload.

#### 3B. Android (minimal updater switch + end-to-end proof)

**Step 1 — Tiny updater PR (`android/app/src/main/java/com/muunkk/connectanytime/Updates.kt`):**
- Change `REPO = "muunkk/connectanytime-mobile"` (`API`, `PAGES`, and `DOWNLOADS` all derive from it; `ReleaseFeed.newest` already ignores any release lacking a valid `ConnectAnytime-<ver>-<code>.apk` asset with a `sha256:` digest, so no `tag_name` filter or pagination is needed).
- Rename the cached feed file from `"release-feed.json"` to `"release-feed-v2.json"` (one-line cache reset so the first check against the new repo never sends the old ETag, with zero changes to `Storage.kt`).
- Update `UpdatesTest`, `UpdaterTest`, and `app/src/test/resources/github-releases.json` for `muunkk/connectanytime-mobile` (keep a test confirming old-repo URLs are rejected).
- Update updater repo references in `android/README.md` and `android/VERIFICATION.md`.

**Step 2 — Ship the switch release:**
1. Bump `versionCode` (strictly above the highest ever exposed on either repo, including any withdrawn release) and `versionName` in `android/app/build.gradle.kts`, merge, push tag `android-vX.Y.Z` on the merge commit, and check out that tag (`git checkout android-vX.Y.Z`) so the APK is built from the exact tagged commit.
2. From `android/`: run `Vendor/build-deps.sh`, `./gradlew testDebugUnitTest lintDebug lintRelease assembleDebug assembleRelease`, `python3 scripts/verify-release.py`, `scripts/verify-native.sh`, `scripts/test-strap-cookie.sh`, and the explicit `apksigner ... | grep -q b5e81629...080035e` check.
3. Copy `app-release.apk` to `app/build/outputs/apk/release/ConnectAnytime-<versionName>-<versionCode>.apk` (stays gitignored).
4. Create a **draft** prerelease on the monorepo:
   `gh release create android-vX.Y.Z --repo muunkk/connectanytime-mobile --verify-tag --prerelease --draft ...`
   Verify via `gh api` that the asset has `state: "uploaded"` and a `sha256:` digest matching `shasum -a 256` locally, then publish (`--draft=false`) and confirm it appears in the public `/releases` feed.
5. **Install the switch build on your phone (choose one):**
   - **Option A (`adb install -r` — simplest for one phone):** Run `adb install -r ConnectAnytime-<versionName>-<versionCode>.apk`. Preserves all settings, hotspot config, and saved session with zero extra release work on the old repo. *(Expected: shell becomes installer of record, so Step 3's in-app update will prompt for one confirmation tap; silent updates resume after that.)*
   - **Option B (bridge prerelease on old repo — zero sideloading):** Publish that exact same APK once via draft → verify digest → publish on `muunkk/connectanytime-android` (`vX.Y.Z`, targeting the old repo's `main` since the monorepo commit doesn't exist there—so omit `--verify-tag` and note the monorepo commit SHA in the release body). Tap **Check for updates** → **Install** in the app (keeps the app as installer of record).
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
- [ ] **Phase 0:** Merge iOS `overnight-diagnostics` → `main`; merge/ship Android pending branches on the old repo and confirm auto-update on phone.
- [ ] **Phase 1:** `git filter-repo` (HTTPS) into `/Users/mini/Master/code/connectanytime-mobile`; clean `build-deps.sh` runs; verify trees/tags, iOS signed entitlements + tests, and Android build + `verify-release.py` + cert fingerprint.
- [ ] **Phase 2:** Push `main` + `ios-v*`/`android-v*` tags to `muunkk/connectanytime-mobile`, create labels, add T3 project, update docs.
- [ ] **Phase 3:** Ship iOS TestFlight from `ios/` → verify on iPhone → transfer iOS issues & archive old iOS repo. Ship Android `REPO` switch (`adb install -r` or bridge prerelease) → prove next monorepo-only in-app update → transfer Android issues & archive old Android repo.
- [ ] **Phase 4:** Extract `engine/openconnect/patches/` (4a, with forced clean iOS rebuild), then reconcile patch drift one PR at a time (4b).
