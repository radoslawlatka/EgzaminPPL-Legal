# Report a Problem — Functional Design (client UX)

Product-owner functional spec for the user-facing reporting flows. Builds on
[concept.md](concept.md) (storage, schema, review pipeline — all unchanged here).
Scope: entry points, flows, states, copy. Not covered: visual design tokens,
implementation plan.

## 1. Decisions locked (2026-07-07, Radek)

| Concept open question | Decision |
|---|---|
| §8.1 App-bug/feedback lane | **Settings `mailto:` only.** No in-app app-bug form; Firestore stays content-only. |
| §8.2 Taxonomy | **3 reasons** (`wrong_answer`, `bad_explanation`, `question_error`) — locked; baked into the doc id. *Amended 2026-07-07 (copy audit): + `other` catch-all; adding reasons is the cheap direction (§4 of the concept).* |
| §8.3 Vote ↔ report | **Independent controls.** No down-vote nudge, no coupling. |
| New: entry-point style | **Icon, as minimalistic as possible** — matches the icon-only `VoteBar` language. |
| New: feature toggles | **Two independent remote toggles**, one per lane: `problem_reporting` (content reports) and `contact_us` (Settings contact row). See §5. |
| New: payload hardening | Rules-level size/type/shape enforcement — no multi-MB reports, no attachments, no nested/malformed payloads. See §6. |
| New (PO review 2026-07-07): reports per question | **Multiple allowed, bounded.** Up to one report per reason — the doc id caps one install to ≤4 tiny docs per (question, revision); the same reason overwrites idempotently. The flag is re-openable and the re-opened sheet marks reasons already filed (local cue). One-and-done was rejected: a question can carry two distinct defects, and the doc-id cap already removes the spam risk one-and-done would guard against. No in-app withdraw and `other` collapsing distinct issues into one doc are accepted alpha limitations. |
| §8.4 Free-text privacy sign-off | **Still open** — gate before flag-on, does not block build. |

## 2. Entry points

### EP1 — Flag icon in the explanation footer (content reports)

A quiet outlined-flag `IconButton` in the same footer row as the vote thumbs,
**start-aligned** (thumbs stay end-aligned). Same visual language as `VoteBar`
(`ui/components/explanation/VoteBar.kt`): 22 dp icon in a 48 dp target,
`onSurfaceVariant` tint, no label, outlined→filled swap once reported,
`LongPress` haptic on action.

```
│  Wyjaśnienie                        │
│  Ciśnienie standardowe wg ISA …     │
│                                     │
│  [⚑]                    [👍] [👎]  │
└─────────────────────────────────────┘
```

Spatial separation from the thumbs is deliberate: report is a higher-intent,
"negative-path" action — keeping it away from the vote pair avoids mis-taps and
keeps the thumbs' rate-this-explanation grouping clean.

The footer renders inside `ExplanationCard`, so EP1 automatically appears on
**both existing surfaces**:

- **Live learning** — `QuestionScreen` (explanation appears after a wrong answer).
- **Review** — `QuestionPreviewScreen` via `SessionResultsScreen` (explanation
  appears for every outcome: correct, wrong, skipped).

Coverage argument: a user who *disagrees with the key* always gets the
"incorrect" verdict → explanation → flag, live. A user who answered correctly
but spotted a typo reports from the results review, where every question is one
tap away. This is why no additional entry point (top-bar action, overflow menu)
is needed — consistent with the minimalism requirement.

### EP2 — Settings → "Send feedback" row (app bugs + general feedback)

New `SettingsRow` in the **About** section, above the App-version row. Icon
`Icons.Outlined.MailOutline` (or `Feedback`), standard `AccentIconChip`
treatment. Tap opens the OS email composer via `LocalUriHandler` with a
`mailto:` URL prefilled with subject and body (see §4).

### Non-entry points (deliberate)

- **Live exam** — no report UI. The key isn't revealed mid-exam, so there is
  nothing to report against; `ExplanationCard` never renders there anyway.
  Post-exam review (EP1) covers it.
