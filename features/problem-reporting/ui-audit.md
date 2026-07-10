# Problem-reporting UI/UX audit — 2026-07-07

A designer pass over the shipped reporting surfaces (report sheet, footer flag, Settings feedback row),
run after Phases 1–6 landed. Six issues raised by Radek; findings R1–R6 below with the decisions taken.
All fixes shipped together in the `feature/problem-reporting-ui-polish` PR.

## Findings

### R1 — Sheet title read as a fourth reason row · Medium · FIXED
`SheetTitle` (titleMedium) and `AppBottomSheetRow` labels (bodyLarge) are the same optical size and share
the 24dp inset. The pickers disambiguate with leading icon chips; the reason rows had no leading slot, so
"Zgłoś problem" scanned as an unselectable option. **Fix:** leading radio buttons on the reason rows (see
R3) — with a control column present, the title unambiguously reads as a header. No typography change.

### R2 — Send below the fold · High · FIXED
`AppBottomSheet` used the default `ModalBottomSheet` state, so content taller than the half-expanded stop
opened partially expanded and hid the Send button — a form whose commit control must be discovered by
dragging. **Fix:** `skipPartiallyExpanded` opt-in on `AppBottomSheet`; the report sheet sets it because
its commit sits at the bottom. Pickers keep the default (every row is itself the commit).

### R3 — Labels re-wrapped when the trailing check appeared · Medium · FIXED
The trailing check is only composed when selected, so selecting stole 24dp from the weighted label column
and long PL labels reflowed. The pickers never show this (selection dismisses the sheet in the same
frame); a select-then-confirm form shows it on every tap. **Fix:** `leadingRadio` variant on
`AppBottomSheetRow` — a constant-width leading `RadioButton` that also matches the row's announced
`Role.RadioButton` (the trailing-check visual contradicted the spoken semantics). The M3-correct control
for choose-then-confirm.

### R4 — No submission feedback · Medium · FIXED (decision: snackbar)
The filled flag was the only confirmation, and it flips exactly while the sheet's dismiss animation and
scrim fade compete for attention — classic change blindness, ending in a "did it send?" doubt loop.
**Decision:** M3 snackbar (not a platform toast), copy **"Dziękujemy za zgłoszenie" / "Thanks for your
report"** — gratitude framing because the write is fire-and-forget and offline-queued, so "sent" could
overclaim. Implementation: `snackbarHost` slot on `ScrollableScaffoldFrame`/`ScrollableScaffold` (the M3
Scaffold slot equivalent — rides above the floating pill bar, or the navigation-bar inset when there is
no bar), plus a shared `rememberConfirmingReportHandler` wrapper used by both surfaces.

### R5 — Flag vs bug icon · KEPT the flag
The flag is the cross-app convention for "report this content" (YouTube/Maps/Instagram); the bug glyph
signals *software* malfunction, is already used by the Settings Debug row, and would route app-bug
traffic into a sheet whose three reasons can't express "the app crashed" — breaking the feature's
two-lane model (content → flag, app bugs → Settings "Wyślij opinię"). Outlined→filled also mirrors the
vote thumbs' state grammar in the same footer.

### R6 — Settings hue duplication (Feedback = Language = Teal) · Low · FIXED
Per-row hue is the row's identity in the Gemini-style grouped list; two Teal rows dulled it. The License
row's hue varies with the selected licence (Blue/Green/Amber/Red), which rules out Indigo (≈ PPL(A) Blue
in dark), Cyan (≈ Teal), Green/Red (licence collisions) and leaves **Purple** — collision-free in release
builds for every licence in both themes. The Debug row (debug builds only) moved Purple → Rust so
internal builds are clean too. Pre-existing note: Theme (Amber) collides with the License row when SPL is
selected; unavoidable while the licence row carries the licence's own brand colour — the invariant we
guarantee is that *fixed* rows never collide with each other.

## Root cause, in one line
R1/R2/R3 were one mistake: reusing the instant-select picker row pattern (tap → check → dismiss) for a
select-then-confirm form. The pattern's assumptions — short content, iconed rows, selection commits —
each broke as a separate visible defect.

## Test fallout
The instrumented run surfaced four pre-existing device-test failures unrelated to this PR: report-flag
assertions hardcoded English strings, but instrumented tests resolve `stringResource` in the device
locale (the emulator runs Polish). Fixed by resolving expected strings through the resource system inside
`setContent`. Full device suite 112/112 after the fix.

## Verification
- Host suite 461/461; instrumented suite 112/112 (was 107/112 before the locale fix).
- Emulator, both themes: sheet opens fully expanded with Send visible; radios; zero reflow selecting the
  long PL reason; snackbar above the pill bar (Question) and above the gesture bar (Preview); Settings
  hues Purple/Rust distinct from all neighbours.
- `:composeApp:compileKotlinIosSimulatorArm64` green (shared Compose — same UI on iOS).
