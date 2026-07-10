# Report a Problem — Implementation Plan

Developer-executable, file-level plan. Inputs: [concept.md](concept.md) (storage/schema/review)
and [functional-design.md](functional-design.md) (UX/flows/copy). The feature is a near-clone of the
shipped **content-voting** stack; every unfamiliar decision below is anchored to a concrete existing
file. Verified against `development` (2026-07-07).

Run shared tests on the JVM host: `./gradlew :composeApp:testAndroidHostTest`. iOS compile check:
`./gradlew :composeApp:compileKotlinIosSimulatorArm64`. Emulator DI launch: `pl.egzaminppl.app.debug`.

---

## 1. Overview + architecture-decision deltas

Layers cloned from voting: `ProblemReason` (domain enum) + `ProblemReportRepository` interface +
`Submit`/`ObserveProblemReportUseCase` → `ProblemReportRepositoryImpl` →
`FirestoreProblemReportDataSource` (GitLive, best-effort write). `ProblemReportController` mirrors
`ContentVoteController`, owned by `QuestionViewModel` + `QuestionPreviewViewModel`. EP2 is Settings-only.

Deltas the specs left to the architect (decided here):

1. **Server timestamp type.** Rules require `createdAt == request.time`, so the client cannot send a
   local clock value — it must send the server-timestamp sentinel. GitLive 2.1.0 exposes
   `dev.gitlive.firebase.firestore.Timestamp.ServerTimestamp : BaseTimestamp` (serialized via
   `BaseTimestampSerializer` → `FieldValue.serverTimestamp`). DTO field:
   `createdAt: BaseTimestamp = Timestamp.ServerTimestamp`. This is the one gitlive type allowed into a
   DTO; votes had no timestamp so this is new. `.set(dto)` must run with the default
   `encodeDefaults = true` so both `createdAt` and the constant `type` default are written (rules
   `hasAll` them). **Verify** the gitlive `set` default on the emulator.

2. **Platform name for EP2.** No platform-name provider exists (`BuildInfo` only has `isDebugBuild`).
   Extend the already-platform-specific `AppVersionProvider` with `platformName(): String`
   ("Android" / "iOS"). Leanest option — no new DI binding. Minor ISP cost (version + platform on one
   interface) accepted; the alternative `DeviceInfoProvider` is a concept *Could*, unneeded for a name.

3. **mailto percent-encoding.** No common URL encoder in the codebase. Add a pure-Kotlin RFC-3986
   percent-encoder in `commonMain` (`ui/settings/FeedbackMailto.kt`) — host-testable, no platform dep.

4. **Reported-state cue = one DataStore-backed mechanism.** Instead of a separate in-memory (Must) and
   persisted (Should) path, use the vote pattern: a DataStore key is the single source of truth, so the
   cue is both in-session and persisted at once (idempotent re-submits make lag harmless). Key
   `report_{code}_{lang}_{rev}` holds a **string set** of submitted reason names; rev in the key means
   a regenerated explanation auto-resets (no stale-rev logic needed). `observeReported` = set non-empty.

5. **Identity promotion.** Rename `data/contentvote/VoterIdProvider.kt` → `data/identity/InstallIdProvider.kt`,
   **keep the DataStore key `"voter_id"`** (existing installs keep identity). Consumed by both
   `ContentVoteRepositoryImpl` (ctor swap) and the new report repository.

6. **Target shape.** `ProblemReportTarget(code, rev)` mirrors `ContentVoteTarget`; `categoryId` is a
   controller ctor param (constant per screen). `code` = **`question.code`** (the ULC join key the
   review script needs), `rev` = explanation rev. `license`/`lang` are pulled from providers in the use
   case (exactly like `SubmitContentVoteUseCase` pulls `lang`).

7. **Note sanitization** (cap 500; keep `\n` and printable text; drop C0/DEL/C1 control chars; trim a
   surrogate half split by the cap) is a pure function in the use case — testable; rules remain the
   enforcement boundary.

8. **RC-configurable support email (EP2).** The support address is a Remote Config **string**
   (`support_email`) so it can be rotated without an app update, with the same address compiled in as the
   fetch-failure/offline/first-launch default. It is a config value, not a feature flag, so it stays out
   of the `FeatureFlag` enum and the boolean-only `RemoteFeatureFlagSource`. A dedicated
   `SupportEmailProvider` (domain) / `RemoteConfigSupportEmailProvider` (data) reads the same
   `Firebase.remoteConfig` singleton the flag source already configures and fetches — no second RC
   configurator. Verified: GitLive `FirebaseRemoteConfigValue.asString()` is the string-read counterpart
   to the `.asBoolean()` the flag source uses; a blank/unknown key returns `""`, handled by
   `resolveSupportEmail(remote) = remote.ifBlank { DEFAULT_SUPPORT_EMAIL }`. See Phase 4.

Discrepancy vs brief: brief says `FirestoreContentVoteDataSource or equivalent, ContentVoteDto` live in
`data/source/` — confirmed. Brief lists `domain/contentvote/` — it does **not** exist; vote domain is
`domain/model/ContentVote.kt` + `domain/repository/ContentVoteRepository.kt` +
`domain/usecase/contentvote/`. Plan follows the real layout: model in `domain/model/`, use cases in
`domain/usecase/problemreport/`. No new libraries needed (`libs.versions.toml` unchanged).

---

## 2. Phased plan (each phase compiles, tests, merges independently)