- **Home / category screens** — rejected; a content report without a concrete
  question attached is an un-triageable blob (that's what EP2 email is for).
- **Correct answer in live learning** — no flag (no explanation card renders).
  Accepted: covered by review, and the wrong-key case always surfaces as
  incorrect. Revisit only if triage shows typo reports aren't arriving.

## 3. Flow A — content report (EP1)

**Happy path: 3 taps** (flag → reason → send). No navigation; a modal sheet
over the current screen. Built on the shared `AppBottomSheet`.

1. **Tap flag** → `AppBottomSheet` opens: title "Report a problem"
   (`SheetTitle`), then:
   - **4 reason rows** — `AppBottomSheetRow` style, `Role.RadioButton` with a
     leading radio (select-then-confirm, not an instant-commit picker). Unlike
     the pickers, selecting does **not** dismiss — it enables Send. A reason
     already filed for this revision carries a quiet trailing "Reported" badge.
   - **Note field** — optional, single `OutlinedTextField`, placeholder
     "Add details (optional)", 2–4 lines, hard 500-char cap in
     `onValueChange`, counter in `supportingText` shown from 400 chars.
   - **Privacy microcopy** — one muted line: "We can't reply to reports.
     Please don't include personal data."
   - **Send button** — full-width, disabled until a reason is selected.
2. **Tap Send** → haptic; fire-and-forget `SubmitProblemReportUseCase` (auto
   context per concept §3.3: code, categoryId, license, lang, rev, appVersion,
   reporterId, createdAt); sheet dismisses **immediately**; flag animates
   outlined→filled.
3. **Confirmation = filled flag + a transient snackbar** ("Thanks for your
   report" / "Dziękujemy za zgłoszenie"), raised above the bottom bar via the
   scaffold's snackbar slot — the sheet dismisses in the same frame, so the
   snackbar acknowledges a change the filled flag alone is easy to miss.
   Filled-flag contentDescription also announces the reported state.

**Offline:** UX is byte-for-byte identical — the write sits in Firestore's
offline queue and flushes on reconnect. No spinner, no error state, no retry UI
(best-effort, same contract as votes).

**Re-report:** tapping the filled flag reopens the sheet. Reasons this install
already filed for the on-screen revision carry a quiet "Reported"/"Zgłoszono"
badge (from the local cue — the collection is read-denied), so the sheet shows
what's been reported without pre-selecting anything. An already-reported reason
stays selectable: re-sending it overwrites the existing doc (deterministic id →
idempotent; wipes maintainer `status` = intentional reopen). A different reason
creates a second slot — allowed, the flag stays filled. There is no in-app
withdraw (a rev regeneration clears the cue), and `other` collapses distinct
free-text issues into one doc (latest note wins) — both accepted for alpha.

**Dismiss** (swipe down / back / scrim tap) without Send → no write, no state
change.

**Reported-state cue:** a DataStore set of submitted reasons keyed by
`code`/`lang`/`rev` (mirroring the local vote store) both fills the flag
(non-empty) and drives the per-reason badges on re-open. Harmless if it lags,
since re-submits are idempotent.

## 4. Flow B — feedback email (EP2)

Tap "Send feedback" → OS email composer opens with:

- **To:** `radoslaw.latka.dev@gmail.com`, resolved at runtime from Remote
  Config (string param `support_email`; the same address is compiled in as the
  fetch-failure default) — the destination can be rotated without an app
  update.
- **Subject:** `EgzaminPPL feedback (Android, v1.4.0)` — platform + versionName.
- **Body (localized):** prefilled context block, then a blank line for prose:

  ```
  App version: 1.4.0 (Android)
  License: PPL(A)
  Language: PL

  Describe the problem or suggestion:
  ```

Platform/OS **is** captured here (unlike content reports) because app bugs are
environment-dependent. The user sees and controls everything sent — email is
the consent-based lane.

**No email app installed** (rare on iOS, possible on Android): catch the
failure and fall back to copying the support address to the clipboard via the
existing `ClipboardCopy` helper (Android 13+ shows the OS "copied" toast;
concept-level parity with the App-version row).

## 5. Feature toggles

**Two independent Remote Config toggles**, one per lane, both in the
`FeatureFlag` enum with `default = false` (dark launch):

| Toggle | Gates | Why its own switch |
|---|---|---|
| `ProblemReporting("problem_reporting")` | EP1 — flag icon + report sheet on both surfaces (the Firestore write path) | A junk-report wave, a rules regression, or triage overload can switch off the write path instantly without touching the contact channel. |
| `ContactUs("contact_us")` | EP2 — Settings "Send feedback" row (mailto lane) | The email row can be pulled independently (inbox abuse, address rotation) without losing structured content reports. |

Gating stays at the ViewModel, exactly like content voting: read
`featureFlagService.isEnabled(...)`, expose a nullable handler, UI omits the
element when null (`onReport = null` → no flag icon; Settings VM `contact_us`
off → no feedback row). Both off = the whole feature invisible.

Dependency note: EP1 rides `ExplanationCard`, so it also implicitly requires
`explanation_enabled`. Questions whose explanation hasn't loaded (error/absent)
have no footer → no flag. Accepted at alpha.

Kill-switch ladder for the write path: `problem_reporting` off = soft kill
(UI gone); redeploying `firestore.rules` with `allow write: if false` = hard
kill (stops writes from already-open sessions too). See concept §5.

## 6. Security & payload hardening

Threat model for a client-writable, read-denied collection with no server
code: oversized documents, attachments, malformed/nested payloads, and
injection into downstream tooling. Defense layers, outermost first:

1. **App Check gate** — writes accepted only from attested app builds
   (existing layer; iOS installer gap inherited from `contentVotes`,
   concept §6).
2. **Schema whitelist (rules)** — `hasOnly`/`hasAll` pins the exact key set;
   any extra key is rejected, so **attachments are impossible by
   construction** — the schema simply has no field that could carry one.
3. **Type pins (rules)** — every field is `is string` (except
   `createdAt == request.time`, which pins it to a server timestamp). Nested
   maps/lists — "JSON in JSON" — fail the type check and reject the write. A
   *string containing* JSON/markup text can still be sent, but it is inert
   (see layer 6).
4. **Length caps on every field (rules)** — not just the note:
   `note.size() <= 500`, `code <= 16`, `license <= 16`, `lang <= 8`,
   `rev <= 64`, `appVersion <= 32`, `reporterId <= 64`, and `reason` must be
   `in ['wrong_answer', 'bad_explanation', 'question_error', 'other']`. This bounds
   the whole document to roughly 1 KB — **a multi-MB report cannot be
   written**. (Firestore's own 1 MiB/doc ceiling stands behind it as a
   platform backstop.)
5. **Identity binding (rules)** — the doc id must equal
   `reporterId:code:lang:rev:reason` recomposed from the payload fields, so a
   payload can't masquerade under someone else's slot and ids stay
   length-bounded too.
6. **Inert content downstream** — report text is never parsed, evaluated,
   rendered as HTML, or interpolated into queries anywhere (Firestore is not
   SQL; there is no query-injection surface). The only consumer is
   `scripts/report_review.py`, which must treat `note` as opaque text and
   strip/escape control and ANSI sequences before printing to the terminal.
   If a GitHub-issue bridge is ever added, markdown-escape there too.
7. **Client-side normalization** — hard 500-char cap in `onValueChange` and
   stripping of ASCII control characters (newlines kept) before submit. UX
   nicety only; the rules are the enforcement boundary.

`read: false`, `delete: false` for clients throughout — a malicious writer can
never read anything back, and the per-install slot bound (concept §6) caps
total junk volume per identity.

## 7. Component impact map

| Piece | Change |
|---|---|
| `ExplanationCard` footer / `VoteBar` | Extend footer row: optional report slot (start) + existing thumbs (end). Render flag iff `onReport != null`. |
| **New** `ProblemReportSheet` | `ui/components/` — AppBottomSheet content: title, 3 radio rows, note field, microcopy, Send. Shared by both surfaces. |
| **New** report controller | Mirror `ContentVoteController`: owned by `QuestionViewModel` + `QuestionPreviewViewModel`, exposes reported-state + `onReport`. |
| `SettingsScreen` | New row in About section; `mailto:` compose + clipboard fallback. |
| `FeatureFlag.kt` | Add `ProblemReporting` + `ContactUs`. |
| `firestore.rules` | New `problemReports` block per §6 (type pins, per-field caps, id binding). |
| `strings.xml` (EN + PL) | New `report_*` / `settings_item_feedback_*` keys (§9). |

## 8. User stories & acceptance criteria (client MVP)

- **S1 (Must)** — As a student who got a question wrong and believes the key is
  wrong, I can report it without leaving the question. *AC:* flag visible on
  live explanation; 3-tap path; write lands in `problemReports` with full auto
  context; works offline; flag fills.
- **S2 (Must)** — As a student reviewing results, I can report any question,
  including ones I answered correctly. *AC:* flag on `QuestionPreviewScreen`
  for every outcome; same sheet and behavior as S1.
- **S3 (Must)** — As a reporter, I see that I already reported this question
  this session and can amend. *AC:* filled flag after submit; reopening and
  re-sending same reason overwrites (no dupes); different reason allowed.
- **S4 (Must)** — As a user with an app bug or an idea, I can email the
  maintainer from Settings. *AC:* composer opens prefilled (subject +
  context body); no-mail-app fallback copies the address.
- **S5 (Must)** — Dark launch, per lane. *AC:* `problem_reporting` off hides
  the flag icon on both surfaces; `contact_us` off hides the Settings row;
  each toggle flips its lane independently, without an app update.
- **S6 (Must)** — Hardened writes. *AC:* rules reject any report with extra
  keys, non-string fields, out-of-enum reason, or any field over its length
  cap (§6); a conforming report is ~1 KB and multi-MB payloads are impossible.
- **Should** — persistent reported-state cue across launches.
- **Could** — local echo of last note on reopen; in-sheet "thanks" morph
  before dismiss.
- **Won't** — screenshots, reply channel, report status in client, snackbar
  infrastructure, down-vote→report nudge, in-app app-bug form.

## 9. Copy (EN / PL)

| Key | EN | PL |
|---|---|---|
| `report_flag_content_description` | Report a problem with this question | Zgłoś problem z tym pytaniem |
| `report_flag_reported_content_description` | Problem reported. Tap to report again | Problem zgłoszony. Dotknij, aby zgłosić ponownie |
| `report_sheet_title` | Report a problem | Zgłoś problem |
| `report_reason_question_error` | There's an error in the question | W treści pytania jest błąd |
| `report_reason_bad_explanation` | The explanation is wrong or unclear | Wyjaśnienie jest błędne lub niejasne |
| `report_reason_wrong_answer` | The answer marked as correct is wrong | Odpowiedź oznaczona jako poprawna jest błędna |
| `report_reason_other` | Something else | Inny problem |
| `report_reason_already_reported` | Reported | Zgłoszono |
| `report_note_placeholder` | Add details (optional) | Dodaj szczegóły (opcjonalnie) |
| `report_privacy_hint` | We can't reply to reports. Please don't include personal data. | Nie możemy odpowiadać na zgłoszenia. Nie podawaj danych osobowych. |
| `report_submitted_confirmation` | Thanks for your report | Dziękujemy za zgłoszenie |
| `report_submit` | Send | Wyślij |
| `settings_item_feedback_title` | Send feedback | Prześlij opinię |
| `settings_item_feedback_subtitle` | Report a bug or suggestion | Zgłoś błąd lub sugestię |
| `feedback_email_subject` | EgzaminPPL feedback (%1$s, v%2$s) | Opinia o aplikacji EgzaminPPL (%1$s, v%2$s) |
| `feedback_email_body_*` | per §4 template | per §4 template |

Reason rows are listed in display order (see the concept's §4 rationale). Reason labels
are user-language framings of the fixed enum values; the enum (`question_error` /
`bad_explanation` / `wrong_answer` / `other`) is what's stored. Full copy rationale:
`copy-audit.md`.

## 10. Accessibility & haptics

- Flag: 48 dp target, `contentDescription` per state, no `selected` semantics
  (it's an action, not a toggle) — state conveyed via description swap.
- Sheet: reason rows `Role.RadioButton` + selected semantics (picker parity);
  Send disabled state announced; note field labeled; sheet is dismissible by
  standard gestures + back.
- Haptics: `LongPress` on Send, matching `VoteBar`'s vote haptic.
- Both themes; note-field + IME insets handled (sheet content `imePadding`).

## 11. Open items

1. **Support email address for EP2** — ✅ resolved 2026-07-07:
   `radoslaw.latka.dev@gmail.com`, delivered via the Remote Config string
   param `support_email` with the address compiled in as the default, so it
   can be rotated later without shipping a new binary.
2. **Privacy sign-off (concept §8.4)** — free-text storage under the
   voting-style legitimate-interest basis; privacy policy EN+PL + store
   data-safety labels before flag-on. Gates the `problem_reporting` flip, not
   the build (and not `contact_us` — the mailto lane stores nothing). Now a
   dedicated phase in [implementation-plan.md](implementation-plan.md).
