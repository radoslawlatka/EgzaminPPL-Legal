# Content Voting — Solution Spec

*Feature slug: `content-voting` · Owner: Product + Architecture · Status: Phase 1 client implemented (flag-gated, dark-launched) on `feature/content-voting`*
*Date: 2026-06-17 · Phase 1 target: up/down voting on explanations*
*Rev 2026-06-18 — UX audit folded in: minimalist (icon-only) control; both surfaces tagged with a `surface` field; clarity-framed microcopy (§2, §6.6, §6.11). Phase 1 client built: domain/data/UI/DI + rules extended; compiles (commonMain/commonTest/commonUiTest), unit tests green, Koin graph verified by emulator launch. `content_voting` flag defaults off. Remaining: deploy `firestore.rules`; pipeline fast-follow (§7).*
*Rev 2026-06-20 — TTL backstop **dropped** (§6.6/§7.5): an inactivity timer would delete still-valid signal and the data is pseudonymous, so retention is bound by the §7.4 revision-cleanup + in-app withdrawal. No `updatedAt`/`expireAt` written. Privacy §6 (EN+PL) softened to a general "kept only as long as needed" + user withdrawal — it no longer promises revision-based deletion, so §7.4 is recommended hygiene to bound growth, not a hard pre-flag policy gate. No TTL claim anywhere. The speculative `appVersion` and now-orphaned `updatedAt` optional fields were removed entirely from the schema, the `firestore.rules` whitelist, and the `firestore.indexes.json` exemptions — vote schema is now exactly `target/code/lang/rev/voterId/vote/surface`.*

> **Scope note.** This document captures the agreed design for content up/down voting. It commits to
> (a) the smallest shippable slice — voting on explanations — and (b) a data model that is
> *revision-ready* from day one, so the regeneration feedback loop and old-revision cleanup land
> later **without a vote-key migration**. The revisioned-feedback lifecycle (§7) is pipeline/team-side
> work that ships after the Phase 1 client.

---

## 1. Objective

To drive the best possible content quality, let users up/down vote content. The instrument is a
quality **signal for the content team**, not a social feature — it tells us *which AI-authored
explanations are wrong, unclear, or unhelpful* so we can regenerate them, closing the loop with the
existing `scripts/explanation_*.py` pipeline.

**Phase 1 = the objective only:** a thumbs-up / thumbs-down control on **explanations**. Everything
else is deferred (§8).

---

## 2. Key product decisions

1. **Votes are a private signal, not social proof.** The UI only ever shows *the user's own* vote.
   No aggregate counts are shown to users. Rationale: an exam-prep audience must trust content by
   correctness, not popularity; public counts invite gaming; and a private signal removes any need
   for live distributed counters — which is what keeps the architecture free of Cloud Functions.
2. **Target = explanations only (Phase 1).** Explanations are AI-authored and quality-variable, and
   we own a pipeline to act on the feedback. Questions (sourced from official ULC PDFs) are a later,
   one-word widening of the same mechanism (§8), not a re-architecture.
3. **Feedback is per content *revision*.** A vote belongs to the specific revision of the content the
   user saw, so regenerating an explanation starts a clean feedback bucket instead of inheriting the
   old version's downvotes (§7).
4. **Revision bump is an editorial decision, not a content hash.** A trivial fix (missing comma)
   preserves votes; only a material change resets them (§7.2).
5. **Minimalist control: thumbs only, no prompt text.** Two icon buttons, no "was this helpful?"
   sentence and no "thanks" line. Rationale (UX audit): the explanation's job is to *teach*; the
   rating is secondary and must look it. A full-sentence prompt firing on every wrong answer, across
   two surfaces, breeds prompt-fatigue and banner-blindness; icon-only has no layout shift and ages
   better. The semantics ("rate the explanation's clarity") live in the accessibility
   `contentDescription`, not on screen (§6.11).
