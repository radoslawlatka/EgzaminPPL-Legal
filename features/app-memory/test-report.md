# F1 — Give the App a Memory: Test Report

*Feature slug: `app-memory` · Branch: `feature/app-memory` · QA run: 2026-06-15*
*Suite task: `:composeApp:testAndroidHostTest` (JVM host; `iosSimulatorArm64Test` fails to link FirebaseCore)*

---

## 1. Acceptance-Criteria Coverage Matrix

Legend: **✅ Covered** = at least one test; **⚠ Weak** = existing test is thin; **❌ Gap** = no test before this run; **NUT** = Not Unit-Testable (manual / e2e).

### Epic A — Durable answer-event log

| Criterion | Covering test(s) | Status | Notes |
|---|---|---|---|
| **A1.1** event written with all §6 fields | `LogAnswerEventUseCaseTest.GIVEN flag on WHEN invoked THEN one event with all forward-compat fields` | ✅ | |
| **A1.2** survives process death | NUT | NUT | Requires a physical relaunch — unit tests cannot exercise process death. |
| **A1.3** re-answer appends; earlier event not overwritten | `AnswerLogStoreImplTest.GIVEN repeated answers WHEN append THEN log is append-only` | ✅ | |
| **A1.4** Favourites mode → nothing logged | `LogAnswerEventUseCaseTest.GIVEN Favourites mode WHEN invoked THEN nothing is logged` | ✅ | |
| **A1.5** Exam logs at finish (not per selection) | `LogAnswerEventUseCaseTest.GIVEN a batch WHEN invoked THEN one event per graded answer` | ✅ | The batch overload is the Exam write site. |
| **A1.6** no null-license event | `LogAnswerEventUseCaseTest.GIVEN no license WHEN invoked THEN no orphan event is written` | ✅ | |
| **A2.1** coverage = distinct questions seen | `MasteryTest.GIVEN distinct questions WHEN toSubjectProgress THEN coverage is distinct seen` | ✅ | |
| **A2.2** mastery = most-recent-attempt-wins | `MasteryTest.GIVEN most recent attempt correct/incorrect` | ✅ | |
| **A2.3** wrong→right: Z counts as mastered | `MasteryTest.GIVEN most recent attempt correct WHEN toSubjectProgress` | ✅ | |
| **A2.4** never-practised → zero without error | `MasteryTest.GIVEN no events WHEN toSubjectProgress THEN zero counts` | ✅ | |
| **A2.5** per-licence scoping; switch restores | `AnswerLogStoreImplTest.GIVEN events in different scopes` + `GIVEN switch back to a scope` | ✅ | |

### Epic B — Session-history store

| Criterion | Covering test(s) | Status | Notes |
|---|---|---|---|
| **B1.1** two results both retrievable | `SessionHistoryStoreImplTest.GIVEN two finished sessions WHEN stored THEN both resolve by id` | ✅ | |
| **B1.2** results screen unchanged (get by id) | `SessionHistoryStoreImplTest.GIVEN a stored result WHEN read by id THEN id finishedAt license and lang are stamped` | ✅ | |
| **B1.3** parent result survives after MistakeReview | `SessionHistoryStoreImplTest.GIVEN a parent result WHEN a mistake review is stored` | ✅ | |
| **B1.4** corrupt entry skipped, others readable | `SessionHistoryStoreImplTest.GIVEN a corrupt bucket WHEN reading another id THEN skipped` | ✅ | |
| **B2.1** no migration (app unreleased) | Deliberate architectural decision — no migration exists | N/A | Architecture doc: no migration. |
| **B2.2/B2.3** legacy empty state / no crash | N/A — legacy store deleted | N/A | App unreleased; `PersistedSessionResultStore` deleted. |
| **B3.1** bound exists and is enforced | `SessionHistoryStoreImplTest.GIVEN more results than the cap WHEN stored THEN only the most recent retained` | ✅ | |
| **B3.2** pruning preserves mastery / correctness | `AnswerLogStoreImplTest.GIVEN more events than the cap WHEN appended THEN retention preserves latest per question` | ✅ | |

