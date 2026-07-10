# P1 — Mistake-driven study · Technical Architecture

*Feature slug: `mistake-driven-study` · Owner: Architecture · Status: design (input to implementation)*
*Input contract: `docs/features/mistake-driven-study/requirements.md` (OQ-1 DECIDED 2026-06-15; scope = both epics, Epic W first)*
*Builds on: F1 "app-memory" — `docs/features/app-memory/architecture.md` (DONE on this branch)*

---

## 0. Problem statement

F1 gave the app a durable, per-question answer log and shipped the read primitive both halves of this
feature consume — `AnswerLogRepository.questionMastery(licenseId, categoryId)`
(`domain/appmemory/AnswerLogRepository.kt:26`). Per PR #33 this primitive is now **language-agnostic**:
a question answered in either language folds to one mastery entry (latest attempt wins), scoped per
license only. This feature is where we finally consume it, in two independently-shippable epics:

- **Epic W — Weighted Quick review (FIRST).** `ShortLearningQuestionStrategy` today is
  `allQuestions.shuffled().take(questionCount)` (`ShortLearningQuestionStrategy.kt:19`). Reweight that
  selection by priority bands **unseen → wrong → old-correct**. Same entry point, same screen, same 10
  questions, **zero new UI**.
- **Epic E — Cross-session "My mistakes" pool (SECOND).** `MistakeReviewQuestionStrategy` today reads
  one `resultId` from `SessionResultStore` and filters that single result's `mistakeQuestionIds`
  (`MistakeReviewQuestionStrategy.kt:17-21`). Add a **second source**: the persistent pool of
  not-mastered-on-latest questions across all sessions, surfaced as a fifth mode-selection card and
  entered as a `MistakeReview` session.

This document specifies HOW. **There is no new stored data and no migration** (§1): every fact both
epics need already lives in F1's answer log and is read through `questionMastery(...)`. Everything is
pure `commonMain` — no `expect`/`actual` — so NFR-6 (identical Android/iOS) is satisfied by
construction.

### How this fits the existing substrate (verified against the code)

- **Selection seam = the strategy interface.** Every mode resolves a `QuestionLoadingStrategy`
  (`domain/usecase/question/QuestionLoadingStrategy.kt`: `suspend fun loadQuestions(categoryId): List<Question>`).
  The `LearningMode → (strategy, machine)` mapping is the `when(mode)` in `AppModule.kt:264-279`. We
  **generalise this seam, never special-case it**: Epic W swaps the body of `ShortLearningQuestionStrategy`;
  Epic E adds one `QuestionLoadingStrategy` implementation for the pool. Both keep `loadQuestions`'s
  exact contract, so the session machines, `SessionViewModel`, and nav are untouched.
- **The pool definition is one read.** "Pool = questions **not** mastered on latest attempt" is the
  inverse of F1's mastery primitive. `questionMastery(...)` returns `QuestionMastery` per question with
  `isMasteredOnLatestAttempt` and `lastAttemptMillis` (`domain/appmemory/QuestionMastery.kt`). Epic W's
  Band 2 **is** Epic E's pool — defined once (§2.1), consumed by both.
- **Mode-selection card pattern is mature.** `ModeSelectionViewModel` injects
  `ObserveFavouriteCountUseCase`, collects it into `latestFavouriteCount`, and patches
  `Content.favouriteCount` reactively; the *Saved* card is `enabled = favouriteCount > 0` with a count
  chip (`ModeSelectionViewModel.kt:38-43`, `ModeSelectionScreen.kt:111-121`). The favourite-count use
  case re-subscribes on language switch via `languageProvider.observeLanguage().flatMapLatest { … }`
  (`ObserveFavouriteCountUseCase.kt:17-23`). The fifth card follows this pattern **exactly**.
- **Flag gating is mature.** Both epics gate on the existing `FeatureFlag.Progress` (OQ-4). The model
  is: write side uses `isEnabled` synchronously (`LogAnswerEventUseCase.kt:48`); read side combines
  `observe(Progress)` reactively and short-circuits to empty when off
  (`ObserveSubjectProgressUseCase.kt:30-43`). We reuse both idioms verbatim.
- **Question source.** `GetQuestionsUseCase(categoryId)` resolves the live bank for the active license
  (`GetQuestionsUseCase.kt:12-13`). It is the denominator for Band 1 "unseen" (W) and the materialiser
  for the pool ids (E).

---

## 1. Overview & layering — no new stored data, no migration

