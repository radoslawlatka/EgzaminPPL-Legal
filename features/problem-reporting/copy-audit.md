# Problem-reporting copy audit — 2026-07-07

A copywriter pass over every user-facing string in the reporting feature (report sheet, footer
flag, snackbar, Settings feedback row, feedback email), in both languages, each string read in its
on-screen context. Follows the same-day UI/UX audit (`ui-audit.md`). Findings C1–C7 plus one
product addition; decisions taken by Radek are marked.

## Findings

### C1 — "Poprawna odpowiedź jest błędna" is self-contradictory · Medium · FIXED
An oxymoron in both languages ("The correct answer is wrong") — the sentence asserts and negates
*correct* about the same noun. The intended claim is about the *marking*, so the fix makes the
marking explicit: **PL "Odpowiedź oznaczona jako poprawna jest błędna" / EN "The answer marked as
correct is wrong"**. (The concept doc §4 had it right all along — "The marked correct answer is
wrong"; the implementation drifted.)

**Decision (Radek), reorder:** the reason list now leads with `question_error` — ULC sources both
the questions and their answer keys, so a wrong key is the *least* likely defect and sits last
among the fixed reasons. Display order = `ProblemReason` declaration order.

**Decision (Radek), new reason:** a catch-all **`other` — PL "Inny problem" / EN "Something
else"** — closes the list. Touches the enum, the rules whitelist, the conformance suite, the
triage CLI's reason set, and the feature docs (§8.2 amendment in the functional design).

### C2 — "Wyślij opinię" → "Prześlij opinię" · Low-Medium · FIXED
"Prześlij opinię" is Google's own Android localization of *Send feedback* (Gmail, Maps, Play,
system settings) — the native idiom in a Google-styled settings list. Side benefit: the Settings
lane no longer shares its headword with the report sheet's "Wyślij" button. EN stays
"Send feedback".

### C3 — Settings feedback subtitle too long (Radek's bullet 3) · Low · FIXED
"Zgłoś błąd lub zaproponuj usprawnienie" / "Report a bug or suggest an improvement" ran long for
a row subtitle. Shortened, parallel in both languages: **PL "Zgłoś błąd lub sugestię" / EN
"Report a bug or suggestion"** (~35% shorter, imperative kept).

### C4 — Reported-flag description promised an edit · Low · FIXED
"Tap to change your report" / "Dotknij, aby zmienić zgłoszenie" — but resubmitting never edits:
the same reason overwrites idempotently, a different reason files a second report. Accurate and
simpler: **"Tap to report again" / "Dotknij, aby zgłosić ponownie"**.

### C5 — Privacy hint diverged between languages · Low · FIXED
EN "We **can't** reply" states a capability (honest — reports carry no contact channel); PL "Nie
odpowiadamy" read as cold policy. PL aligned to the honest framing: **"Nie możemy odpowiadać na
zgłoszenia."**

### C6 — PL reason-list parallelism · Nit · FIXED
Reasons were two sentences plus a noun phrase ("Błąd w treści pytania"). With C1 a sentence, the
set is now uniformly sentences: **"W treści pytania jest błąd"**. EN was already all-sentences.

### C7 — Email subject "Opinia EgzaminPPL" · Low, PL grammar · FIXED
Bare juxtaposition reads as a genitive — "EgzaminPPL's opinion". Fixed: **"Opinia o aplikacji
EgzaminPPL (…)"**. EN subject unchanged.

## Verified good (unchanged)
Reason 2 matches the on-screen section header ("Wyjaśnienie"/"Explanation") exactly; sheet title
and flag description agree ("Zgłoś problem…"); the snackbar keeps its deliberate gratitude framing
("Dziękujemy za zgłoszenie" — the write is offline-queued, "wysłano" could overclaim); note
placeholder, counter and Send are clean. The two lanes keep distinct vocabularies — content =
*problem/zgłoszenie*, app = *opinia* — which quietly reinforces the two-lane model.

## Shipped copy (final)
See functional-design §9 — the table there is updated to this audit's outcome and is the
authoritative copy record.