### Epic C — Home-tile progress

| Criterion | Covering test(s) | Status | Notes |
|---|---|---|---|
| **C1.1** practised tile differs from never-practised | `CategoriesViewModelTest.GIVEN flag on with events WHEN content loads THEN practised tile is Populated` | ✅ | |
| **C1.2** empty state not scary / no "0%" when never practised | `CategoriesViewModelTest.GIVEN flag on WHEN subject never practised THEN tile is Empty` | ✅ | |
| **C1.3** updates after session without restart | `CategoriesViewModelProgressEdgeCaseTest.GIVEN flag on WHEN new event appended THEN tile updates live` | ✅ (NEW) | |
| **C1.4** licence switch re-scopes tiles | `CategoriesViewModelTest.GIVEN flag on WHEN license switches THEN progress re-scopes` | ✅ | |
| **C1.5** skeleton first, no stale numbers | NUT | NUT | Requires Compose rendering to verify shimmer. |

### Epic D — Feature flag

| Criterion | Covering test(s) | Status | Notes |
|---|---|---|---|
| **D1.1** flag off → no progress, grid as today | `CategoriesViewModelTest.GIVEN flag off WHEN content loads THEN every tile is Hidden` | ✅ | |
| **D1.2** flag on → full F1 behaviour active | `CategoriesViewModelTest.GIVEN flag on with events WHEN content loads THEN tile is Populated` | ✅ | |
| **D1.3** debug ForceOn/Off | `CompositeFeatureFlagServiceTest.Progress debug override forces on/off regardless of remote` | ✅ | |
| **D1.4** flag toggle at runtime flips grid | `CategoriesViewModelTest.GIVEN flag off WHEN toggled on at runtime THEN tiles flip to progress` | ✅ | |
| **D1.5** flag turned on for first time → empty, no crash | `CategoriesViewModelProgressEdgeCaseTest.GIVEN flag just turned on with no prior history WHEN content loads THEN tiles are Empty` | ✅ (NEW) | |

### NFRs

| NFR | Covering test(s) | Status | Notes |
|---|---|---|---|
| **NFR-2** write off critical path / no jank | NUT | NUT | Requires timing measurement in a real device run. |
| **NFR-3** per-licence + per-language scoping | `AnswerLogRetentionBoundaryTest.GIVEN events for same license but different languages` (×3 tests) + `SessionHistoryBoundaryTest.GIVEN results stored under different languages` | ✅ (NEW) | |
| **NFR-5** corrupt bucket skipped, no crash | `AnswerLogStoreImplTest.GIVEN a corrupt bucket` + `SessionHistoryBoundaryTest.GIVEN the active scope bucket is corrupt` + `CategoriesViewModelTest.GIVEN log read fails THEN Unavailable` | ✅ | |
| **NFR-6** cross-platform | N/A — all logic in `commonMain`; same code runs both | N/A | |
| **NFR-7** bounded storage | `SessionHistoryStoreImplTest.cap+5` + `AnswerLogStoreImplTest.cap+50` + `AnswerLogRetentionPolicyTest` | ✅ | |

---

## 2. New Tests Added

### `MasteryEdgeCaseTest` (10 tests) — NEW FILE

Location: `composeApp/src/commonTest/kotlin/pl/egzaminppl/app/domain/appmemory/MasteryEdgeCaseTest.kt`

