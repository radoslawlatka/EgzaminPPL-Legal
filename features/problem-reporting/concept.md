# Report a Problem — High-Level Concept

Status: **concept approved for UI drafting** — 2026-07-07. Nothing implemented.
Roles consulted: Product Owner (taxonomy, triage workflow, MVP cut) and Software
Architect (storage options, cost model, rules/schema sketch). Codebase facts verified
against `development`.

## 1. Summary

Users can report problems they find in the app or in the content (wrong "correct"
answer, invalid explanation). The recommended design is a near-clone of the merged
**content-voting** feature:

- **Content reports** are written to a new top-level, read-denied, client-writable
  Firestore collection **`problemReports`**, gated by App Check + strict rules
  validation, identified by the existing anonymous per-install UUID, dark-launched
  behind a new Remote Config flag.
- **App bugs / general feedback** go through a **Settings → "Send feedback" `mailto:`**
  escape hatch (crashes are already auto-captured by Crashlytics). No app-bug storage
  in Firestore for the MVP; the schema leaves the door open.
- The maintainer reviews reports with a new **Admin-SDK Python CLI**
  (`scripts/report_review.py`) — grouped by question `code`, joined with question text,
  resolved by annotation write-back. No dashboard, no Cloud Functions, no Blaze.

Infra cost: **zero marginal** — stays on the Spark plan (alpha report volume is orders
of magnitude below the 20k writes/day free tier; 1 GiB ≈ ~1M report docs).

## 2. Why (product rationale)

- **Content trust is the product.** A wrong answer key actively teaches the student the
  wrong thing and can cost them the real ULC exam — the highest-harm defect class.
- Alpha content is young and AI-assisted; defects are *expected*, and the person best
  positioned to catch each one is the student hitting that exact question in context.
- Content voting captures only ambient sentiment (private thumbs, no reason, no text).
  A report is a different intent: an **actionable defect claim**. Same blueprint,
  separate channel.
- Goals: turn engaged alpha users into a distributed content-QA net at ~zero cost; give
  the solo maintainer a deduped, code-grouped, prioritizable defect stream.
- **Non-goals:** support desk / ticketing, two-way dialogue, public bug tracker,
  real-time alerting, admin dashboard, anything requiring Blaze or user accounts.

## 3. The three questions

### 3.1 Where are reports stored? (options ranked)

| # | Option | Verdict |
|---|--------|---------|
| 1 | **Firestore `problemReports` collection** (clone of `contentVotes`) | **RECOMMENDED.** Zero new infra (Spark-safe), offline queue flushes on reconnect (students study offline), structured & queryable for triage, pseudonymous (no PII), full iOS parity via GitLive SDK, proven abuse gate (App Check + rules validation). |
| 2 | Firestore + periodic digest (GitHub Actions / local cron running the review script) | Same core as #1; adopt later only if *remembering to run the CLI* becomes the bottleneck. Never Cloud Functions. |
| 3 | `mailto:` prefilled email | Zero infra but: exposes the user's real email address (unavoidable PII), no offline queue, unstructured (manual inbox triage), fails silently when no mail client is configured. **Kept only as the Settings escape hatch for general/app feedback**, where a reply channel is actually useful. |
| 4 | Google Form (prefilled URL) | Needs live connectivity (fatal for offline studiers), adds Google Forms as a new GDPR processor, data disconnected from the content pipeline. Skip. |
| 5 | GitHub Issues from the client | **Non-starter** — requires embedding a write token in a distributed binary. GitHub is viable only as a later maintainer-side bridge script on top of #1 (token stays in Actions secrets / local machine). |

### 3.2 How does the maintainer review reports?

