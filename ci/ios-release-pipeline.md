# iOS release pipeline — implementation & configuration plan

Status: **Pipeline PROVEN end-to-end 2026-07-10** — Phase 0 (enrollment, team `GRYN97UG27`, ASC app
record + agreements accepted), Phase 1 (test target, versioning, `ci_scripts/` — validated live on
Xcode Cloud), and Phase 3 (managed signing; archive → App Store export → **TestFlight upload
succeeded**) are DONE. Remaining: finalize Phase 2 workflow topology (repoint archive workflow from
`development` to `release` push, add the PR-test workflow) + Phase 4 guardrails. Owner: Radek.
Last updated: 2026-07-10.

This document scopes the work to bring iOS to parity with the existing Android CI/CD
pipeline: **PR tests** and a **signed release build delivered to testers**. It reflects three
decisions already made:

| Decision | Choice |
|---|---|
| Apple Developer Program | **Not enrolled yet** — enrollment is the blocking prerequisite |
| iOS CI host | **Xcode Cloud** (25 free Mac-hours/month, native signing + TestFlight) |
| Distribution depth | **Auto-upload to TestFlight** internal testing (no auto App Store submit) |

---

## 1. Where we are today

**Android pipeline (the target to mirror):**

- `ci.yml` — every PR: host-JVM `commonTest` + Android debug compile smoke (Linux, free).
- `ui-tests.yml` — PRs **into `release` only**: Android emulator UI + launch integration tests.
- `build.yml` — push to `release` / `v*` tag / manual: signed **AAB + R8 mapping** → artifacts +
  a GitHub "build ledger" Release. `VERSION_CODE = github.run_number`, `GIT_SHA = github.sha`.
  **No auto-publish** — manual Play upload.
- `version-bump.yml` — bumps `VERSION_NAME` in `version.properties`, tags `vX.Y.Z`.

**iOS state — the gaps:**

1. **No iOS anywhere in CI** — no simulator build, no test run, no archive/IPA.
2. **The UI test is an orphan.** `iosApp/iosAppUITests/FirstScreenLaunchTest.swift` exists, but
   `project.pbxproj` has **only one native target** (the app). There is no UI-test bundle target and
   the shared scheme's `TestAction` is empty — nothing runs this file. It must be wired before any
   iOS test can execute.
3. **No distribution signing.** `CODE_SIGN_STYLE = Automatic`, `DEVELOPMENT_TEAM = ${TEAM_ID}`, and
   `TEAM_ID=` is **blank** in `iosApp/Configuration/Config.xcconfig`.
4. **Versioning diverges.** iOS `MARKETING_VERSION=1.0.0` / `CURRENT_PROJECT_VERSION=1` vs Android
   `version.properties` `VERSION_NAME=1.1.0` / `VERSION_CODE=1`. No single source of truth.
5. **Already in good shape:** Firebase via SPM (`firebase-ios-sdk` 12.11.0+); `GoogleService-Info.plist`
   committed (no secret needed); build phases already run the Crashlytics dSYM script, the
   Debug/Release plist copy, and `./gradlew :composeApp:embedAndSignAppleFrameworkForXcode` to
   produce the Kotlin/Native framework. Xcode 16 project (`objectVersion 77`), iOS 15 deployment
   target, iPhone-only (`TARGETED_DEVICE_FAMILY = 1`).

---

## 2. Cost analysis — staying inside free limits

| Item | Cost | Notes |
|---|---|---|
| Apple Developer Program | **$99 / year — unavoidable** | Hard requirement for signing, TestFlight, App Store, **and Xcode Cloud**. No free substitute exists. |
| Xcode Cloud | **$0** | 25 compute-hours/month included with the membership. Ample for a solo dev if triggers are gated (below). |
| GitHub Actions | **$0 (unchanged)** | Everything stays on free Linux runners. **No macOS minutes consumed** — this is the whole reason to pick Xcode Cloud over GitHub-hosted macOS, which on a *private* repo bills at 10× (~200 free min/month, one archive eats it). |
| **Net new recurring** | **$99/yr Apple + $0 CI** | The $99 is mandatory for *any* iOS distribution regardless of CI vendor. |

