# Test Report — P1 Mistake-driven study

*Date: 2026-06-15 · Feature slug: `mistake-driven-study` · QA verification of the implemented feature*

## Execution summary

| Run | Task | Tests | Failures | Errors |
|---|---|---|---|---|
| Host JVM | `:composeApp:testAndroidHostTest` | **383** | **0** | **0** |
| Build | `:androidApp:assembleDebug` | — | BUILD SUCCESSFUL | — |

Pre-existing tests: 348. New tests added during QA: **35** across new edge-case files
(`BandingEdgeCaseTest`, `ObserveMistakesPoolCountEdgeCaseTest`, `MistakeDrivenStudyEdgeCaseTest`),
on top of the 36 the implementer wrote.

## Acceptance-criteria coverage

| Criterion | Verifying test(s) | Status |
|---|---|---|
| **E1.1** — latest-wrong → in pool | `BandingTest`, `BandingEdgeCaseTest` | PASS |
| **E1.2** — latest-correct → not in pool | `BandingTest`, `BandingEdgeCaseTest` (out-of-order timestamps) | PASS |
| **E1.3** — unseen → not in pool | `BandingTest` | PASS |
| **E1.4** — Saved bookmark never enters pool (G2) | `BandingEdgeCaseTest` (×2), `MistakeDrivenStudyEdgeCaseTest`, `ObserveMistakesPoolCountEdgeCaseTest` | PASS |
| **E1.5** — licence+language scope isolation | `ObserveMistakesPoolCountUseCaseTest`, `ObserveMistakesPoolCountEdgeCaseTest` (×3), `MistakeDrivenStudyEdgeCaseTest` (×4) | PASS |
| **E1.6** — pooled id absent from bank is skipped | `BandingTest`, `PoolMistakeReviewQuestionStrategyTest` | PASS |
| **E2.1** — correct in any mode → graduated | `BandingTest`, `MistakeDrivenStudyEdgeCaseTest` (Exam + MistakeReview modes) | PASS |
| **E2.2** — graduated then wrong again → back in pool | `BandingTest`, `MistakeDrivenStudyEdgeCaseTest` | PASS |
| **E2.3** — "twice" rule delta (OQ-1 deferred) | Documented; no MVP test needed | NOT IN SCOPE |
| **E3.1** — flag on + pool≥1 → count chip | `ModeSelectionViewModelTest`, `ObserveMistakesPoolCountUseCaseTest` | PASS |
| **E3.2** — flag on + empty pool → 0 (disabled) | `ModeSelectionViewModelTest`, `ObserveMistakesPoolCountUseCaseTest`, `ObserveMistakesPoolCountEdgeCaseTest` | PASS |
| **E3.3** — flag off → card hidden (null) | `ModeSelectionViewModelTest`, `ObserveMistakesPoolCountUseCaseTest` | PASS |
| **E3.4** — loading → initial state is Loading | `ModeSelectionViewModelTest` | PASS |
| **E3.5** — scope switch → count re-scopes | `ObserveMistakesPoolCountUseCaseTest`, `ObserveMistakesPoolCountEdgeCaseTest` (×3) | PASS |
| **E4.1** — tap → session in bank order | `PoolMistakeReviewQuestionStrategyTest`, `BandingTest` | PASS |
| **E4.2** — MistakeReview answers graduate (count drops) | `ModeSelectionViewModelTest`, `ObserveMistakesPoolCountUseCaseTest` | PASS |
| **E4.3** — per-session results-screen CTA untouched | `MistakeReviewQuestionStrategyTest`, `MistakeDrivenStudyEdgeCaseTest` (regression) | PASS |
| **E4.4** — stale chip never blocks entry | Session materialises fresh at load; not unit-testable at this layer | NOT UNIT-TESTABLE |
| **E4.5** — empty pool at session start → cleared not crash | `PoolMistakeReviewQuestionStrategyTest` (read failure → empty) | PASS |
| **W1.1** — strict band order (unseen→wrong→old) | `BandingTest`, `BandingEdgeCaseTest` (7+5+5 partial fill) | PASS |
| **W1.2** — within-band shuffle, across-band priority | `BandingTest`, `BandingEdgeCaseTest` | PASS |
| **W1.3** — <10 total → no duplicate padding; 0 → empty | `BandingTest`, `BandingEdgeCaseTest`, `MistakeDrivenStudyEdgeCaseTest` | PASS |
| **W1.4** — all mastered → 10 oldest from Band 3 | `BandingTest`, `BandingEdgeCaseTest`, `MistakeDrivenStudyEdgeCaseTest` | PASS |
| **W1.5** — scope switch → banding re-scopes | `MistakeDrivenStudyEdgeCaseTest` (language + licence) | PASS |
| **W2.1** — wrong question stays eligible in Band 2 | `BandingEdgeCaseTest` (repeated `selectWeighted`) | PASS |
| **W2.2** — correct answer → moves to Band 3 | `BandingTest`, `BandingEdgeCaseTest` | PASS |
| **W2.3** — "twice" rule delta (OQ-1 deferred) | Not in MVP scope | NOT IN SCOPE |
| **W3.1** — flag off → random, no log read | `ShortLearningQuestionStrategyTest`, `MistakeDrivenStudyEdgeCaseTest` | PASS |
| **W3.2** — zero history → all Band 1 (= random) | `BandingTest`, `ShortLearningQuestionStrategyTest` | PASS |
| **W3.3** — read failure → random fallback, playable set | `ShortLearningQuestionStrategyTest`, `MistakeDrivenStudyEdgeCaseTest` | PASS |
| **NFR-1** — read perf / no perceptible stall | Timing; one bucket decode on suspend path | NOT UNIT-TESTABLE |
| **NFR-2** — flag off is true no-op (no log read) | `ShortLearningQuestionStrategyTest`, `ObserveMistakesPoolCountUseCaseTest`, `PoolMistakeReviewQuestionStrategyTest` | PASS |
| **NFR-3** — per-licence + per-language scoping | `ObserveMistakesPoolCountUseCaseTest` + edge tests (×6) | PASS |
| **NFR-4** — local-only, no network | By construction (no network dependency) | NOT UNIT-TESTABLE |
| **NFR-5** — resilient: failure → null not 0; corrupt → skip | `ObserveMistakesPoolCountEdgeCaseTest` (assertNull on failure), `PoolMistakeReviewQuestionStrategyTest` | PASS |
| **NFR-6** — cross-platform identical behaviour | Logic in `commonMain`; iOS framework compiles; runtime not testable here | NOT UNIT-TESTABLE |
| **NFR-7** — accessibility of the new card | Compose semantics / TalkBack — device only | NOT UNIT-TESTABLE |