6. **Both surfaces, tagged by `surface`.** The control renders on the live `QuestionScreen` (high
   volume, but the user just failed — a hot, biased moment) *and* the review `QuestionPreviewScreen`
   (lower volume, calm, deliberate). Each vote records `surface: "live" | "review"` so the content
   team can segment or discard hot-state votes. Because both surfaces share the same deterministic doc
   id, a calm review vote **overwrites** the same user's earlier reflex live vote — considered
   judgment supersedes the reflex, for free. *Note:* explanations are, by current product design, the
   wrong-answer debrief — they never render for a correct answer on **any** surface, so the voter
   population is always "users who got this question wrong." Widening to correct-answerers is a
   separate product change, out of scope.
7. **Rate clarity, not helpfulness.** The audience is definitionally people the explanation hasn't
   (yet) helped, so "was this helpful?" mis-frames it. Copy/`contentDescription` ask whether the
   explanation was *clear / understandable* ("Czy wyjaśnienie było zrozumiałe?").

---

## 3. Scope

**In scope (Phase 1, client):**
- Minimalist thumbs-up / thumbs-down control (icon-only, no prompt/thanks text — §6.11) on the
  explanation card, on **both** the live `QuestionScreen` (after a wrong answer) and the review
  `QuestionPreviewScreen`. The control only renders in the card's `Content` state — never over a
  loading spinner or an error.
- Each vote tagged with `surface: "live" | "review"`.
- Set / clear / switch the vote; own vote persists locally and shows instantly (optimistic, offline).
- One device counts once per explanation per revision (idempotent).
- Vote mirrored to Firestore as a private per-device signal.
- Accessibility: each thumb carries a stable, clarity-framed `contentDescription` plus a `selected`
  semantics flag; the selected state is conveyed visually by the icon's outline→fill and a `primary`
  tint (no container — the fill is the non-color cue) plus a light haptic, not by any on-screen text
  (§6.11).
- Firestore security rules permitting the write (§6.7).
- Gated behind a `content_voting` feature flag.
- Revision-ready vote key (`rev` segment present from day one).

**Deferred (post-Phase-1):**
- ❌ Downvote reason chips
- ❌ Voting on questions (error reports)
- ❌ On-screen prompt / "thanks" microcopy / snackbar (deliberately omitted — decision §2.5)
- ❌ Voting on correct-answer explanations (would require showing explanations to correct-answerers —
  a separate product change, §2.6)
- ❌ Pipeline: `rev` stamping on write, per-rev report, old-rev cleanup (§7)
- ❌ Rules unit-test harness (needs Node/Jest tooling — weigh later)

---

## 4. User stories (Phase 1)

**US-1 — Rate an explanation (Must)**
> As a learner reading an explanation, I can tap thumbs-up or thumbs-down to signal whether the
> explanation was clear.

Acceptance:
- The control appears in the explanation card's `Content` state — on the live `QuestionScreen` after
  a wrong answer, and in the review `QuestionPreviewScreen`. It does **not** appear over the card's
  loading or error state.
- Two icon buttons only — no prompt sentence, no "thanks" line, no container. The selected thumb is a
  filled icon in the `primary` tint; a screen reader announces its label and selected/unselected state.
- Up/down selects the choice; tapping the lit thumb again clears it; tapping the opposite switches it.
- My choice persists across restarts and is visible offline immediately (optimistic).
- One device counts once per explanation per revision, regardless of repeated taps or which surface I
  vote from; a later vote on one surface overwrites my earlier vote on the other.
- A remote write failure never blocks the optimistic UI (local state holds; logged to Crashlytics).

**US-2 — Surface worst explanations to the team (Must, internal, pipeline-side)**
> As the content owner, I can rank explanations by downvotes for the *current revision* to prioritise
> regeneration.

Acceptance: a query/script over `contentVotes` filtered to each explanation's live `rev` outputs
ranked offending `code`s. (Lands with §7 pipeline work.)

---

## 5. Architecture — constraint adherence

The constraint: **introduce no new technology/infrastructure until absolutely necessary.**

| Need | Existing asset reused | New? |
|---|---|---|
| Backend store | Firestore (gitlive KMP wrapper) + persistent cache | No |
| Abuse protection | Firebase App Check (Play Integrity), already installed | No |
| Offline write queue | Firestore SDK offline persistence (already on) | No |
| Own-vote local state | DataStore Preferences (same pattern as Favourites) | No |
| Rollout control | `CompositeFeatureFlagService` + Remote Config | No |
| Feedback loop | `scripts/explanation_*.py` already read/write Firestore | No |
| Old-revision cleanup | Server SDK (service account, already used by scripts) | No |
| Voter identity | one locally-generated UUID | new code, no new infra |