### Phase 0 — Identity promotion (PR #A, prep, no behavior change)
- **Move** `data/contentvote/VoterIdProvider.kt` → `data/identity/InstallIdProvider.kt`; rename class
  `VoterIdProvider` → `InstallIdProvider`; keep `stringPreferencesKey("voter_id")` and the `get()` body.
- **Modify** `data/repository/ContentVoteRepositoryImpl.kt` — ctor param type + import.
- **Modify** `di/AppModule.kt` — `singleOf(::VoterIdProvider)` → `singleOf(::InstallIdProvider)`, import.
- **Move/rename test** `data/contentvote/VoterIdProviderTest.kt` → `data/identity/InstallIdProviderTest.kt`;
  update class + import. `ContentVoteRepositoryImplTest.kt` — swap `VoterIdProvider(store)` →
  `InstallIdProvider(store)`.
- **Verify:** `:composeApp:testAndroidHostTest`; `:composeApp:compileKotlinIosSimulatorArm64`; emulator
  launch (DI graph binds; vote still works).

### Phase 1 — Domain + data write path (PR #B, headless, dark)
Create:
- `domain/model/ProblemReason.kt`
  ```kotlin
  enum class ProblemReason(val firestoreValue: String) {
      WrongAnswer("wrong_answer"), BadExplanation("bad_explanation"), QuestionError("question_error")
  }
  ```
- `domain/repository/ProblemReportRepository.kt`
  ```kotlin
  interface ProblemReportRepository {
      fun observeReported(code: String, lang: String, rev: String): Flow<Boolean>
      suspend fun submitReport(
          code: String, categoryId: String, license: String, lang: String, rev: String,
          reason: ProblemReason, note: String,
      )
  }
  ```
- `domain/usecase/problemreport/SubmitProblemReportUseCase.kt` — ctor
  `(repo, languageProvider: LanguageProvider, licenseProvider: LicenseProvider)`;
  `invoke(code, categoryId, rev, reason, note)` sanitizes note, resolves
  `license = licenseProvider.current()?.id.orEmpty()`, `lang = languageProvider.currentLanguage().code`,
  delegates. Sanitizer caps at 500, trims a trailing split surrogate half, then
  `filter { it == '\n' || it.code in 0x20..0x7E || it.code >= 0xA0 }` (drops C0-except-newline, DEL, C1).
- `domain/usecase/problemreport/ObserveProblemReportUseCase.kt` — mirrors `ObserveContentVoteUseCase`:
  `languageProvider.observeLanguage().flatMapLatest { repo.observeReported(code, it.code, rev) }`.
- `data/source/dto/ProblemReportDto.kt`
  ```kotlin
  @Serializable
  internal data class ProblemReportDto(
      val type: String = "content",
      val reason: String, val code: String, val categoryId: String,
      val license: String, val lang: String, val rev: String, val note: String,
      val appVersion: String, val reporterId: String,
      val createdAt: BaseTimestamp = Timestamp.ServerTimestamp,
  )
  ```
- `data/source/ProblemReportDataSource.kt` — `internal interface { suspend fun setReport(dto) }`
  (write-only; no delete — client `delete: false`).
- `data/source/FirestoreProblemReportDataSource.kt` — mirrors `FirestoreContentVoteDataSource`. The id is
  built by a pure, host-testable `reportDocId(dto)` (so it can be checked against the rules id binding
  without a live Firestore) and includes **`lang`** — a report is language-specific, so PL and EN must not
  collide onto one doc:
  ```kotlin
  internal fun reportDocId(dto: ProblemReportDto) =
      "${dto.reporterId}:${dto.code}:${dto.lang}:${dto.rev}:${dto.reason}"
  override suspend fun setReport(dto: ProblemReportDto) =
      db.collection(FirestoreCollections.PROBLEM_REPORTS).document(reportDocId(dto)).set(dto)
  ```
- `data/repository/ProblemReportRepositoryImpl.kt` — ctor
  `(dataStore, dataSource, installIdProvider, appVersionProvider)`. A single `normalizeRev` owner applies
  `rev.ifBlank { "_" }` to the DTO `rev`, the doc id, and the local cue key so the rules binding holds;
  `submitReport` builds the DTO and writes best-effort in `try/catch` + `e.rethrowIfCancellation()` +
  `Logger.e` (copy vote impl), then
  records the local cue set. `observeReported` = `dataStore.data.map { it[reportKey(code,lang,rev)]?.isNotEmpty() == true }`.
  Key: `stringSetPreferencesKey("report_${code}_${lang}_${rev}")`; on submit add `reason.firestoreValue`.

Modify:
- `data/source/FirestoreCollections.kt` — `const val PROBLEM_REPORTS = "problemReports"`.
- `di/AppModule.kt` — add:
  ```kotlin
  singleOf(::FirestoreProblemReportDataSource) bind ProblemReportDataSource::class
  singleOf(::ProblemReportRepositoryImpl) bind ProblemReportRepository::class
  factoryOf(::ObserveProblemReportUseCase)
  factoryOf(::SubmitProblemReportUseCase)
  ```
  (`ProblemReportRepositoryImpl` has no defaulted ctor args → `singleOf` is safe; verify at launch.)