| # | Test name | Gap filled |
|---|---|---|
| 1 | `GIVEN wrong then right then wrong WHEN toSubjectProgress THEN most recent wrong attempt wins` | A2 triple-flip sequence |
| 2 | `GIVEN right then wrong then right WHEN toSubjectProgress THEN most recent correct attempt wins` | A2 triple-flip sequence |
| 3 | `GIVEN two events with the same timestamp WHEN toSubjectProgress THEN last in append order wins` | A2 tie-breaking at identical timestamp |
| 4 | `GIVEN two events with the same timestamp reversed WHEN toSubjectProgress THEN last in list wins` | A2 tie-breaking reversed order |
| 5 | `GIVEN a question answered in multiple modes WHEN toSubjectProgress THEN mode does not affect mastery` | A2 multi-mode mastery |
| 6 | `GIVEN a question answered in multiple modes WHEN toQuestionMastery THEN lastMode reflects the latest event` | lastMode captured correctly |
| 7 | `GIVEN one question answered many times WHEN toSubjectProgress THEN seenCount is 1` | Coverage counts distinct questions (100× churn) |
| 8 | `GIVEN every question answered wrong WHEN toSubjectProgress THEN seenCount positive and masteredCount is zero` | A2 all-wrong yields Populated(0, n) not Empty |
| 9 | `GIVEN mastery folds over a floor-only event list WHEN toSubjectProgress THEN counts stay exact` | B3 floor-preserving invariant |
| 10 | `GIVEN single event list WHEN toSubjectProgress THEN works correctly` | Boundary: exactly 1 event |

### `AnswerLogRetentionPolicyTest` (8 tests) — NEW FILE

Location: `composeApp/src/commonTest/kotlin/pl/egzaminppl/app/data/appmemory/AnswerLogRetentionPolicyTest.kt`

| # | Test name | Gap filled |
|---|---|---|
| 1 | `GIVEN events below the cap WHEN prune THEN list returned unchanged` | B3 boundary: below cap |
| 2 | `GIVEN exactly cap distinct-question events WHEN prune THEN all events retained` | B3 boundary: exactly at cap |
| 3 | `GIVEN cap plus one churn events on one question WHEN prune THEN latest attempt retained` | B3 boundary: cap+1 churn |
| 4 | `GIVEN floor larger than the cap WHEN prune THEN floor wins` | B3 floor > cap: mastery wins |
| 5 | `GIVEN a mixed list after pruning WHEN inspected THEN oldest-first order preserved` | Append order preserved after pruning |
| 6 | `GIVEN empty event list WHEN prune THEN empty list returned` | Empty input |
| 7 | `GIVEN a single event WHEN prune THEN returned unchanged` | Single-event input |
| 8 | `GIVEN one unique question and many churn events WHEN prune THEN recent churn fills budget` | Budget-fill logic |

### `AnswerLogRetentionBoundaryTest` (9 tests) — NEW FILE

Location: `composeApp/src/commonTest/kotlin/pl/egzaminppl/app/data/appmemory/AnswerLogRetentionBoundaryTest.kt`

| # | Test name | Gap filled |
|---|---|---|
| 1 | `GIVEN exactly cap events WHEN appended THEN no pruning occurs` | B3 boundary: exactly cap via store |
| 2 | `GIVEN cap minus one events WHEN appended THEN all events retained` | B3 boundary: cap−1 |
| 3 | `GIVEN cap plus one events WHEN appended THEN latest per question preserved` | B3 boundary: cap+1 floor |
| 4 | `GIVEN cap plus one churn events on one question WHEN appended THEN latest attempt wins` | B3 + mastery correctness |
| 5 | `GIVEN events for same license but different languages WHEN observe THEN no bleed across languages` | NFR-3 language isolation |
| 6 | `GIVEN events for same language but different licenses WHEN observe THEN no bleed across licenses` | NFR-3 license isolation |
| 7 | `GIVEN events for all three scope combos WHEN all scopes observed THEN each is isolated` | NFR-3 full three-axis isolation |
| 8 | `GIVEN switching scope and back WHEN observe THEN original scope is intact` | NFR-3 switch-and-back |
| 9 | `GIVEN no events WHEN observing an empty scope THEN empty list returned without error` | A2.4 empty-scope edge case |

### `SessionHistoryBoundaryTest` (11 tests) — NEW FILE

Location: `composeApp/src/commonTest/kotlin/pl/egzaminppl/app/data/session/SessionHistoryBoundaryTest.kt`