The only genuinely new code: one collection, one repository + datasource, a DataStore mirror, a
feature flag, a small Compose control, and the rules block. No new infra column.

---

## 6. Architecture — detail

### 6.1 Identity without auth
There is **no Firebase Auth** in the project, and we don't add it. For idempotency we need a stable
per-install id: a random UUID persisted in DataStore (`voter_id`), exposed via `VoterIdProvider`
(a Koin `single`). Generated in `commonMain` — zero new dependency. Survives restarts; resets on
uninstall/clear-data (acceptable for a quality signal).

**The UUID must be created atomically** to avoid a latent double-vote race:

```kotlin
// WRONG — two concurrent first-votes both read null, both generate, one clobbers the other,
// briefly yielding two ids for one install → two vote docs.
val id = store.data.first()[KEY] ?: uuid().also { store.edit { it[KEY] = uuid() } }

// CORRECT — edit is serialized by DataStore; the second caller sees the first's value and no-ops.
val result = store.edit { p -> if (p[KEY] == null) p[KEY] = uuid() }
return result[KEY]!!
```

### 6.2 Content identity + revision
Explanations are stored license-agnostically at `/explanations/{code}/langs/{lang}`. A vote is keyed
by **`code` + `lang` + `rev`** (target fixed to `explanation`), *not* by question `id` or license —
so a shared question's explanation is voted on once regardless of which license the user studies.

`rev` is a per-explanation content revision carried on the explanation doc and tagged onto every
vote (§7). It is present in the key from Phase 1 even before the pipeline emits real values;
bootstrap value `"1"` until the pipeline starts stamping revisions.

> **Pipeline contract:** `rev` must be written to Firestore as a **string** (`"2"`), never a number.
> The client decodes the explanation doc into `ExplanationDocDto.rev: String`; a numeric `rev` makes
> gitlive throw on decode and fails the *entire* explanation fetch for that doc. Whatever format §11
> settles on (monotonic int vs. short hash), serialise the value as a string. Keep it free of the `:`
> delimiter too — it is the separator in the vote doc id `target:code:lang:rev:voterId` (§6.6) and in
> the local mirror value `"{rev}:{direction}"` (§6.4).

### 6.3 Data model (domain)
```kotlin
enum class VoteDirection { UP, DOWN }            // absence = not voted

interface ContentVoteRepository {
    fun observeVote(code: String, lang: String, rev: String): Flow<VoteDirection?>  // own vote
    suspend fun submitVote(code: String, lang: String, rev: String, direction: VoteDirection?)
}
```
Use cases (match the `factoryOf` style): `ObserveContentVoteUseCase`, `SubmitContentVoteUseCase`.
`Explanation` gains a `rev: String` field, sourced from the doc (§6.6).

### 6.4 Local store (UI source of truth)
DataStore key `vote_{code}_{lang}` → value `"{rev}:{direction}"`. `observeVote` returns the
direction **only if the stored rev equals the current rev**, else neutral. Mirrors
`FavouriteRepositoryImpl`. This delivers the "control resets on a new revision" UX (§7).

### 6.5 Remote store (the signal)
`ContentVoteRepositoryImpl` writes:
1. **Local DataStore** (awaited) — drives the UI; instant, offline, free.
2. **`FirestoreContentVoteDataSource`** (fire-and-forget) — rides the SDK's offline queue; failures
   never block the UI. `submitVote` with a direction **sets** the doc; with `null` (cleared)
   **deletes** it.

### 6.6 Firestore schema — flat, idempotent, dedup by construction
Collection **`contentVotes`** (top-level). **Deterministic doc id** =
`"{target}:{code}:{lang}:{rev}:{voterId}"`.