Tests (commonTest):
- `data/source/FakeProblemReportDataSource.kt` — records `setCalls: MutableList<ProblemReportDto>`, `failOnWrite`.
- `data/repository/FakeProblemReportRepository.kt` — for controller/VM tests.
- `data/repository/ProblemReportRepositoryImplTest.kt` — mirrors `ContentVoteRepositoryImplTest`:
  submit records one DTO with mapped fields; `observeReported` true after submit / false for other rev;
  second distinct reason keeps the set non-empty; remote failure never breaks the local cue; blank rev →
  `_` in id + payload.
- `domain/usecase/problemreport/SubmitProblemReportUseCaseTest.kt` — license/lang wiring; note truncation
  to 500; control-char stripping keeps newlines.
- **Verify:** `:composeApp:testAndroidHostTest`; iOS compile (BaseTimestamp/Timestamp resolve on native);
  emulator launch (`ProblemReportRepositoryImpl` binds; no crash).

### Phase 2 — Flags + controller + UI components (PR #C, dark, flags default false)
Modify:
- `domain/featureflag/FeatureFlag.kt` — add
  `ProblemReporting("problem_reporting", false)`, `ContactUs("contact_us", false)`. (RemoteConfig
  defaults auto-register via `FeatureFlag.entries` in `RemoteConfigFeatureFlagSource`.)

Create:
- `ui/problemreport/ProblemReportController.kt` — mirrors `ContentVoteController`:
  ```kotlin
  data class ProblemReportTarget(val code: String, val rev: String)
  class ProblemReportController(
      private val scope: CoroutineScope,
      targets: Flow<ProblemReportTarget?>,
      private val categoryId: String,
      private val observeProblemReport: ObserveProblemReportUseCase,
      private val submitProblemReport: SubmitProblemReportUseCase,
      val isEnabled: Boolean,
  ) {
      private val target = targets.distinctUntilChanged().stateIn(scope, Eagerly, null)
      val reportedState: StateFlow<Boolean> = target
          .flatMapLatest { t -> if (t == null) flowOf(false) else observeProblemReport(t.code, t.rev) }
          .stateIn(scope, WhileSubscribed(5_000), false)
      fun onReport(reason: ProblemReason, note: String) {
          if (!isEnabled) return
          val t = target.value ?: return
          scope.launch { submitProblemReport(t.code, categoryId, t.rev, reason, note) }
      }
  }
  ```
- `ui/components/ProblemReportSheet.kt` — `AppBottomSheet` content, `onDismiss`, `onSubmit`. Local
  `reason: ProblemReason?` + `note: String` remember-state. 3 `AppBottomSheetRow`s in a
  `Modifier.selectableGroup()` (`Role.RadioButton`, selected check, tap **does not** dismiss). Note:
  `OutlinedTextField`, `onValueChange = { if (it.length <= 500) note = it }`, `supportingText` counter
  shown from 400. Muted privacy line. Full-width `Button(enabled = reason != null)` → `LongPress`
  haptic, `onSubmit(reason!!, note)`, `onDismiss()`. Content `Modifier.imePadding()`. Reason label
  resolver `ProblemReason.labelRes()` local `when`.
- `ui/components/explanation/ReportFlagButton.kt` — mirrors `VoteBar`'s `ThumbButton`: 48dp `IconButton`,
  22dp glyph, `onSurfaceVariant`→`primary` tint, `Icons.Outlined.Flag`↔`Icons.Filled.Flag`,
  `contentDescription` swap per `reported`, **no** `selected` semantics (action, not toggle). Owns
  `var showSheet by remember`; renders `ProblemReportSheet` when open; `onSubmit` forwards to
  `onReport(reason, note)`.

Modify:
- `ui/components/explanation/ExplanationCard.kt` — add params
  `reported: Boolean = false, onReport: ((ProblemReason, String) -> Unit)? = null`. In the `Content`
  branch footer:
  ```kotlin
  if (onReport != null) {
      Row(Modifier.fillMaxWidth(), verticalAlignment = Alignment.CenterVertically) {
          ReportFlagButton(reported = reported, onReport = onReport)
          if (onVote != null) VoteBar(vote, onVote, Modifier.weight(1f))
          else Spacer(Modifier.weight(1f))
      }
  } else if (onVote != null) {
      VoteBar(vote = vote, onVote = onVote)   // unchanged voting-only path
  }
  ```
- `ui/UiTestTags.kt` — add `object Report { const val FLAG = "report_flag"; const val SHEET = "report_sheet";
  const val SEND = "report_send"; fun reason(v: String) = "report_reason_$v" }`.

Tests:
- `commonUiTest ui/components/ProblemReportSheetTest.kt` — Send disabled until a reason is tapped;
  selecting a reason does not dismiss; typing past 500 is clamped; `onSubmit` carries reason + note.
- `commonUiTest ui/components/explanation/ReportFlagButtonTest.kt` — outlined vs filled contentDescription
  per `reported`; tap opens the sheet.
- Extend `commonUiTest .../explanation/ExplanationCardTest.kt` — flag renders iff `onReport != null`;
  co-exists with thumbs.
- `ui/problemreport/ProblemReportControllerTest.kt` — null target ⇒ `reportedState` false + `onReport`
  no-ops; `isEnabled = false` ⇒ submit never called (`TestScope`, `FakeProblemReportRepository`).
- **Verify:** host + iOS compile; run app, force-enable flag via debug override, eyeball the sheet both
  themes + IME.