**Staying under the 25-hour Xcode Cloud budget.** KMP builds are slow — a cold run compiles the
Kotlin/Native framework and (for configuration) evaluates the `androidApp` module. Budget guard:

- Gate the **PR test** workflow to **PRs targeting `release` only**, mirroring `ui-tests.yml`.
  Day-to-day PRs into `development` keep their fast, free Linux checks; shared business logic is
  already covered cross-platform by host-JVM `commonTest`. iOS CI only needs to prove *the app links
  and launches on a simulator* + *archives*.
- Reserve the **archive → TestFlight** workflow for pushes to `release`, mirroring `build.yml`.
- Rough budget: even at a pessimistic ~20 min/run, 25 h ≈ 75 runs/month. Gated as above, real usage
  is a handful of runs per release — comfortably free.

---

## 3. The dependency gate

```
Phase 0: Apple Developer Program enrollment  ($99/yr, ~24–48h approval)
    │  (blocks EVERYTHING below — Xcode Cloud + TestFlight both need membership)
    ├──────────────► Phase 2: Xcode Cloud workflows
    │                Phase 3: signing + TestFlight delivery
    │                Phase 4: guardrails / parity
    │
Phase 1: project prep  ── can be done NOW, free, no account ──┐
    (wire test target, versioning, ci_scripts, local validation)
                                                              └──► feeds Phase 2
```

Do **Phase 1 now** (free) so that the moment enrollment clears, Phases 2–4 are pure configuration.

---

## 4. Phase 0 — Enrollment & App Store Connect (blocking, external)

1. Enrol in the **Apple Developer Program** (Individual is fine): <https://developer.apple.com/programs/enroll/>.
   Approval typically 24–48h. Record the **Team ID** (10-char, e.g. `ABCDE12345`).
2. In **App Store Connect**:
   - Register the App ID / bundle identifier **`pl.egzaminppl.app`** (Certificates, IDs & Profiles).
     Enable the capabilities the app uses (Push is **not** used; App Check/Firestore need none special).
   - Create the **app record**: name *Egzamin PPL*, primary language, bundle ID `pl.egzaminppl.app`, SKU.
   - Under **TestFlight**, create an **Internal Testing** group and add yourself as a tester.
3. Note: the debug bundle id `pl.egzaminppl.app.debug` does **not** need an App Store record — it never ships.

**Deliverable:** membership active, Team ID known, app record + TestFlight internal group exist.

---

## 5. Phase 1 — Project prep (do now, free, no account required)

All of this compiles and tests on the **simulator**, which needs no signing and no membership. It can
be validated entirely on your Mac today.

### 5a. Wire the UI-test target (makes the orphan test runnable)

In Xcode (hand-editing `project.pbxproj` is error-prone — do this in the GUI):

1. **File ▸ New ▸ Target ▸ UI Testing Bundle**, name **`iosAppUITests`**, Target to Test = `iosApp`.
2. Delete the auto-generated placeholder `.swift`; **add the existing** `FirstScreenLaunchTest.swift`
   to the new target (verify Target Membership).
