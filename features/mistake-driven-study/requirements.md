# P1 — Mistake-driven study

*Feature slug: `mistake-driven-study` · Owner: Product · Status: requirements (pre-design)*
*Source: UX audit 2026-06-11 §2.F2, §3, §4 ("P1 — Mistake-driven study"), §5 step 4 · Date: 2026-06-15*
*Builds on: F1 "app-memory" (DONE on this branch) — `docs/features/app-memory/requirements.md`*

> **Reviewer note.** This document defines WHAT and WHY, not HOW. No DataStore key, query shape, or
> screen layout is prescribed — those belong to the architect and designer. The two sub-features
> below are the **first two follow-up iterations** F1 was explicitly built to enable (F1 §10 release
> plan). F1 already ships the read primitive both consume: `AnswerLogRepository.questionMastery(...)`
> and `observeSubjectProgress(...)`, whose doc comment states it is "the foundation for the
> cross-session mistakes-pool and weighted-review features — not consumed in-app yet." This feature
> is where we consume it.
>
> **The one decision that needs the reviewer's call** is OQ-1: the audit wishes wrong questions
> resurface "until answered correctly **twice**", but the data model F1 ships
> (`QuestionMastery.isMasteredOnLatestAttempt`) only cheaply supports **"correct on the latest
> attempt"**. "Twice" needs streak tracking the model does not have today. See §9 OQ-1 — the rest of
> the doc proceeds on the cheap, no-migration MVP rule and flags exactly where "twice" would cost
> more.

---

## 1. Problem — who hurts, when, and how much

F1 gave the app a memory; it is currently consumed by exactly one surface (the home-tile progress
signal). The student-pilot's effort is now durably recorded but **the two highest-leverage uses of
that memory are still missing**, and both are about studying smarter, not just remembering:

- **"Let me practise everything I've ever gotten wrong" is still impossible across sessions.**
  The audit (§2.F2) notes the redesign's *Review mistakes* CTA only spans the **one stored session**
  it was launched from — `MistakeReviewQuestionStrategy` reads a single `resultId` from the
  `SessionResultStore` and filters that result's `mistakeQuestionIds`
  (`MistakeReviewQuestionStrategy.kt`). A mistake made two sessions ago is invisible. There is no
  standing "these are my open mistakes" surface, even though F1 now records every wrong answer.

- **Quick review wastes the user's scarce study time on questions they already know.** Quick review
  (`LearningMode.ShortLearning`) picks **10 fully random** questions
  (`ShortLearningQuestionStrategy.kt`: `allQuestions.shuffled().take(questionCount)`). It is just as
  likely to re-serve a question the user has aced five times as one they have never seen or keep
  getting wrong. The audit calls reweighting this "dramatically more useful at zero UI cost" — same
  entry point, same 10 questions, smarter selection.

**Job-to-be-done:** *"When I sit down to study, target my weak spots — resurface the questions I keep
getting wrong and the ones I've never seen — so my limited time moves the subjects that will
actually stop me passing, instead of re-drilling what I already know."*