| # | Test name | Gap filled |
|---|---|---|
| 1 | `GIVEN exactly 25 results WHEN stored THEN all 25 retained` | B3 boundary: exactly cap |
| 2 | `GIVEN 26 results WHEN stored THEN oldest pruned, newest 25 retained` | B3 boundary: cap+1 |
| 3 | `GIVEN a parent result WHEN two mistake reviews stored THEN parent retrievable` | B1.3 extended: two reviews |
| 4 | `GIVEN results stored under different languages WHEN reading by id THEN each resolves` | NFR-3 language scoping |
| 5 | `GIVEN results stored under different licenses WHEN reading by id THEN each resolves` | NFR-3 license scoping |
| 6 | `GIVEN the active scope bucket is corrupt WHEN get called THEN null, no crash` | NFR-5 corrupt-active-scope |
| 7 | `GIVEN one corrupt and one valid bucket WHEN get for valid THEN it resolves` | NFR-5 partial corruption |
| 8 | `GIVEN result stored at known time WHEN read THEN all forward-compat fields present` | §6 id/finishedAt/licenseId/lang stamped |
| 9 | `GIVEN no license selected WHEN store called THEN result not findable via get` | A1.6/§5.2 null-license guard |
| 10 | `GIVEN multiple stored results WHEN ids collected THEN all ids are distinct` | B1.1 id uniqueness |
| 11 | `GIVEN cap results then one more WHEN second-newest requested THEN it resolves` | B3 newest-N survival |

### `CategoriesViewModelProgressEdgeCaseTest` (8 tests) — NEW FILE

Location: `composeApp/src/commonTest/kotlin/pl/egzaminppl/app/ui/categories/CategoriesViewModelProgressEdgeCaseTest.kt`

| # | Test name | Gap filled |
|---|---|---|
| 1 | `GIVEN every answer was wrong WHEN flag on THEN tile is Populated with masteredCount zero` | C1 design §3.3: all-wrong → Populated(0%) |
| 2 | `GIVEN category with zero total questions WHEN event recorded THEN mastery fraction is 0f not NaN` | Zero-denominator safety |
| 3 | `GIVEN flag just turned on with no history WHEN content loads THEN all tiles Empty` | D1.5: first flag-on |
| 4 | `GIVEN flag on WHEN new event appended after first render THEN tile updates live` | C1.3 live update |
| 5 | `GIVEN events under PL WHEN language switches to EN THEN tile becomes Empty` | NFR-3 language-scope via VM |
| 6 | `GIVEN flag on WHEN license switches and back THEN original progress restored` | C1.4 + NFR-3 switch-and-back |
| 7 | `GIVEN flag on then turned off WHEN flag toggled THEN tiles revert to Hidden` | D1.4 reactive flag-off |
| 8 | `GIVEN all questions mastered WHEN flag on THEN mastery fraction in valid range` | TileProgress.Populated.mastery arithmetic |

### `ObserveSubjectProgressUseCaseEdgeCaseTest` (5 tests) — NEW FILE

Location: `composeApp/src/commonTest/kotlin/pl/egzaminppl/app/domain/usecase/appmemory/ObserveSubjectProgressUseCaseEdgeCaseTest.kt`

| # | Test name | Gap filled |
|---|---|---|
| 1 | `GIVEN flag on but no license selected WHEN observed THEN empty list emitted without error` | Null-license guard in use case |
| 2 | `GIVEN flag on with no events WHEN observed THEN empty list emitted` | D1.5 use-case level |
| 3 | `GIVEN events for PL and EN WHEN language switches THEN use case reflects new scope` | NFR-3 language switch via use case |
| 4 | `GIVEN no explicit flag override WHEN observed THEN Progress defaults off` | D1 default-off |
| 5 | `GIVEN events for ppl_a and ppl_h WHEN license switches THEN only active scope returned` | NFR-3 license switch via use case |

---

## 3. Observed Suite Result

```
Task :composeApp:testAndroidHostTest

BUILD SUCCESSFUL in 2s
20 actionable tasks: 3 executed, 17 up-to-date

Test counts:
  Pre-existing: 259 tests, 0 failures
  New (this run): 51 tests, 0 failures
  Total: 310 tests, 0 failures, 0 errors
```

