# F1 — Give the App a Memory · Technical Architecture

*Feature slug: `app-memory` · Owner: Architecture · Status: design (input to implementation)*
*Input contract: `docs/features/app-memory/requirements.md` (accepted 2026-06-15, all OQ confirmed)*
*Parallel deliverable: `docs/features/app-memory/design.md` (designer — home grid visuals)*

---

## 0. Problem statement

The app forgets everything when a session ends. `PersistedSessionResultStore` keeps **one**
overwrite slot and there is **no per-question answer log**. F1 builds the durable memory substrate
the audit's whole §4 roadmap depends on, plus the single visible proof (per-subject progress on the
home grid), shipped **dark behind one feature flag**.

This document specifies HOW: the domain models (with all §6 forward-compat fields), the repository
read/write contracts, the DataStore-backed persistence + migration + retention, the
off-critical-path write mechanism, the feature-flag entry and gating points, the home-grid UI-state
model, DI wiring, package layout, per-license scoping, and a phased plan.

> **Stats are now language-agnostic (PR #33).** The answer-log read/bucket scope is **per-license
> only**; the `lang` segment was dropped from the bucket key (prefix bumped to `answer_log_v2_`) and
> from the repository read signatures. A question answered in PL or EN folds to one mastery entry, so
> stats survive a UI-language switch. `AnswerEvent` still records `lang` for analytics, but it is no
> longer part of the bucket key or read scope. Where this document still says "per-license+language",
> read it as "per-license"; the language-scoping passages below are retained only as the original
> design narrative.

Everything is **pure `commonMain`** (no `expect`/`actual` — the existing shared `DataStore<Preferences>`,
`Clock`, `LicenseProvider`, `LanguageProvider` cover every platform need). NFR-6 is satisfied by
construction.

### How F1 fits the existing substrate (verified against the code)

- Persistence is **DataStore `Preferences`** everywhere; user data is keyed `prefix_licenseId_lang_…`
  (`LearningSessionRepositoryImpl`, `FavouriteRepositoryImpl`). F1 mirrors this exactly (NFR-3).
- Serialization is `Json { ignoreUnknownKeys = true }` with `try/decode/Logger.e/return null` on
  failure (NFR-5 corruption pattern already established).
- Result writes happen at exactly two sites: `QuestionViewModel.finish()` and
  `ExamViewModel.validateAndFinish()`, both calling `resultStore.store(...)` inside a
  `viewModelScope.launch`.
- The grading transition that produces an `AnswerRecord` lives in the **session machines**
  (`LearningSessionMachine.selectOption` creates the record; `ExamSessionMachine` only marks
  `Selected` and grades at finish via `ValidateExamUseCase`). This is the natural hook for answer-event
  logging.
- The feature-flag system is mature: `FeatureFlag` enum → `CompositeFeatureFlagService` (remote +
  debug override, observed reactively) consumed via `isEnabled` (e.g. `GetExplanationUseCase` returns
  `null` when off) and `observe` (AppViewModel/DebugViewModel). F1 reuses this verbatim.
- The home grid is `CategoriesViewModel` → `CategoriesScreen` → `AppTile`, already a sealed
  `CategoriesUiState` with `Loading/Content/Error` combining several flows; it already has a shimmer
  skeleton. F1 extends this, it does not replace it.

---

## 1. Feature flag (Epic D / OQ-10) — first, because everything lands behind it

### 1.1 The one new enum entry

> **Decommissioned (PR #23):** the `Progress` / `progress_enabled` flag was removed once the feature
> shipped — it is now permanently on. The flag-gating described in this section is historical; the
> per-subject progress and answer-log behaviour now runs unconditionally.

Add a single entry to the existing `FeatureFlag` enum (`domain/featureflag/FeatureFlag.kt`):

```kotlin
enum class FeatureFlag(val key: String, val default: Boolean) {
    Explanation(key = "explanation_enabled", default = false),
    Progress(key = "progress_enabled", default = false), // F1 — "Give the app a memory"
}
```

**Name decision: `Progress`, remote key `progress_enabled`, default `false`.**
Rationale: the *flag* names the user-visible promise (per-subject progress), matching how
`Explanation` names a user-visible surface rather than its plumbing. The underlying packages are
named `appmemory` (§4) so the code is self-describing; the flag is the public/remote handle. Single
flag, gates all of F1 (OQ-10), debug-overridable and reactively observed for free — it flows through
`CompositeFeatureFlagService`, `DebugScreen`/`DebugViewModel` (which iterate `FeatureFlag.entries`),
and remote config with **zero** changes to those files.

### 1.2 Gating points (gate the whole feature — data AND UI, OQ-10)

The flag is read/observed at exactly three seams. When off, behaviour is **byte-for-byte today's**.

| Seam | Mechanism | When flag OFF | When flag ON |
|---|---|---|---|
| **Answer-event write** | `LogAnswerEventUseCase` checks `isEnabled(Progress)` first | No-op: no event written, no store touched | Appends event off critical path |
| **Home read** | `CategoriesViewModel` combines `featureFlagService.observe(Progress)`; `ObserveSubjectProgressUseCase` short-circuits to "no progress" | Tiles render exactly as today (no progress overlay) | Tiles render coverage/mastery |

> **Result persistence is NOT gated.** The multi-result session store always runs — the results screen
> needs it regardless of the flag (it is pre-F1 functionality). Replacing the single slot with a
> keyed, bounded list is invisible to the user when the flag is off (no UI lists history yet); the
> only behavioural effect is the MistakeReview-parent bug fix. This is an accepted, minor relaxation
> of "flag off = byte-for-byte today".

Key consequences:
- **Reactive (D1.4):** the home read combines `observe(Progress)`. A debug ForceOn/Off or a remote
  refresh flips the grid with no restart, exactly like the app already observes flags.
- **No hidden data trail (OQ-10):** logging is gated at the *use-case* boundary, so OFF writes
  nothing. Turning the flag ON later simply starts from an empty history (D1.5) — no migration, no
  crash.
- **The write seam uses `isEnabled` (synchronous one-shot)**, not `observe`: a write is a point-in-time
  decision on the answer-grading path and must not suspend on a flow. The read seam uses `observe`
  because the grid must react.

---

## 2. Domain models

All new models live in `domain/appmemory/`. They are pure Kotlin, `@Serializable`, no framework deps.

### 2.1 Answer event (Epic A, §6 per-answer table)

```kotlin
package pl.egzaminppl.app.domain.appmemory

@Serializable
data class AnswerEvent(
    val questionId: String,        // coverage / mastery / mistakes pool / weighted review
    val questionCode: String,      // placard code — search & cross-license shared-code analytics (OQ-9)
    val categoryId: String,        // per-subject coverage/mastery, readiness, category hub
    val licenseId: String,         // per-license scoping (NFR-3)
    val lang: String,              // bank size differs per language; keyed like all user data
    val mode: LearningMode,        // readiness ~ Exam; weighting; review-churn exclusion (already serializable)
    val isCorrect: Boolean,        // mastery, scores, mistakes
    val selectedOptionId: String,  // reconstruct attempt; future per-distractor analytics
    val correctOptionId: String,   // same
    val timestampMillis: Long,     // most-recent-attempt mastery, "old" ordering, streaks, trend, pruning
)
```

Every §6 per-event fact is present. `mode` reuses the existing serializable `LearningMode`. The model
is intentionally flat (cheap to serialize as a JSON array element). Nothing here is derived — mastery
is **computed on read** (OQ-3), never stored.

### 2.2 Augmented finished-session result (Epic B, §6 per-result table)

`SessionResult` today has **no** timestamp, **no** license/lang, and **no** stable id (the id is
minted by the store and is ephemeral). F1 must add these. Two viable approaches:

- **(A) Add fields to `SessionResult` directly** — simplest, but `SessionResult` is also the
  transient payload the results screen reads, and `toSessionResult` builds it without scope context.
- **(B) Introduce a `SessionRecord` envelope** wrapping `SessionResult` + the new metadata.

**Decision: (A) augment `SessionResult` with nullable, defaulted fields.** Rationale: backward- and
forward-compatible deserialization is free (`ignoreUnknownKeys` + defaults — exactly how
`LearningSession.updatedAt` was added), it keeps one result type across the app (no envelope/unwrap
churn in `SessionResultsViewModel`, `MistakeReviewQuestionStrategy`), and the new fields are genuinely
*about the result*. The store assigns the id and stamps it into the payload so it persists (§6: "id
must persist, not be ephemeral").

```kotlin
@Serializable
data class SessionResult(
    val mode: LearningMode,
    val categoryId: String,
    val totalQuestions: Int,
    val correctCount: Int,
    val questionResults: List<QuestionResult>,
    val examDuration: ExamDuration? = null,
    val examPassCount: Int? = null,
    // ── F1 additions (all defaulted ⇒ legacy payloads still deserialize) ──
    val id: String? = null,            // stable id, persisted into the payload (§6)
    val finishedAtMillis: Long? = null,// exam history list/trend, "last exam score", recency
    val licenseId: String? = null,     // scope history per license
    val lang: String? = null,          // correct grouping per language
)
```

The two highest-leverage adds (§6 closing note) — **timestamp** and **license+language** — are now on
every answer event *and* every result. `mode` and the score inputs were already present.

### 2.3 Read-surface result types (the generously-designed read API, §8 "Should")

Computed on read, never stored. Shaped so the deferred category-hub / readiness / weighted-review can
consume them directly without a new read pass.

```kotlin
package pl.egzaminppl.app.domain.appmemory

/** Per-subject coverage & mastery for the active license (language-agnostic, PR #33). */
data class SubjectProgress(
    val categoryId: String,
    val seenCount: Int,        // distinct questions answered at least once (coverage, A2.1)
    val masteredCount: Int,    // distinct questions correct on most-recent attempt (A2.2/3)
    // totalCount is NOT stored here — see §6 note below
)

/**
 * Per-question mastery snapshot derived from the latest event per question.
 * The read primitive the NEXT project (cross-session mistakes pool + weighted Quick review)
 * consumes directly. Retained by product decision (2026-06-15); not used by F1 itself.
 */
data class QuestionMastery(
    val questionId: String,
    val isMasteredOnLatestAttempt: Boolean,
    val lastAttemptMillis: Long,   // "old" ordering for weighted review
    val lastMode: LearningMode,    // exclude review-churn or weight by source later
)
```

> **Coverage denominator.** `SubjectProgress` deliberately omits the subject's *total* question count.
> Coverage % = `seenCount / category.questionCount(Language.PL)` — the Polish superset count, used
> regardless of the active UI language (language-agnostic stats, PR #33): a question answered in either
> language counts once. That denominator is already owned by `Category.questionCount`, available in
> `CategoriesViewModel`. Keeping the bank total out of
> the log read avoids coupling the answer log to catalog content and keeps the read a pure aggregation
> over events. The ViewModel combines the two.

---

## 3. Repository interfaces (domain)

Two **segregated** interfaces (ISP): an append/read answer log, and a session-history store. They are
separate because their lifecycles, retention policies, and consumers differ. The session-history
interface *replaces* the role of today's `SessionResultStore` for history but the **`SessionResultStore`
interface is kept** so the results screen and `MistakeReviewQuestionStrategy` are untouched (see §5.3).

### 3.1 `AnswerLogRepository` (Epic A — write + generous read)

```kotlin
package pl.egzaminppl.app.domain.appmemory

interface AnswerLogRepository {

    // ── WRITE (A1) ──
    /** Append one graded answer event. Append-only (OQ-2); never overwrites a prior event. */
    suspend fun append(event: AnswerEvent)

    // ── READ (A2), scoped to active license by the caller's arg (language-agnostic, PR #33) ──

    /** Per-subject coverage & mastery for every category in licenseId. Drives Epic C. */
    fun observeSubjectProgress(licenseId: String): Flow<List<SubjectProgress>>

    /**
     * Per-question mastery for one subject — the read primitive the NEXT project (cross-session
     * mistakes pool + weighted Quick review) consumes. Not used by F1; retained by explicit product
     * decision (2026-06-15) because it is exactly what that project needs and adding it later would
     * otherwise be an interface change. Cheap: it returns the per-question intermediate that
     * `observeSubjectProgress` already folds over (`Mastery.kt`).
     */
    suspend fun questionMastery(licenseId: String, categoryId: String): List<QuestionMastery>
}
```

- `observeSubjectProgress` is a **Flow** so the grid updates without restart (C1.3) and reacts to a
  license switch when the caller re-subscribes with a new arg. Reads are language-agnostic (PR #33), so
  a UI-language switch does not re-scope them — stats survive the switch.
- `questionMastery` is retained for the **next project** (cross-session mistakes pool + weighted Quick
  review) — it is the exact per-question primitive those features consume, so keeping it now means
  that project needs no interface change. It is **not** used by F1.
- **Trimmed to hold F1 to minimal scope (reviewer decision 2026-06-15):** the raw-`events` analytics
  escape hatch was removed. Features further out (analytics/trends) can re-add it later as a purely
  additive interface change with **zero data migration** — every field they would read is already
  stored (§6).
- Scoping is **explicit in the signature** (`licenseId`) rather than read from a provider inside
  the repo, so the read is pure and trivially testable, and the caller controls scope on a license
  switch (NFR-3, A2.5). Language is not a read scope (PR #33).

### 3.2 Session-result store — keep the existing interface, replace the implementation

**Decision (reviewer 2026-06-15): no separate `SessionHistoryRepository` interface.** The app is
unreleased, so there is no migration and no need for a dual store. The new multi-result store simply
**implements the existing `SessionResultStore`** (`store`/`get`), so the four consumers
(`QuestionViewModel`, `ExamViewModel`, `SessionResultsViewModel`, `MistakeReviewQuestionStrategy`)
are unchanged. `PersistedSessionResultStore` (single slot) is **deleted**.

```kotlin
package pl.egzaminppl.app.domain.session

interface SessionResultStore {           // unchanged
    suspend fun store(result: SessionResult): String  // appends to a keyed, bounded list; returns stable id
    suspend fun get(id: String): SessionResult?       // by id; null if absent/corrupt, never throws
}
```

The new impl keeps a per-`(license, lang)` list keyed by stable id (so results are never overwritten —
this is what fixes the MistakeReview-destroys-parent bug) and stamps `id`/`finishedAt`/`license`/`lang`
into the payload on `store`. The exam-history list read (`observe(license, lang)`) is **not** added now;
it serves the deferred exam-history view and can be added to the interface later additively with zero
data migration.

---

## 4. Package structure

```
domain/appmemory/
  AnswerEvent.kt                 # §2.1
  SubjectProgress.kt             # §2.3
  QuestionMastery.kt             # §2.3
  AnswerLogRepository.kt         # §3.1
  SessionHistoryRepository.kt    # §3.2
  Mastery.kt                     # pure fn: List<AnswerEvent> -> List<QuestionMastery>/SubjectProgress (OQ-3 logic)

domain/usecase/appmemory/
  LogAnswerEventUseCase.kt       # flag-gated append (A1, Epic D)
  RecordSessionResultUseCase.kt  # flag-gated history write + legacy slot (B1, Epic D)
  ObserveSubjectProgressUseCase.kt # flag-gated read for the home grid (A2 + C1 + Epic D)

data/appmemory/
  AnswerLogStoreImpl.kt          # DataStore-backed AnswerLogRepository (§5.1)
  SessionHistoryStoreImpl.kt     # DataStore-backed SessionHistoryRepository (§5.2)
  AnswerLogRetentionPolicy.kt    # pruning (B3 / §5.4)

domain/model/SessionResult.kt    # MODIFIED — add id/finishedAt/license/lang (§2.2)
domain/featureflag/FeatureFlag.kt# MODIFIED — add Progress (§1.1)
ui/categories/CategoriesViewModel.kt # MODIFIED — progress state (§6)
```

Mastery math lives in a **pure top-level function file** (`Mastery.kt`) so the OQ-3 definition
("correct on most recent attempt") is unit-tested in isolation and the definition can evolve (e.g.
"correct twice") without touching storage — the whole point of computing on read.

---

## 5. Data layer

### 5.1 Answer-event log storage shape

Mirror `LearningSessionRepositoryImpl`/`FavouriteRepositoryImpl`: a single `DataStore<Preferences>`,
`Json { ignoreUnknownKeys = true }`, **one key per `(licenseId, categoryId)` bucket** whose
value is a JSON-encoded `List<AnswerEvent>`. Stats are language-agnostic (PR #33): the bucket key has
**no `lang` segment** and the key prefix was bumped to `answer_log_v2_`, so a question answered in PL or
EN folds to one mastery entry.

```
key:   answer_log_v2_{licenseId}_{categoryId}   (stringPreferencesKey)
value: Json.encodeToString(List<AnswerEvent>)   (newest appended at the tail)
```

Decision — **bucket per subject, not one giant blob and not one key per event:**
- Per-event keys would explode the preference map and make the home read (which iterates all keys)
  slow and fragile.
- A single all-history blob would rewrite the entire log on every append (O(total) writes — jank risk
  + lock contention).
- **Per-`(license, category)` bucket** is the sweet spot: an append rewrites only that subject's
  list (bounded by retention, §5.4); the home read decodes ≤9 buckets for the active license; a license
  switch reads a disjoint key set (NFR-3, A2.5 satisfied for free). A UI-language switch does not change
  the key set (PR #33).

**Append** (A1.3 append-only): read the bucket list, `+ event`, apply retention (§5.4), write back —
inside one `dataStore.edit {}` transaction so concurrent appends from different subjects don't clobber
each other (each touches a different key).

**Read** (`observeSubjectProgress`): `dataStore.data.map { prefs -> … }` — for the active license, find
all keys with prefix `answer_log_v2_{licenseId}_`, decode each (skip+log on failure, NFR-5),
fold each subject's events through `Mastery.kt` into `SubjectProgress`. This is a Flow, so it emits on
every write → grid updates live (C1.3).

**Corruption safety (NFR-5, A2 "without error"):** a bad bucket is caught per-key, logged via
`Logger.e`, and skipped; other subjects still compute. A bad single event inside a decodable list is
impossible to isolate (the list either decodes or not), so the unit of corruption is the bucket — an
acceptable, bounded blast radius, and consistent with today's per-key behaviour.

### 5.2 Session-result storage shape (the new `SessionResultStore` impl)

Same engine as the answer log. Replace the single overwrite slot with a **per-`(license, lang)` list
of results**, capped by retention (§5.4). This impl always runs (result persistence is pre-F1
functionality the results screen needs); it is **not** flag-gated.

```
key:   session_history_{licenseId}_{lang}      (stringPreferencesKey)
value: Json.encodeToString(List<SessionResult>) (each carries its own id)
```

`store(result)`: mint a `Uuid`, copy it into the result along with `finishedAtMillis = clock.now`,
the active `licenseId`/`lang` (resolved from the providers), prepend to the scope's list, prune
(§5.4), write; return the id. `get(id)` scans `session_history_*` buckets (few and small — no index
needed) and returns the match or null. Corruption is skipped+logged per bucket, as for the answer log.

> Keying by stable id is what fixes the MistakeReview-destroys-parent bug: the parent and the review
> result coexist in the list and resolve independently, instead of sharing one slot.

### 5.3 Results-screen continuity & the stale nav rationale

The four consumers keep using `SessionResultStore.store/get` unchanged — they don't know the impl
swapped from one slot to a keyed list. No read-through, no second store, no `RecordSessionResultUseCase`.

One coupled comment must be fixed: `AppNavGraph` pops the results screen on "Review mistakes"/"Retake"
with the rationale *"its single-slot result is consumed by the next session, so there is nothing valid
to come back to."* That is no longer true once results are preserved. **Keep the forward-only nav**
(the preserved result exists for the future history view, not for back-navigation) and **update the
comment** to say the pop is a deliberate flow choice. No behaviour change.

### 5.4 Retention / pruning (B3, OQ-5, NFR-7)

Two bounds, both enforced **on write** (the only time the data grows), in `AnswerLogRetentionPolicy`:

**Answer-event log** (per `(license, category)` bucket):
1. **Always preserve the most-recent event per `questionId`** (the mastery/coverage invariant — OQ-3/
   OQ-5: "pruning never corrupts current mastery/coverage"). This is the *floor*.
2. On top of that floor, keep a bounded recent window of older attempts for trend/streak/"old"
   ordering — cap the bucket at **N events (default 500)** by dropping oldest *non-floor* events first.
   Concretely: partition into (a) latest-per-question = keep, (b) the rest sorted newest-first; keep
   from (b) until the bucket reaches N. The floor is never dropped even if it alone exceeds N (mastery
   correctness wins over the size bound).

Because the floor = one event per distinct question, and the bank is ~1775 questions across 9
subjects (≈200/subject), the floor is naturally small; coverage and mastery stay **exact** forever
(A2.1–A2.4), while churn from repeated drilling is bounded.

**Session history** (per `(license, lang)` bucket): keep the most recent **M results (default 25)**,
newest-first; drop the oldest beyond M. Sufficient for "recent exam average" + a short trend
(OQ-5). MistakeReview results are excluded from this list (OQ-7) so they don't consume the cap.

The numbers (N=500, M=25) are defined as named constants and are the firm-bound-exists requirement
(B3); they can be tuned without interface change.

### 5.5 Write path — OFF the answer-grading critical path (NFR-2)

This is the central NFR. Two write sites, two mechanisms, both fire-and-forget on a scope so the UI
state update never awaits the persist.

**Answer-event logging (A1).** The grading decision is made in the session machine
(`LearningSessionMachine` builds the `AnswerRecord`; the exam grades at finish). To keep the machines
pure (they are deliberately side-effect-free, returning `TransitionResult(state, effects)`), logging
is driven from the **ViewModel** the same way `SaveSession`/`DeleteSession`/`Finish` effects already
are — *not* by making the machine do IO.

- **Learning / ShortLearning / MistakeReview:** in `QuestionViewModel`, on the transition that produces
  a fresh `QuestionState.Answered` (detectable exactly as the existing `observeIncorrectAnswers`
  collector detects new answers — a state→answered edge), call
  `viewModelScope.launch { logAnswerEventUseCase(record, …) }`. The UI state is emitted *synchronously*
  by `dispatch()` **before** this launch; the write is fully detached. (Testable per NFR-2: UI state
  updates without awaiting the write.)
- **Exam:** answers are graded only at finish (A1.5). In `validateAndFinish()`, after
  `validateExamUseCase.validate(...)`, launch one `logAnswerEventUseCase` call per graded question
  (batched into a single `append` per subject — exam = one subject), detached from the
  `_uiState.value = Finished(...)` emission.

**Ordering & failure semantics:**
- Appends within one ViewModel are launched on `viewModelScope` and the per-subject bucket write is a
  single `dataStore.edit {}` (serialized by DataStore's own writer). For Learning, answers are
  inherently sequential (you answer current before advancing), so launch order = causal order. We do
  not need a dedicated single-thread actor for F1's volumes; if a strict global ordering is ever
  required, an internal `Channel`-backed writer can be added behind `AnswerLogRepository` without
  changing the interface.
- **A write failure must never surface to the user (NFR-2/NFR-5):** `LogAnswerEventUseCase` wraps the
  append in `try/catch`, `rethrowIfCancellation()`, `Logger.e`, and swallows. A lost event degrades
  progress slightly; it never crashes or blocks grading. This matches the app's
  deserialize-and-log-on-failure ethos.
- **Cold-start orphan guard (A1.6):** `LogAnswerEventUseCase` resolves `licenseId` via
  `licenseProvider.current()` (nullable). If `current()` is null it **returns without writing** — the
  log never contains a null-license event. (Unreachable from a live session, which requires a license,
  but enforced defensively.) It still resolves `lang` via `languageProvider` to record it on the
  `AnswerEvent` for analytics, but `lang` is **not** part of the bucket key (PR #33).

**Result write (B1).** `finish()`/`validateAndFinish()` keep calling `resultStore.store(result)` in
`viewModelScope.launch`, awaiting the id to navigate to the results screen — exactly as today. Only
the bound *implementation* changes (single slot → keyed bounded list), and it stamps
`id`/`finishedAt`/`license`/`lang` internally. No use-case wrapper, no VM change at the write site.

---

## 6. UI-state model for the home grid (Epic C)

The designer owns visuals (`design.md`); the architecture owns the **state set** the ViewModel
exposes. The model below is a **superset** of any sensible progress tile (loading, empty, populated,
error) and explicitly carries the **flag-off baseline**.

`CategoriesViewModel` adds the progress flow to its existing `combine`:

```kotlin
// new inputs added to the existing combine(...)
featureFlagService.observe(FeatureFlag.Progress),          // reactive flag (D1.4)
observeSubjectProgressUseCase(),                            // Flow<SubjectProgressMap> (already flag-gated)
```

`ObserveSubjectProgressUseCase` internally `combine`s the flag with `AnswerLogRepository
.observeSubjectProgress(activeLicense)` and emits the appropriate state (language-agnostic, PR #33).
Per-tile state:

```kotlin
sealed interface TileProgress {
    /** Flag off → today's look exactly; tiles render with no progress affordance (D1.1). */
    data object Hidden : TileProgress
    /** Flag on, progress not yet read → skeleton/placeholder for the progress region (C1.5, NFR-1). */
    data object Loading : TileProgress
    /** Flag on, never practised this subject in this license → zero/empty, not a scary "0%" (C1.2). */
    data object Empty : TileProgress
    /** Flag on, has data → mastery-led signal; coverage available for a two-part visual (OQ-4). */
    data class Populated(
        val masteredCount: Int,
        val seenCount: Int,
        val totalCount: Int,        // = Category.questionCount(Language.PL) — Polish superset (PR #33)
    ) : TileProgress
    /** Flag on but the log read failed/corrupt → fall back to today's look, never block the grid (NFR-5). */
    data object Unavailable : TileProgress
}
```

State derivation (the superset the designer can rely on):

| Condition | Tile state |
|---|---|
| `Progress` flag OFF | `Hidden` (grid identical to today, D1.1) |
| flag ON, progress flow still emitting initial/loading | `Loading` (existing shimmer pattern, C1.5) |
| flag ON, subject has `seenCount == 0` | `Empty` (zero state, C1.2) |
| flag ON, subject has events | `Populated(mastered, seen, total)` (mastery-led, OQ-4) |
| flag ON, log read threw/corrupt | `Unavailable` (graceful fallback, NFR-5) |

Integration into the existing sealed `CategoriesUiState`:
- Carry a `Map<categoryId, TileProgress>` on `CategoriesUiState.Content` (defaulting to all-`Hidden`
  so existing call sites and the flag-off path compile and render unchanged).
- `CategoriesUiState.Loading` keeps its current shape; per-tile progress `Loading` only applies once
  categories themselves are `Content` (the grid skeleton already covers the whole-screen load — NFR-1
  "skeleton first, then data" is preserved: the grid never blocks on the history read because the
  progress flow has its own `Loading`/`Hidden` initial value and the `combine` does not wait on it).
- License switch (C1.4): `observeSubjectProgressUseCase` re-derives from the license provider's
  flow, so a switch re-subscribes the log read to the new `license` key set and tiles reflect
  the new scope; switching back restores (NFR-3, A2.5). A UI-language switch does **not** re-scope the
  read — stats are language-agnostic (PR #33) and survive the switch.

**`AppTile` change:** add an optional `progress: TileProgress = TileProgress.Hidden` slot the tile
renders below/around its footer. `Hidden` renders nothing → existing tiles unchanged. This is the only
`AppTile` API addition; the visual treatment of `Populated`/`Empty`/`Loading` is the designer's
(`design.md`).

---

## 7. DI wiring (`di/AppModule.kt`)

Pure additions (no removals — the legacy `PersistedSessionResultStore` stays for the off-path and
read-through, §5.3):

```kotlin
// Repositories
singleOf(::AnswerLogStoreImpl) bind AnswerLogRepository::class
// Replace the single slot: the new keyed, bounded impl IS the SessionResultStore.
// Delete the PersistedSessionResultStore binding (and the class).
singleOf(::SessionHistoryStoreImpl) bind SessionResultStore::class

// Use cases (factories, like the rest)
factoryOf(::LogAnswerEventUseCase)        // Phase A (already wired)
factoryOf(::ObserveSubjectProgressUseCase) // Phase C
```

ViewModel wiring:
- `QuestionViewModel` / `ExamViewModel` — **no change** in Phase B; they already have
  `LogAnswerEventUseCase` (Phase A) and keep calling the unchanged `resultStore.store`/`get`.
- `CategoriesViewModel` ← `ObserveSubjectProgressUseCase` (Phase C).

`SessionHistoryStoreImpl` deps: the shared `DataStore<Preferences>`, `LicenseProvider`,
`LanguageProvider`, `Clock`, and a retention bound.
`ObserveSubjectProgressUseCase` deps: `AnswerLogRepository`, `FeatureFlagService`, `LicenseProvider`,
`CategoryRepository` (for the `totalCount` denominator). It no longer scopes by language (PR #33);
`LanguageProvider` is not a read-scoping dependency.

All stores reuse the **single shared `DataStore<Preferences>`** already provided by the platform
modules (`PlatformModule.android.kt`/`PlatformModule.ios.kt`) — no new DataStore, no `expect`/`actual`.

---

## 8. Migration — none (reviewer decision 2026-06-15)

The app is **unreleased**, so there is no persisted user data to preserve and **no migration**. The
legacy `PersistedSessionResultStore` and its keys (`last_session_result_id`, `last_session_result`)
are **deleted** outright rather than kept as a read-through. New stores use new key namespaces
(`answer_log_*`, `session_history_*`), and the new `SessionResult` fields are defaulted, so nothing
needs transforming. (The defaulted-field approach still means a *future* additive field change needs
no migration either — but that is a property of the design, not a step F1 performs.)

---

## 9. Per-license scoping (NFR-3) — switch behaviour

> **Language-agnostic stats (PR #33).** The answer-log read/bucket scope is **per-license only**. The
> `lang` segment was dropped from the bucket key (prefix bumped to `answer_log_v2_`) and from the read
> signatures, so a question answered in PL or EN folds to one mastery entry and stats survive a
> UI-language switch. (Session history below is unchanged and still keyed `(license, lang)`.)

- **Storage:** every answer-log key embeds `{licenseId}` (no `lang`). Different licenses are physically
  disjoint key sets, so they never interfere (A2.5).
- **Write:** `LogAnswerEventUseCase`/`RecordSessionResultUseCase` resolve the *active* license at write
  time from `LicenseProvider` (same call the session machinery already uses in `SaveSessionUseCase`).
  `LogAnswerEventUseCase` still resolves `lang` to record it on the `AnswerEvent` for analytics, but
  `lang` is not part of the answer-log bucket key (PR #33).
- **Read:** `ObserveSubjectProgressUseCase` combines the license provider flow, so a license change
  re-derives the active `license` and re-subscribes the log read to that license's keys. The grid
  re-renders with the new license's progress; switching back surfaces the original data intact
  (C1.4, A2.5). A UI-language switch does **not** re-scope the read.
- **Language switch nuance:** stats are language-agnostic — a question answered in either language
  counts once and progress survives a UI-language switch (PR #33). Favourites/sessions remain keyed by
  language; only the answer-log read scope dropped it.

---

## 10. SOLID / quality check

- **Dependency arrows inward:** `domain/appmemory` is pure Kotlin (models + interfaces + a pure mastery
  function). `data/appmemory` implements the interfaces; ViewModels depend on use cases → domain
  interfaces. No data type leaks into UI; no framework in domain. ✔
- **SRP:** `AnswerLogRepository` (events) and `SessionHistoryRepository` (results) are separate;
  retention is its own policy object; mastery math is its own pure function; each use case is one
  operation. ✔
- **OCP:** the **next project** (cross-session mistakes pool + weighted Quick review) consumes the
  already-defined `questionMastery` read with **no modification** of F1 code and no migration. Features
  further out (exam-history list, analytics/trends, readiness) will add their own reads (`events`,
  history `observe`, …) later — a purely additive interface change with zero data migration, since
  every field they need is already stored (§6 contract). Adding a logged mode = no code change (mode is
  a stored field). ✔
- **LSP:** fakes (§11) are fully substitutable; no `NotImplementedError`. ✔
- **ISP:** two focused repository interfaces, not one god store. ✔
- **DIP:** ViewModels depend on use cases and domain interfaces; Koin injects the data impls. ✔
- **Anti-patterns avoided:** machines stay pure (no IO), logging is a ViewModel effect; no business
  logic in composables; the flag gate is one decision per seam; adding F1 touches new files plus the
  two write ViewModels + `CategoriesViewModel` + enum — each change is additive and bounded.

---

## 11. Phased implementation plan

Ordered so the **flag is wired first** (everything lands dark, D1.1), then write→read→store→UI, each
phase independently testable on `commonTest` (JVM via `:composeApp:testAndroidHostTest` — see the
host-runner note; `iosSimulatorArm64Test` fails to link FirebaseCore here). Tests use
`kotlin.test.*` + `runTest`, one-line fake init, GIVEN/WHEN/THEN per the kmp-unit-testing skill.

### Phase 0 — Feature flag (Epic D) · *do first, unblocks dark merge*
- **Create/modify:** `FeatureFlag.kt` (+`Progress`).
- **Interfaces:** none new — reuses `FeatureFlagService`/override/remote/Debug verbatim.
- **Acceptance:** `Progress` appears in `DebugScreen` with Default/ForceOn/ForceOff; resolves `false`
  by default; observable.
- **Tests:** extend `CompositeFeatureFlagServiceTest` to cover `Progress` (default off, override
  on/off, remote precedence) — mostly free since the service iterates `entries`.
- **Depends on:** nothing.

### Phase A — Answer-event log: write + read (Epic A) · *flag-gated from line one*
- **Create:** `AnswerEvent`, `SubjectProgress`, `QuestionMastery`, `Mastery.kt`,
  `AnswerLogRepository`, `AnswerLogStoreImpl`, `AnswerLogRetentionPolicy`, `LogAnswerEventUseCase`,
  `ObserveSubjectProgressUseCase`.
- **Modify:** `QuestionViewModel.finish`/answer edge + `ExamViewModel.validateAndFinish` to call
  `LogAnswerEventUseCase` (detached). DI: bind repo + use cases.
- **Acceptance:** A1.1–A1.6 (event written with all §6 fields, survives process death, append-only,
  non-logged Favourites writes nothing, exam logs at finish, no null-license event); A2.1–A2.5
  (coverage = distinct seen, mastery = most-recent-attempt-wins, empty = zero, per-license scoping);
  NFR-2 (UI state updates without awaiting write).
- **Tests:** `MasteryTest` (pure: most-recent wins, wrong-then-right, empty); `AnswerLogStoreImplTest`
  (over `FakePreferencesDataStore`: append-only, per-scope isolation, corrupt-bucket skipped,
  retention preserves latest-per-question; `questionMastery` returns per-question latest state —
  retained for the next project, so it is implemented and tested even though F1 does not call it);
  `LogAnswerEventUseCaseTest` (flag off = no-op, null license = no write, Favourites excluded);
  `ObserveSubjectProgressUseCaseTest` (flag off = no progress; reacts to switch). New fakes:
  `FakeAnswerLogRepository`, extend `FakeFeatureFlagService` to allow per-flag override.
- **Depends on:** Phase 0.

### Phase B — Multi-result session store + retention (Epic B) · *no migration, not flag-gated*
- **Create:** `SessionHistoryStoreImpl` implementing the existing `SessionResultStore` (keyed,
  bounded list; stamps `id`/`finishedAt`/`license`/`lang`); a history retention bound (sibling of
  `AnswerLogRetentionPolicy`, recent-M per scope).
- **Modify:** `SessionResult` (+`id`/`finishedAtMillis`/`licenseId`/`lang`, all defaulted);
  `di/AppModule.kt` (bind `SessionHistoryStoreImpl` to `SessionResultStore`, delete the
  `PersistedSessionResultStore` binding); fix the stale single-slot comment in `AppNavGraph` (keep
  the pop). **Delete `PersistedSessionResultStore`.** `QuestionViewModel`/`ExamViewModel` write site
  unchanged.
- **Acceptance:** B1.1 (multiple results retained, none overwritten), B1.2 (results screen unchanged),
  B1.3 (**parent result preserved after MistakeReview**), B1.4 (corrupt entry skipped); B3
  (bounded — oldest pruned beyond M per scope).
- **Tests:** `SessionHistoryStoreImplTest` (store/get by id, retain M, corrupt-skip, per-scope
  isolation, metadata stamped); a regression test for B1.3 (store parent → store review →
  parent still `get`-able). Update `FakeSessionResultStore` only if needed (already a keyed map).
- **Depends on:** nothing new (the `SessionResultStore` interface already exists). Independent of
  Phases A and C.

### Phase C — Home-tile progress (Epic C) · *the only visible slice*
- **Create:** `TileProgress` (in `ui/categories`); progress region in `AppTile`.
- **Modify:** `CategoriesViewModel` (combine flag + `ObserveSubjectProgressUseCase`, derive
  `Map<categoryId, TileProgress>` on `Content`, denominator from `Category.questionCount`);
  `CategoriesScreen`/`CategoryItem` pass `progress` to `AppTile`. DI: VM deps.
- **Acceptance:** C1.1–C1.5 (practised vs never-practised distinct & correct; zero state not scary;
  updates without restart; license switch re-scopes; skeleton-first never blocks). D1.1/D1.4 (off =
  today's look; reacts to toggle live).
- **Tests:** `CategoriesViewModelTest` — flag off → all tiles `Hidden`; flag on + events →
  `Populated` with correct mastered/seen/total; never-practised → `Empty`; log read failure →
  `Unavailable`; license switch flips scope; toggle flips live. (Uses `FakeAnswerLogRepository`,
  `FakeFeatureFlagService`, `FakeLicenseProvider`, `FakeLanguageProvider`.)
- **Depends on:** Phase A (read), Phase 0 (flag). Independent of Phase B.

> **Landing dark:** Phases 0→A→B→C can all merge with `Progress` defaulting `false` ⇒ zero user-visible
> change until remote/ debug flip. C is the proof; A/B are the foundation reused by every deferred §5
> feature with no migration (§6 contract upheld).

---

## 12. Risks & trade-offs

- **Per-subject bucket vs. event-stream store.** Chosen bucket-per-`(license,category)` over a
  per-event key map (read/scale) and a single blob (write amplification/jank). Trade-off: corruption
  granularity is the bucket, not the event — bounded and consistent with today's per-key model.
- **Augmenting `SessionResult` vs. an envelope.** Chosen in-place defaulted fields for zero-migration
  deserialization and no churn in result consumers. Trade-off: `SessionResult` now carries scope
  metadata that is `null` for the transient just-finished case until `record` stamps it — acceptable,
  and the store owns stamping.
- **Mastery computed on read (OQ-3).** Trade-off: recompute cost on every grid read. Mitigated by the
  bucket shape (≤9 buckets, retention-bounded) and the floor-preserving prune keeping the per-question
  latest set small. If NFR-1 ever bites on very large logs, an in-memory derived cache can sit behind
  `AnswerLogRepository` with no interface change (OCP).
- **Write ordering.** F1 relies on launch-order = causal-order (true for sequential Learning answers)
  plus DataStore's serialized writer. A strict global actor is deferred until a feature needs it; the
  interface already hides this.
- **Flag-on starts empty (D1.5, OQ-10).** Correct for a dark launch but means the very first enable
  shows no back-history. Documented expected behaviour, not a bug.