### Phase 3 — Wire content surfaces (PR #D, still dark)
Modify:
- `ui/question/QuestionViewModel.kt` — add ctor params `observeProblemReportUseCase`,
  `submitProblemReportUseCase`. Build `ProblemReportController` positionally (like `voteController`),
  `targets` = the same `combine(sessionStateFlow, explanationStates)` mapping to
  `ProblemReportTarget(it.code, it.rev)`, `categoryId = categoryId`,
  `isEnabled = featureFlagService.isEnabled(FeatureFlag.ProblemReporting)`. Expose
  `val isReportingEnabled get() = reportController.isEnabled`,
  `val reportedState get() = reportController.reportedState`,
  `fun onReport(reason, note) = reportController.onReport(reason, note)`.
- `ui/questionpreview/QuestionPreviewViewModel.kt` — same, `targets` from `_uiState.map { ... }` like
  `voteController`, `categoryId = categoryId`.
- `di/AppModule.kt` — pass `observeProblemReportUseCase = get()`, `submitProblemReportUseCase = get()`
  into the `questionViewModel(...)` helper and the `QuestionPreviewViewModel { }` block.
- `ui/question/QuestionScreen.kt` — collect `reported by viewModel.reportedState.collectAsState()`; pass
  `reported = reported`, `onReport = if (viewModel.isReportingEnabled) viewModel::onReport else null` down
  through `QuestionContent` → `ExplanationCard`.
- `ui/questionpreview/QuestionPreviewScreen.kt` — same into `QuestionPreviewContent` → `ExplanationCard`.

Tests: extend `commonUiTest .../QuestionContentTest.kt` + `.../QuestionPreviewContentTest.kt` — flag shown
when `onReport != null`. VM host tests bypass Koin, so **emulator launch is mandatory** here (new ctor
params + positional controller): open a wrong-answer explanation live and a preview in review, confirm
the flag appears with the flag override on and is absent with it off.

### Phase 4 — Settings feedback lane EP2 (PR #E, independent flag)
Support address is **RC-configurable** (rotatable without an app update; delta 8): a Remote Config
string `support_email`, with the same address compiled in as the fetch-failure/offline/first-launch
default. It is a config *value*, not a feature flag, so it does **not** join the `FeatureFlag` enum and
must not pollute the boolean-only `RemoteFeatureFlagSource`. A dedicated provider is cleaner than
extending `RemoteConfigFeatureFlagSource`; it reads the **same** `Firebase.remoteConfig` singleton, so it
rides that class's existing single `setDefaults`/`fetchAndActivate` cycle with no second RC configurator.

Create:
- `domain/service/SupportEmailProvider.kt` — `interface SupportEmailProvider { fun current(): String }`
  (next to `AppVersionProvider`/`LicenseProvider`).
- `data/service/RemoteConfigSupportEmailProvider.kt` — `internal class RemoteConfigSupportEmailProvider :
  SupportEmailProvider`, reading GitLive's cross-platform string API
  `Firebase.remoteConfig.getValue(SUPPORT_EMAIL_KEY).asString()` (verified `FirebaseRemoteConfigValue`
  exposes `.asString()`; the flag source already reads `.asBoolean()` the same way). Compiled default +
  key are the single config point, in this file:
  ```kotlin
  internal const val SUPPORT_EMAIL_KEY = "support_email"
  internal const val DEFAULT_SUPPORT_EMAIL = "radoslaw.latka.dev@gmail.com"
  // Pure, host-testable fallback: blank (unknown key / pre-fetch / offline) → compiled default.
  internal fun resolveSupportEmail(remote: String): String = remote.ifBlank { DEFAULT_SUPPORT_EMAIL }
  override fun current(): String = resolveSupportEmail(
      runCatching { remoteConfig.getValue(SUPPORT_EMAIL_KEY).asString() }.getOrDefault(""),
  )
  ```
  `.ifBlank { }` covers first-launch/offline/fetch-failure without touching the flag source's
  `setDefaults` (an unknown key returns `""`); registering an RC in-app default is therefore optional and
  deliberately omitted to keep the flag-source enum-scoped.
- `ui/settings/FeedbackMailto.kt` — pure `internal fun percentEncode(String): String` (RFC-3986,
  encode everything except unreserved) + `internal fun buildFeedbackMailto(email, subject, body): String`
  → `"mailto:$email?subject=${percentEncode(subject)}&body=${percentEncode(body)}"`.

Modify:
- `domain/service/AppVersionProvider.kt` — add `fun platformName(): String`.
- `androidMain .../AndroidAppVersionProvider.kt` — `override fun platformName() = "Android"`.
- `iosMain .../IosAppVersionProvider.kt` — `override fun platformName() = "iOS"`.
- `ui/settings/SettingsViewModel.kt` — add ctor params `featureFlagService: FeatureFlagService`
  (**not** currently injected) and `supportEmailProvider: SupportEmailProvider`. Add to
  `SettingsUiState.Content`: `isFeedbackEnabled: Boolean` (=`featureFlagService.isEnabled(ContactUs)`),
  `platform: String` (=`appVersionProvider.platformName()`), `supportEmail: String`
  (=`supportEmailProvider.current()`).
- `di/AppModule.kt` — add `singleOf(::RemoteConfigSupportEmailProvider) bind SupportEmailProvider::class`
  (no-arg ctor → `singleOf` safe). `SettingsViewModel` is `viewModelOf(::SettingsViewModel)`; the two new
  params (`FeatureFlagService`, `SupportEmailProvider`) resolve automatically (both are `single`). Confirm.