| Concern | Layer | What changes |
|---|---|---|
| Pool / band definition (pure functions over `List<Question>` + `List<QuestionMastery>`) | **domain** (`domain/appmemory/Banding.kt`) | NEW pure file, sibling of `Mastery.kt` |
| "Pool = not-mastered-on-latest" read + pool count flow | **domain** (use cases) | NEW use cases over the existing `AnswerLogRepository` |
| Weighted selection | **domain** (`ShortLearningQuestionStrategy`) | MODIFY: random → banded, with random fallback |
| Pool-sourced MistakeReview | **domain** (`PoolMistakeReviewQuestionStrategy`) | NEW `QuestionLoadingStrategy` impl |
| Fifth card state + count | **UI** (`ModeSelectionViewModel` / `ModeSelectionScreen`) | MODIFY: add a `MistakesPool` state slot + card, mirroring Saved |
| Entry navigation | **UI** (`AppNavGraph`, `AppRoute`) | MODIFY: add a pool-sourced `MistakeReview` route variant |
| Storage | **data** | **NONE** — reads F1's `answer_log_*` buckets through `AnswerLogRepository` |

Dependency arrows stay inward: the new pure functions and use cases live in `domain`; the strategies
depend only on domain interfaces (`AnswerLogRepository`, `QuestionRepository`/`GetQuestionsUseCase`,
`LicenseProvider`); the ViewModel depends on use cases. **No data-layer change at
all** — confirming §6 of the requirements (no migration, everything reads the log). The only
persistence touched is the *read* path of F1's `AnswerLogStoreImpl`, unchanged.

**Flag-off is a true no-op (NFR-2):** with `Progress` off, the band-selection use case returns the
random fallback **without reading the log**, and the pool-count use case emits "hidden" without
reading the log — exactly mirroring `ObserveSubjectProgressUseCase`'s short-circuit.

---

## 2. Domain interfaces & use cases

### 2.1 The shared pool/band primitive — defined once (`domain/appmemory/Banding.kt`)

The whole feature reduces to **classifying a subject's question bank into three priority bands using
the per-question mastery list**. This is a pure fold, a sibling of F1's `Mastery.kt`, unit-tested in
isolation, with the OQ-1 graduation rule as its **only** behavioural knob (§8 deferred-"twice" slot).

```kotlin
package pl.egzaminppl.app.domain.appmemory

import pl.egzaminppl.app.domain.model.Question

/** Study-priority band for one question, highest priority first. */
enum class StudyBand { UNSEEN, WRONG, OLD_CORRECT }

/** A question paired with its band and the recency used to order Band 3 (Long.MAX for unseen/wrong). */
data class BandedQuestion(
    val question: Question,
    val band: StudyBand,
    val lastAttemptMillis: Long,
)

/**
 * Classify a subject's bank against its mastery list. Pure: no IO, no scope resolution.
 *  - UNSEEN      → question has no mastery entry (no answer event in scope).
 *  - WRONG       → has a mastery entry, NOT mastered on latest attempt  ← this set == Epic E's pool.
 *  - OLD_CORRECT → has a mastery entry, mastered on latest attempt.
 * Graduation rule (OQ-1 MVP) lives ONLY in `isMasteredOnLatestAttempt`; see §8 for the "twice" slot.
 */
fun bandQuestions(
    questions: List<Question>,
    mastery: List<QuestionMastery>,
): List<BandedQuestion> {
    val masteryById = mastery.associateBy { it.questionId }
    return questions.map { q ->
        when (val m = masteryById[q.id]) {
            null -> BandedQuestion(q, StudyBand.UNSEEN, Long.MAX_VALUE)
            else -> BandedQuestion(
                question = q,
                band = if (m.isMasteredOnLatestAttempt) StudyBand.OLD_CORRECT else StudyBand.WRONG,
                lastAttemptMillis = m.lastAttemptMillis,
            )
        }
    }
}

/** The mistakes pool for one subject: bank questions whose latest attempt was wrong, in bank order. */
fun mistakesPool(
    questions: List<Question>,
    mastery: List<QuestionMastery>,
): List<Question> =
    bandQuestions(questions, mastery)
        .filter { it.band == StudyBand.WRONG }
        .map { it.question }

/**
 * Weighted Quick-review selection: fill `count` slots strictly band-by-band (OQ-7, W1.1),
 * shuffling WITHIN each band (W1.2, across-band priority preserved), Band 3 ordered oldest-first.
 * `shuffle` is injected so tests are deterministic; production passes `{ it.shuffled() }`.
 */
fun selectWeighted(
    questions: List<Question>,
    mastery: List<QuestionMastery>,
    count: Int,
    shuffle: (List<BandedQuestion>) -> List<BandedQuestion> = { it.shuffled() },
): List<Question> {
    val banded = bandQuestions(questions, mastery)
    val unseen = shuffle(banded.filter { it.band == StudyBand.UNSEEN })
    val wrong = shuffle(banded.filter { it.band == StudyBand.WRONG })
    // Band 3: oldest lastAttemptMillis first (spaced refresh, W1.1); shuffle only ties if desired.
    val oldCorrect = banded.filter { it.band == StudyBand.OLD_CORRECT }.sortedBy { it.lastAttemptMillis }
    return (unseen + wrong + oldCorrect).take(count).map { it.question }
}
```