One test-defect (not a product defect) was found and fixed during authoring: a test in
`ObserveSubjectProgressUseCaseEdgeCaseTest` incorrectly asserted `categoryId == "q_pl"` when the
category id is always `"cat"`. The intent (scope isolation) was already verified by the preceding
`seenCount` assertion. The spurious assertion was removed. The suite then turned green.

---

## 4. Defects Found

**No production defects.** All acceptance criteria that are testable at unit level pass. Three
items require clarification:

### DEFECT CANDIDATE — D1.1: Result persistence is not flag-gated (architecture note, not a defect)

The architecture document explicitly records that `SessionHistoryStoreImpl` is **not** flag-gated
(the results screen needs it regardless). The requirement D1.1 ("off = byte-for-byte today's") is
therefore a soft relaxation for the results-screen path. This is a documented and accepted decision,
not a defect. It is flagged here so the reviewer can confirm the understanding is accurate.

### OBSERVATION — same-timestamp tie-breaking is last-writer-wins (not a defect)

`latestEventPerQuestion` uses `>=` so a second event with the **same millisecond timestamp**
overwrites the first. This means if two events share a timestamp, the last one appended wins. No
acceptance criterion specifies tie-breaking behaviour, but the implementation is consistent (the
last-appended event is always the one retained). Documented for future feature authors.

### OBSERVATION — null-license `store` call returns an id that `get` cannot resolve

If `licenseProvider.current()` is `null` at result-store time, `SessionHistoryStoreImpl.store`
mints an id and returns it, but writes no bucket, so `get(id)` returns null. The architecture
document calls this path unreachable in production ("a live session always has a license"). The test
`GIVEN no license selected WHEN store called THEN result not findable via get` documents and pins
this behaviour so a future change cannot accidentally start writing null-license buckets.

---

## 5. Criteria Not Testable at Unit Level (manual / e2e verification required)

| Criterion | Why not unit-testable | Recommended manual step |
|---|---|---|
| A1.2 — event survives process death | Requires physical app kill + relaunch cycle; DataStore is the real persistence layer that cannot be exercised in-JVM with `FakePreferencesDataStore` | Install debug build with Progress flag on, answer a question, kill the process from the app switcher, relaunch, verify the home tile shows updated progress. |
| C1.5 — skeleton shows during load, never stale data | Requires Compose rendering and timing; the shimmer composable and its flag-gated bar placeholder need visual inspection | Screenshot test or manual check: observe that tiles show shimmer on first cold start with the flag on. |
| NFR-1 — grid not blocked on history read | Requires timing on a mid-range device | Measure `observeSubjectProgress` latency with a multi-week log (≈9 buckets × 500 events = 4500 events) on a device. Target: initial grid visible in <100 ms. |
| NFR-2 — write off critical path / no jank | Requires a frame-timing measurement (Systrace or Perfetto) | Answer a question rapidly on a low-end device; confirm no frame drop during the grading animation. |
| NFR-6 — identical behaviour on iOS | Requires running the iOS target; `iosSimulatorArm64Test` fails to link FirebaseCore in this environment | CI on GitHub Actions with the iOS simulator target. |
| D1.3 — Debug screen ForceOn/ForceOff visible in UI | Requires a debug build and the debug screen to be visible | Open debug screen, confirm `Progress` flag row is present, toggle it, observe grid reaction. |

---

## 6. Remaining Risk Assessment

**Lowest-risk areas:** Epic A write/read logic (Mastery.kt is pure; comprehensively tested),
Session-history store (keyed-list + retention; covered at boundary), feature-flag integration
(CompositeFeatureFlagService already had coverage; Progress cases added).

**Moderate risk:** The live-update path (C1.3) is tested against the Fake, not the real DataStore.
`AnswerLogStoreImpl.observeSubjectProgress` emits on every DataStore write via `dataStore.data.map`.
The real DataStore guarantees this; the fake replicates it via MutableStateFlow. Risk is low but
not zero.

**Highest-risk untested area:** End-to-end process-death durability (A1.2). The entire value
proposition of F1 is that events survive a kill. Unit tests exercise the DataStore-backed store
against a fake (in-memory) DataStore, so they do not exercise the actual file-system persistence.
This must be verified on a real device as described in §5.