Pull-based, batched (~weekly, no SLA) — same posture as vote review. New CLI
`scripts/report_review.py` using the established Admin SDK + `service-account.json`
pattern (Admin SDK bypasses rules, so `read: false` doesn't block it):

- List open reports (**`status` absent or ≠ `resolved`**), **grouped by `code`**,
  sorted by distinct-reporter count — the worst questions float up.
- **Join question text + stored correct answer** (from Firestore `/questions/...`) so a
  `wrong_answer` report can be adjudicated in place against the ULC source.
- **Explanation reports self-clean:** reports carry the explanation `rev`; when a fix
  bumps the rev, older reports show as stale and are auto-filtered. `wrong_answer` /
  `question_error` reports have no rev to bump — resolved manually via annotation.
- **Mark reviewed:** CLI writes a `status` field (`resolved` / `dismissed`) via Admin
  SDK. The client rules key-whitelist keeps the client schema locked while the server
  annotates freely. Preferred over a local state file (survives machines) and over
  deleting docs (keeps history). Note: a client re-submit overwrites the doc and wipes
  `status` — **a re-report naturally reopens a resolved item**, which is exactly the
  right signal ("you marked it fixed but it's still wrong").
- **Retention hygiene:** periodic Admin-SDK delete of resolved reports older than N
  days (mirrors content-voting's cleanup).

Per-type action: `wrong_answer` → verify vs ULC, fix key/membership (top priority);
`bad_explanation` → re-author + bump `rev`; `question_error` → fix ingestion;
mailto feedback → normal inbox.

There is **no feedback loop to the reporter** (no accounts) — accepted limitation. The
only user-facing close is the in-app "Thanks, reported" confirmation; the silent payoff
is the content getting fixed.

### 3.3 What does a report include?

Principle: content-report context identifies *which content*; environment data is noise
there and is deliberately **not** collected.

**Auto-captured (content report):**

| Field | Why |
|-------|-----|
| `code` | THE join key — stable ULC content key, same question across licenses |
| `categoryId` | fix-path routing |
| `license`, `lang` | context (which membership map / language variant) |
| `rev` | scopes the report to an explanation revision → self-cleaning triage |
| `reason` | the triage lever (see §4) |
| `appVersion` | rules out already-fixed client rendering issues |
| `reporterId` | anonymous per-install UUID (existing `voter_id`) — dedup only |
| `createdAt` | recency (rules-pinned to `request.time`) |

Explicitly **not** captured: OS, device model, platform (environment-independent
defects), and **no contact email** — it would be unused PII (no reply loop exists),
heavier GDPR duty, and friction. A user who wants to be reachable uses the Settings
mailto (consent-based self-disclosure).

**User-provided:** reason — one tap, **required**; free text — **optional** (the
auto-context already pinpoints question + rev), capped at **500 chars**, enforced both
client-side and in rules via `note.size()`.

## 4. Report taxonomy

Two lanes, two entry points:

**Content lane** — on the question/explanation surface, 4 reasons: three mapping 1:1 to
a fix path plus a catch-all. Listed in display order — ULC sources the answer keys, so
`wrong_answer` is the least likely defect and sits last before the catch-all:

1. `question_error` — "There's an error in the question" (OCR/typo/missing text →
   fix ingestion)
2. `bad_explanation` — "The explanation is wrong or unclear" (→ re-author, bump `rev`)
3. `wrong_answer` — "The answer marked as correct is wrong" (highest severity → verify
   vs ULC, fix key/membership)
4. `other` — "Something else" (hand-triaged catch-all; the note carries the substance)

**App/general lane** — Settings → "Send feedback" `mailto:` with prefilled subject/body
(app version, platform). No in-app app-bug category picker: Crashlytics already
captures crashes with stacks; non-crash bugs are prose and benefit from a reply channel,
which email has and Firestore doesn't.

Deferred reason: `broken_media` — add only when explanation media ships. The taxonomy is
**baked into the doc id**, so adding reasons is cheap but renaming/removing is not —
lock it with the UI draft.

## 5. Architecture sketch (for the implementation phase)

Copy the `contentVotes` stack:

- **Doc id (deterministic):** `reporterId:code:lang:rev:reason` (`'_'` placeholder for a
  missing rev). `lang` is in the id because a report is language-specific (a bad PL explanation
  is a distinct defect from a bad EN one), so the two must not collide onto one doc. One install
  holds at most one report per (question, language, revision, reason) — free dedup, bounded abuse
  blast radius, idempotent re-submits, last-write-wins.
- **DTO:** flat payload `{type:'content', reason, code, categoryId, license, lang, rev,
  note, appVersion, reporterId, createdAt}` — `type` is a constant for now (mirrors
  votes' `target:'explanation'`), reserved so an app lane could move into the same
  collection later without a new collection.
- **Rules:** clone the `contentVotes` block — App Check gate, `hasOnly`/`hasAll` key
  whitelist, enum validation (`reason in [...]`), `note.size() <= 500`,
  `createdAt == request.time`, doc-id-to-payload binding. `read: false`,
  `delete: false`. **Harden every field**, not just the note: type-pin each field
  (`is string`; `createdAt` pinned via `== request.time`) so nested maps/lists
  ("JSON in JSON") and non-string payloads are rejected, and length-cap each string
  (`code`/`license`/`lang`/`rev`/`appVersion`/`reporterId` all bounded) so a conforming
  document is ~1 KB — abnormally large (multi-MB) reports are unwritable, with
  Firestore's 1 MiB/doc platform ceiling as backstop. The key whitelist means
  attachments are impossible by construction (no field can carry one). See
  functional-design.md §6 for the layered threat model.
- **Identity:** promote `VoterIdProvider` → shared **`InstallIdProvider`** (neutral
  package, e.g. `data/identity/`), keeping the DataStore key `voter_id` so existing
  installs keep their identity. Rename/move, not a behavior change.
- **Layers:** `ProblemReport` domain model + `ProblemReportRepository` interface +
  `SubmitProblemReportUseCase` → `ProblemReportRepositoryImpl` →
  `FirestoreProblemReportDataSource` (GitLive, best-effort write, offline queue). Do
  NOT fold into the vote repository. Optional DataStore set of submitted ids for an
  "already reported" UI cue (the collection is read-denied, so no read-back).
- **Flags:** two independent Remote Config toggles in `FeatureFlag.kt`, one per lane —
  `ProblemReporting("problem_reporting", default = false)` gates the content-report
  write path (EP1), `ContactUs("contact_us", default = false)` gates the Settings
  mailto row (EP2). Ship dark; each lane can be enabled or killed on its own.
- **Kill switches:** per-lane toggle off = soft kill (hides that lane's UI only);
  redeploying `firestore.rules` with `allow write: if false` = hard kill (stops all
  report writes, including from already-open sessions).

## 6. Abuse, privacy, risks

- **Enforceable without server code:** per-install slot cap via deterministic id
  (≤ #questions × #reasons docs per install), per-field size and type caps (§5 — bounds
  a document to ~1 KB, rejects nested/malformed payloads and any extra key that could
  smuggle an attachment), App Check. **Not enforceable:** per-IP/time throttling.
  Residual risk: a determined actor can fill their slot space with size-capped garbage
  they cannot read back — acceptable at alpha; junk is filtered at triage.
- **Injection surface:** none by design — report text is never parsed, executed,
  rendered as HTML, or interpolated into queries (Firestore is not SQL). The one
  consumer, `scripts/report_review.py`, must treat `note` as opaque text and
  strip/escape control and ANSI sequences before printing to a terminal; if a
  GitHub-issue bridge is added later, markdown-escape there as well.
- **iOS App Check gap:** iOS has no App Check installer today, so `request.app != null`
  is not actually enforced there — **identical to the existing `contentVotes`
  exposure**, not a new risk. Ship consistent with votes; fixing the gap benefits both.
- **Free-text PII:** users may paste emails/names. Mitigations: microcopy ("we can't
  reply — please don't include personal data"), 500-char cap, never displayed back,
  retention cleanup. **Privacy policy (EN+PL) must be updated before flag-on**, same
  gate as voting.
- **Spark quota:** non-issue at alpha (~1 KB docs; ~1M to reach 1 GiB).

## 7. MVP cut (MoSCoW)

**Must**
- Report entry point on the question/explanation surface (both live and review/preview
  surfaces — a user who answered *correctly* can still spot a wrong key).
- 4 content reasons (3 fixed + `other` catch-all) + optional capped free text; one-tap
  reason required.
- Auto-captured context per §3.3; write to `problemReports` per §5.
- Reuse install UUID (`InstallIdProvider`) — no new identity system.
- In-app confirmation; no reply.
- **`scripts/report_review.py`** (group-by-code, question-text join, rev-staleness
  filter, status write-back) — without the review path the reports are write-only and
  worthless.
- Two Remote Config toggles, dark launch: `problem_reporting` (content lane) and
  `contact_us` (Settings mailto lane), independently switchable.
- Settings `mailto:` escape hatch for app/general feedback.
- Privacy policy update (EN+PL) before flag-on.

**Should** — CLI count/sort aggregation view; "already reported" UI cue.

**Could** — GitHub-issue bridge / Actions digest on top of the CLI; `DeviceInfoProvider`
expect/actual (only if an in-app app-bug lane is ever added).

**Won't (explicit defers)** — screenshots/attachments, status lifecycle in the client,
reply-to-reporter, rate limiting beyond the above, admin dashboard, in-app app-bug
category picker, `broken_media` reason (until media ships).

## 8. Open product questions (for Radek)

1. **App-lane home:** MVP routes app bugs to `mailto:` only (recommended — leanest,
   reply-capable, Crashlytics covers crashes). Alternative: fold an `app` type with
   free text into `problemReports` (anonymous + offline-queued, but un-triageable blobs
   and more privacy surface). Schema reserves the option either way.
2. **Taxonomy final call:** 3 content reasons as proposed, or collapse to 2 (drop
   `question_error`)? 3 recommended — each maps to a distinct fix path.
3. **Relationship to voting:** keep report and vote as separate controls (recommended —
   report is higher-intent), or let a down-vote prompt "want to tell us what's wrong?"
4. **Free-text storage sign-off:** confirm storing capped, never-displayed free text
   under the voting-style legitimate-interest basis (privacy copy + store data-safety
   labels before flag-on).

## 9. Success measures (alpha = qualitative-leaning)

- Actionability rate (% of reports leading to a real content fix).
- Signal concentration (do codes with ≥2 distinct reporters correlate with genuine
  defects?).
- Time-to-content-fix for confirmed `wrong_answer` reports — the one number that
  matters (days, not weeks).
- Volume/week as a health check only (near-zero → discoverability problem; flood →
  abuse or systemic content issue).

## 10. References

- Precedent spec: `docs/content-voting.md` (privacy §6, retention §7.4, review-script
  intent).
- Blueprint code: `firestore.rules` (`contentVotes` block),
  `data/contentvote/VoterIdProvider.kt`, `data/source/dto/ContentVoteDto.kt`,
  `domain/featureflag/FeatureFlag.kt`, `scripts/` (Admin SDK bootstrap pattern).
- Context available on the question screens today: `question.code`, `categoryId`,
  explanation `rev`, `LicenseProvider`, `LanguageProvider`, `AppVersionProvider`.