```
contentVotes/{deterministicId}
  target:     "explanation"
  code:       "PL0421"
  lang:       "pl"
  rev:        "1"
  voterId:    "<uuid>"
  vote:       -1 | 1           // down | up
  surface:    "live" | "review"  // where the (last) vote was cast — for signal segmentation
```
A device always writes/overwrites its own single doc per (explanation, revision): re-tapping is a
no-op overwrite, switching overwrites in place, clearing deletes it. **One device = one vote**, with
no counter, no transaction, no read-before-write. `surface` is **not** part of the doc id, so a vote
cast in review overwrites the same user's earlier live vote in place and records `"review"` as the
final origin (the calm, considered vote wins — §2.6). It is metadata for the team's offline
segmentation, never read by the client.

### 6.7 Security rules (first client-writable collection)
Every other collection is `write: if false` (ingested via the server SDK, which bypasses rules).
`contentVotes` is the first that clients write. Already added to `firestore.rules`:

```
match /contentVotes/{voteId} {
  allow read: if false;                         // private — no scraping, no social proof

  allow create, update: if isFromVerifiedApp()  // App Check gates the write (no auth exists)
    && request.resource.data.keys().hasOnly(
         ['target','code','lang','rev','voterId','vote','surface'])
    && request.resource.data.keys().hasAll(['target','code','lang','rev','voterId','vote'])
    && request.resource.data.target == 'explanation'
    && request.resource.data.vote in [-1, 1]
    && request.resource.data.code is string
    && request.resource.data.lang is string
    && request.resource.data.rev is string
    && request.resource.data.voterId is string
    && (!('surface' in request.resource.data.keys())
         || request.resource.data.surface in ['live', 'review'])
    && voteId == request.resource.data.target + ':' + request.resource.data.code + ':'
         + request.resource.data.lang + ':' + request.resource.data.rev + ':'
         + request.resource.data.voterId;

  allow delete: if isFromVerifiedApp();         // clear a vote; id is unguessable
}
```

> **Status (rev 2026-06-18):** the rule above is now committed to `firestore.rules` with the `rev`
> segment in the id binding and `surface` whitelisted as optional — the gap flagged in the previous
> revision is closed. The speculative `appVersion`/`updatedAt` fields were **removed** (2026-06-20) —
> neither is whitelisted, indexed, or written; the vote schema is exactly
> `target/code/lang/rev/voterId/vote/surface`. The rule still needs **deploying** via the Firebase CLI
> now that `firebase.json` is committed — see §7.7.

Validation guarantees: no client reads; App Check on every write; doc id bound to payload (one
install → one deterministic doc); `vote ∈ {-1,1}`; field whitelist; Phase-1 lockdown to
`target == 'explanation'`.

### 6.8 Offline
Local DataStore drives the UI with zero network dependency; the Firestore mirror rides the SDK's
existing offline write queue and flushes on reconnect. Fits the current
`feature/offline-resilience-phase1` theme with no special-casing.

### 6.9 Integration touchpoints
- **DI (`AppModule.kt`):** `singleOf(::VoterIdProvider)`; `FirestoreContentVoteDataSource` bound to
  `ContentVoteDataSource`; `ContentVoteRepositoryImpl` bound to `ContentVoteRepository`; two
  `factoryOf` use cases. Add `CONTENT_VOTES = "contentVotes"` to `FirestoreCollections`.
- **ViewModels:** `QuestionViewModel` and `QuestionPreviewViewModel` expose
  `voteState: StateFlow<VoteDirection?>` and `onVote(direction)` — same shape as the existing
  favourite wiring already in those VMs. Each VM passes its own `surface` (`"live"` / `"review"`) into
  `SubmitContentVoteUseCase`; the UI never sees `surface`.
- **UI:** a new `VoteBar` composable in `ExplanationCard`'s footer — see the full spec in §6.11.
  `ExplanationCard` gains a nullable vote slot (state + `onVote`); it renders `VoteBar` only in the
  `Content` branch, so the footer never shows over the loading/error states. It lives in `commonMain`
  Compose, so Android and iOS get it from one implementation (satisfies the "both platforms" rule).
- **Config:** `content_voting` flag via the existing `FeatureFlagService` — controlled rollout +
  kill switch.