## Defects

**0 defects.** All 383 tests pass. Items verified adversarially:

- **NFR-5 null-vs-zero contract is correct.** `ObserveMistakesPoolCountUseCase` emits `null` on read
  failure (`.catch { emit(null) }`) and `0` on a genuinely empty pool (`?: 0`). Test
  *"GIVEN flag on and log read throws WHEN observed THEN emits null not zero"* passes — the most
  dangerous candidate (a `0` on failure would falsely tell a user with real mistakes they are clear).
- **G2 Saved/My-mistakes independence is structural.** `mistakesPool` / `PoolMistakeReviewQuestionStrategy`
  take no `FavouriteRepository` — independence is enforced by the type system, and confirmed by explicit
  tests at the pure-function and strategy layers.
- **Out-of-order timestamps.** `latestEventPerQuestion` uses `>=` with append order as tiebreak; an
  older correct event does not mask a newer wrong event (both directions tested).
- **Band 3 tie-ordering** is stable (`sortedBy`, stable in Kotlin).
- **E4.3 regression** clean — `MistakeReviewQuestionStrategy` reads only its stored `resultId`,
  independent of `AnswerLogRepository`.
- **`coerceAtLeast(0)` guard** correct — `seenCount - masteredCount` cannot go negative with valid F1 data.

## Items requiring manual / device verification

- **NFR-1 performance** — mode-selection chip must not block. One bucket decode on a suspend path,
  equivalent to today's `shuffled().take(10)`. Risk: LOW.
- **NFR-6 iOS runtime** — `iosSimulatorArm64Test` cannot link FirebaseCore in this env; logic is pure
  `commonMain`. Verify in Xcode. Risk: LOW.
- **NFR-7 accessibility** — TalkBack/VoiceOver descriptions; design spec complete, Material3
  `mergeDescendants` handles it. Risk: LOW.
- **E4.4 stale chip / race** — chip from reactive `observeSubjectProgress`, session materialises fresh;
  tolerated by the requirement, never crashes. Risk: LOW.
- **Koin DI wiring** — host tests bypass `AppModule`; the `singleOf` defaulted-param gotcha applies to
  `ShortLearningQuestionStrategy`'s positional construction. **Riskiest untested area** — verified by
  device smoke-test with `FeatureFlag.Progress` forced on (done 2026-06-15).
