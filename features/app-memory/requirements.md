# F1 — Give the App a Memory

*Feature slug: `app-memory` · Owner: Product · Status: requirements (pre-design)*
*Source: UX audit 2026-06-11 §2.F1, §3, §4, §5 · Date: 2026-06-15*

> **Reviewer note.** This document defines WHAT and WHY, not HOW. No data-model schema, DataStore
> keys, table layout, or screen design is prescribed here — those belong to the architect and
> designer. What this document *does* commit to is: (a) the smallest shippable, user-visible slice;
> and (b) the **set of facts every answer/session must remember now** so the audit's downstream
> features land later **without a data migration**. The forward-compatibility table in §6 is the
> heart of the ask.

---

## 1. Problem — who hurts, when, and how much

The app is a polished drill trainer, but **it forgets everything the moment a session ends.**
`PersistedSessionResultStore` keeps exactly one result in a single overwrite slot; there is no
per-question answer log. Concretely:

- **The student-pilot cannot answer the only question that matters: "Am I ready for the exam?"**
  Readiness for a real ULC sitting is per-subject (75% pass mark, weakest-link). With no history,
  the app cannot say anything about where the user stands.
- **Home has zero progress signal.** A brand-new user and someone who has mastered 80% of the
  bank see an identical category grid. The Continue card only helps while a single session is
  unfinished.
- **Past exam attempts are unrecoverable.** No score history, no trend, no improvement signal.
  Finishing one session overwrites the previous result entirely.
- **"Practice everything I've ever gotten wrong" is impossible.** The mistakes pool lives inside
  one session; the new MistakeReview mode reads the single slot, so finishing a review overwrites
  the very result it came from. The cross-session pool the audit wants cannot be built.
- **An OS kill mid-exam silently loses up to ~50 minutes of work** (learning sessions persist;
  exams do not).

**Job-to-be-done:** *"When I sit down to study, help me see what I've already done and where I'm
weak, so I can spend my limited time on the subjects that will actually stop me passing — and
never lose work I've put in."*

F1 is the **architectural prerequisite** for nearly all of audit §4 (statistics & progress,
exam readiness, exam history, cross-session mistakes pool, weighted Quick review, streaks). It is
step 2 of the audit's suggested sequence. The product strategy is explicit: *the same screens we
already have (home grid, mode selection, results) become a progress system almost for free once
the app has a memory.*

---

## 2. Target users

**Primary persona — the student-pilot.** An adult studying for a ULC theory exam over several
weeks, typically on one licence at a time (PPL(A), PPL(H), SPL, or BPL), across nine subject
categories. They study in short, repeated sessions, return to the app many times, and ultimately
sit a real multi-subject exam where each subject must independently clear 75%.

Their two recurring questions, in order:
1. **"Am I ready?"** (per subject, weakest-link)
2. **"What should I do now?"** (which subject / which questions to drill next)

**Context of use:** mobile, often offline or on poor connectivity (the question bank is cached;
study happens anywhere). Sessions are frequent and accumulate over weeks, so the stored history
grows continuously and must stay bounded.

---

## 3. Goals & success metrics

F1 is primarily a **data-foundation** feature, but it ships with one concrete user-visible proof
that the memory works. Metrics are split accordingly.

### Foundation goals (measurable in-app / in tests)

| # | Goal | Success metric |
|---|------|----------------|
| G1 | Durable per-answer event log exists | After any answer is graded in a logged mode, an answer event with all required fields (§6) is retrievable after process death. Verified by test + manual relaunch. |
| G2 | Multiple session results are retained | After finishing N sessions (N ≥ 2), all N results remain retrievable; none is overwritten by a later session. |
| G3 | No regression to existing flows | 100% of existing session/result/resume tests still pass; answering a question still feels instant (see NFR-2). |
| G4 | Forward-compatible data | Every field in the §6 forward-compatibility table is captured at write time, so each named downstream feature can be built later with **zero migration**. Verified by reviewing that each "unlocks" row maps to a stored field. |