### 6.10 Idempotency & race analysis (summary)
- **Ballot stuffing by a normal user:** prevented — deterministic doc id, one id per install.
- **Classic voting race (concurrent counter increment):** does not exist — no shared counter; votes
  are per-voter docs aggregated offline.
- **Rapid taps same device:** Firestore applies a single client's writes to one doc in order →
  last-tap-wins; the atomic DataStore mirror is the UI truth regardless.
- **voterId creation race:** closed by the atomic `edit` in §6.1.
- **Honest limits (no auth):** reinstall / multi-device / forged UUIDs can produce extra votes.
  Acceptable for a quality signal; escalation lever = Firebase **Anonymous Auth** + rule
  `request.auth.uid == voterId`, added only if abuse metrics justify it.

### 6.11 Vote control (`VoteBar`) — UI spec

A minimalist, icon-only control (decision §2.5). It sits right-aligned in `ExplanationCard`'s footer.
**No container and no divider** — the enclosing card already separates the footer from the explanation
blocks, so grouping is done with the card's existing spacing, not a hairline (a graphic-design audit
found a full-bleed divider over two small right-aligned icons unbalanced, and an internal divider
inside an already-contained surface over-structured). Lives in `commonMain` → one implementation for
Android + iOS.

```
        explanation blocks …
                                  (card spacing only — no divider)
                          👍   👎     ← neutral: outlined, onSurfaceVariant
                         [👍]  👎     ← up selected: filled icon, primary tint (no container)
                          👍  [👎]    ← down selected: filled icon, primary tint
```

**Components & states**
- Two `IconButton`s, `Icons.Outlined.ThumbUp/ThumbDown` ↔ `Icons.Filled.ThumbUp/ThumbDown`.
- Neutral = outlined icon, `onSurfaceVariant` tint. Selected = **filled** icon, `primary` tint —
  **no container**. State is carried by the icon's outline→fill (a shape/luminance cue that survives
  grayscale / color-blindness) *and* the tint, so the non-color requirement (audit m3/M4) still holds
  without a disc. Never red, for up or down (audit m3).
- Glyph is **22dp** (quieter than the 24dp default) to read as secondary to the content; the touch
  target stays the full `IconButton` ≥48dp.
- At most one selected at a time.
- No prompt text, no count, no "thanks" line (§2.5). The only feedback is the icon state + haptic.

**Interaction**
- Tap unselected → select that direction (optimistic; local mirror flips instantly).
- Tap the selected thumb → clear (deletes the vote doc).
- Tap the opposite thumb → switch.
- A light haptic tick on every state change confirms the tap registered (the substitute for the
  removed text confirmation).

**Layout / touch**
- Each button keeps the ≥48×48dp `IconButton` touch target even though the glyph is 22dp.
- The row is nudged with `offset(x = 8.dp)` to cancel the trailing `IconButton`'s internal end inset,
  so the last thumb optically aligns with the card's content edge.
- Text-stable: the outline→fill swap is the same glyph size, so nothing resizes/jumps on selection
  (audit M3).
- Rendered only in the card's `Content` state — gated by the caller, never over loading/error
  (audit M2).

**Accessibility (load-bearing for an icon-only control — audit M4)**
- Each thumb has a **stable**, clarity-framed `contentDescription` (§2.7): *"Oceń to wyjaśnienie jako
  zrozumiałe"* / *"…jako niezrozumiałe"* (PL + default string resources). The description does **not**
  swap on selection — that avoids churn and a description-rewrite.
- Selected state is exposed separately via the `selected` semantics flag on each button, so
  TalkBack/VoiceOver announce selected/not-selected on top of the stable label.
- Because there is no visible confirmation text, that state announcement **is** the confirmation for
  non-sighted users.

**As built:** two `IconButton`s, glyph `outlined ↔ filled` at 22dp, `onSurfaceVariant ↔ primary` tint,
no container, no divider, row `offset(x = 8.dp)`, `LocalHapticFeedback` tick on toggle,
`Modifier.semantics { selected }`. **Test tags:** `UiTestTags.Vote.UP`, `UiTestTags.Vote.DOWN`.

---

## 7. Revisioned feedback lifecycle (pipeline/team-side)