- `ui/settings/SettingsScreen.kt` — in the About `SettingsSection`, when `state.isFeedbackEnabled`,
  prepend a feedback `SettingsRow` above `AppVersionRow` (section shapes recompute by index). Row:
  `Icons.Outlined.MailOutline`, `AccentIconChip` tone (e.g. `AccentTones.Teal.forTheme()`), title
  `settings_item_feedback_title`, subtitle `settings_item_feedback_subtitle`, `onClick = openFeedback`.
  `openFeedback` builds subject/body via `stringResource(feedback_email_subject, platform, version)` +
  `stringResource(feedback_email_body, version, platform, licenseDisplay, langCode)` (licenseDisplay =
  `stringResource(state.selectedLicense.displayNameRes())`), then:
  ```kotlin
  runCatching { uriHandler.openUri(buildFeedbackMailto(state.supportEmail, subject, body)) }
      .onFailure { scope.launch { clipboard.setPlainText(FEEDBACK_CLIP_LABEL, state.supportEmail) } }
  ```
  (reuses `LocalClipboard` + `setPlainText` already imported in this file; the address is the RC-resolved
  `state.supportEmail`, not a hardcoded const).

Tests:
- `ui/settings/FeedbackMailtoTest.kt` (commonTest) — spaces→`%20`, newline→`%0A`, `@`/`:` preserved-or-
  encoded consistently, full URL shape.
- `data/service/RemoteConfigSupportEmailProviderTest.kt` (commonTest) — pure `resolveSupportEmail`:
  blank/whitespace → `DEFAULT_SUPPORT_EMAIL`, non-blank RC value passes through. The live
  `Firebase.remoteConfig.getValue` read is **not** host-testable (touches Firebase init, like
  `RemoteConfigFeatureFlagSource`) → covered by the emulator step, not a unit test.
- **Verify:** host + iOS compile (GitLive `asString()` is commonMain); emulator with `contact_us`
  override on → tap opens composer prefilled to the default address; set the RC `support_email` param to a
  different value, `refresh`, confirm the composer uses it; simulate no mail app (uninstall Gmail or force
  failure) → address lands on clipboard (OS toast).