3. Signing for the test target: **Automatic**, no explicit team (simulator tests don't sign).
4. Edit the shared **`iosApp` scheme ▸ Test action** → add `iosAppUITests` to the test targets.
   (The scheme is shared under `xcshareddata/xcschemes/` and is tracked by git per `.gitignore`.)
5. Commit the changed `project.pbxproj` + scheme.

**Local validation (free):**

```bash
xcodebuild test \
  -project iosApp/iosApp.xcodeproj \
  -scheme iosApp \
  -destination 'platform=iOS Simulator,name=iPhone 16'
```

This exercises the full path CI will use: Gradle builds the Kotlin framework via the existing build
phase, the app links, launches, and `FirstScreenLaunchTest` asserts the first screen appears.

### 5b. Unify versioning (single source of truth = `version.properties`)

Goal: **one marketing version across platforms**, monotonic per-platform build numbers.

- Bring the committed baseline into line so local Xcode builds match Android: set
  `MARKETING_VERSION` in `iosApp/Configuration/Config.xcconfig` to `1.1.0` (= current `VERSION_NAME`).
- In CI, derive both values (see `ci_pre_xcodebuild.sh` in 5c): `MARKETING_VERSION` from
  `version.properties:VERSION_NAME`, and `CURRENT_PROJECT_VERSION` from Xcode Cloud's
  `$CI_BUILD_NUMBER` (monotonic — the iOS analogue of Android's `VERSION_CODE = github.run_number`).
- `version-bump.yml` already owns `VERSION_NAME`, so bumping the marketing version stays a single
  action that now flows to **both** stores.

### 5c. Add Xcode Cloud custom build scripts

Xcode Cloud auto-runs any executable named `ci_post_clone.sh` / `ci_pre_xcodebuild.sh` /
`ci_post_xcodebuild.sh` in a `ci_scripts/` folder. These make the KMP build work on a clean Mac
image and wire versioning + Crashlytics symbols.

> ⚠️ **LOCATION GOTCHA (cost us two failed runs).** The `ci_scripts/` folder must sit **in the same
> directory as the `.xcodeproj` — i.e. `iosApp/ci_scripts/`, NOT the repository root.** At the repo
> root Xcode Cloud silently never runs the scripts, and the build fails downstream with the *exact
> same* error every time (for us: "Unable to locate a Java Runtime"). If a script "isn't running,"
> check its location first.

> ⚠️ **#1 integration risk.** The iOS app links a Gradle-produced Kotlin framework. Running Gradle
> needs a **JDK**, and configuring the build **evaluates the `androidApp` module**, which fails with
> *"SDK location not found"* unless an **Android SDK** location is provided (confirmed behaviour of
> this repo). So the clean Xcode Cloud image must get a JDK **and** an Android SDK before
> `embedAndSignAppleFrameworkForXcode` runs. Validate this script path early — it's the part most
> likely to need a tweak for the current Xcode Cloud image.

`ci_scripts/ci_post_clone.sh` (installs JDK 17 + Android cmdline-tools, points Gradle at them):

```sh
#!/bin/sh
# NB: no `pipefail` — `yes | sdkmanager` makes `yes` die by SIGPIPE, which pipefail
# would surface as a spurious failure.
set -eu

# --- JDK 17: install AND export JAVA_HOME for this script ---
brew install --quiet openjdk@17
# sdkmanager below is a Java program → this script needs JAVA_HOME (java_home can't see a brew keg).
# The build phase resolves Java independently via its own fallback (a separate process).
export JAVA_HOME="$(brew --prefix openjdk@17)/libexec/openjdk.jdk/Contents/Home"

# --- Android SDK (Gradle configures the composeApp/androidApp Android modules) ---
# The pinned bundle is too old to know API 37, so bootstrap the CURRENT cmdline-tools with it.
ANDROID_HOME="$HOME/android-sdk"; mkdir -p "$ANDROID_HOME"
# Compile SDK from the catalog (single source of truth), so a bump there needs no change here.
COMPILE_SDK="$(sed -nE 's/^[[:space:]]*android-compileSdk[[:space:]]*=[[:space:]]*"?([0-9]+)"?.*/\1/p' \
  "$CI_PRIMARY_REPOSITORY_PATH/gradle/libs.versions.toml")"
curl -fsSL "https://dl.google.com/android/repository/commandlinetools-mac-11076708_latest.zip" -o /tmp/cmdtools.zip
unzip -q -o /tmp/cmdtools.zip -d /tmp/cmdtools
BOOT="/tmp/cmdtools/cmdline-tools/bin/sdkmanager"
yes | "$BOOT" --sdk_root="$ANDROID_HOME" --licenses >/dev/null 2>&1 || true
"$BOOT" --sdk_root="$ANDROID_HOME" "cmdline-tools;latest" >/dev/null
SDKMANAGER="$ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager"
yes | "$SDKMANAGER" --licenses >/dev/null 2>&1 || true
"$SDKMANAGER" "platform-tools" "platforms;android-${COMPILE_SDK}" >/dev/null 2>&1 \
  || echo "note: AGP will auto-resolve the compile SDK"

# Gradle reads the Android SDK location from local.properties (sdk.dir).
echo "sdk.dir=$ANDROID_HOME" >> "$CI_PRIMARY_REPOSITORY_PATH/local.properties"
```

> Note on Java resolution: the stock "Compile Kotlin Framework" build phase did
> `export JAVA_HOME=$(/usr/libexec/java_home)`, which fails on the CI image (a bare `brew install`
> keg isn't registered in `/Library/Java/JavaVirtualMachines`, and `org.gradle.java.home` in
> `local.properties` is ignored — Gradle reads that only from `gradle.properties`). So the build
> phase was **modified** to resolve Java robustly: honor a valid `$JAVA_HOME`, else `java_home`
> (local dev — Android Studio JBR), else fall back to `/opt/homebrew/opt/openjdk@17` (the CI brew
> keg). This makes it independent of `sudo` and `java_home`; the `sudo -n ln` in `ci_post_clone` is
> now just best-effort insurance. Verified locally: `xcodebuild build` still green.

`ci_scripts/ci_pre_xcodebuild.sh` (versioning):

```sh
#!/bin/sh
set -eu
REPO="$CI_PRIMARY_REPOSITORY_PATH"
CONFIG="$REPO/iosApp/Configuration/Config.xcconfig"

VERSION_NAME="$(grep -E '^VERSION_NAME=' "$REPO/version.properties" | cut -d= -f2- | tr -d '[:space:]')"
[ -n "$VERSION_NAME" ] || { echo "error: VERSION_NAME not found" >&2; exit 1; }

# The values live in Config.xcconfig (not the pbxproj / a static Info.plist), so rewrite them
# there directly — agvtool edits the pbxproj and would miss the xcconfig. Marketing version from
# the shared source of truth; build number from Xcode Cloud's monotonic CI_BUILD_NUMBER.
sed -i '' -E "s/^MARKETING_VERSION=.*/MARKETING_VERSION=$VERSION_NAME/" "$CONFIG"
sed -i '' -E "s/^CURRENT_PROJECT_VERSION=.*/CURRENT_PROJECT_VERSION=$CI_BUILD_NUMBER/" "$CONFIG"
```

`ci_scripts/ci_post_xcodebuild.sh` (upload Crashlytics dSYMs on archive — iOS analogue of Android's
mapping/native-symbol upload):

```sh
#!/bin/sh
set -euo pipefail
# Only meaningful for archive builds (which produce dSYMs).
[ -n "${CI_ARCHIVE_PATH:-}" ] || exit 0

REPO="$CI_PRIMARY_REPOSITORY_PATH"
UPLOAD="$CI_DERIVED_DATA_PATH/SourcePackages/checkouts/firebase-ios-sdk/Crashlytics/upload-symbols"
DSYMS="$CI_ARCHIVE_PATH/dSYMs"
[ -x "$UPLOAD" ] && [ -d "$DSYMS" ] || exit 0

"$UPLOAD" -gsp "$REPO/iosApp/iosApp/GoogleService-Info.plist" -p ios "$DSYMS"
```

Make them executable and commit:

```bash
chmod +x ci_scripts/ci_*.sh
```

**Deliverable of Phase 1:** test target wired + green locally; versioning unified; `ci_scripts/`
committed; Kotlin framework builds headlessly. All free, all before enrollment.

---

## 6. Phase 2 — Xcode Cloud workflows (after Phase 0)

Set `TEAM_ID` in `Config.xcconfig` to the enrolled Team ID (so local archives resolve a team;
Xcode Cloud manages signing itself). Then in **Xcode ▸ Product ▸ Xcode Cloud ▸ Create Workflow**
(or App Store Connect ▸ your app ▸ Xcode Cloud):

1. **Grant repo access** — install/authorise the Xcode Cloud GitHub app on the **private** repo.
   This is what lets Xcode Cloud clone it and post build **status back to GitHub PRs**.
2. Pin the **Xcode version** to match local (Xcode 16.x, matching `objectVersion 77`).

**Workflow A — `iOS PR Tests`**
- **Start condition:** Pull Request changes, **target branch = `release`** (mirror `ui-tests.yml`
  gating to conserve hours).
- **Action:** **Test** — scheme `iosApp`, destination *iOS Simulator (latest)*.
- **No archive.** Posts a required-able status check to the PR.

**Workflow B — `iOS Release → TestFlight`**
- **Start condition:** Branch changes, **branch = `release`** (mirror `build.yml`).
- **Actions:** **Archive** (Release) — optionally a **Test** action first for a final gate.
- **Archive ▸ Distribution Preparation = "App Store Connect"** (the current UI offers None /
  TestFlight (Internal Testing Only) / App Store Connect). App Store Connect keeps the build
  store-promotable *and* still feeds TestFlight; Internal-Testing-Only builds can never be promoted
  to the store.
- **Post-action:** **TestFlight Internal Testing** → the internal group from Phase 0.
- **Environment: leave "Clean" UNCHECKED.** Clean disables all caching, so every run re-downloads
  the Firebase binary xcframeworks from dl.google.com — slower and re-exposed to the CDN timeouts
  that `ci_post_clone.sh`'s retry loop was added to survive.
- Runs `ci_pre_xcodebuild.sh` (versioning) and `ci_post_xcodebuild.sh` (dSYMs) automatically.

> **Expected-failure note (App Store Connect mode):** Xcode Cloud runs THREE exports — app-store,
> plus auto-generated **ad-hoc** and **development** side-exports. With zero registered devices the
> two side-exports always fail with *"No profiles for 'pl.egzaminppl.app' were found."* This is
> **benign**: the build is green as long as the app-store export succeeds and the TestFlight
> post-action uploads. Do not "fix" it by registering profiles/devices.

Optional: a tag-driven variant on `v*` if you prefer tags over branch pushes — but `build.yml`'s
primary trigger is the `release` push, so branch-push keeps the two platforms symmetric.

---

## 7. Phase 3 — Signing & delivery (after Phase 0)

- **Use Xcode Cloud managed (automatic) signing.** It provisions the distribution certificate and
  provisioning profile for `pl.egzaminppl.app` itself. **No certs/profiles/API keys stored in the
  repo or GitHub secrets** — a major simplification vs a fastlane-match setup, and the main payoff of
  choosing Xcode Cloud. (For contrast: a GitHub-Actions macOS path would have needed an App Store
  Connect API key + `fastlane match` on a private cert repo.)
- After a release run, the build appears in **TestFlight** automatically once Apple finishes
  processing; internal testers are notified. Promotion to the App Store stays a **manual** decision
  in App Store Connect — matching the "no auto-publish" philosophy of the Android pipeline.

---

## 8. Phase 4 — Guardrails & Android parity

- **Branch protection:** once `iOS PR Tests` is green a few times, add its check as a **required
  status** on `release` PRs, alongside the Android emulator check.
- **Crashlytics:** confirm the iOS app shows up in the Firebase console with symbolicated crashes
  (validates `ci_post_xcodebuild.sh`).
- **Versioning flow:** `version-bump.yml` → `VERSION_NAME` → picked up by `ci_pre_xcodebuild.sh` on
  the next release run. One bump, both platforms.
- **Docs:** link this file from `README.md` next to the CI badges. (Xcode Cloud has no Actions-style
  status badge; link to App Store Connect instead if desired.)

### Parity summary

| Concern | Android (today) | iOS (this plan) |
|---|---|---|
| PR tests | `ci.yml` unit + compile (every PR); `ui-tests.yml` emulator (→ release PRs) | Xcode Cloud **Workflow A**: simulator test on → release PRs |
| Release build | `build.yml`: signed AAB + mapping | Xcode Cloud **Workflow B**: signed archive |
| Delivery | Manual Play upload | **Auto → TestFlight** internal (manual store promote) |
| Build number | `github.run_number` | `CI_BUILD_NUMBER` |
| Marketing version | `version.properties:VERSION_NAME` | same, via `ci_pre_xcodebuild.sh` |
| Crash symbols | NDK symbols + R8 mapping in AAB/Release | dSYMs → Firebase via `ci_post_xcodebuild.sh` |
| Signing secrets | Keystore in GitHub secrets | **None** — Xcode Cloud managed signing |
| CI cost | Free (Linux) | Free (25 Mac-h/mo) + $99/yr membership |

---

## 9. Risks & gotchas

1. **Enrollment gates everything.** Xcode Cloud *and* TestFlight both need the paid account; nothing
   in Phases 2–4 is possible until Phase 0 clears. Phase 1 is the only pre-enrollment work.
2. **KMP on a clean Mac image** (§5c) — JDK + Android SDK provisioning is the most fragile step;
   validate the `ci_post_clone.sh` path on the first Xcode Cloud run and adjust for the current image.
3. **Kotlin/Native build time vs the 25-h budget** — cold framework compiles are slow and Xcode Cloud
   caches SPM/Swift better than arbitrary Gradle/Kotlin caches. Keep triggers gated (§2); watch usage
   in App Store Connect the first month.
4. **Monotonic build numbers** — TestFlight rejects a duplicate `CURRENT_PROJECT_VERSION` for the same
   marketing version. `CI_BUILD_NUMBER` guarantees monotonicity; don't hand-set it.
5. **The test target must actually be created** — the `.swift` file alone runs nothing (§5a).
6. **iPhone-only, iOS 15+** — `TARGETED_DEVICE_FAMILY = 1`; App Review will test on iPhone only.
   Fine for the current scope, just be intentional if iPad is wanted later.
7. **ASC agreements silently gate ALL uploads** (hit 2026-07-09/10). A fresh enrollment leaves the
   *Paid & Free Applications Agreement* in "New" state; until accepted, every upload fails with
   `Failed to find an account with App Store Connect access for team teamID='…', teamName='(null)'`
   — while archive and even the app-store **export still succeed**, so the build looks nearly green.
   `teamName='(null)'` is the smoking gun. Accept the agreement terms (bank/tax forms not needed for
   a free app) at ASC ▸ Business ▸ Agreements, then allow propagation (hours → up to 48 h after
   enrollment). Switching Distribution Preparation modes does NOT help — both modes upload to ASC.
8. **Automatic signing needs ≥1 registered device to archive LOCALLY** — even though App Store
   distribution signing itself needs no devices. `CODE_SIGN_STYLE = Automatic` insists on managing a
   *development* profile, which cannot exist with zero devices; hardcoding
   `CODE_SIGN_IDENTITY = "Apple Distribution"` under automatic signing is rejected as "conflicting
   provisioning settings". Local paths: register one device (automatic then works; Organizer
   re-signs for distribution on upload), or switch to full manual signing. **Xcode Cloud managed
   signing needs no devices at all** — proven by a green archive + export with zero registered.
```