### 7.1 Why
The explanation doc is **overwritten in place** on regeneration. Without a revision dimension, a
fixed explanation inherits its predecessor's downvotes and looks broken — the first regeneration
corrupts the very signal it's meant to produce. Folding `rev` into the vote key gives each revision a
clean bucket while retaining the old one for a before/after story.

### 7.2 `rev` is an editorial bump, preserve-by-default — NOT a content hash
A hash answers *"did the bytes change?"*; the question that should reset votes is *"did the meaning
change enough that old feedback is invalid?"* — only a human can answer that. A missing comma changes
a hash but not the meaning. So `rev` bumps **only when the editor declares the change material**:

| Change | rev | Votes |
|---|---|---|
| Comma / typo / whitespace | unchanged | carried over ✅ |
| Reworded, same meaning | editor's call (usually unchanged) | usually carried over |
| Wrong formula / source / rewritten answer | **bumped** | reset to clean slate ✅ |

The architecture is unchanged by this choice — only the *policy producing `rev`* differs.

### 7.3 Pipeline flag
`explanation_*.py` update path, default off:
- `explanation_update.py CODE` → content updated, **rev preserved** (the comma case).
- `explanation_update.py CODE --new-revision` → **rev bumped**, feedback reset (the rewrite case).

The safe common case is zero-effort; resetting feedback is deliberate. Failure modes are asymmetric
and favor preserve-by-default: forgetting to bump lingers briefly (recoverable); auto-hash would
reset on every comma (permanent signal loss).

Optional later guardrail: keep a mechanical `contentHash` on the doc **only** for pipeline
idempotency/audit (skip no-op uploads), never for the vote key; have the tool *suggest* a bump
heuristically (diff touches a `Formula`/`Source`/`Image` block → suggest `--new-revision`) with human
override.

### 7.4 Old-revision cleanup
On a bump: **archive the aggregate, then delete the raw rows.**
- **Keep** a one-line summary per closed rev — `{rev, up, down, net, totalVotes, closedAt}` — as a
  `revisionHistory` array on the explanation doc (or a `/explanations/{code}/langs/{lang}/revStats/{rev}`
  subdoc). This preserves the "v1: 38 down → v2: 1 down" proof.
- **Delete** the per-voter `contentVotes` docs for the old rev — they'd otherwise accumulate forever
  on every regeneration (unbounded growth + stale per-install data).

Mechanism — the existing server SDK in the bump tool (no Cloud Functions). Order:
1. Aggregate old-rev votes → write summary to `revisionHistory`.
2. Write new explanation content with the new `rev`.
3. Batch-delete `contentVotes where code == CODE and lang == L and rev == oldRev` (≤500/batch, loop).

**Race-safe:** v1 and v2 votes have different doc ids, so deleting `rev == v1` never touches a live
v2 vote; new voting and cleanup don't contend. One stray case — a stale-cached user voting v1 *after*
cleanup — resurrects a single orphan doc; with the TTL backstop dropped (§7.5) it is removed by the
next cleanup run for that code rather than by a timer (negligible leakage).

### 7.5 Retention bound & index
- **No TTL.** A time-based Firestore TTL was considered and **dropped** (2026-06-20): an inactivity
  timer would delete still-valid signal on an explanation nobody happened to re-vote recently, and the
  data is pseudonymous (random per-install `voterId`, non-identifying), so content-lifecycle retention
  is both sufficient and better aligned to purpose. **Retention is bound by the §7.4 revision-cleanup**
  (old-rev votes archived + batch-deleted on a material bump) plus in-app per-vote withdrawal (§6.5).
  Consequence: votes for a *never-revised* explanation persist as long as that content is live —
  acceptable for a non-identifying quality signal. The §7.4 cleanup bounds growth and honours
  data-minimisation; since the privacy policy no longer promises revision-based deletion (§6 was
  softened to "kept only as long as needed" + in-app withdrawal), shipping §7.4 is recommended hygiene
  rather than a hard policy gate. No `updatedAt` / `expireAt` field is written (§6.6).