> **Why band order over `lastAttemptMillis` for Band 3, but shuffle for Bands 1–2:** the requirements
> ask for spaced refresh (oldest-mastered-first, W1.1) but within-band variety for unseen/wrong (W1.2).
> Band 3 ordering is therefore deterministic-by-recency; the W1.2 shuffle applies to the higher bands.
> See §8 risk on determinism vs. shuffle.

### 2.2 Epic W — band selection consumed by the strategy (NOT a new use case)

Decision: **the banding logic is the pure function in §2.1; the strategy calls it directly.** No
`SelectWeightedQuestionsUseCase` — that would be a one-line pass-through wrapping a pure function, and
the existing strategies already own their own data resolution (`ShortLearningQuestionStrategy` already
injects `QuestionRepository` + `LicenseProvider`). Generalising the strategy keeps the
`LearningMode → strategy` map (`AppModule.kt:267-268`) and the `QuestionLoadingStrategy` contract
intact. The strategy gains two dependencies (the log + the flag) and a fallback:

```kotlin
class ShortLearningQuestionStrategy(
    private val questionRepository: QuestionRepository,
    private val licenseProvider: LicenseProvider,
    private val answerLogRepository: AnswerLogRepository,   // NEW
    private val featureFlagService: FeatureFlagService,     // NEW
    private val questionCount: Int = DEFAULT_SHORT_LEARNING_QUESTION_COUNT,
) : QuestionLoadingStrategy {

    override suspend fun loadQuestions(categoryId: String): List<Question> {
        val licenseId = licenseProvider.requireCurrent().id
        val all = questionRepository.getQuestionsByCategory(licenseId, categoryId)
        // Flag off → byte-for-byte today; no log read (W3.1, NFR-2).
        if (!featureFlagService.isEnabled(FeatureFlag.Progress)) {
            return all.shuffled().take(questionCount)
        }
        return try {
            val mastery = answerLogRepository.questionMastery(licenseId, categoryId)
            selectWeighted(all, mastery, questionCount)            // §2.1
        } catch (e: Exception) {
            e.rethrowIfCancellation()
            Logger.e(e, "Weighted Quick review failed; random fallback (categoryId=$categoryId)")
            all.shuffled().take(questionCount)                     // W3.3 graceful degrade
        }
    }
}
```

This satisfies W1 (band order), W2 (wrong resurfaces until `isMasteredOnLatestAttempt` flips), W3.1
(flag off = random), W3.2 (zero history ⇒ all Band 1 ⇒ shuffled unseen = today's behaviour for a fresh
subject — falls out of `selectWeighted` with empty mastery), W3.3 (read failure ⇒ random), and W1.3
(`take(count)` never pads with duplicates). NFR-1: one `questionMastery(...)` read = one bucket decode;
the bank is ≤ a few hundred per category, so selection is as instant as `shuffled().take(10)` today.

### 2.3 Epic E — pool count (UI) and pool materialisation (session)

Two use cases, both flag-gated, mirroring the favourite-count + materialise split:

**(a) `ObserveMistakesPoolCountUseCase`** — feeds the fifth card's count chip, the analogue of
`ObserveFavouriteCountUseCase`. It must be reactive to flag and license switches. Stats are
language-agnostic (PR #33): the count is scoped by license only and **survives a UI-language switch
unchanged**, so no language re-scoping belongs here.

```kotlin
package pl.egzaminppl.app.domain.usecase.appmemory

/**
 * Reactive count of the My-mistakes pool for one subject. Emits null when the Progress flag is off or
 * no license is selected → the card is HIDDEN (NFR-2: no log read on the off path). Emits 0 when the
 * pool is empty → the card is DISABLED. Emits the count when populated → ENABLED with a chip.
 */
class ObserveMistakesPoolCountUseCase(
    private val answerLogRepository: AnswerLogRepository,
    private val getQuestionsUseCase: GetQuestionsUseCase,
    private val featureFlagService: FeatureFlagService,
    private val licenseProvider: LicenseProvider,
) {
    operator fun invoke(categoryId: String): Flow<Int?>   // null = hidden, 0 = empty, n = populated
}
```

Implementation shape (mirrors `ObserveSubjectProgressUseCase.kt:30-43` + `ObserveFavouriteCountUseCase`):
`combine(observe(Progress), licenseProvider.observe())` → a nullable `Scope`; `distinctUntilChanged()`;
`flatMapLatest { scope -> if (scope == null) flowOf(null) else
answerLogRepository.observeSubjectProgress(...).map { … } }`. The pool **count** is derivable cheaply
from `SubjectProgress` for the subject (`seenCount - masteredCount` = not-mastered-on-latest count),
**without** loading the bank — so the chip never needs the question list. We reuse the existing reactive
`observeSubjectProgress(licenseId)` Flow so the count updates live as the user graduates questions
(E2 ⇒ smaller count, E4.2). The bank is only loaded at session entry (b).

> **Why `seenCount - masteredCount` and not a new repo read:** `SubjectProgress.seenCount` = distinct
> questions with any event; `masteredCount` = those correct on latest attempt (`Mastery.kt:25-32`).
> Their difference is exactly "seen but not mastered on latest" = the pool size. Both counts are
> language-agnostic and license-scoped (PR #33); the progress denominator is the Polish superset count
> `category.questionCount(Language.PL)`, never a per-language count. This reuses the already-emitting
> home-grid Flow with **zero** new repository surface, and is cheaper than the bank cross-reference
> (NFR-1: the chip resolves from ≤9 already-decoded buckets, never blocks the screen).

**(b) `PoolMistakeReviewQuestionStrategy`** — the new `QuestionLoadingStrategy` that materialises the
pool into questions for the session (E4.1). Decision: **a new sibling strategy, not a parameterised
`MistakeReviewQuestionStrategy`.** The two sources have different inputs (one `resultId` + store vs.
mastery log + bank) and different lifecycles (frozen session snapshot vs. live pool); ISP says keep
them as two focused implementations of the same interface rather than one strategy with a
`source: enum` branch. Both produce questions in **bank order**, matching today's MistakeReview
(`MistakeReviewQuestionStrategy.kt:20`, requirement E4.1).

```kotlin
package pl.egzaminppl.app.domain.usecase.question

/**
 * Loads the persistent My-mistakes pool for a subject (not-mastered-on-latest across all sessions),
 * in question-bank order. Resolves the pool fresh at session start (E4.4 stale-chip tolerance). A
 * pooled id missing from the live bank is skipped (E1.6) because `bandQuestions` only ever bands
 * questions that exist in the bank. Flag off / read failure → empty list (handled by the caller as a
 * cleared MistakeReview, E4.5).
 */
class PoolMistakeReviewQuestionStrategy(
    private val answerLogRepository: AnswerLogRepository,
    private val getQuestionsUseCase: GetQuestionsUseCase,
    private val featureFlagService: FeatureFlagService,
    private val licenseProvider: LicenseProvider,
) : QuestionLoadingStrategy {

    override suspend fun loadQuestions(categoryId: String): List<Question> {
        if (!featureFlagService.isEnabled(FeatureFlag.Progress)) return emptyList()
        return try {
            val licenseId = licenseProvider.requireCurrent().id
            val all = getQuestionsUseCase(categoryId)
            val mastery = answerLogRepository.questionMastery(licenseId, categoryId)
            mistakesPool(all, mastery)            // §2.1 — bank order, missing ids inherently skipped
        } catch (e: Exception) {
            e.rethrowIfCancellation()
            Logger.e(e, "Pool MistakeReview load failed (categoryId=$categoryId)")
            emptyList()                            // E4.5 / NFR-5: degrade to cleared, never crash
        }
    }
}
```

### 2.4 The "pool = not-mastered-on-latest" read, consumed by both epics

Defined once in §2.1 as `mistakesPool(...)` / the `WRONG` band of `bandQuestions(...)`. Epic W's
`selectWeighted` and Epic E's `mistakesPool` are the **same fold** over the same inputs (bank +
`questionMastery`). The requirements' note that "Epic W's Band 2 == Epic E's pool" is enforced
structurally: `mistakesPool` is literally `bandQuestions(...).filter { it.band == WRONG }`. Build Epic
W first ⇒ `Banding.kt` exists and is tested ⇒ Epic E reuses it with no new logic.

---

## 3. Data layer

**No new storage, no new repository methods.** Both epics compose two existing reads:

1. `AnswerLogRepository.questionMastery(licenseId, categoryId)` → `List<QuestionMastery>`
   (`AnswerLogStoreImpl.kt:64-72`): one bucket decode, latest-event-per-question fold (`Mastery.kt`).
   Language-agnostic (PR #33): one bucket per (license, category), so a question answered in either
   language folds to one entry, latest attempt wins.
2. The subject's bank: `QuestionRepository.getQuestionsByCategory(...)` via `GetQuestionsUseCase`.

**Composition for Band 1 "unseen":** unseen = `bank.id ∉ mastery.questionId`. The bank is the
denominator (the catalog owns it, exactly as F1 keeps `totalCount` out of `SubjectProgress`); the
mastery list is the seen set. `bandQuestions` joins them by id (§2.1) — an O(bank) map lookup.

**Pool count for the chip** does *not* load the bank: it derives `seenCount - masteredCount` from the
already-emitting `observeSubjectProgress(...)` Flow (§2.3a). The bank is only materialised at session
entry.

**Performance (NFR-1).** Largest categories are a few hundred questions; bank ~1775 across 9
categories. Each read is a single-bucket decode + one linear pass — comparable to `shuffled().take(10)`
today. Mode selection **never blocks on the read**: the card uses the existing
`ModeSelectionUiState.Loading` skeleton first, then the count flow resolves asynchronously (exactly the
favourite-count flow, `ModeSelectionViewModel.kt:38-43`). Quick-review selection is one extra bucket
decode on the existing `loadQuestions` suspend path, already off the UI thread.

**Resilience (NFR-5).**
- Corrupt bucket: `AnswerLogStoreImpl.decodeBucket` already catches, logs, and returns `emptyList()`
  (`AnswerLogStoreImpl.kt:74-82`). An empty mastery list ⇒ every question is Band 1 (unseen) for W
  (random fallback in effect) and an empty pool for E (card disabled / session skipped). No crash.
- Read throws: both strategies wrap the read in `try/catch + rethrowIfCancellation()` → W degrades to
  `shuffled().take(10)` (W3.3), E degrades to empty (E4.5). The count flow surfaces a read failure as
  "hidden/empty" in the VM (§4), never an error card.
- Missing pooled id (E1.6): impossible to leak — `bandQuestions`/`mistakesPool` only ever band
  questions present in the live `bank`, so a pooled id absent from the bank is silently excluded.

---

## 4. UI / Presentation

### Epic W — no UI

W is invisible to the UI: `ShortLearningQuestionStrategy.loadQuestions` keeps its signature, so
`SessionViewModel` / `QuestionScreen` / nav are untouched. **No designer work for W.**

### Epic E — the fifth "My mistakes" card

`ModeSelectionViewModel` gains a `MistakesPool` state slot, collected the same way as
`favouriteCount` (`ModeSelectionViewModel.kt:36-43`):

```kotlin
// inject ObserveMistakesPoolCountUseCase; in init:
viewModelScope.launch {
    observeMistakesPoolCountUseCase(categoryId).collect { count ->   // null | 0 | n
        latestMistakesPoolCount = count
        updateContent { copy(mistakesPoolCount = count) }
    }
}
```

`ModeSelectionUiState.Content` gains `val mistakesPoolCount: Int? = null` (defaulted ⇒ existing call
sites + flag-off compile and render unchanged). The card descriptor list in `ModeSelectionScreen`
(`learningModes(...)`, `ModeSelectionScreen.kt:77-122`) appends a fifth `ModeDescriptor` for
`LearningMode.MistakeReview` **only when `mistakesPoolCount != null`** (flag on).

**The five UI states the designer must handle** (cross-check against E3):

| # | Condition | `mistakesPoolCount` | Card treatment | Requirement |
|---|---|---|---|---|
| 1 | Loading (state still `ModeSelectionUiState.Loading`) | — | Existing screen skeleton; no card yet, never a stale count | E3.4 / NFR-1 |
| 2 | Flag OFF (default) | `null` | **Card not rendered** — byte-for-byte the four-card layout | E3.3 / NFR-2 |
| 3 | Flag ON, pool empty | `0` | **Disabled** card + hint copy ("Questions you answer incorrectly are collected here") | E3.2 |
| 4 | Flag ON, pool populated | `n > 0` | **Enabled** card + count chip = `n` (mirrors Saved `enabled = count > 0`) | E3.1 |
| 5 | Flag ON, read failure | treated as `null` (hidden) **or** `0` (disabled) | Degrade to **hidden/disabled**, never an error card | NFR-5 |

> Decision for state 5: the count use case maps a read failure to **`null` (hidden)** so a transient
> read error never surfaces a broken-looking card; the card simply behaves as if the feature is off
> until the next emission. (Disabled-`0` is an acceptable alternative the designer can choose; the VM
> contract is "error ⇒ not a populated card.")

**Card descriptor** mirrors the Saved card exactly (`ModeSelectionScreen.kt:111-121`): icon (e.g.
`Icons.Outlined.ErrorOutline` or `FactCheck`), accent color, `enabled = count > 0`, plural description
copy, and a count chip via the existing `footer`/`TonalChip` path. NFR-7 (a11y): the card carries a
content description and its count/empty state is conveyed, identical to the other four cards.

**Entry navigation (E4).** Tapping the card must start a pool-sourced `MistakeReview` **without a
`resultId`** (the pool is not a stored session). Two changes:

1. `onModeSelected` in `ModeSelectionViewModel.kt:73-95` currently `error(...)`s on `MistakeReview`
   (because it was results-only). Allow it: `LearningMode.MistakeReview -> onNavigate(mode, null)`.
2. `AppNavGraph` mode-selection branch (`AppNavGraph.kt:152-153`) currently `error(...)`s too.
   Route it to a **new `AppRoute.MistakeReviewPool(categoryId)`** (no `resultId`), distinct from the
   existing `AppRoute.MistakeReview(categoryId, resultId)` used by the results-screen CTA. The new
   route resolves a ViewModel bound to `PoolMistakeReviewQuestionStrategy` (§6).

This keeps the **per-session results-screen CTA unchanged (OQ-5, E4.3)**: `AppRoute.MistakeReview` +
`MistakeReviewQuestionStrategy` are not touched.

**E4.5 race (pool emptied between render and tap).** Today `SessionViewModel.loadQuestions` maps an
empty strategy result to `errorState()` (`SessionViewModel.kt:67-68`) — the same path the existing
per-session MistakeReview already uses when its result is fully correct. The pool-sourced session
inherits this: an emptied pool resolves to the existing cleared/empty MistakeReview path, never a crash
(E4.5 explicitly accepts "handled exactly like a fully-cleared MistakeReview today"). The card's
`enabled = count > 0` gate already prevents entry in the common case; the race is the residual covered
by the empty-strategy path.

---

## 5. Package structure

Mirrors F1's `appmemory` packages and the existing strategy package. New files in **bold**.

```
domain/appmemory/
  Banding.kt                         # NEW — StudyBand, BandedQuestion, bandQuestions, mistakesPool, selectWeighted (§2.1)
  AnswerLogRepository.kt             # unchanged — questionMastery(...) consumed as-is
  QuestionMastery.kt / Mastery.kt    # unchanged

domain/usecase/appmemory/
  ObserveMistakesPoolCountUseCase.kt # NEW — reactive Int? count for the fifth card (§2.3a)

domain/usecase/question/
  ShortLearningQuestionStrategy.kt   # MODIFIED — banded selection + random fallback (§2.2)
  PoolMistakeReviewQuestionStrategy.kt # NEW — QuestionLoadingStrategy over the pool (§2.3b)
  MistakeReviewQuestionStrategy.kt   # unchanged — per-session CTA (OQ-5)
  QuestionLoadingStrategy.kt         # unchanged contract

ui/modeselection/
  ModeSelectionViewModel.kt          # MODIFIED — mistakesPoolCount slot + collector (§4)
  ModeSelectionScreen.kt             # MODIFIED — fifth ModeDescriptor when flag on (§4)

AppNavGraph.kt                       # MODIFIED — AppRoute.MistakeReviewPool + mode-selection branch (§4)
di/AppModule.kt                      # MODIFIED — wiring (§6)
```

No `data/` changes. No `expect`/`actual`. Tests land in `commonTest` mirroring these packages
(`domain/appmemory/BandingTest.kt`, `domain/usecase/question/…StrategyTest.kt`,
`ui/modeselection/ModeSelectionViewModelTest.kt`).

---

## 6. Koin wiring (`di/AppModule.kt`)

### Epic W — modify the `ShortLearning` strategy construction

`AppModule.kt:267-268` currently `ShortLearningQuestionStrategy(get(), get())`. The strategy is built
**positionally inside `questionViewModel(...)`**, not via `singleOf`/`factoryOf`, so the gotcha below
does not bite here — but the new ctor args must be supplied explicitly in declared order:

```kotlin
LearningMode.ShortLearning ->
    ShortLearningQuestionStrategy(
        questionRepository = get(),
        licenseProvider = get(),
        answerLogRepository = get(),      // bound by F1: single<AnswerLogRepository> { … }
        featureFlagService = get(),
        // questionCount defaults to DEFAULT_SHORT_LEARNING_QUESTION_COUNT
    ) to LearningSessionMachine()
```

### Epic E — pool-count use case + pool strategy + the new route's ViewModel

```kotlin
// Use case (factory, like the rest):
factoryOf(::ObserveMistakesPoolCountUseCase)

// ModeSelectionViewModel gains the new use case dependency (positional viewModel { }):
viewModel { (categoryId: String) ->
    ModeSelectionViewModel(
        categoryId = categoryId,
        getCategoryUseCase = get(),
        getExamDurationUseCase = get(),
        getSessionUseCase = get(),
        deleteSessionUseCase = get(),
        observeFavouriteCountUseCase = get(),
        observeMistakesPoolCountUseCase = get(),   // NEW
        languageProvider = get(),
    )
}

// New pool-sourced MistakeReview ViewModel (no resultId), in questionViewModel(...):
viewModel(qualifier = named("mistakeReviewPool")) { (categoryId: String) ->
    questionViewModel(categoryId, sessionId = null, mode = LearningMode.MistakeReview, poolSourced = true)
}
```

In `questionViewModel(...)` (`AppModule.kt:258-298`), extend the `MistakeReview` branch to pick the
strategy by source:

```kotlin
LearningMode.MistakeReview ->
    if (poolSourced) {
        PoolMistakeReviewQuestionStrategy(
            answerLogRepository = get(),
            getQuestionsUseCase = get(),
            featureFlagService = get(),
            licenseProvider = get(),
        ) to FavouritesSessionMachine()
    } else {
        val sourceResultId = requireNotNull(resultId) { … }   // unchanged per-session CTA
        MistakeReviewQuestionStrategy(sourceResultId, get(), get()) to FavouritesSessionMachine()
    }
```

> ⚠️ **`singleOf` defaulted-param gotcha (F1 lesson).** `singleOf(::X)`/`factoryOf(::X)` **ignore
> defaulted constructor params** — they only wire the leading non-defaulted ones. This is why F1 binds
> `AnswerLogStoreImpl` with an explicit lambda (`single<AnswerLogRepository> { AnswerLogStoreImpl(dataStore = get()) }`,
> `AppModule.kt:154`) because of its defaulted `retentionPolicy`. Applied here:
> - `ShortLearningQuestionStrategy` has a defaulted `questionCount` — it is **already** constructed via
>   an explicit positional call inside `questionViewModel(...)`, **not** `singleOf`, so it is safe.
>   **Do not "tidy" it into a `singleOf`/`factoryOf`** or the count/flag/log deps with defaults would be
>   dropped. Keep the explicit construction.
> - `PoolMistakeReviewQuestionStrategy` and `ObserveMistakesPoolCountUseCase` have **no** defaulted
>   params, so `factoryOf(::ObserveMistakesPoolCountUseCase)` is safe; the strategy is built explicitly
>   inside `questionViewModel(...)` anyway.
> - **Verify on a real device, not just host tests** (F1 lesson: VM/host tests bypass the Koin graph —
>   `factoryOf` arity mismatches only surface at resolve time on the emulator).

All new dependencies (`AnswerLogRepository`, `FeatureFlagService`, `LicenseProvider`,
`LanguageProvider`, `GetQuestionsUseCase`) are already bound by F1/the app — no new platform module,
no new DataStore.

---

## 7. Phased plan

Ordered per OQ-3 / §10: **Epic W first (no UI), then Epic E.** Both gate on `FeatureFlag.Progress`, so
both land dark (flag default `false`) and ship with no rollout switch. Tests run on `commonTest`
(`:composeApp:testAndroidHostTest`, the host runner — `iosSimulatorArm64Test` fails to link
FirebaseCore here), `kotlin.test.*` + `runTest`, GIVEN/WHEN/THEN per the kmp-unit-testing skill.

### Phase 1 — Epic W: weighted Quick review (no UI)

**Add:** `domain/appmemory/Banding.kt` (§2.1).
**Modify:** `ShortLearningQuestionStrategy.kt` (banded + fallback, §2.2); `AppModule.kt` (extra ctor
args, §6).

**Pure functions to unit-test (`BandingTest.kt`) — Given/When/Then 1:1 with W1/W2/W3:**
- `bandQuestions`: unseen (no mastery entry) → `UNSEEN`; mastery not-mastered → `WRONG`; mastered →
  `OLD_CORRECT` with its `lastAttemptMillis`.
- `selectWeighted` **band ordering** (W1.1/W1.2, inject identity `shuffle` for determinism): strict
  unseen → wrong → old-correct fill; a Band 2 question is never chosen over an available Band 1 (W1.2).
- `selectWeighted` **Band 3 recency** (W1.1): all-mastered subject → oldest `lastAttemptMillis` first.
- **graduation** (W2.2): a question with `isMasteredOnLatestAttempt = true` lands in Band 3, not Band 2.
- **fewer than count** (W1.3): `< 10` total → returns all, no duplicates.
- **all mastered** (W1.4): Bands 1–2 empty → 10 from Band 3.

**Strategy paths to test (`ShortLearningQuestionStrategyTest.kt`, using `FakeAnswerLogRepository` +
`FakeFeatureFlagService`):**
- **flag OFF** (W3.1): returns `take(count)` of a shuffle; **`questionMastery` never called** (assert
  via fake) — NFR-2.
- **flag ON, zero history** (W3.2): all unseen ⇒ 10 shuffled unseen (= today).
- **flag ON, mixed history**: selection in band order against a seeded mastery list.
- **read failure** (W3.3): `FakeAnswerLogRepository.failReads = true` (already supported,
  `FakeAnswerLogRepository.kt:24`) — extend it to throw from `questionMastery` too → strategy returns
  random fallback, no throw.

**Acceptance:** W1.1–W1.5, W2.1–W2.2, W3.1–W3.3. Verify on device that ShortLearning still starts
instantly with the flag forced on (DebugScreen).

### Phase 2 — Epic E: My-mistakes pool + fifth card

**Add:** `domain/usecase/appmemory/ObserveMistakesPoolCountUseCase.kt` (§2.3a);
`domain/usecase/question/PoolMistakeReviewQuestionStrategy.kt` (§2.3b).
**Modify:** `ModeSelectionViewModel.kt` (count slot + collector, allow `MistakeReview` in
`onModeSelected`); `ModeSelectionScreen.kt` (fifth descriptor when flag on); `AppNavGraph.kt`
(`AppRoute.MistakeReviewPool` + branch); `AppModule.kt` (wiring, §6). UI applied to **both** Android
and iOS per CLAUDE.md (shared `commonMain` card + both run the same `App()`).

**Pure functions to unit-test:** `mistakesPool` is already covered by `BandingTest` (it is the `WRONG`
filter); add explicit pool cases mirroring E1: latest-wrong → in pool (E1.1); latest-correct after
earlier wrong → not in pool (E1.2, graduation E2.1); unseen → not in pool (E1.3); wrong-again after
graduation → back in pool (E2.2); missing-from-bank id → excluded (E1.6); bank order preserved (E4.1).

**Strategy paths to test (`PoolMistakeReviewQuestionStrategyTest.kt`):** flag OFF → empty; populated
pool → bank-order questions; read failure → empty (E4.5 / NFR-5); pooled id absent from bank → skipped.

**Use-case paths to test (`ObserveMistakesPoolCountUseCaseTest.kt`, mirroring favourite-count + the
F1 `ObserveSubjectProgressUseCaseTest`):** flag OFF → emits `null`, no log read (NFR-2, E3.3); flag ON
empty pool → `0` (E3.2); flag ON populated → `n` (E3.1); license switch re-scopes the count (E3.5,
NFR-3); a UI-language switch leaves the count unchanged (language-agnostic stats, PR #33); graduating a
question lowers the count on the next emission (E2 / E4.2).

**VM/UI state to test (`ModeSelectionViewModelTest.kt`):** the five states of §4 (loading / hidden /
disabled / populated / error-as-hidden); `onModeSelected(MistakeReview)` navigates with `null`
sessionId (no `error(...)`); the per-session results-screen path is untouched (E4.3 — assert the
existing `AppRoute.MistakeReview(categoryId, resultId)` flow still resolves
`MistakeReviewQuestionStrategy`).

**Acceptance:** E1.1–E1.6, E2.1–E2.2, E3.1–E3.5, E4.1–E4.5. Device-verify (F1 lesson): flag forced on,
answer some questions wrong, confirm the card appears with the right count, shrinks after graduating,
and the results-screen CTA still works.

---

## 8. Risks & trade-offs

- **Deferred "twice" graduation slot (OQ-1).** The entire graduation rule is the single predicate
  `QuestionMastery.isMasteredOnLatestAttempt` read inside `bandQuestions` (§2.1). If the reviewer later
  adopts "correct twice", **only `bandQuestions` changes** — it would consume a richer read primitive
  (a sibling of `questionMastery` returning the last-N outcomes / a streak, derivable on read from the
  append-only `AnswerEvent` history with **no migration**, §6 of requirements) and band `WRONG` until
  two consecutive corrects. No strategy, use case, VM, nav, or storage changes. This is the additive,
  no-migration slot the requirements call for (E2.3, W2.3). It is **not** built now.
- **Determinism vs. shuffle in W1.2.** Within-band shuffle (variety) competes with across-band priority
  (correctness) and with testability. Resolved by: across-band order is **always** strict (priority is
  correctness, never randomised); within-band order is shuffled via an **injected `shuffle` lambda** so
  production gets variety while tests inject identity for deterministic Given/When/Then. Band 3 is
  ordered by recency (spaced refresh), not shuffled, so "oldest-mastered-first" is verifiable. Risk: if
  W1.2 (Should, per §8 MoSCoW) is cut, pass identity in production too — the feature still works
  (deterministic in-band order), only variety is lost.
- **Pool count via `seenCount - masteredCount` vs. a dedicated pool read.** Chosen the derived count
  off the already-emitting `observeSubjectProgress` Flow (no bank load for the chip, reactive for free).
  Trade-off: the count and the materialised session use two different reads (count from
  `SubjectProgress`, session from `questionMastery` + bank), so a slightly stale chip is possible
  (E4.4 explicitly tolerates this — the session materialises fresh at start). Acceptable and documented.
- **New strategy vs. parameterised MistakeReview.** Chose a second `QuestionLoadingStrategy` impl
  (`PoolMistakeReviewQuestionStrategy`) over a `source` flag on the existing strategy (ISP + LSP: two
  focused, fully-substitutable implementations vs. one with a branch). Cost: a parallel `named(...)`
  route + branch in `questionViewModel`. Benefit: the per-session CTA code is byte-for-byte untouched
  (OQ-5 de-risked) and each strategy has one reason to change.
- **Empty-strategy → `errorState()` today (E4.5).** `SessionViewModel.loadQuestions` currently maps an
  empty load to `errorState()`, not a dedicated empty screen. The pool-sourced session inherits the
  same behaviour the per-session MistakeReview already has when a result is fully correct, so this is
  **pre-existing, consistent, and non-crashing** — but if product wants a friendlier "you've cleared
  this subject" empty state (a §8 "Could"), that is a `SessionViewModel`-level change affecting both
  MistakeReview sources, out of this feature's Must scope.
- **Strategy now reads the log on the hot session-start path (NFR-1).** One extra single-bucket decode
  per ShortLearning start. Bounded (≤ a few hundred questions, one bucket) and on the existing suspend
  path — negligible vs. `shuffled()`. If a very large log ever bites, an in-memory mastery cache can sit
  behind `AnswerLogRepository` with no interface change (OCP), exactly as F1's §12 notes.