### User-visible goal (the MVP proof slice)

| # | Goal | Success metric |
|---|------|----------------|
| G5 | The user can *see* that the app now remembers their effort | A returning user sees a per-subject progress signal on the home grid that distinguishes a never-touched subject from one they have practised, and the signal updates after a session. (Exact visual is the designer's call; the *distinction must be perceivable and correct*.) |

**Explicit non-metric:** F1 does **not** commit to a readiness verdict, trend lines, streaks, or a
stats dashboard. Those are downstream features (§5 Won't / §6).

---

## 4. Epics & user stories

Three epics. Epic A and Epic B are the invisible foundation (Must). Epic C is the single visible
proof slice (Must — small). Everything else is deferred (§5).

> All acceptance criteria are written in Given/When/Then so QA can map each line 1:1 to a
> `commonTest` case. "Logged mode" = the set of modes that record answer events, resolved in
> Open Question OQ-1 (recommended: Learning, ShortLearning, Exam, MistakeReview — **not** Favourites).

---

### Epic A — Durable answer-event log

The system records every graded answer as a durable, append-only event, scoped per licence and
language, carrying enough context for all downstream features.

#### A1. Record an answer event when a question is graded (Must)

**As a** student-pilot
**I want** every answer I commit in practice to be remembered permanently
**So that** the app can later tell me my coverage, my mastery, and my mistakes across all my study

**Acceptance criteria:**

1. **Given** I am in a logged mode and have not yet answered the current question
   **When** my answer is graded (correct or incorrect)
   **Then** exactly one answer event is recorded carrying at minimum: question identity, subject
   category, active licence, active language, the mode, whether it was correct, and the wall-clock
   time it happened (full field list in §6).

2. **Given** I answered a question and the app process is then killed
   **When** I relaunch the app
   **Then** that answer event is still retrievable with all its fields intact.

3. **Given** I answer the same question again later (same or different session)
   **When** the second answer is graded
   **Then** a **new** answer event is appended; the earlier event is **not** overwritten
   (the log is append-only — see OQ-2).

4. **Given** I am in a non-logged mode (per OQ-1, e.g. Favourites)
   **When** I answer a question
   **Then** no answer event is recorded.

5. **Given** I have selected an option in Exam mode but the exam has not finished
   **When** the app records events
   **Then** Exam answer events are recorded at exam finish (when answers are graded), not on each
   tentative selection (Exam grades only at the end — this matches today's flow).

6. **Given** no licence is selected (cold-start edge)
   **When** the system would record an answer event
   **Then** no orphan event without a licence is written (this state should not be reachable from a
   live session, but the log must never contain a null-licence event).

#### A2. Read coverage and mastery per subject from the log (Must — read API only)

**As a** the home screen (on behalf of the student)
**I want** to ask "how many questions in this subject have I seen, and how many do I currently have
right?"
**So that** the grid can show a real progress signal (Epic C) and future features can reuse the same read

**Acceptance criteria:**

1. **Given** a subject in which I have answered K distinct questions at least once
   **When** coverage is computed for that subject (current licence + language)
   **Then** coverage = K distinct questions seen (not total events).

2. **Given** a subject where my most recent answer to question X was correct and to question Y was
   incorrect
   **When** mastery is computed
   **Then** question X counts as mastered and Y does not — mastery is **"correct on the user's most
   recent attempt"** (the audit's definition), computed on read from the log (see OQ-3).

3. **Given** I answered question Z wrong, then later answered Z correctly
   **When** mastery is computed
   **Then** Z counts as mastered (most-recent-attempt wins).

4. **Given** a subject I have never practised in the current licence + language
   **When** coverage and mastery are computed
   **Then** both return zero (the empty state), without error.

5. **Given** I switch to a different licence
   **When** coverage and mastery are computed
   **Then** they reflect only the active licence's events; the other licence's history is unaffected
   and reappears when I switch back (per-licence scoping — see NFR-3).

---

### Epic B — Session-history store

The system retains a list of finished session results (exam attempts in particular), replacing the
single overwrite slot, while not breaking users who already have one stored result.

#### B1. Retain multiple finished session results (Must)

**As a** student-pilot
**I want** my finished sessions — especially exam attempts — to be kept, not overwritten
**So that** I can later review past attempts and the app can show a trend

**Acceptance criteria:**

1. **Given** I finish a session and a result is stored
   **When** I finish a second, different session
   **Then** both results are independently retrievable by their ids; the first is not overwritten.

2. **Given** I open the results screen for a just-finished session
   **When** I navigate away and the result is later read again by its id
   **Then** the same result is returned (the results screen keeps working exactly as today,
   including MistakeReview launched from results).

3. **Given** I finish a MistakeReview launched from a parent result
   **When** the review's own result is stored
   **Then** the parent session's result is **still** retrievable afterwards (today it is lost — this
   regression must be fixed; see OQ-7 for whether MistakeReview results are themselves retained).

4. **Given** a stored result payload is corrupt / undeserializable
   **When** it is read
   **Then** the system skips it gracefully (returns "missing" for that id, logs the error) and other
   results remain readable — no crash.

#### B2. Migrate the existing single-slot result without breaking current users (Must)

**As an** existing user who already has one stored result
**I want** the app to keep working after the update
**So that** an in-progress upgrade never crashes or wipes my last result unexpectedly

**Acceptance criteria:**

1. **Given** the app has exactly one result stored in the legacy single-slot format
   **When** I update to the F1 build and open the app
   **Then** the app launches without crashing and the legacy result is either carried into the new
   history or discarded **gracefully** (recommended in OQ-6), never causing an error dialog or crash.

2. **Given** the legacy slot is empty (fresh install or never finished a session)
   **When** I open the F1 build
   **Then** the history is simply empty; no error.

3. **Given** the legacy result is read by a still-running results screen during/after migration
   **When** it is requested by id
   **Then** it resolves to a valid result or a clean "missing" state — never a crash.

#### B3. Bound history growth (Must)

**As the** business
**I want** stored history to stay within a sane size as the user studies for weeks
**So that** the app's local storage and read performance do not degrade over a long study campaign

**Acceptance criteria:**

1. **Given** the answer-event log and session history grow over time (bank is ~1775 questions; many
   sessions over weeks)
   **When** storage is written
   **Then** the retained data stays within a defined bound (cap by count and/or age — see OQ-5)
   such that the foundation read APIs in A2 still return within NFR-1.

2. **Given** the configured retention bound is exceeded
   **When** new data is written
   **Then** the oldest data beyond the bound is pruned **without** corrupting coverage/mastery for
   currently-relevant questions (pruning policy must preserve "most recent attempt per question" —
   see OQ-3/OQ-5).

> Note: B3's exact bound is an Open Question, but the *requirement that a bound exists and is
> enforced* is a Must — unbounded growth on a key-value store over a multi-week campaign is a real
> risk.

---

### Epic C — Visible proof: per-subject progress on the home grid (Must — smallest slice)

The audit's own suggestion for the cheapest visible proof is home-tile progress (§3.F3,
"Tile progress"). We adopt exactly that as the MVP's user-facing slice. **This is deliberately the
*only* user-visible feature in F1** — readiness, history views, stats, and the mistakes pool are
all deferred.

#### C1. Show a per-subject progress signal on the home grid (Must)

**As a** returning student-pilot
**I want** the home category grid to show how far along I am in each subject
**So that** a brand-new user and a power user no longer see the identical screen, and I get
immediate evidence the app remembers my work

**Acceptance criteria:**

1. **Given** I have practised some questions in subject S (current licence + language)
   **When** I open the home screen
   **Then** subject S's tile shows a progress signal derived from coverage and/or mastery (A2) that
   is visibly different from a never-practised subject. (The exact visual — bar, percentage, ring —
   is the designer's decision; the requirement is that the distinction is perceivable and correct.)

2. **Given** I have never practised subject T in the current licence + language
   **When** I open the home screen
   **Then** subject T's tile shows the zero/empty progress state (no error, no misleading "0%" that
   reads as failure — wording/visual is the designer's call).

3. **Given** I finish a session in subject S
   **When** I return to the home screen
   **Then** subject S's tile reflects the updated progress without needing an app restart.

4. **Given** I switch licence on the home screen
   **When** the grid re-renders
   **Then** each tile's progress reflects the newly active licence's history (per-licence scoping).

5. **Given** the progress data is still loading
   **When** the grid first appears
   **Then** tiles use the existing skeleton/placeholder pattern and never show stale or wrong
   progress (no jank, no flash of incorrect numbers).

> Scope guard for C1: F1 ships the **coverage/mastery** progress signal only. It does **not** ship a
> readiness verdict on the tile ("at exam standard"), which depends on exam-attempt aggregation and
> the readiness-strictness decisions still open in the Home redesign brief. That is a downstream
> feature (§5).

---

---

### Epic D — Rollout gating behind a feature flag (Must)

F1 ships **dark by default**, gated behind a feature flag that mirrors the existing `Explanation`
flag exactly: a `FeatureFlag` enum entry resolved by `CompositeFeatureFlagService` from Firebase
Remote Config (production) plus debug overrides (debug builds can ForceOn/ForceOff via `DebugScreen`),
**default `false`**. This lets us merge F1 and enable it remotely / per-build without a release.

#### D1. Gate the entire F1 feature behind a flag (Must)

**As the** business / release owner
**I want** the whole memory feature to sit behind a single feature flag that defaults off
**So that** F1 can be merged safely and rolled out (or rolled back) remotely, exactly like explanations

**Acceptance criteria:**

1. **Given** the F1 flag is **off** (the default)
   **When** I use the app
   **Then** behaviour is **identical to today**: the home grid shows no per-subject progress signal,
   and the app behaves as the pre-F1 build from the user's point of view (no observable F1 surface).

2. **Given** the F1 flag is **on**
   **When** I use the app
   **Then** the full F1 behaviour is active: answer events are logged, multiple session results are
   retained, and the home grid shows per-subject progress (Epic C).

3. **Given** I am on a debug build
   **When** I open the debug screen
   **Then** I can ForceOn / ForceOff the F1 flag and the app reflects the change (consistent with the
   existing `Explanation` override behaviour).

4. **Given** the flag is toggled at runtime (debug override or a remote-config refresh)
   **When** the home screen is visible
   **Then** the grid reacts to the flag without requiring an app restart (the flag is **observed**,
   not only read once), consistent with how the app already observes flags.

5. **Given** the flag has been **off** and is then turned **on**
   **When** F1 becomes active
   **Then** there is simply no prior history to show (empty/zero state); turning the flag on does not
   crash or require migration. (Whether answer logging should run even while the *visible* surface is
   off is resolved in OQ-10.)

> The exact `FeatureFlag` enum name and remote-config key are the architect's call (suggested:
> a single `Progress`/`AppMemory`-style flag, key `*_enabled`, default `false`); the requirement is
> *one* flag gates *all* of F1, defaulting off, debug-overridable, observed reactively.

---

## 5. Out of scope for F1 (explicit non-goals)

These are the audit's downstream features. They are **Won't (this iteration)**, but the F1 data
model **must anticipate them** (§6). Listing them here prevents scope creep while making the
forward-compatibility contract explicit.

- **Exam-readiness indicator / verdict** ("X of 9 subjects at exam standard"). Depends on
  exam-attempt aggregation + unresolved readiness-strictness/wording decisions (Home brief).
- **Category hub / dashboard** (progress ring, last exam score, mistakes count, pass mark on the
  exam card). New screen work; deferred.
- **Exam history view** (list of attempts: date, score, pass/fail, time; trend line).
- **Cross-session "My mistakes" pool** + fifth mode card on mode selection.
- **Weighted Quick review** (unseen → wrong → old ordering; light spaced repetition until correct
  twice). Today Quick review is `shuffled().take(10)`.
- **Study streak / daily goal.** (Also previously rejected for Home specifically — engagement ≠
  readiness.)
- **Exam crash recovery.** Important and listed alongside F1 in the audit's step 2, but it is a
  *resume-state* concern, not an answer-log concern. **Recommended: track as a sibling story, not
  inside F1**, to keep F1's scope tight (see OQ-8). F1 must not block it.
- **Question search, "full mock exam day", bulk "save all wrong answers".** Audit P2; untouched.
- **Question explanations.** Separate workstream; out of scope by owner decision.
- **Backend sync of progress / cross-device history.** F1 is local-only (no progress backend exists
  today; Firestore holds read-only content only — see NFR-4).

---

## 6. Forward-compatibility — the data the log MUST capture now

This is the core of the request: design F1's data so the deferred features in §5 ship later
**without a migration.** Each row names a fact the foundation must store **now** and which future
feature it unlocks. (Field *names* are illustrative — the architect owns the schema; the
*information* is the requirement.)

### Per answer event (Epic A)

| Captured fact | Why / what it unlocks later | Required in F1? |
|---|---|---|
| **Question identity** (question id) | Coverage (distinct seen), mastery, mistakes pool, weighted review | **Yes** |
| **Question code** (the placard code) | Question search by code; cross-license shared-code analytics (codes are shared across licences) | **Yes** (cheap to store, hard to backfill) |
| **Subject category** | Per-category coverage/mastery, readiness per subject, category hub | **Yes** |
| **Licence** (active license id) | Per-licence scoping; correct progress on licence switch | **Yes** |
| **Language** (active lang code) | Bank size differs per language; existing app keys all user data by licence+lang | **Yes** |
| **Mode** (Learning / ShortLearning / Exam / MistakeReview) | Readiness derived primarily from **Exam** attempts; weighting review by source; excluding review-churn from mastery if desired | **Yes** |
| **Correct / incorrect** | Mastery, mistakes pool, scores | **Yes** |
| **Selected option id + correct option id** | Reconstruct a past attempt for review; future per-distractor analytics | **Yes** (already in `AnswerRecord`) |
| **Timestamp** (wall-clock) | "old" ordering in weighted review, streaks, trend over time, "most recent attempt" for mastery, retention/pruning | **Yes** |

> **Mastery is computed on read** from these events ("correct on most recent attempt"), not stored
> as a separate flag — so the definition can evolve (e.g. "correct twice") without a migration
> (OQ-3).

### Per finished session result (Epic B)

The current `SessionResult` already holds mode, categoryId, totals, per-question results, exam
duration, and exam pass count. F1 must additionally ensure each retained result carries:

| Captured fact | Why / what it unlocks later | Required in F1? |
|---|---|---|
| **Finished-at timestamp** | Exam history list & trend line, "last exam score", recency | **Yes** (not stored today) |
| **Licence + language** | Scope exam history per licence; correct grouping | **Yes** (not stored today) |
| **Mode** (already present) | Separate exam attempts from practice in history | Already present |
| **Score + pass count + verdict inputs** (already present) | Pass/fail history, readiness rolling average | Already present (`correctCount`, `examPassCount`, `ExamDuration`) |
| **Stable result id** | Address a specific past attempt; link review-from-history | **Yes** (store already returns an id; it must persist, not be ephemeral) |

> Two missing-today fields stand out as the highest-leverage adds: **timestamp** and
> **licence+language** on every answer event *and* every session result. Without them, exam history,
> trends, streaks, per-licence scoping, and "old" ordering all require a later migration. Capturing
> them now is the single most important forward-compatibility decision in F1.

---

## 7. Non-functional requirements

- **NFR-1 — Read performance.** Computing per-subject coverage and mastery for the home grid
  (Epic C) over a full multi-week log must complete fast enough that the grid is not blocked: the
  grid renders its skeleton immediately and resolves real progress without a perceptible stall on a
  mid-range device. (Concrete bound to be confirmed in design; the testable requirement is "grid
  never blocks on history read — skeleton first, then data.")
- **NFR-2 — Write must not jank the answer.** Recording an answer event must not delay or block the
  answer-grading interaction. Answering a question stays as responsive as today; event persistence
  happens off the interaction's critical path. (Testable: answer UI state updates without awaiting
  the write.)
- **NFR-3 — Per-licence + per-language scoping.** All coverage, mastery, history, and the home
  signal are scoped to the active licence **and** language, consistent with how sessions and
  favourites are already keyed (`licenseId_lang_…`). Switching licence/language shows that
  scope's data; switching back restores the original.
- **NFR-4 — Local-only, private.** All F1 data is stored locally on device. No network call, no
  backend sync, no PII leaves the device. (Verified: the app's only backend — Firestore — serves
  read-only content collections; there is no user-progress collection. F1 introduces none.)
- **NFR-5 — Resilient to corruption.** A single corrupt stored entry must never crash the app or
  poison the whole log/history; it is skipped and logged (consistent with existing
  deserialize-and-log-on-failure behaviour).
- **NFR-6 — Cross-platform.** Behaviour is identical on Android and iOS (shared `commonMain`
  logic over the shared DataStore); no platform-specific divergence in what is remembered.
- **NFR-7 — Bounded storage.** History/log size is capped (B3 / OQ-5) so a long campaign does not
  grow storage or read time unboundedly.

---

## 8. Prioritisation

### MoSCoW (within F1)

**Must — the release is pointless without these:**
- A1 Record durable answer events with all §6 fields
- A2 Read coverage & mastery from the log (read API)
- B1 Retain multiple session results (replace single slot)
- B2 Safe migration from the legacy single-slot store
- B3 Bounded history growth (a bound exists and is enforced)
- C1 Per-subject progress on the home grid (the one visible proof)
- D1 Gate all of F1 behind a feature flag, default off, debug-overridable, observed reactively
- NFR-2 (no answer jank), NFR-3 (per-licence scoping), NFR-4 (local-only), NFR-5 (corruption-safe)

**Should — valuable, ships without if pressured:**
- Carry the legacy single-slot result *forward* into history rather than discarding it (B2 currently
  recommends graceful discard; preserving it is a "should").
- Expose the answer-log read API in the shape the **next project** (cross-session mistakes pool +
  weighted Quick review) will consume — i.e. retain the per-question mastery read now. **Reviewer
  decision 2026-06-15:** keep *only* that primitive; trim broader analytics / exam-history-list reads
  to hold F1 to strict minimal scope (they are additive later with zero data migration).

**Could — first to be cut:**
- A lightweight, non-blocking telemetry/log of write failures beyond what NFR-5 requires.
- A debug/dev affordance to inspect or clear the local history (developer quality-of-life only).

**Won't (this iteration) — but data model must anticipate (see §5 / §6):**
- Readiness indicator, category hub, exam history view, cross-session mistakes pool, weighted Quick
  review, streaks/daily goal, exam crash recovery, search, mock-exam-day, bulk save-wrong.

> Honesty check: Musts span the foundation + one small visible slice. The visible surface is
> deliberately a *single* tile signal; the bulk of the work is invisible plumbing that exists to be
> reused. This is the correct shape for a foundation feature.

### RICE — why F1 before the features it unlocks

| Item | Reach | Impact | Confidence | Effort | Score | Note |
|---|---|---|---|---|---|---|
| **F1 (this)** | All active users | 3 (massive — unblocks the headline feature set) | 80% | medium | high | Prerequisite multiplier; nothing in §4 ships without it |
| Readiness indicator | All | 3 | 50% | high | — | Blocked on F1 **and** open strictness/wording decisions |
| Cross-session mistakes pool | All | 2 | 80% | medium | — | Blocked on F1 |
| Weighted Quick review | All | 2 | 80% | low | — | Blocked on F1; very cheap *after* F1 |

F1's value is overwhelmingly its **enabling** role; building any §4 feature first would force the
same log to be built anyway, plus a migration. Sequencing F1 first is the lowest-total-effort path.

---

## 9. Open questions & assumptions

> **Reviewer decision (2026-06-15): requirements accepted as-is.** MVP scope approved (foundation +
> single home-tile progress proof, no new screens); exam crash recovery stays a separate sibling
> story; the home tile signal is mastery-led; and **all** recommended answers below (OQ-1…OQ-9) are
> **confirmed**. One addition was requested and is now a Must: **the whole feature must be gated by a
> feature toggle, mirroring the explanations toggle** — captured as Epic D / story D1, with its own
> gate-scope question OQ-10 below.

Each has a **recommended answer** so the reviewer can simply confirm or override. F1 proceeds on the
recommended answer unless told otherwise. **All OQs below are confirmed** per the reviewer decision
above.

- **OQ-1 — Which modes record answer events?**
  *Recommendation:* Log **Learning, ShortLearning (Quick review), Exam, MistakeReview**. Do **not**
  log **Favourites/Saved** (it is a curated revisit surface, not fresh practice; logging it would
  inflate coverage with questions the user already chose to bookmark). Mode is stored on every
  event, so this can be revisited per-feature on read without a migration. *Proceeding on: log all
  modes except Favourites.*

- **OQ-2 — Re-answering: append a new event or update mastery in place?**
  *Recommendation:* **Append a new event** (append-only log). Mastery is derived on read as
  "correct on most recent attempt." This keeps history for trends/streaks and makes the definition
  changeable later. *Proceeding on: append-only.*

- **OQ-3 — Is mastery stored or computed on read?**
  *Recommendation:* **Computed on read** as "correct on the user's most recent attempt for that
  question" (the audit's definition). Storing a derived flag risks drift and locks the definition.
  Pruning (OQ-5) must preserve the most-recent attempt per still-relevant question. *Proceeding on:
  computed on read, most-recent-attempt-wins.*

- **OQ-4 — Does the home tile show coverage, mastery, or both?**
  *Recommendation:* Drive the C1 signal from **mastery** ("how much do I currently have right"),
  because that is closest to the user's readiness mindset; coverage is also captured and available
  to the designer if they want a two-part indicator. Final visual is the designer's. *Proceeding on:
  mastery-led signal, coverage available.*

- **OQ-5 — Retention bound (count and/or age)?**
  *Recommendation:* Cap **session-result history** at a recent-N per licence+category (e.g. keep the
  last ~20 results per subject, sufficient for "recent exam average" and a short trend) and bound
  the **answer-event log** by keeping, at minimum, the **latest attempt per question** plus a recent
  window, so mastery and coverage stay exact while old churn is pruned. Exact numbers to confirm in
  design. The firm requirement: a bound exists and pruning never corrupts current mastery/coverage.
  *Proceeding on: bounded, most-recent-attempt-per-question always preserved.*

- **OQ-6 — Migration: preserve or discard the one legacy result?**
  *Recommendation:* **Discard gracefully** (the single legacy result is at most one transient
  just-finished result; preserving it adds migration risk for negligible value). It is a "should" to
  carry it forward if cheap. The Must is: no crash, no data wipe surprise. *Proceeding on: graceful
  discard, no crash.*

- **OQ-7 — Are MistakeReview results themselves retained in history?**
  *Recommendation:* MistakeReview **answer events** are logged (so a correct re-answer updates
  mastery and the mistakes pool empties), but its **session result** does **not** need to appear in
  the user-facing exam/practice history list later (it is a derived drill, not a standalone attempt).
  Critically, B1.3 requires that finishing a MistakeReview no longer destroys its parent result.
  *Proceeding on: log review answers; review results not surfaced as standalone history; parent
  result preserved.*

- **OQ-8 — Is exam crash recovery part of F1?**
  *Recommendation:* **No — split it into a sibling story.** It is a resume-state concern (persist
  exam session + wall-clock deadline + offer resume), not an answer-log concern, and the audit notes
  the hard part (wall-clock timer) is already done. Keeping it out keeps F1 tight; F1 must not block
  it. *Proceeding on: tracked separately, not in F1 scope.*

- **OQ-9 — Does coverage/mastery account for the bank being shared by question code across
  licences?**
  *Recommendation:* For F1, scope strictly **per active licence + language** (consistent with how
  sessions and favourites are keyed today). We *store* the question code so cross-licence credit
  ("you answered this shared question under another licence") can be added later without migration,
  but F1 does **not** implement cross-licence credit. *Proceeding on: per-licence scope now, code
  stored for later.*

- **OQ-10 — Does the feature flag gate the whole feature (data + UI) or only the visible surface?**
  *Recommendation:* The flag gates the **whole feature**: when off, no answer events are logged, the
  history store is not engaged, and the home grid shows no progress — behaviour is byte-for-byte the
  pre-F1 experience (D1.1). Gating only the UI while silently logging in the background would build a
  hidden data trail before the feature is sanctioned and complicate the "behaves exactly as today"
  guarantee. Trade-off: when the flag is first switched on there is no back-history (D1.5) — which is
  the correct, expected behaviour for a dark-launched feature. *Proceeding on: flag gates all of F1
  (data + UI); off = today's behaviour exactly.* **(Confirmed.)**

---

## 10. Release plan

**MVP slice (single shippable release):**
- Epic A (A1 record, A2 read) + Epic B (B1 multi-result, B2 migration, B3 bound) + Epic C1 (home-tile
  progress) + the Must NFRs.
- Result: the app remembers every practice answer and every finished session, scoped per licence;
  the home grid proves it with a per-subject progress signal; existing users upgrade without a crash;
  storage stays bounded. **No new screens.**

**Follow-up iterations (each now a thin layer on F1's log — no migration):**
1. **Weighted Quick review** — reorder Quick review by unseen → wrong → old using the log. Cheapest,
   highest-leverage first follow-up.
2. **Cross-session mistakes pool + fifth mode card** — graduate the Review-mistakes CTA to the
   persistent pool.
3. **Category hub + exam history view** — last exam score, attempts list, pass mark on the exam card,
   per-subject trend.
4. **Exam-readiness indicator** — rolling exam-average vs. 75% pass mark, weakest-link verdict
   (pending readiness-strictness/wording decisions from the Home brief).
5. **Exam crash recovery** (sibling story, OQ-8) and remaining P2 items as appetite allows.

---

## Definition of Ready check (for the Must stories)

- [x] Acceptance criteria in Given/When/Then, including empty/zero, corrupt, offline-by-design
      (local-only), and licence-switch unhappy paths.
- [x] Out-of-scope explicitly stated (§5) and forward-compatibility pinned (§6).
- [x] Blocking open questions resolved — **confirmed by reviewer 2026-06-15** (all of §9 OQ-1…OQ-10
      accepted on the recommended answers; feature-flag gating added as Must story D1).
- [x] Each story is a few days of work; oversized foundation work split into A/B read vs. write and
      a separate visible slice (C).