- Indexes live in `firestore.indexes.json` (committed; deploy via the Firebase CLI — §7.7):
  - One **composite index** `contentVotes (code, lang, rev, vote)`. It serves the cleanup query and
    the count *via its prefix* `(code, lang, rev)`, and lets `sum('vote')` run against the index
    (§7.6). One index covers everything in the core stats + cleanup path.
  - **Single-field index exemptions** for `target` and `voterId` — fields the team never queries on.
    Firestore auto-indexes *every* field (asc+desc) by default; on a high-volume votes collection those
    unused single-field indexes are wasted storage and add write latency. Exempting them is the one
    concrete write-side cost fix.
- Rules are unchanged: cleanup is server-SDK only; clients never delete another rev's votes.

### 7.6 Statistics queries (aggregation) — the cost-safe recipe
Votes are a private signal, so stats are computed **offline by the team** (server SDK / console), not
in the app — never read the collection client-side. The one rule that matters for cost: **use
aggregation queries, never fetch-and-count.**

Fetching every vote doc to count them bills **1 read per document** — a 5,000-vote explanation costs
5,000 reads each time you check it; the whole 2,487-explanation catalog runs into six figures per pass.
Instead, Firestore's server-side aggregations bill **~1 read per 1,000 index entries scanned**, so the
same 5,000 votes cost ~5 reads — a ~1000× reduction.

Up/down for one explanation revision = **two aggregations** over `where code==X, lang==Y, rev==Z`:
- `total = count()`
- `net   = sum('vote')`   *(up = +1, down = −1, so net = up − down)*
- then **up = (total + net) / 2**, **down = (total − net) / 2**

Neither filters on `vote`, so the single composite index `(code, lang, rev, vote)` serves both (the
equality filter rides its `(code, lang, rev)` prefix; `sum` reads `vote` from the index). Ranking the
catalog by downvotes = run this pair per `(code, lang, rev)` → ~2 reads each → ~5k reads for the whole
catalog: a cheap periodic job.

**Surface segmentation** (live vs review, §2.6): add `surface==` to the filter. That needs an extra
composite index `contentVotes (code, lang, rev, surface, vote)` — left out of the committed
`firestore.indexes.json` to keep default index cost down; add it when the team actually starts
segmenting.

**When this is *not* the right shape:** showing **live counts to users**. Aggregating on read (even
cheaply) doesn't fit real-time display — that needs a maintained rollup doc, which reintroduces
counter contention / a Cloud Function and is correctly out of scope for a private signal (§10).

### 7.7 Deploying rules + indexes (Firebase CLI)
Config is committed at repo root: `firebase.json` (wires the two), `firestore.rules`,
`firestore.indexes.json`, `.firebaserc` (default project `egzamin-ppl`).

```bash
# one-time
npm install -g firebase-tools        # or: brew install firebase-cli
firebase login                        # run as `! firebase login` in this session for an interactive login

# from the repo root (where firebase.json lives)
firebase deploy --only firestore:rules     # instant
firebase deploy --only firestore:indexes   # kicks off an index build (minutes → hours on backfill)
# or both: firebase deploy --only firestore

firebase firestore:indexes                 # inspect deployed indexes / build status
```

Notes:
- The CLI reads `.firebaserc` for the project; override with `--project egzamin-ppl` if needed.
- Index **builds are async** — the deploy returns immediately; aggregation queries that need the new
  index error until the build finishes. Watch status in the console (Firestore → Indexes) or via the
  command above.
- Single-field **exemptions** deploy together with indexes; applying them drops the auto-created
  single-field indexes for those fields.
- TTL is **not used** — it was dropped in favour of the §7.4 revision-cleanup (§7.5); there is no TTL
  policy to configure.
- These config files are not in `.gitignore`; commit them so the rules/indexes are versioned with the
  code that depends on them.

---

## 8. Build list & phasing

**Phase 1 (client) — the objective — ✅ implemented on `feature/content-voting`:**
- ✅ Domain: `VoteDirection` + `VoteSurface`; `ContentVoteRepository`; `ObserveContentVoteUseCase`,
  `SubmitContentVoteUseCase` (takes a `surface` arg).
- ✅ Data: atomic `VoterIdProvider`; `ContentVoteRepositoryImpl` (DataStore mirror + remote set/delete,
  writes `surface`); `FirestoreContentVoteDataSource`; `CONTENT_VOTES` constant; `rev` on
  `Explanation` + DTO.