This is audit §5 step 4 ("Cross-session mistakes pool + weighted Quick review — upgrade the existing
Review-mistakes CTA and add the fifth mode card"). It is a thin layer on F1's log: **no migration,
and the weighted-review half needs no new screens at all.**

---

## 2. Target users

**Primary persona — the student-pilot** (same as F1). An adult studying for a ULC theory exam over
several weeks, typically one licence at a time (PPL(A), PPL(H), SPL, BPL), across nine subject
categories. Studies in short, repeated sessions; returns to the app many times; ultimately sits a
multi-subject exam where each subject must independently clear the 75% pass mark (weakest-link).

For *this* feature the persona's relevant behaviour is:
- They have already accumulated answer history (the feature is empty/no-op for a brand-new user — see
  §3 G-states and §9 OQ-4).
- Their recurring "what should I do now?" question (F1 §2) is answered most directly by "drill what
  you keep missing" and "stop re-seeing what you've mastered."

**Context of use:** mobile, often offline (bank is cached, all F1 data is local). History grows over
weeks. The mistakes pool can be large early in a campaign and should shrink as the user improves —
that shrinking *is* the progress signal.

---

## 3. Goals & success metrics

Two sub-features, scored separately so they can ship separately (§8, §9 OQ-3).

### Sub-feature 1 — Cross-session "My mistakes" pool (fifth mode card)

| # | Goal | Success metric |
|---|------|----------------|
| G1 | The user can practise every question they currently have wrong, across all sessions | From mode selection, the user can enter a MistakeReview session whose question set = exactly the questions not-mastered-on-latest-attempt in that subject (current licence + language), verified by test against a seeded log. |
| G2 | The pool is auto-collected and distinct from manual Saved | A wrong answer enters the pool with no user action; a *Saved* bookmark never enters the pool and vice-versa (the two surfaces are independent — `FavouriteRepository` vs. `AnswerLogRepository`). |
| G3 | A mastered question leaves the pool | After the user answers a pooled question correctly (graduation rule per §9 OQ-1), it no longer appears in the pool on the next read. Verified by test. |
| G4 | The card honestly reflects the pool size | The fifth card shows a count chip equal to the current pool size for the subject, and is disabled (with hint copy) when the pool is empty — mirroring the *Saved* card's disabled-when-empty pattern (`ModeSelectionScreen.kt`). |

### Sub-feature 2 — Weighted Quick review

| # | Goal | Success metric |
|---|------|----------------|
| G5 | Quick review prioritises the questions that move readiness | Given a subject with a mix of unseen / wrong / mastered questions and history, the 10 selected questions are drawn in priority order **unseen → wrong → old-correct**, verified by test against a seeded log (definition in story W1). |
| G6 | Wrong questions resurface until handled | A question answered wrong is eligible to reappear in a later Quick review until it meets the graduation rule (§9 OQ-1); once graduated it drops to the lowest priority band. Verified by test. |
| G7 | Quick review never regresses for users with no history | With the flag off, or zero history for the subject, Quick review still returns 10 questions exactly as today (random fallback) — no empty Quick review, no error. Verified by test. |

**Explicit non-metric:** neither sub-feature commits to a statistics dashboard, a readiness verdict,
streaks, or any SRS scheduling beyond the three-band ordering. Those are §5 out-of-scope.

---

## 4. Epics & user stories

Two epics, one per sub-feature, so they can ship in either order (§9 OQ-3 recommends weighted Quick
review first — cheapest, no new screen). All acceptance criteria are Given/When/Then so QA maps each
line 1:1 to a `commonTest` case.

> Throughout: **"mastered"** means the question meets the graduation rule resolved in §9 OQ-1.
> The recommended MVP rule is **"correct on the user's latest attempt"** —
> `QuestionMastery.isMasteredOnLatestAttempt`, which F1 already computes (`Mastery.kt`). Wherever the
> audit's "correct twice" rule would change behaviour, it is called out inline.

---

### Epic E — Cross-session "My mistakes" pool

The system surfaces, as a fifth mode-selection card, the set of questions the user currently has
wrong in a subject (latest-attempt incorrect or never-correct), auto-collected across all sessions,
and lets them drill exactly that set in a MistakeReview session.

#### E1. Define pool membership from the answer log (Must)

**As a** student-pilot
**I want** every question I currently have wrong in a subject collected automatically
**So that** I can practise my real, standing weak spots without manually bookmarking anything

**Priority:** Must

**Acceptance criteria:**

1. **Given** the Progress flag is on and, in subject S (current licence + language), my latest
   attempt at question X was incorrect
   **When** the My-mistakes pool for S is computed
   **Then** X is in the pool.

2. **Given** my latest attempt at question Y in S was correct (regardless of earlier wrong attempts)
   **When** the pool is computed
   **Then** Y is **not** in the pool — membership is driven by `questionMastery(...)`:
   not-mastered-on-latest-attempt (the inverse of F1's mastery, OQ-1 MVP rule).

3. **Given** a question Z in S I have never answered
   **When** the pool is computed
   **Then** Z is **not** in the pool (unseen ≠ wrong; the pool is "things I got wrong," not "things I
   haven't done" — those belong to weighted Quick review, Epic W).

4. **Given** I have a *Saved* bookmark on a question I answered correctly
   **When** the pool is computed
   **Then** that question is not in the pool (Saved and My-mistakes are independent surfaces; G2).

5. **Given** I switch licence or language
   **When** the pool is computed
   **Then** it reflects only the active licence+language scope (NFR-3), consistent with how
   `questionMastery(licenseId, lang, categoryId)` is already scoped (`AnswerLogRepository.kt`).

6. **Given** a question is in the pool but no longer exists in the current question bank (e.g. a
   content update removed it)
   **When** the pool is materialised into questions for a session
   **Then** the missing question is skipped without error (the pool is a set of ids resolved against
   the live bank, like `MistakeReviewQuestionStrategy` already filters by `it.id in mistakeIds`).

#### E2. Graduate a question out of the pool when mastered (Must)

**As a** student-pilot
**I want** a question to leave my mistakes pool once I get it right
**So that** the pool shrinks as I improve and stays a list of work that still needs doing

**Priority:** Must

**Acceptance criteria:**

1. **Given** question X is in the pool and I answer X correctly in any logged mode (Learning,
   ShortLearning, Exam, MistakeReview)
   **When** the pool is next computed
   **Then** X is no longer in the pool (MVP rule: latest attempt correct = graduated, §9 OQ-1).

2. **Given** question X has graduated and I later answer it incorrectly again
   **When** the pool is next computed
   **Then** X is back in the pool (membership always reflects the latest attempt; no permanent
   removal).

3. **Given** the audit's "correctly twice" rule is adopted instead (§9 OQ-1 override)
   **When** I answer a pooled question correctly once
   **Then** it **remains** in the pool until a second consecutive correct answer — explicitly out of
   the MVP unless OQ-1 is overridden; this criterion documents the delta, not MVP behaviour.

#### E3. Surface My-mistakes as a fifth mode card with a count chip (Must)

**As a** student-pilot on the mode-selection screen
**I want** a "My mistakes" card showing how many questions I currently have wrong in this subject
**So that** I can see and enter my cross-session review the same way I enter Learning, Quick review,
Exam, and Saved

**Priority:** Must

**Acceptance criteria:**

1. **Given** the Progress flag is on and the pool for the subject is non-empty
   **When** I open mode selection
   **Then** a fifth "My mistakes" card appears below the existing four, enabled, with a count chip
   equal to the pool size (mirroring the *Saved* card's `chipCount`/disabled pattern in
   `ModeSelectionScreen.kt` and `ModeSelectionViewModel`'s `favouriteCount`).

2. **Given** the pool for the subject is empty (flag on, but nothing currently wrong)
   **When** I open mode selection
   **Then** the My-mistakes card is **disabled** with a zero/empty state and hint copy explaining how
   it fills (e.g. "Questions you answer incorrectly are collected here") — same treatment the audit
   asked for on the empty Saved tile (§3).

3. **Given** the Progress flag is **off** (the default)
   **When** I open mode selection
   **Then** the My-mistakes card is **not shown at all** — the screen is byte-for-byte the pre-feature
   four-card layout (the pool reads from the flag-gated log, which is empty/inert when off; consistent
   with F1 OQ-10 and `ObserveSubjectProgressUseCase` returning empty when the flag is off).

4. **Given** the pool count is still loading
   **When** mode selection first appears
   **Then** the screen uses the existing loading/skeleton pattern and never shows a stale or wrong
   count (consistent with `ModeSelectionUiState.Loading`).

5. **Given** I switch licence or language while on/returning to mode selection
   **When** the card re-renders
   **Then** the count reflects the newly active scope (NFR-3), like the favourite count already
   reacts to `languageProvider.observeLanguage()` in `ModeSelectionViewModel`.

#### E4. Enter a MistakeReview session sourced from the persistent pool (Must)

**As a** student-pilot
**I want** tapping "My mistakes" to start a review of exactly my current pool for that subject
**So that** I drill everything I've gotten wrong across sessions in one go

**Priority:** Must

**Acceptance criteria:**

1. **Given** the My-mistakes card is enabled
   **When** I tap it
   **Then** a `MistakeReview` session starts containing exactly the current pool's questions for that
   subject (current licence + language), in question-bank order (consistent with today's
   `MistakeReviewQuestionStrategy` ordering).

2. **Given** I am in the pool-sourced MistakeReview and I answer questions
   **When** my answers are graded
   **Then** they are logged as MistakeReview answer events (F1 already logs this mode —
   `LogAnswerEventUseCase` excludes only Favourites), so correct answers graduate questions out of
   the pool (E2) and the next visit to the card shows a smaller count.

3. **Given** the existing **per-session** Review-mistakes CTA on the results screen
   **When** I finish a session and tap it
   **Then** it continues to work as today (per-session, sourced from that one `resultId`) — this
   feature **adds** the persistent-pool entry point at mode selection; it does **not** remove or change
   the results-screen CTA (audit §2.F2: "graduate the CTA to the persistent pool" is interpreted as
   *add the standing surface*, not *rip out the post-session shortcut*; see §9 OQ-5).

4. **Given** I open the My-mistakes card, then in a parallel/earlier flow the pool changes (e.g. the
   count was 12 when the screen loaded but is now 10)
   **When** the session starts
   **Then** it uses the pool as resolved at session start; a slightly stale chip count never blocks or
   crashes entry (the session materialises the pool fresh on load).

5. **Given** the pool became empty between the card rendering and entry (race)
   **When** the session would start
   **Then** it resolves to an empty/cleared state handled exactly like a fully-cleared MistakeReview
   today (the results/empty path already exists), never a crash.

---

### Epic W — Weighted Quick review

The system reweights Quick review's 10-question selection by study priority (unseen → wrong → old)
using F1's log, so wrong questions resurface until mastered — **same entry point, same screen, same
10 questions, no UI change.**

#### W1. Select Quick review questions by priority bands (Must)

**As a** student-pilot
**I want** Quick review to favour questions I've never seen and questions I keep getting wrong
**So that** my 10-question burst targets my weak spots instead of re-drilling what I know

**Priority:** Must

**Acceptance criteria:**

1. **Given** the Progress flag is on and subject S has history
   **When** Quick review builds its 10-question set
   **Then** questions are selected in this priority order until 10 are filled:
   - **Band 1 — unseen:** questions with no answer event in S (current licence + language).
   - **Band 2 — wrong:** seen questions not mastered on latest attempt (the pool of Epic E).
   - **Band 3 — old-correct:** mastered questions, oldest `lastAttemptMillis` first (spaced refresh).

2. **Given** a band has more questions than the remaining slots
   **When** that band is sampled
   **Then** the picks within the band are shuffled (so repeated Quick reviews vary), but the **band
   order is always respected** — a Band 2 question is never chosen over an available Band 1 question.
   (Within-band randomisation, across-band priority.)

3. **Given** subject S has fewer than 10 questions in total
   **When** Quick review builds its set
   **Then** it returns all available questions (≤ 10), as today (`minOf(totalCount, 10)` is already
   reflected in the card chip) — never pads with duplicates.

4. **Given** all of subject S is mastered (Bands 1 and 2 empty)
   **When** Quick review builds its set
   **Then** it returns 10 from Band 3 (oldest-mastered-first) — Quick review is never empty for a
   subject that has any questions.

5. **Given** I switch licence or language
   **When** Quick review builds its set
   **Then** banding uses the newly active scope's history (NFR-3).

#### W2. Resurface wrong questions until mastered (Must)

**As a** student-pilot
**I want** a question I got wrong to keep coming back in Quick review until I get it right
**So that** I can't accidentally leave a weak spot un-drilled

**Priority:** Must

**Acceptance criteria:**

1. **Given** I answered question X wrong (X is in Band 2)
   **When** I run Quick review repeatedly without ever answering X correctly
   **Then** X remains eligible in Band 2 and is prioritised above mastered (Band 3) questions on each
   run until handled.

2. **Given** I answer X correctly (graduation rule, §9 OQ-1)
   **When** Quick review next builds its set
   **Then** X moves to Band 3 (old-correct) and is no longer prioritised as a mistake.

3. **Given** the "correctly twice" rule is adopted (§9 OQ-1 override)
   **When** I answer X correctly once
   **Then** X stays in Band 2 until a second consecutive correct — out of MVP scope unless OQ-1 is
   overridden; documented here as the delta.

#### W3. Fall back safely with no history or flag off (Must)

**As a** brand-new student-pilot, or any user with the feature off
**I want** Quick review to keep working exactly as it does today
**So that** the smarter selection never makes the feature worse for someone with no data

**Priority:** Must

**Acceptance criteria:**

1. **Given** the Progress flag is **off** (the default)
   **When** I run Quick review
   **Then** selection is the current behaviour: 10 random questions
   (`shuffled().take(10)`) — no log read, no banding, identical to today.

2. **Given** the Progress flag is on but subject S has **zero** answer history
   **When** I run Quick review
   **Then** every question is Band 1 (unseen), so the result is 10 (shuffled) unseen questions —
   which is exactly the current random behaviour for a fresh subject, no regression.

3. **Given** the log read fails or is unavailable at selection time
   **When** Quick review builds its set
   **Then** it degrades gracefully to the random fallback (W3.1) rather than failing to start the
   session — Quick review must always return a playable set.

---

## 5. Out of scope for this feature (explicit non-goals)

These are deferred. Listing them prevents scope creep.

- **Statistics & progress dashboard / category hub / exam-readiness verdict / exam history view.**
  This is the *other* P1 in the audit ("Statistics & progress"), a separate feature set
  (audit §4, §5 step 3). This feature consumes the log; it does not build dashboards or a readiness
  indicator.
- **Heavy spaced-repetition algorithms (SM-2, Anki-style intervals, ease factors, due-date
  scheduling).** The audit asks for "**light** spaced repetition" — the three-band ordering in W1 is
  the whole of it. No interval math, no per-question scheduling state.
- **Exam crash recovery.** F1's sibling story (F1 §5 / OQ-8); a resume-state concern, untouched here.
- **Category hub redesign / promoting mode selection to a dashboard** (audit §3). We add one card to
  the existing screen; we do not restructure it.
- **"Save all wrong answers" bulk action, question search, full mock-exam-day** (audit P2).
- **Surfacing the pool on the home grid** (e.g. a mistakes-count badge per tile). The home tile shows
  F1's mastery progress; a mistakes badge is a possible later enhancement, not this feature.
- **Cross-licence credit for shared question codes.** F1 stores `questionCode` for this, but scope
  stays strictly per active licence + language (F1 OQ-9 / NFR-3); not implemented here.
- **A new "mastered twice / streak" data field**, *unless* §9 OQ-1 is resolved in favour of the
  "twice" rule — in which case it moves in scope (and is no longer free; see OQ-1).
- **Backend sync.** Local-only, like F1 (NFR-4).

---

## 6. Data — what this feature needs (already captured by F1)

This feature requires **no new stored data and no migration.** Everything it reads exists in F1's
answer-event log:

| Needed fact | Where it already lives | Used by |
|---|---|---|
| Per-question latest-attempt correctness | `QuestionMastery.isMasteredOnLatestAttempt` via `AnswerLogRepository.questionMastery(licenseId, lang, categoryId)` | E1 pool membership, E2 graduation, W1 Band 2 |
| Which questions have been seen at all | answer events exist for the question (`SubjectProgress.seenCount` / per-question mastery list) | W1 Band 1 (unseen) |
| Recency of last attempt | `QuestionMastery.lastAttemptMillis` | W1 Band 3 (old-correct ordering) |
| Per-licence + per-language scope | scope args on `questionMastery(...)` / `observeSubjectProgress(...)` | NFR-3, E1.5, E3.5, W1.5 |
| The full question set of a subject (denominator for "unseen") | `QuestionRepository.getQuestionsByCategory(...)` via `GetQuestionsUseCase` | W1 Band 1 (questions with no event), E4 materialising the pool |

> **The one gap:** the "correct twice" graduation rule (§9 OQ-1). F1's model is latest-attempt only;
> it does not record a correct-streak. The MVP rule ("correct on latest attempt") is free. "Twice"
> would require either deriving a streak from the full event history on read (the events are there —
> `AnswerEvent.timestampMillis` ordering — so it is computable without a migration, but it is more
> read work and a richer read primitive than `questionMastery(...)` exposes today) or storing a
> streak. This is the reviewer's call at the checkpoint.

---

## 7. Non-functional requirements

- **NFR-1 — Read performance across the bank.** Computing pool membership (Epic E) and Quick review
  banding (Epic W) reads `questionMastery(...)` for a subject and cross-references the subject's
  question list (largest categories are a few hundred questions; bank ~1775 across 9 categories).
  The mode-selection chip must not block the screen — it renders the existing skeleton first and
  resolves the count without a perceptible stall (testable: mode selection never blocks on the pool
  read — skeleton first, then count, like the favourite-count flow). Quick review's selection must
  complete fast enough that starting a session feels as instant as today's `shuffled().take(10)`.
- **NFR-2 — Flag-off is a true no-op.** With Progress off: no pool is computed, no fifth card is
  shown, and Quick review uses the random fallback. No log read happens when the flag is off
  (consistent with `ObserveSubjectProgressUseCase` and `LogAnswerEventUseCase` short-circuiting on
  `FeatureFlag.Progress`). No hidden data trail.
- **NFR-3 — Per-licence + per-language scoping.** Pool and banding are scoped to the active licence
  **and** language, consistent with F1 and with how sessions/favourites are keyed
  (`licenseId_lang_…`). Switching scope shows that scope's data; switching back restores the original.
- **NFR-4 — Local-only, private.** All reads are from the local F1 log; no network call, no backend
  sync, no PII leaves the device (consistent with F1 NFR-4).
- **NFR-5 — Resilient.** A corrupt/unreadable log entry or a pooled id missing from the live bank is
  skipped, not fatal (E1.6, W3.3) — consistent with F1's deserialize-and-log behaviour. Pool and
  Quick review always resolve to a valid (possibly empty/random) set, never a crash.
- **NFR-6 — Cross-platform.** Identical behaviour on Android and iOS (shared `commonMain`
  selection logic over the shared F1 log). The fifth card must be applied to both apps per
  CLAUDE.md's "UI changes apply to both" rule.
- **NFR-7 — Accessibility of the new card.** The My-mistakes card carries a content description and
  its count/empty state is conveyed to screen readers, consistent with the accessibility pass the
  audit recorded as done (§3) and the labelling of the other mode cards.

---

## 8. Prioritisation

### MoSCoW (within this feature)

**Must — the feature is pointless without these:**
- W1 Weighted Quick review band selection (unseen → wrong → old)
- W2 Resurface wrong questions until mastered
- W3 Safe fallback (flag off / no history / read failure → 10 questions, no regression)
- E1 Pool membership definition from the log
- E2 Graduation out of the pool when mastered
- E3 Fifth "My mistakes" card with count chip + disabled/hidden states
- E4 Enter pool-sourced MistakeReview, without breaking the existing results-screen CTA
- NFR-2 (flag-off no-op), NFR-3 (scoping), NFR-5 (resilience), NFR-6 (both platforms), NFR-7 (a11y)

**Should — valuable, ships without if pressured:**
- Within-band shuffling so repeated Quick reviews vary (W1.2). Functionally the feature works with a
  deterministic in-band order; randomisation is a polish that prevents monotony. Recommend keeping.
- Hint copy on the empty My-mistakes card (E3.2) beyond a bare disabled state.

**Could — first to be cut:**
- A "you've cleared all your mistakes in this subject" celebratory empty state beyond the plain
  disabled card.
- Exposing the pool count anywhere other than the mode card (e.g. echoing it on the results screen).

**Won't (this iteration) — see §5:**
- Stats dashboard, readiness verdict, exam history, category hub, SRS scheduling, the "correctly
  twice" graduation rule (unless OQ-1 is overridden), home-tile mistakes badge, cross-licence credit,
  bulk save-wrong, search, mock-exam-day, backend sync.

> Honesty check: Musts are split across two independently-shippable epics; each epic's Musts are the
> irreducible core of that epic. Weighted Quick review (Epic W) is shippable with **zero** new UI and
> is the cheapest, highest-leverage half — recommend shipping it first (§9 OQ-3, §10).

### RICE — sequencing within the feature and against the alternative

| Item | Reach | Impact | Confidence | Effort | Note |
|---|---|---|---|---|---|
| **Weighted Quick review (Epic W)** | All active users (Quick review is a core mode) | 2 (high) | 80% | low | No new screen; reuses `ShortLearningQuestionStrategy` seam + F1 read. Ship first. |
| **My-mistakes pool + card (Epic E)** | All active users with history | 2 (high) | 80% | medium | New card + state on an existing screen; pool-sourced strategy. Ship second. |
| Stats/readiness dashboard (the *other* P1) | All | 3 | 50% | high | Deferred; needs design decisions on readiness strictness (Home brief). Not this feature. |

Both halves are cheap *because* F1 already paid the data cost. Building them now, while the log read
primitive is fresh and tested, is the lowest-total-effort path (F1 §10 explicitly sequenced these as
follow-ups 1 and 2).

---

## 9. Open questions & assumptions

Each has a **recommended answer**; the feature proceeds on it unless the reviewer overrides. **OQ-1
is the load-bearing one** and the reason this doc goes to a checkpoint.

- **OQ-1 — Graduation rule: "correct on latest attempt" (cheap, what F1 supports) vs. "correct
  twice" (what the audit wishes)?** ✅ **DECIDED 2026-06-15 — reviewer chose "correct on latest
  attempt" (the MVP rule below). "Twice" is deferred as an additive, no-migration refinement.**
  *The tradeoff:* F1's `QuestionMastery` tracks **only** latest-attempt correctness
  (`isMasteredOnLatestAttempt`, `Mastery.kt`). The "correct on latest attempt" rule is therefore
  **free** — it is literally the inverse of the mastery primitive F1 already ships and tests, with no
  new data and no migration. The audit's "until answered correctly **twice**" needs a correct-streak
  the model does not have. The streak *is* derivable on read from the full append-only event history
  (the events and their timestamps exist — `AnswerEvent` — so still **no migration**), but it is a
  new, heavier read primitive (`questionMastery(...)` would need a sibling that returns streak/last-N
  outcomes) plus more read work per question. Storing a streak flag is the other option and risks
  definition drift (F1 OQ-3 deliberately avoided stored derived flags).
  *Recommendation:* **Ship the MVP on "correct on latest attempt."** It delivers the entire user
  value ("practise what I currently have wrong; stop re-seeing what I know"), is zero-cost on F1's
  data, and "twice" is an additive read-side refinement later with **no migration** if the
  product-cost data shows users graduate questions too easily. Every story above is written so the
  rule is the *only* thing that changes if "twice" is later adopted (E2.3, W2.3 document the delta).
  *DECIDED: latest-attempt rule for MVP; "twice" deferred, additive, no-migration.*

- **OQ-2 — Is the My-mistakes pool per-category or all-categories-at-once for a licence?**
  *Recommendation:* **Per-category**, surfaced on each subject's mode-selection screen, exactly like
  every other mode card (which is already per-category — `ModeSelectionViewModel(categoryId, …)`).
  This keeps it consistent with the screen it lives on, keeps the read scoped (NFR-1), and matches the
  user's mental model ("review my Meteorology mistakes"). An all-subjects "review every mistake"
  surface would need a new entry point (home) and a cross-category session the app does not have today
  — that is closer to the deferred "mock-exam-day" and is out of scope (§5).
  *Proceeding on: per-category pool, on mode selection.*

- **OQ-3 — Do the two sub-features ship together or separately?**
  *Recommendation:* **Separately, weighted Quick review (Epic W) first.** It is lower-risk (no new
  screen, pure selection-logic change behind the existing entry point), cheaper, and independently
  valuable; the audit singles it out as "dramatically more useful at zero UI cost." The My-mistakes
  card (Epic E) follows. Both are gated by the same Progress flag, so neither needs its own rollout
  switch. They share the pool definition (Epic E's "not mastered on latest attempt" = Quick review's
  Band 2), so building W first establishes that shared read, and E reuses it.
  *Proceeding on: ship Epic W, then Epic E; same Progress flag gates both.*

- **OQ-4 — Reuse the existing `FeatureFlag.Progress`, or add a new flag?**
  *Recommendation:* **Reuse `FeatureFlag.Progress`.** This feature only *reads* the log F1's Progress
  flag already gates; with the flag off the log is empty/inert, so the pool is empty and Quick review
  falls back to random anyway (W3.1, E3.3). A separate flag would let the surfaces appear while their
  data source is off — incoherent. If the reviewer wants to dark-launch these surfaces independently
  of F1's home-tile progress, a second flag (e.g. `MistakeStudy`) can gate *only the UI surfaces* (the
  fifth card + the banding switch) while still reading the Progress-gated log; flag this as the only
  reason to add one.
  *Proceeding on: reuse `FeatureFlag.Progress`; add a UI-only flag only if independent rollout is
  required.*

- **OQ-5 — Does the persistent-pool card replace the results-screen per-session Review-mistakes CTA?**
  *Recommendation:* **No — add, don't replace.** The results-screen CTA (F2, shipped) is a
  post-session shortcut ("clean up what you just missed") and is genuinely useful in that moment; the
  fifth card is a standing "review everything I've ever missed" surface. They serve different moments.
  The audit's "graduate the CTA to the persistent pool" is read as *introduce the persistent surface*,
  not *delete the shortcut*. Keeping both also de-risks Epic E (the existing flow is untouched).
  *Proceeding on: keep the results-screen CTA as-is; add the mode-selection card.*

- **OQ-6 — Should Exam answers count toward the pool / graduation, or only practice modes?**
  *Recommendation:* **Count all logged modes (Learning, ShortLearning, Exam, MistakeReview).** F1
  logs all of these (only Favourites is excluded — `LogAnswerEventUseCase`). A question you got wrong
  *on an exam* is exactly a mistake worth pooling, and getting it right on a later exam is a real
  graduation. Excluding Exam would make the pool lag reality. Mode is stored per event, so a future
  "only count practice" refinement is possible on read without a migration.
  *Proceeding on: all logged modes feed the pool and graduation.*

- **OQ-7 — Within Quick review, may a single run mix bands (e.g. 4 unseen + 4 wrong + 2 old), or
  fill strictly band-by-band?**
  *Recommendation:* **Fill strictly band-by-band** (W1: exhaust Band 1, then Band 2, then Band 3
  until 10). It is the simplest faithful reading of the audit's "unseen → wrong → old" ordering, is
  trivially testable, and naturally produces a healthy mix early in a campaign (lots of unseen) that
  shifts toward wrong/old as coverage grows. A fixed quota per band (e.g. always reserve some "wrong"
  slots) is a tunable refinement, deferred.
  *Proceeding on: strict band order, no per-band quotas.*

---

## 10. Release plan

**Iteration 1 — Weighted Quick review (Epic W). No new screens.**
- Reweight `ShortLearning` selection by unseen → wrong → old using F1's `questionMastery(...)`, with
  the random fallback for flag-off / no-history / read-failure (W1–W3). Same entry point, same 10
  questions, smarter set. Cheapest, highest-leverage, lowest-risk — ship first.
- Result: a returning user's Quick review starts targeting weak spots immediately, with zero UI work
  and no migration.

**Iteration 2 — Cross-session "My mistakes" pool + fifth mode card (Epic E).**
- Materialise the pool (= Band 2 from Iteration 1: not-mastered-on-latest-attempt, per category) and
  surface it as a fifth, count-chipped, disabled-when-empty, hidden-when-flag-off card on mode
  selection; entering it starts a MistakeReview session sourced from the pool (E1–E4). The existing
  results-screen Review-mistakes CTA is untouched.
- Result: a standing "practise everything I've ever gotten wrong" surface that shrinks as the user
  improves.

**Both iterations are gated by the existing `FeatureFlag.Progress` (OQ-4); no new rollout switch.
Neither requires a data migration (§6).**

---

## Definition of Ready check (for the Must stories)

- [x] Acceptance criteria in Given/When/Then, including empty/zero (E1.3, E3.2, W3.2), flag-off
      (E3.3, W3.1), missing-question / read-failure (E1.6, W3.3), and licence/language-switch
      (E1.5, E3.5, W1.5) unhappy paths.
- [x] Out-of-scope explicitly stated (§5); no new data / no migration pinned (§6).
- [x] **OQ-1 (graduation rule) DECIDED 2026-06-15: "correct on latest attempt"** — MVP confirmed by
      reviewer; "twice" documented as an additive, no-migration delta (E2.3, W2.3). Scope confirmed:
      both epics, Epic W first (OQ-3). OQ-2…OQ-7 proceed on the recommended answers.
- [x] Each story is a few days of work; the two sub-features are split into independently-shippable
      epics (§9 OQ-3), and Epic W ships with no UI.