### Phase 5 — Rules + review CLI + copy (PR #F, ships with/after client)
- `firestore.rules` — the `problemReports` block is **already added (PR #58, §3)** and host-guarded by
  `ProblemReportContractTest`. Phase 5 adds the emulator write-conformance test (a conforming write
  succeeds; over-cap `categoryId` / missing `createdAt` / bad `reason` / id mismatch are rejected). Deploy
  is a **separate rollout step**, not a code merge.
- `scripts/report_review.py` — new (§4). The CLI splits the id on the first 4 colons (5-part id, §3).
- `composeResources/values/strings.xml` + `values-pl/strings.xml` — all keys from functional-design §9
  (§ below). Add these in Phase 2/4 alongside the UI that consumes them; listed here for completeness.

### Phase 6 — Privacy policy & store data-safety (PR #G, gates `problem_reporting` flag-on)
The privacy policy is **in-repo**: `docs/legal/privacy-en.md` + `docs/legal/privacy-pl.md`, republished by
`.github/workflows/publish-privacy-policy.yml` on change. So the policy edits are a real PR; the store
form/label changes are external console deliverables inside the same phase. This phase has no code and
does not gate the merge of Phases A–F — it gates the **`problem_reporting` flag flip** only. `contact_us`
is **not** gated by it: the mailto lane sends nothing to our storage (the email leaves via the user's own
mail client, consent-based), so EP2 can be enabled without a policy change.

Modify (PR #G, model the clause on the existing content-voting wording — §3 table row, §4 legal basis,
§6 retention, §7 rights):
- `docs/legal/privacy-en.md` + `docs/legal/privacy-pl.md` — bump "Last updated"; add a **problem-reports**
  row/clause covering:
  - **What is stored** (Cloud Firestore collection `problemReports`): the per-install Voter/Install ID
    (the same random UUID already disclosed for votes), the reported question's content `code`,
    `categoryId`, `license`, `lang`, explanation `rev`, the chosen `reason`, an **optional free-text note
    (≤ 500 chars)** the user types, the `appVersion`, and a server-set timestamp. No name, email, device
    model, OS, or location.
  - **Purpose:** content quality — identifying and triaging content defects (wrong answer key, faulty
    explanation, question error).
  - **Legal basis:** legitimate interest (Art. 6(1)(f) GDPR), mirroring the votes clause — improving
    content quality and preventing manipulation of the feedback.
  - **Retention:** resolved reports are purged after N days via `report_review.py --purge` (state the N
    once decided; content-voting §7.5 keeps this as hygiene, not a hard promise — keep the wording
    consistent, "kept only as long as useful").
  - **Explicitly:** there is **no reply channel** to a report, and users are told **not to include
    personal data** in the note (matches the in-app privacy microcopy, functional-design §3/§9).
- `.github/workflows/publish-privacy-policy.yml` — no change expected (auto-republishes on `docs/legal/**`
  change); confirm the trigger path covers both files.

External (console deliverables, no PR) — captured as an executable runbook in
[store-data-safety.md](store-data-safety.md):
- **Google Play Data safety form delta:** under "Data collected", add/adjust the App-activity /
  App-info-and-performance categories to reflect an optional free-text **"other user-generated content"**
  item (the note) and the existing device-scoped identifier; mark it **not linked to identity**, collected
  for **App functionality / analytics-adjacent content quality**, **not shared**, **not used for tracking**.
  The install-ID/diagnostic entries already exist for voting; the new item is the optional note.
- **Apple App Store privacy labels delta:** add **"User Content" → Other User Content"** (the note) under
  *Data Not Linked to You*, purpose *App Functionality*; the identifier entry already exists for votes.

Exit criterion (**hard gate for `problem_reporting`**): both markdown files updated + published, and both
store forms updated, **before** the `problem_reporting` RC flag is enabled. Verify the published EN/PL URLs
render the new clause. (Privacy sign-off is functional-design §11.2 / concept §8.4.)

---

## 3. `firestore.rules` — `problemReports` block

**Implemented in PR #58** (the block is in `firestore.rules`, after `contentVotes`). Cloned style, hardened
per functional-design §6 (type-pin + per-field cap + enum + id binding; `read: false`, `delete: false`).
Deploy stays a **separate rollout step** (§6) — a code merge does not push rules. Two corrections vs the
original draft, applied both here and in the client: `lang` is part of the id binding (a report is
language-specific, so PL and EN must not collide), and the `categoryId` cap is **32** (the longest slug,
`aircraft_general_knowledge`, is 26). The shipped block:

```
    match /problemReports/{reportId} {
      allow read: if false;
      allow delete: if false;

      allow create, update: if isFromVerifiedApp()
        && request.resource.data.keys().hasOnly(
             ['type', 'reason', 'code', 'categoryId', 'license', 'lang', 'rev',
              'note', 'appVersion', 'reporterId', 'createdAt'])
        && request.resource.data.keys().hasAll(
             ['type', 'reason', 'code', 'categoryId', 'license', 'lang', 'rev',
              'note', 'appVersion', 'reporterId', 'createdAt'])
        && request.resource.data.type == 'content'
        && request.resource.data.reason in ['wrong_answer', 'bad_explanation', 'question_error']
        && request.resource.data.code is string && request.resource.data.code.size() <= 16
        && request.resource.data.categoryId is string && request.resource.data.categoryId.size() <= 32
        && request.resource.data.license is string && request.resource.data.license.size() <= 16
        && request.resource.data.lang is string && request.resource.data.lang.size() <= 8
        && request.resource.data.rev is string && request.resource.data.rev.size() <= 64
        && request.resource.data.note is string && request.resource.data.note.size() <= 500
        && request.resource.data.appVersion is string && request.resource.data.appVersion.size() <= 32
        && request.resource.data.reporterId is string && request.resource.data.reporterId.size() <= 64
        && request.resource.data.createdAt == request.time
        && reportId == request.resource.data.reporterId + ':' + request.resource.data.code + ':'
             + request.resource.data.lang + ':' + request.resource.data.rev + ':'
             + request.resource.data.reason;
    }
```

Note: the client keeps `code`/`lang`/`rev` colon-free (ULC codes, language codes, and revs are; the id
split in the CLI assumes it). Client↔rules parity is guarded on the host by `ProblemReportContractTest`
(doc-id shape + the DTO field set vs the whitelist above); the live write-conformance test
(a conforming write succeeds; over-cap `categoryId` / missing `createdAt` / bad `reason` / id mismatch are
rejected) is emulator-only — see §5 / the rollout gate.

---

## 4. `scripts/report_review.py` — CLI spec

Bootstrap identical to `feed_generator.py`: self-venv, `firebase_admin` +
`credentials.Certificate(service_account.json)` + `firestore.client()` (Admin SDK bypasses rules → reads
the read-denied collection). `argparse` subcommands/flags:

- `list` (default) — read all `problemReports` where `status` is absent or `!= 'resolved'`
  (`collection('problemReports').stream()`, filter in Python — no composite index needed, alpha volume).
  Group by `code`; per group compute distinct-reporter count (`{r['reporterId']}`); sort groups
  descending by that count. For each group join `/questions/{categoryId}/langs/{lang}/chunks` to print
  question text + stored `correctOptionId`/answer, and per report print `reason`, sanitized `note`,
  `appVersion`, `createdAt`. Sanitize note before printing: strip ANSI CSI + C0/C1 control chars
  (`re.sub(r'[\x00-\x1f\x7f-\x9f]', ' ', note)`), truncate for display.
- Rev-staleness auto-filter: for `bad_explanation` reports, fetch the current explanation rev via
  `/explanationMaps/{categoryId}` (code→key) then `/explanations/{key}/langs/{lang}.rev`; hide reports
  whose stored `rev` != current (a rev bump already addressed them). `wrong_answer`/`question_error`
  have no rev lever — always shown until resolved.
- `--resolve <reportId> [--status resolved|dismissed]` — Admin-SDK write of `status` (+ `reviewedAt`).
  The client key-whitelist locks the client schema; the server annotates freely.
- `--purge --older-than <N>` — delete `status == resolved` docs with `reviewedAt` older than N days
  (retention hygiene, mirrors vote cleanup).
- `--service-account <path>` (default `service-account.json`), `--json` (machine output).

Output sketch:
```
CODE PL0421  ▲3 reporters  cat=air_law lang=pl
  Q: "Ciśnienie standardowe wg ISA na poziomie morza wynosi…"   ✓ correct: B
  • wrong_answer  v1.4.0  2026-07-06  "odpowiedź B jest zła, powinno być C"
  • wrong_answer  v1.4.0  2026-07-05  (no note)
  • question_error v1.4.0 2026-07-05  "literówka w treści"
```

---

## 5. Test strategy

Unit matrix (commonTest, host):

| Layer | Test | Reuses | New fake |
|---|---|---|---|
| data source (write id) | `ProblemReportRepositoryImplTest` asserts DTO fields + id | `FakePreferencesDataStore` | `FakeProblemReportDataSource` |
| repository (local cue) | same class: observe true/false, rev-scope, remote-fail resilience | — | — |
| use case (Submit) | `SubmitProblemReportUseCaseTest`: license/lang wiring, note cap + control-char strip | fake license/lang providers (see vote tests) | `FakeProblemReportRepository` |
| controller | `ProblemReportControllerTest`: null target no-op, flag-gate | `TestScope`, `runTest` | `FakeProblemReportRepository` |
| mailto | `FeedbackMailtoTest`: percent-encoding + URL shape | — | — |
| support email | `RemoteConfigSupportEmailProviderTest`: pure `resolveSupportEmail` blank→default, non-blank pass-through (live RC read is emulator-only) | — | — |
| identity | `InstallIdProviderTest` (moved) | `FakePreferencesDataStore` | — |

UI (commonUiTest): `ProblemReportSheetTest`, `ReportFlagButtonTest`, extend `ExplanationCardTest`,
`QuestionContentTest`, `QuestionPreviewContentTest`.

Emulator (mandatory, Koin bypassed by host tests — see [Run app to verify Koin]): app boots (new
bindings resolve, `singleOf(::ProblemReportRepositoryImpl)` has no defaulted args), flag override on →
flag on both surfaces, sheet submit fills flag and survives relaunch (persisted cue), offline submit
queues then flushes, `contact_us` on → composer opens / clipboard fallback. Firestore emulator +
`firestore.rules`: a conforming write succeeds; extra key / non-string field / >cap note / bad reason /
id-payload mismatch all rejected (S6).

Skip `iosSimulatorArm64Test` locally (FirebaseCore link failure); rely on
`:composeApp:compileKotlinIosSimulatorArm64` for native-API safety (BaseTimestamp, clipboard).

---

## 6. Rollout checklist

1. Merge PRs in order **A → B → C → D → E → F**, each targeting `development` (never `release`).
   Everything ships dark (`problem_reporting`, `contact_us` default false; flags absent in RC = defaults).
   **Phase 6 (PR #G, privacy)** can land any time in parallel; it gates the `problem_reporting` flip, not
   the code merges.
2. Deploy `firestore.rules` (adds `problemReports`) — do this **before** flipping `problem_reporting` so
   already-updated clients can write. Verify against the Firestore emulator first.
3. Remote Config: create booleans `problem_reporting` and `contact_us` (default false, keep off) **and**
   the string param `support_email` = `radoslaw.latka.dev@gmail.com` (matches the compiled default; can be
   rotated later without an app update — it seeds EP2's address).
4. Privacy gate: complete **Phase 6** (EN+PL policy clause published + Play Data-safety + App Store labels)
   **before** enabling `problem_reporting`. Blocks that flag flip only; does not block the build or
   `contact_us`.
5. `support_email` needs no code change to set — create/adjust the RC string param in the Firebase console.
   Enabling `contact_us` is no longer blocked on an address decision (default is already the real inbox).
6. Dark-launch verification (flags on for internal build only): flag on both surfaces, 3-tap submit,
   filled-flag cue persists, offline queue flush, re-report overwrite (no dupe), composer prefill +
   clipboard fallback, and a live `support_email` RC override picked up after `refresh`. Then enable per
   lane in production RC independently.
7. Kill ladder if needed: RC flag off = soft kill (UI gone); redeploy rules `allow ... : if false` =
   hard kill (stops open-session writes).

---

## 7. Risks & gotchas

| # | Risk / gotcha | Mitigation |
|---|---|---|
| 1 | GitLive offline write "fire-and-forget": `.set()` suspends until the SDK **enqueues** locally (returns while offline), not until server ack — same as votes. | Copy the vote path exactly: `try/catch` + `rethrowIfCancellation` + `Logger.e`; local cue is written first so the flag fills regardless. |
| 2 | `createdAt == request.time` needs the server sentinel; a local clock value is rejected. | `createdAt: BaseTimestamp = Timestamp.ServerTimestamp`; ensure `.set(dto)` keeps `encodeDefaults = true` (writes `createdAt` + constant `type`). Verify on emulator. |
| 3 | mailto subject/body URL-encoding — raw spaces/newlines/`(`/`)` break `openUri` and truncate the body. | `percentEncode` everything but unreserved; unit-tested. Newline → `%0A`. |
| 4 | ModalBottomSheet + IME: the note field is hidden by the keyboard. | Sheet content `Modifier.imePadding()` (functional-design §10); verify on device. |
| 5 | `'_'` rev placeholder + `':'` in id components — a colon in `code`/`lang`/`rev` would desync the id from the rules recomposition and break the CLI split. | Single `normalizeRev` owner puts `rev.ifBlank { "_" }` into the id, the DTO `rev`, and the local cue key; codes/langs/revs are colon-free in practice — document the invariant; the 5-part id (`reporterId:code:lang:rev:reason`) means the CLI splits on the first 4 colons defensively. |
| 6 | Two flag read points (`ProblemReporting` at both VMs, `ContactUs` at Settings VM) drift. | Both read `featureFlagService.isEnabled(...)` once at VM init (vote precedent); nullable handler → UI omits element when disabled. |
| 7 | `singleOf` ignores defaulted ctor params. | `ProblemReportRepositoryImpl` has **no** defaulted args (safe); controllers are built positionally inside VMs, never via DI. Emulator launch confirms. |
| 8 | `SettingsViewModel` gains `FeatureFlagService` + `SupportEmailProvider` params — `viewModelOf` must still resolve. | Both are existing `single`s; `viewModelOf(::SettingsViewModel)` auto-resolves. Confirm at launch. |
| 9 | Rev is per-(code, lang); a lang switch could show a stale cue. | Local key includes `lang`; `ObserveProblemReportUseCase` re-keys on `observeLanguage()` (vote precedent). |
| 10 | Host tests constructing `ProblemReportDto` reference `Timestamp.ServerTimestamp` (an object). | It's inert until serialization; repo/use-case tests use the fake source (no serialize) → safe on JVM host. |
| 11 | Stale RC cache serves the **compiled** `support_email` before/without a successful fetch (first launch, offline, or a not-yet-set RC param). | Acceptable — the compiled default **is** the real inbox (`radoslaw.latka.dev@gmail.com`), so the composer always opens to a valid address; `resolveSupportEmail` `.ifBlank`-falls back and the address is picked up live on the next fetch. |
| 12 | `SupportEmailProvider` shares `RemoteConfigFeatureFlagSource`'s `Firebase.remoteConfig` singleton — it does **not** run its own fetch. | By design (one RC config object per app); it reads activated values after the flag source's `fetchAndActivate`. Never add a second configurator/fetch. |

---

## New file tree (all additions)

```
composeApp/src/commonMain/kotlin/pl/egzaminppl/app/
  data/identity/InstallIdProvider.kt                          (moved from data/contentvote/VoterIdProvider.kt)
  data/source/dto/ProblemReportDto.kt
  data/source/ProblemReportDataSource.kt
  data/source/FirestoreProblemReportDataSource.kt
  data/repository/ProblemReportRepositoryImpl.kt
  domain/model/ProblemReason.kt
  domain/repository/ProblemReportRepository.kt
  domain/usecase/problemreport/SubmitProblemReportUseCase.kt
  domain/usecase/problemreport/ObserveProblemReportUseCase.kt
  ui/problemreport/ProblemReportController.kt
  ui/components/ProblemReportSheet.kt
  ui/components/explanation/ReportFlagButton.kt
  domain/service/SupportEmailProvider.kt
  data/service/RemoteConfigSupportEmailProvider.kt              (+ DEFAULT_SUPPORT_EMAIL / SUPPORT_EMAIL_KEY consts)
  ui/settings/FeedbackMailto.kt

composeApp/src/commonTest/kotlin/pl/egzaminppl/app/
  data/identity/InstallIdProviderTest.kt                      (moved from data/contentvote/VoterIdProviderTest.kt)
  data/source/FakeProblemReportDataSource.kt
  data/repository/FakeProblemReportRepository.kt
  data/repository/ProblemReportRepositoryImplTest.kt
  domain/usecase/problemreport/SubmitProblemReportUseCaseTest.kt
  ui/problemreport/ProblemReportControllerTest.kt
  data/service/RemoteConfigSupportEmailProviderTest.kt
  ui/settings/FeedbackMailtoTest.kt

composeApp/src/commonUiTest/kotlin/pl/egzaminppl/app/
  ui/components/ProblemReportSheetTest.kt
  ui/components/explanation/ReportFlagButtonTest.kt

scripts/report_review.py
```

Modified: `di/AppModule.kt`, `domain/featureflag/FeatureFlag.kt`, `domain/service/AppVersionProvider.kt`,
`androidMain/.../AndroidAppVersionProvider.kt`, `iosMain/.../IosAppVersionProvider.kt`,
`data/repository/ContentVoteRepositoryImpl.kt`, `data/source/FirestoreCollections.kt`,
`ui/question/QuestionViewModel.kt`, `ui/question/QuestionScreen.kt`,
`ui/questionpreview/QuestionPreviewViewModel.kt`, `ui/questionpreview/QuestionPreviewScreen.kt`,
`ui/components/explanation/ExplanationCard.kt`, `ui/settings/SettingsViewModel.kt`,
`ui/settings/SettingsScreen.kt`, `ui/UiTestTags.kt`, `firestore.rules`,
`composeResources/values/strings.xml`, `composeResources/values-pl/strings.xml`,
`docs/legal/privacy-en.md`, `docs/legal/privacy-pl.md` (Phase 6).

### Strings to add (functional-design §9, EN / PL)
`report_flag_content_description`, `report_flag_reported_content_description`, `report_sheet_title`,
`report_reason_wrong_answer`, `report_reason_bad_explanation`, `report_reason_question_error`,
`report_note_placeholder`, `report_note_counter` (`%1$d/500`), `report_privacy_hint`, `report_submit`,
`settings_item_feedback_title`, `settings_item_feedback_subtitle`,
`feedback_email_subject` (`EgzaminPPL feedback (%1$s, v%2$s)`),
`feedback_email_body` (`App version: %1$s (%2$s)\nLicense: %3$s\nLanguage: %4$s\n\nDescribe the problem or suggestion:\n`).