- ✅ Rules: `contentVotes` block extended in `firestore.rules` with the `rev` id segment and the
  optional `surface` field (§6.7).
- ✅ Firestore config committed: `firebase.json`, `firestore.indexes.json` (composite
  `(code, lang, rev, vote)` + exemptions for `target`/`voterId`), `.firebaserc`
  → project `egzamin-ppl`. **Deploy still pending** — run `firebase deploy --only firestore` (§7.7).
- ✅ UI: minimalist icon-only `VoteBar` (§6.11) in `ExplanationCard`'s footer, gated to the `Content`
  state; wired into `QuestionViewModel` (`surface = LIVE`) + `QuestionPreviewViewModel`
  (`surface = REVIEW`); selected = filled icon + `primary` tint (no container/divider) + haptic; stable PL/default
  `contentDescription` + `selected` semantics; `UiTestTags.Vote.*`.
- ✅ Config: `content_voting` feature flag (defaults off → dark launch).
- ✅ Tests: `FakeContentVoteRepository`/`FakeContentVoteDataSource`; `ContentVoteRepositoryImplTest`
  (set/switch/clear, `surface`+`vote` mapping, rev-mismatch→neutral, remote-failure swallowed);
  `VoterIdProviderTest` (stability, persistence, N-parallel→one id); `QuestionViewModelTest` updated.
  Verified: `commonMain`/`commonTest`/`commonUiTest` compile, host unit tests green, **emulator launch**
  → Koin graph resolves, no crash.
- ⏳ Not yet: dedicated `VoteBar` UI test (absent in loading/error; toggle/clear) — deferred to QA pass;
  visual check with the flag enabled.

**Fast-follow (pipeline/team):**
- `rev` stamping on explanation write; editorial `--new-revision` bump policy.
- Old-rev archive + batch-delete (the enforced retention bound — no TTL); `(code, lang, rev)` index.
- Per-rev downvote report feeding `explanation_*.py` regeneration.

**Later (Could):**
- Downvote reason chips. Voting on questions (`target == 'question'`). Undo snackbar.

**Won't (now):** public counts, comments/free text, Cloud-Functions aggregation, Firebase Auth.

---

## 9. Success metrics
- **Coverage:** % of viewed explanations receiving ≥1 vote.
- **Signal:** downvote ratio per explanation (current rev); count crossing a "needs review" threshold.
- **Loop:** explanations regenerated per month *because of* votes, and the downvote-ratio delta after.
- **Health:** vote write error rate; App Check rejection rate (abuse indicator).

---

## 10. Risks & escalation levers ("until absolutely necessary")
| Risk | Mitigation now | Escalate to (only on evidence) |
|---|---|---|
| Vote stuffing / forged voterIds | App Check + deterministic doc id | Firebase Anonymous Auth (`auth.uid == voterId`) |
| Want live counts in-app | Out of scope by design | Cloud Functions counter / scheduled aggregation |
| Aggregation outgrows ad-hoc query | Server-SDK script over the flat collection | Firestore aggregation queries / scheduled export |
| Old-rev rows accumulate | Bump-time archive + batch delete (server SDK) | — |

Each lever stays within Firebase and is pulled only on metric evidence.

---

## 11. Open decisions

*Resolved in rev 2026-06-18:* control style (minimalist icon-only, §2.5/§6.11), surface coverage
(both, tagged via `surface`, §2.6), microcopy framing (clarity not helpfulness, §2.7).

Still open:
- Final visual polish of `VoteBar` — haptic intensity per platform, optional micro-motion on the
  outline→fill toggle (the structure/treatment is fixed in §6.11; this is the implementation finish).
- `rev` value format once the pipeline stamps it (monotonic int vs. short content-hash *as the bump
  value* — but bump policy stays editorial regardless, §7.2).
- Whether question voting is ever wanted (would flip `target` validation in the rules).
- Whether to keep the live (`surface == "live"`) votes in scoring once we have data, or treat them as
  a secondary signal if they prove noisier than review votes (§2.6).
