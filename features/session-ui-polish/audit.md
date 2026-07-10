# Session Screens UI Audit — Question, Exam, Results & Preview

- **Date:** 2026-07-03
- **Build audited:** branch `development` @ `bbb1c7b`, debug build on a Pixel-class emulator (1080×2400)
- **Scope:** the full session flow — `QuestionScreen` (serving five modes via VM qualifiers: Nauka, Szybka powtórka, Zapisane, per-session mistake review, Moje błędy), `ExamScreen` (20 questions / 30 min), `SessionResultsScreen`, and `QuestionPreviewScreen` — plus their shared components (`QuestionComponents`, `SessionProgressContent`, `ModalSideSheet`, `ExplanationCard`/`ExplanationBlocks`, `BottomPillBar`, `ExamTimerDisplay`, `ScrollableScaffold`, `CenteredContent`), in both themes, including the explicit question of whether the ModeSelection sky-remnant gradient should extend to these screens
- **Method:** live screenshots captured on-device across all five learning modes, the exam (question, side sheet, exit/finish dialogs, results), results and preview, then a multi-lens agent audit (color/theming, typography, iconography, layout, interaction/motion, accessibility, cross-app consistency, gradient feasibility) — **55 findings raised, every one adversarially verified against source and by re-sampling pixels from the captures; all 55 confirmed** (a handful carry an explicitly-hedged sub-claim, noted inline). Quality bar: ModeSelection and the hero screens as left by their polish passes, and the same best-in-class benchmark set
- **Evidence:** live on-device captures (dark + light) at 1080×2400 — the pixel samples and contrast ratios quoted below were measured on those originals

> File/line references are to the audited commit and will drift.

## Verdict

The architecture is the flow's greatest asset: one component file serves five learning modes, the exam, the preview, and the results list, so nearly every fix below lands once and repairs every surface. But the session flow visibly predates the Night Flight polish, and its eight high-severity issues cluster into two families. The first is a **color-channel family**: the light-theme quiz palette fails WCAG on every surface where it carries text, correct-vs-wrong is encoded by green-vs-red hue alone end to end (reveal, side sheet, results list), and in the dark exam sheet answered-vs-unanswered chips measure 1.08:1 apart — the sheet cannot do its one job. The second is a **semantics family**: answer options, the modal side sheet, and the progress rows are all silent or misleading to screen readers, so a TalkBack user cannot tell what they selected, what was correct, or which questions they got wrong anywhere in the app. On top of these sit two mechanical defects — question changes inherit the previous question's scroll offset with zero transition, and long formulas clip silently mid-expression. The rest of the list is largely mechanical ports of patterns the heroes and ModeSelection already shipped (edge fades, entrance choreography, mono numerals, heading semantics, reduced-motion gating), plus one deliberate scene decision the product owner asked about, answered below.

## The gradient background: yes across the whole session flow

**Recommendation: extract ModeSelection's sky remnant into a shared `Modifier.skyRemnant()` and apply it to the learning-mode question screens, the question preview, and (taller) the results screen.** *(These are two of the confirmed medium findings, G1/G2 in the summary table.)*

> **Product decision (2026-07-03, post-audit):** the exam **also** carries the sky remnant. The audit below argued for keeping it intentionally flat as an "instrument mode"; the product owner chose consistency and simplicity — one gradient language across every session surface — so the implementation applies `skyRemnant()` to the exam question screen at the same 0.11f as the learning screens. The "flat = under test" note in the *Exam stays flat* bullet is superseded.

**The cut is real.** `AppNavGraph.kt:367-371` — `OpaqueScreen` paints plain `MaterialTheme.colorScheme.surface` for every session route, so the scene chain (Categories full shader → ModeSelection 11% remnant → session) terminates in an unstyled hard cut. The captures show a dead-black band above the progress bar in dark theme where ModeSelection carries the remnant, and the same discontinuity in light theme (flat `#F4F6FF` vs. the `#AFC8E8` sky band one screen up).

**The plumbing is already proven.** ModeSelection draws its remnant *inside* its own `OpaqueScreen` (`ModeSelectionScreen.kt:252-271`, a `drawBehind` vertical gradient from `LocalHorizonPalette.current.skyTop` to `surface`, `SKY_REMNANT_FRACTION = 0.11f` at :110). Do **not** unwrap `OpaqueScreen` or make session roots transparent — its kdoc (:358-364) explains it guards predictive-back previews; the remnant draws over the opaque surface exactly as ModeSelection's does, so previews stay correct.

- **Learning + preview:** promote the remnant to `ui/components/SkyRemnant.kt` (a second usage — CLAUDE.md's reuse-first rule requires the promotion) and apply at the default 0.11f on `QuestionScreen`'s root Box (`QuestionScreen.kt:130`) and the preview's scaffold root. The top bar and progress bar sit inside the band; verify the light-theme "Pytanie N z M" `onSurfaceVariant` subtitle holds ≥4.5:1 over `#AFC8E8`.
- **Results:** the session's emotional bookend is currently the flattest surface in the app — `SessionResultsScreen.kt:119` is a bare `Column` on `#0A0A0C` (`Theme.kt:71`) and the top ~35% of the frame around the ring is uniform near-black in the captures. Apply the remnant with a taller fraction (~0.30) so the dusk wash sits behind the `ScoreRing` and merges with its radial glow; keep the ring/verdict colors as-is (the no-red-shaming intent at :418-420 is right), and optionally stagger the verdict chip + CTAs in via `Entrance.kt` after the ring sweep for choreography parity with ModeSelection.
- **Exam stays flat, on purpose:** it is the flow's instrument mode — a concentration surface — and the amber `tertiaryContainer` timer pill (`ExamTimerDisplay.kt:34`) would sit warm-on-warm inside a dusk band. The flat surface becomes semantic, extending the established language: **live sky = home, remnant = navigation and debrief, flat = under test.**

## What's already at the bar (keep)

- **Shared-component consolidation** — `QuestionComponents`, `SessionProgressContent`, `ScrollableScaffold` and the explanation stack serve all seven session surfaces; the fix list below is short in *code* even where it is long in findings.
- **The dark-theme quiz palette passes everywhere it carries text** (7.47/6.07/10.29/7.79:1 on the same constructions that fail in light theme) — the dark tuning is the reference to match, not to touch.
- **No-red-shaming ScoreRing** — a documented decision (`SessionResultsScreen.kt:419-421`): the ring stays neutral on a fail and the verdict lives on the chip. Keep it; only the *pass* state needs durability (M16).
- **ExamVerdictTile triple-encodes** its meaning (tonal container + CheckCircle/Cancel icon + label, `SessionResultsScreen.kt:346`) — exactly the redundancy discipline the rest of the flow should copy (H4).
- **The learning loading skeleton** mirrors the real layout with a live circled close so the chrome never jumps (`QuestionLoadingContent.kt:41-52`) — the documented no-layout-shift contract; the exam variant just needs its timer slot (L16).
- **The patterns for the a11y fixes already exist in-repo:** `FormulaBlock` ships alt text (`ExplanationBlocks.kt:189`), `VoteBar` models selection semantics (`VoteBar.kt:100`), `SectionHeader` carries `heading()` (`SectionHeader.kt:30`), CategoriesScreen shows the one-shot live-region announce.
- **ScoreRing choreography** (sweep + count-up + pass pulse) is reduced-motion aware (`SessionResultsScreen.kt:435-446`) — the only session animation that is (L1).
- **The explanation block vocabulary is right** — mono formula slab with accent bar and real subscripts, primary-tinted KeyTakeaway, inline markup, VideoBlock's honest OpenInNew affordance. The gaps are at the edges (clipping, crop, rhythm), not the system.

## High-severity findings

### H1. Question changes are an abrupt swap that inherits the previous question's scroll position

`ScrollableScaffold.kt:38` creates its scroll state internally (`.verticalScroll(rememberScrollState())`) and nothing ever resets it; the scaffold stays in composition across question changes — the `key(questionState.question.id)` at `QuestionScreen.kt:167` / `ExamScreen.kt:156` wraps only the options block. A grep over the session flow finds no `scrollTo` and no `AnimatedContent`/`Crossfade` on either screen. Exam questions routinely exceed one viewport (option D below the fold in both theme captures), so the user is necessarily scrolled down when tapping "Następne" — the next question opens mid/bottom-scrolled with its code and opening text off-screen, and no transition marks the change.

**Fix:** add a `scrollState: ScrollState` parameter to `ScrollableScaffold`, hoist it in `QuestionContent`/`ExamContent`, and reset per question (`LaunchedEffect(question.id) { scrollState.scrollTo(0) }`, or `key(question.id) { rememberScrollState() }`). Simultaneously replace the bare `key {}` with `AnimatedContent` on the app's existing fade-through specs (`AppNavGraph.kt:375-387`: 210ms in, 90ms delay / 90ms out), snapping under `rememberReducedMotion()` — separate instances per key preserve the existing highlight-leak fix. Preview keeps the default state.

### H2. Long formulas clip silently at the slab edge, ending in a bare division sign

`ExplanationBlocks.kt:195-199` — `maxLines = 1`, `softWrap = false`, `.horizontalScroll(rememberScrollState())` with no fade, ellipsis, scrollbar, or chevron; the Text fills the width, so the line ends flush with the rounded slab and the hidden remainder is undiscoverable. Captured live: a formula renders as "a = [T − D − F_R − W · sin θ] /" terminating exactly at the right card edge — the denominator is off-screen even though the "where:" legend directly below defines "m — mass". The alt at :189 covers screen readers; sighted students read a silently truncated formula.

**Fix:** hoist the scroll state and draw horizontal edge fades over the slab while `canScrollForward/canScrollBackward` (generalize `TopEdgeFade.kt` into a horizontal variant in `ui/components/`); give the scrolled Text ~24dp end padding so a scrollable line never terminates flush; auto-step the mono style bodyLarge→bodyMedium when `onTextLayout` reports overflow so most formulas fit without scrolling at all.

### H3. Light-theme quiz colors fail WCAG on every surface where they carry text

`QuizColors.kt:22-23` — `correct = Color(0xFF2F9A4D)`, `incorrect = Color(0xFFD5453C)`. Measured on the real composited backgrounds: green digits on the tinted tile **2.92:1** (below even the 3:1 non-text floor; ~2.7:1 over the side sheet's `surfaceContainer`), red digits **3.48:1**, the near-white A/B/C/D letters on the badges **3.33:1** (correct) / **4.10:1** (incorrect) — `QuestionComponents.kt:230` puts `colorScheme.surface` (`#F4F6FF`, `Theme.kt:41`) on those mid-tone fills — and the verdict-chip label **3.48:1** fail / **2.92:1** pass (`SessionResultsScreen.kt:344`; digit sites at `SessionProgressContent.kt:149-150`). Dark theme passes everywhere (7.47/6.07/10.29/7.79:1).

**Fix:** darken `LightQuizColors` to tone-30 — correct → ~`0xFF1B6B33` (digit-on-tile 4.64:1, white-on-badge 6.09:1), incorrect → `0xFFB3261E`, which is already `AccentTones.Red` light so the reds stay in lockstep (4.54:1 / 6.06:1). Keep the 0x1F light containers and dark palette unchanged. One edit fixes borders, badges, side-sheet/results digits, and the verdict chip at once (alternatively pick `badgeContent` by badge luminance — the discipline `AccentTones` already enforces).

### H4. Correct vs. wrong is encoded by green-vs-red hue alone, end to end (WCAG 1.4.1)

`QuestionComponents.kt:248-268` — the Correct/CorrectRevealed and Incorrect branches of `toColors()` return structurally identical `OptionColors` (2dp border + tinted fill + filled badge), differing only in `quizColors.correct` vs. `incorrect`; `SessionProgressContent.kt:135-151` — `NumberTile` likewise switches only background/content color, no icon, shape, or text channel. Under hue-collapse the pair is inseparable by lightness: green:red luminance 1.32:1 dark, 1.23:1 light (`QuizColors.kt:22-32`) — verified indistinguishable under deutan/protan simulation of the answered-state and results captures. This defeats the results screen's primary job of scanning which questions were wrong; preview reuses the same states (`QuestionPreviewScreen.kt:131-134`). The app already models the fix: `ExamVerdictTile` pairs color with icon + text.

**Fix:** add a non-color channel in both shared components — in `AnswerOption`, crossfade the badge letter to a check/close icon on reveal (the letter is redundant once graded) or add a trailing 16dp check/X; in `NumberTile`, overlay a small check/X/dash glyph. Being shared, this fixes the reveal, the in-session sheet, and the results list in one place.

### H5. Answer options expose no selection or correctness semantics to screen readers

`QuestionComponents.kt:291-308` — the entire interactive surface is `Surface(onClick = …, enabled = …)` with no semantics block anywhere in the file; a repo-wide grep for `stateDescription` returns zero hits under `ui/question`, `ui/exam`, `ui/components`. In the exam, TalkBack cannot tell which option is selected (`ExamScreen.kt:161` conveys it purely via `Selected` colors); in learning modes, grading silently disables all options (`QuestionScreen.kt:178`) and a *correct* answer has no announced consequence at all (`QuestionScreen.kt:184` shows the explanation only for `Incorrect`). `VoteBar.kt:100` proves the project knows the pattern.

**Fix:** in `AnswerOption` add `Modifier.semantics { role = Role.RadioButton; selected = …; stateDescription = … }` with localized state strings ("wybrana odpowiedź", "poprawna odpowiedź", "twoja błędna odpowiedź"); wrap the option loops (`QuestionScreen.kt:167`, `ExamScreen.kt:156`) in `selectableGroup()`; keep revealed rows focusable by moving state into semantics instead of relying on the disabled node; surface the verdict non-visually (polite live region on a grading caption) so correct answers aren't silent.

### H6. The side sheet is not modal — back misfires through it and its scrim leaks the tree

`ModalSideSheet.kt:74-95` — the sheet and scrim are plain Box siblings (not a Popup/Dialog); the scrim is a `clickable(…, indication = null)` with no `onClickLabel`, role, or `paneTitle`, and nothing clears the semantics underneath, so TalkBack traversal walks into the darkened question behind the scrim. A grep for `BackHandler` under `ui/` hits only `ExamScreen.kt:111` (`BackHandler { showCloseConfirmation = true }`), which fires unconditionally even while the sheet is open — back with the sheet open raises the "Przerwać egzamin?" confirmation *on top of the open sheet*; in learning modes (no handler at all) back pops the whole session.

**Fix:** add `BackHandler(enabled = visible) { onDismiss() }` inside `ModalSideSheet.kt` — composed after ExamScreen's handler (`ExamScreen.kt:169`), it takes precedence while open, restoring "back dismisses the topmost surface" on both screens. Then host the overlay in a focusable Popup (or `semantics { paneTitle = "Postęp" }` on the sheet Surface plus `clearAndSetSemantics`/`invisibleToUser` on background content while visible) and give the scrim an `onClickLabel` ("Zamknij").

### H7. Progress rows announce neither role nor status — a screen-reader user can never find their wrong answers

`SessionProgressContent.kt:96-120` — `SessionProgressItem` has only `.clickable(onClick)`; per-question status lives exclusively in `NumberTile`'s background color (:135-151) and `isCurrent` is a 0.10-alpha tint (:91) + SemiBold (:114). TalkBack reads just "1, Duża wysokość gęstościowa…" for every row, identical whether the question was aced, failed, or skipped. The same component renders the results list (`SessionResultsScreen.kt:198-210`), so the one place the app summarizes performance is inaccessible end to end (WCAG 1.4.1 / 4.1.2).

**Fix:** on the Row add `Modifier.semantics { role = Role.Button; selected = item.isCurrent; stateDescription = … }` built from `item.status`/`isCurrent` with localized strings ("poprawna odpowiedź", "błędna odpowiedź", "bez odpowiedzi", "z odpowiedzią", "bieżące pytanie") — one change fixes side sheet and results together; H4's check/X glyph provides the matching visual channel.

### H8. Dark-theme exam sheet: answered and unanswered chips are 1.08:1 apart

`SessionProgressContent.kt:139` gives Answered a `primary.copy(alpha = 0.16f)` tile vs. Unanswered `surfaceContainerHighest` (:136). Pixel-sampled from the dark exam-sheet capture: answered bg RGB(53,56,70) vs. unanswered RGB(50,51,55) = **1.08:1**; the 12sp digits (174,185,255 vs. 166,168,176) differ in hue only. This fails the 3:1 non-text floor the app itself enforces via `AccentTones` — and the finish dialog ("Masz 18 pytań bez odpowiedzi") expects users to *find* those 18 in this sheet.

**Fix:** give Answered a full-alpha container in `NumberTile` (e.g. `secondaryContainer` + `onSecondaryContainer` digit), or keep the tint but add a redundant non-color cue (small filled check/dot on the tile), so answered rows scan at a glance in both themes.

## Medium findings

### Question reading & layout

- **M1. Question text is always titleLarge regardless of length.** `QuestionComponents.kt:201-205` renders every question at 22sp B612 unconditionally, across all six surfaces. In the light exam capture a bilingual ULC question runs ~19–23 lines to ~65% of screen height before option A appears — options C/D end up clipped or under the pill bar, costly under the 30-min timer. B612 is a cockpit face designed for short labels, not 20-line paragraphs (rendering concern; the string content itself is out of scope). **Fix:** make the style length-adaptive inside `QuestionCard` — titleLarge below ~140–200 chars, stepping to titleMedium/bodyLarge with relaxed lineHeight beyond; one change fixes all modes.
- **M2. Answer text is bodyMedium (14sp) — a 22→14sp cliff below the question.** `QuestionComponents.kt:329`; bilingual options run 2–3 lines at 14sp inside generously padded 20dp-radius pills that visually dwarf their own text. **Fix:** raise to bodyLarge (16sp); with M1 this rebalances the hierarchy toward the decision content.
- **M3. Content shears at a hard line above the "floating" pill bar.** The last option clips mid-card with a dead gap above the pill (visible in the dark exam capture) because `ScrollableScaffold.kt:33-50` places the bar in flow after the `weight(1f)` column, contradicting `BottomPillBar.kt:31-35`'s own "detached floating pill" kdoc — ModeSelection dissolves its edge instead (`ModeSelectionScreen.kt:299-301`). **Fix:** overlay the bar in a Box, add its measured height to bottom content padding, and add a `bottomEdgeFade` (generalize `TopEdgeFade.kt` into an edge-fade with a bottom variant).
- **M4. No content width cap on any session screen.** `widthIn` caps exist only on the heroes (`LicenseSelectionScreen.kt:57` 460dp, `ModeSelectionScreen.kt:314` 480dp); `ScrollableScaffold` and the results LazyColumn/CTAs have none — options, question text and full-width buttons stretch edge-to-edge on tablets/landscape (code-verified; no tablet capture). **Fix:** center-cap once in `ScrollableScaffold` (~560dp) and apply the same to the results list and `ResultActions`.

### Explanation card

- **M5. The explanation — the core learning payload — pops in with zero motion and can appear entirely below the fold.** `QuestionScreen.kt:184-192` is a bare conditional appending `ExplanationCard` last in the scroll column; no `AnimatedVisibility`, no `animateContentSize` for the Loading→Content swap (`ExplanationCard.kt:49,77-87`), and no scroll-into-view anywhere in the codebase. In the answered-state capture only a ~6-line sliver peeks above the pill bar; on a long question the card renders fully off-screen while "Dalej" is enabled. **Fix:** wrap in `AnimatedVisibility(expandVertically + fadeIn, 250ms)` (snap under reduced motion), add `animateContentSize()`, and scroll the card's top edge into view via the H1-hoisted `ScrollState` when the verdict lands.
- **M6. The card is visually a fifth gray option.** `ExplanationCard.kt:53-57` — same `surfaceContainer` fill as an unselected option (`QuestionComponents.kt:235`), near-identical radius (24 vs 20dp), exactly one option-gap (12dp, `ScrollableScaffold.kt:30`) below the last option, and no header or label; both theme captures confirm it scans as another option blob. **Fix:** add a compact header row (16–18dp `Outlined.Lightbulb` + "Wyjaśnienie" in titleSmall/labelLarge), differentiate the container (`surfaceContainerHigh` or a hairline tinted outline à la KeyTakeaway, `ExplanationBlocks.kt:101`), and pass `top = 12.dp` extra at the call sites for a 24dp break from the answer stack.
- **M7. Explanation images crop.** `ExplanationBlocks.kt:224` — `ContentScale.Crop` inside a forced `aspectRatio(… ?: 16:9)` slot (:227, default :206): any diagram off the declared ratio loses its edges — content loss in exactly the block meant for charts and force diagrams (code-derived; no image block in the captures). **Fix:** switch to `ContentScale.Fit` — the slot already paints `surfaceContainerHigh`, so mismatches letterbox gracefully; keep the ratio slot for state stability.
- **M8. A wrong answer with no explanation renders nothing at all.** `QuestionViewModel.kt:282` maps missing/empty to `ExplanationUiState.Empty` and `ExplanationCard.kt:47` early-returns — a documented choice, but with the explanation store mid-regeneration (~1114/2487) the debrief silently appears and vanishes question-to-question with no cue which is intended. **Fix:** until coverage is complete, render a quiet one-line placeholder ("Wyjaśnienie w przygotowaniu", bodySmall, no container); restore the early return once populated.
- **M9. The results→preview loop drops the explanation exactly where it is still owed.** `QuestionPreviewScreen.kt:144-154` — the `when` branches are exclusive: a skipped question gets the unanswered banner *instead of* the explanation, and a correct one gets nothing below the options (deliberate per the comment at :146) — yet the results CTA counts skips among the "błędy" to review, and in-session the debrief is Incorrect-only, making preview the sole re-read surface. **Fix:** render `ExplanationCard` for all three outcomes on this read-only screen (banner *then* card for skips; optionally collapsed behind "Pokaż wyjaśnienie" for correct answers) — `explanationState` is already loaded regardless of verdict.

### Side sheet & chrome

- **M10. Sheet width is hard-coded to 360dp.** `ModalSideSheet.kt:46/102` — no `widthIn`/fraction coercion; on the 411dp test device it already covers ~88% (a ~45–52dp scrim strip in the captures), and on any 360dp-wide phone it is full-bleed: no scrim remains, so tap-outside dismissal silently disappears (arithmetic from the constant). **Fix:** size relative to the window — `fillMaxWidth(0.87f).widthIn(max = SheetWidth)` or `min(SheetWidth, maxWidth − 56.dp)` — deriving offset/scrim math from the measured width; or fall back to a bottom sheet at compact widths per M3 guidance.
- **M11. The sheet toggle is the nav-drawer glyph.** `QuestionComponents.kt:143` uses `Icons.AutoMirrored.Default.MenuOpen` — reads "menu", and its left-pointing chevron references a start-edge drawer while this sheet slides from the right (`ModalSideSheet.kt:101`, `CenterEnd`); the sheet's content is a numbered question checklist. **Fix:** `Outlined.Checklist` or `Outlined.FormatListNumbered` (extended icons already in use), keeping the existing content description.
- **M12. Error/empty states swap the Night Flight chrome for a stock TopAppBar + ArrowBack.** `CenteredContent.kt:164-177` (`BackNavigationScaffold`) is used by `QuestionScreen.kt:52-80` and `ExamScreen.kt:65-73`, while the loading skeleton these states replace keeps the shared `FloatingTopBar` + circled Close X (`QuestionLoadingContent.kt:86-94`) — the same route changes bar style *and* icon semantics mid-flow. **Fix:** rebuild `BackNavigationScaffold` on `FloatingTopBar` with a navigation-icon parameter so session flows keep the X they entered with.

### Results & verdict

- **M13. Skips are counted as "błędy".** `SessionResult.kt:29-34` folds unanswered into `mistakeCount`, so the CTA reads "Przejrzyj 18 błędów" while the list paints those same questions neutral gray (17 of the 18 in the fail capture were never answered); learning modes quietly permit skipping (`BottomPillBar.kt:71-78` has no enabled gating; `LearningSessionMachine.kt:34-46` advances regardless). **Fix:** keep free navigation but make the wording honest ("Przejrzyj %1$d pytań do poprawy" in `values-pl/strings.xml`), and give Unanswered a distinct tile treatment (dashed/outline or "–" badge) so gray unambiguously means "skipped, included in review".
- **M14. The score and pass threshold share one low-emphasis concatenated line.** `SessionResultsScreen.kt:140-147/166-170` — `countText + thresholdSuffix` as a single grey bodyLarge Text with no numeric font, sandwiched between the mono "10%" ring and the mono time chip; the threshold is count-denominated ("próg zaliczenia: 15") while the ring speaks percent, and the line wraps arbitrarily at large font scale. **Fix:** split — score "2/20" emphasized with numeric spans, threshold as its own labelMedium row/chip near the verdict, ideally in the ring's unit ("próg 75% · 15/20").
- **M15. The ring track is nearly invisible.** `SessionResultsScreen.kt:450` uses `surfaceContainerHighest`, measured on-screen at **1.23:1** light / **1.57:1** dark against the surface — a 5% score reads as a floating dot, not a fraction of a whole; `QuestionTopBar`'s progress track shares the token (`QuestionComponents.kt:160`). Mitigated by the counted-up digits and the strong fill (5.65/10.53:1), hence M. **Fix:** a dedicated track tone (~`0xFF8F95AB` light / `0xFF53545C` dark) applied to both ring and bar so they stay one system.
- **M16. The pass state evaporates under reduced motion.** The neutral-on-fail ring is intentional and right (`SessionResultsScreen.kt:419-421`), but green exists only as a 1.3s glow pulse (:442-445) that the reduced-motion path skips entirely (:436-440 snaps and returns) — a pass and a fail then render identical rings. **Fix:** have `ScoreRing` accept the verdict, build the sweep from `quizColors.correct` on a pass, and raise the static glow alpha under reduced motion. No change to the fail ring.
- **M17. Wrong vs. unanswered tiles collapse in dark theme.** `SessionProgressContent.kt:136/141/145/150` — composited maroon (54,30,30) vs. `surfaceContainerHighest` (50,51,55) = **1.22:1**, digits `#FF7A70` vs. `#A6A8B0` = 1.07:1 luminance (hue-only); in the fail capture a wrong tile reads as another gray square, and the skipped-vs-erred distinction vanishes entirely under CVD. **Fix:** structural treatment for Unanswered — transparent bg + 1dp `outlineVariant` border (outlined = empty slot) and/or an em-dash glyph; H4's check/X also resolves the pair.
- **M18. Result rows open the preview but carry zero tap affordance.** `SessionResultsScreen.kt:198-210` reuses the side sheet's row verbatim — transparent background, no chevron, no hint (`SessionProgressContent.kt:90-121`) — so the per-question explanation loop is discovered by accident. **Fix:** an optional trailing slot on `SessionProgressItem` (16–20dp `ChevronRight` in `onSurfaceVariant`), enabled only from results.
- **M19. Ungraded sessions get zero qualitative feedback.** `SessionResultsScreen.kt:159-164` — `passed` is null and the verdict tile renders only for exams; the pulse fires only on `passed == true` (:442-445), so a perfect 100% learning run renders the same bare layout as a 40% one — against the product's motivation goal. **Fix:** a score-band headline for `grade == null` ("Bezbłędnie!" at 100%, encouraging ≥75%, neutral below) and a `celebrate` flag so `mistakeCount == 0` triggers the green pulse.

### Exam flow & dialogs

- **M20. The destructive "Przerwij" is the app's standard positive filled-primary pill.** `ExamScreen.kt:193-205` — a plain `Button`, pixel-identical to the *constructive* finish confirm (`FinishConfirmationDialog.kt:49-58`) and the "Następne" pill; the affordance that means "proceed" everywhere else here destroys the session ("Stracisz dotychczasowe odpowiedzi."). **Fix:** error tonality on the abort (`errorContainer`/`onErrorContainer`) or swap emphasis (filled "Wróć do egzaminu", tonal/error "Przerwij"); apply on both platforms per CLAUDE.md.
- **M21. The countdown pill is invisible to assistive tech and its urgency is color-only.** `ExamTimerDisplay.kt:31-40` — last-minute state is only the errorContainer swap; :52 `contentDescription = null`; :56-63 bare text, no semantics; the per-second mutation on an unlabeled node also risks per-second re-announcement while focused (speculative, TalkBack-version dependent). **Fix:** `clearAndSetSemantics { contentDescription = "Pozostały czas: X minut" }` at minute granularity, plus a one-shot polite live-region "Została minuta" on the transition (pattern at `CategoriesScreen.kt:255-256`).
- **M22. Learning sessions confirm with "Zakończyć egzamin?".** `FinishConfirmationDialog.kt:36` hard-codes the exam title (`values-pl/strings.xml:149`), and `QuestionScreen.kt:203-211` raises it for all five learning modes whenever skips remain. **Fix:** title/message parameters with a learning variant ("Zakończyć sesję?"), exam wording kept in ExamScreen only.

### Consistency & state encoding

- **M23. The icon-in-tonal-pill recipe is hand-rolled twice instead of extending TonalChip.** `ExamTimerDisplay.kt:41-65` and `ExamTimeSummary` (`SessionResultsScreen.kt:374-403`) rebuild the identical structure with drifted padding, container tone, and type scale, while `TonalChip.kt:22-34` owns the pill but lacks an icon slot — the exact copy-paste CLAUDE.md's reuse-first rule forbids; three subtly different pills are visible across the exam, results, and ModeSelection captures. **Fix:** generalize `TonalChip` (optional leading icon + color params) and rebuild both on it.
- **M24. The B612-Mono-numerals convention breaks on the results score line and the sheet count.** `Typography.kt:19-21` reserves the numeric face for "timers, question counters, codes, and scores"; the top-bar counter, ring percent and time chip comply, but `SessionResultsScreen.kt:167-168` and `SessionProgressContent.kt:60-66` render "2/20 · próg…: 15" and "9 / 10" in default sans, sandwiched between mono neighbours in the captures. **Fix:** `LocalAppFonts.current.numeric` on the numeral spans via AnnotatedString.
- **M25. Correct-picked and correct-revealed render identically.** `QuestionComponents.kt:248` shares one `OptionColors` for `Correct` and `CorrectRevealed` (`AnswerOptionState.kt:13` distinguishes them), so which option the user actually picked is only inferable from a red option elsewhere. **Fix:** render `CorrectRevealed` one step lighter (green border + badge, `Unselected` background) so a filled container consistently means "your answer"; combines naturally with H4's icons.

### Accessibility & scaling

- **M26. No heading semantics anywhere in the session flow.** "Wynik", the inline "Pytania" section label (`SessionResultsScreen.kt:122-126/178-183`), "Postęp" (`SessionProgressContent.kt:56-59`) and "Podgląd pytania" (`QuestionPreviewScreen.kt:94-98`) are plain Text — zero `heading()` hits under these packages, while `SectionHeader.kt:30` already ships it. **Fix:** `heading()` on the titles; replace the inline "Pytania" label with the shared `SectionHeader` (reuse rule), which brings the semantics for free.
- **M27. Fixed-size text containers clip at large font scales.** Every CTA is `.height(48.dp)` (`SessionResultsScreen.kt:292/309/330`, `ExamScreen.kt:201/211`, `FinishConfirmationDialog.kt:54/64`, `BottomPillBar.kt:75`) and the letter badge / number tiles are `.size(32.dp)` (`QuestionComponents.kt:314-318`, `SessionProgressContent.kt:153-157`) — at 2.0× the ~28sp letter exceeds its 32dp box. **Fix:** `heightIn(min = 48.dp)` / `defaultMinSize(32.dp, 32.dp)`; verify at Android font scale 2.0.

## Low findings

- **L1. The reduced-motion contract is skipped throughout the session flow** — the sheet's 300ms slide (`ModalSideSheet.kt:66-72`), the nav fade-through (`AppNavGraph.kt:52-55`), the 200ms reveal tweens and the progress animation (`QuestionComponents.kt:283-288/155-157`) all run unconditionally, while 10+ call sites elsewhere gate on `rememberReducedMotion()` and `ReducedMotion.kt:5-9` promises snapping (iOS behavior inferred from the missing gate). *Fix:* snap/0ms specs behind the flag; a small shared `reducedAwareTween()` keeps it consistent.
- **L2. The A/B/C/D badge floats mid-block on 3–6-line options** (`QuestionComponents.kt:312` `CenterVertically`; visible on a 6-line option in the mistake-review capture). *Fix:* top-align the Row and optically center the badge on the first line.
- **L3. Post-reveal, the two irrelevant options keep full resting "tappable" styling** — `toColors()` ignores `enabled` (`QuestionComponents.kt:232-238`), inviting dead taps. *Fix:* ~60% content alpha on unselected options once validated.
- **L4. Two reds can co-occur on a timed-out fail:** verdict chip `quizColors.incorrect` (`SessionResultsScreen.kt:344`) vs. timeout note `colorScheme.error` (:408) — code-evidenced (captures finished early). *Fix:* the timeout note joins the verdict story on `quizColors.incorrect`; keep `error` for live in-exam urgency.
- **L5. The mono question-code placard takes the prime first-line position in the exam** (`QuestionComponents.kt:191-199`, passed at `ExamScreen.kt:150`) above already viewport-filling questions. *Fix:* pass empty code from ExamScreen (the parameter already defaults `""`); keep it in learning/preview.
- **L6. B612 Mono's colon cell typesets every time as "02: 54"** and the results chip mono-typesets the word "Czas:" too (`ExamTimerDisplay.kt:68-75`, `SessionResultsScreen.kt:396-398`; visible in both timer captures). *Fix:* AnnotatedString applying the numeric family to digit groups only, in a shared helper both pills use.
- **L7. Long single-paragraph explanations render ~16 unbroken lines at bodyMedium's default leading** (`ExplanationBlocks.kt:78-84`). *Fix:* a reading-optimized style (`lineHeight = 22.sp` or bodyLarge); paragraph splitting is content-pipeline, out of scope.
- **L8. The verdict tile uses filled CheckCircle/Cancel** while the app renders the same concepts Outlined (`SessionResultsScreen.kt:346` vs. `ModeSelectionScreen.kt:213`, `QuestionScreen.kt:66`) — the lone filled-glyph outlier. *Fix:* swap to Outlined.
- **L9. `QuizIcons.kt` is dead code** (zero call sites; hardcoded `tint = Color.White` would be illegible in light theme if revived) duplicating `AccentIconChip`'s recipe. *Fix:* delete.
- **L10. The sheet's close X is a bare IconButton** (`ModalSideSheet.kt:152-160`) while every other close/back sits in a 48dp tonal circle (`FloatingTopBar.kt:63-73`) — visible doubled in the sheet captures, circled X dimmed behind, bare X in front. *Fix:* the same tonal circle (`surfaceContainerHighest` against the sheet), or extract FloatingTopBar's circled icon as a shared composable.
- **L11. Swipe-to-dismiss ignores fling velocity and has no affordance** (`ModalSideSheet.kt:106-116` decides on position only, threshold 0.5). *Fix:* `AnchoredDraggable` or velocity tracking; optionally a subtle start-edge grabber.
- **L12. PL copy "1 / 20 odpowiedzi" drops the verb and mislabels the total** (`values-pl/strings.xml:135` vs. EN "%1$d / %2$d answered") — the 20 are questions, not answers. *Fix:* "Odpowiedziano na %1$d z %2$d pytań" (or "Z odpowiedzią: %1$d z %2$d").
- **L13. A tappable source link differs from a plain citation by text color alone** (`ExplanationBlocks.kt:149-153`; no underline or glyph — WCAG 1.4.1; code-derived). *Fix:* append the trailing OpenInNew icon VideoBlock already uses (:303-308).
- **L14. Uniform 12dp block spacing floats headings equidistant between sections** (`ExplanationCard.kt:58`; `HeadingBlock` has no asymmetric padding — code-derived, captures show single-section explanations only). *Fix:* +8dp top padding on `HeadingBlock`.
- **L15. A failed exam offers no bottom exit** — `SessionResultsScreen.kt:252-297`: with review-primary + retake, no "Gotowe" renders; exit is only the top-left X on a 2400px screen (both fail captures). *Fix:* append a tertiary "Gotowe" TextButton to the failed-exam stack.
- **L16. The exam loading skeleton omits the timer slot**, so real content lands ~42dp lower — contradicting the skeleton's own no-layout-shift kdoc (`QuestionLoadingContent.kt:41-46` vs. `ExamScreen.kt:64/123-130`, `QuestionComponents.kt:172-180`; code-derived). *Fix:* an optional `belowBar` pass-through with a neutral-fill pill placeholder (recipe at `QuestionLoadingContent.kt:163-183`).
- **L17. The results loading state is a bare spinner with no chrome and no exit** (`SessionResultsScreen.kt:97` passes no `onBack`; `CenteredContent.kt:59-66/182-183`), flashing between submit and results since exam-finish pops inclusively (`AppNavGraph.kt:227-231`); mitigated by the fast local read. *Fix:* pass `onClose`, ideally a light skeleton (FloatingTopBar + neutral ring placeholder).
- **L18. Submitting a fully-answered exam from Q20 is instant and irreversible** — the guard fires only when `unansweredCount > 0` (`ExamScreen.kt:138-144`), and the submit occupies the exact pill position that was "Następne" one tap earlier (`BottomPillBar.kt:78`), while the *lower*-stakes abort IS confirmed; accidental-tap frequency is speculative, hence L. *Fix:* confirm the exam finish unconditionally with a positive message ("Wszystkie pytania mają odpowiedź. Zakończyć i ocenić egzamin?") — the dialog is gaining title/message parameters in M22 anyway; keep learning modes as-is.

## Suggested implementation order

Steps are independently shippable; most touch shared components, so each repairs all six surfaces at once.

1. **Color & channel pass** (H3 + H4 + H8, M15 + M17 + M25, L4): tone-30 light `QuizColors`, check/X/dash glyphs in `AnswerOption` + `NumberTile`, answered-tile fix, track tone, `CorrectRevealed` de-emphasis. Mirrors ModeSelection step 1.
2. **Semantics pass** (H5 + H6 + H7, M21 + M26, L12): option radio semantics + groups, sheet BackHandler/paneTitle/scrim label, row state descriptions, timer label + live region, headings, PL count string. Almost entirely mechanical.
3. **Scroll & motion** (H1 + H2, M3 + M5, L1 + L2 + L3): hoisted ScrollState + AnimatedContent question swap, formula edge fades + end padding, bottom-bar overlay + edge fade, explanation reveal + bring-into-view, reduced-motion gating, badge alignment, post-reveal dimming.
4. **Scene & results** (G1 + G2, M13 + M14 + M16 + M18 + M19, L8 + L15): shared `skyRemnant` on learning/preview/results (exam stays flat), honest review wording + skip tiles, score/threshold split, durable pass state, row chevrons, ungraded headline, outlined verdict icons, failed-exam exit.
5. **Chrome & consistency** (M6 + M10 + M11 + M12 + M20 + M22 + M23 + M24, L5 + L6 + L9 + L10 + L11 + L16 + L17): explanation header + container, adaptive sheet width, checklist glyph, FloatingTopBar error states, destructive abort tonality, dialog parameters, TonalChip generalization, mono spans, placard/colon/dead-code/close-circle/fling/skeleton nits.
6. **Edge cases & robustness** (M1 + M2 + M4 + M7 + M8 + M9 + M27, L7 + L13 + L14 + L18): adaptive question type + bodyLarge answers, width caps, image Fit, explanation placeholder + preview coverage, font-scale sweep at 2.0×, reading leading, source-link glyph, heading rhythm, unconditional exam-finish confirm.

## Summary table

| ID | Finding | Primary fix site |
|---|---|---|
| H1 | Question swap inherits previous scroll offset, no transition | `ScrollableScaffold.kt` |
| H2 | Formulas clip flush at the slab edge, no affordance | `ExplanationBlocks.kt` |
| H3 | Light quiz palette fails 4.5:1/3:1 everywhere it carries text | `QuizColors.kt` |
| H4 | Correct/wrong encoded by hue alone (reveal, sheet, results) | `QuestionComponents.kt`, `SessionProgressContent.kt` |
| H5 | Options expose no selection/correctness semantics | `QuestionComponents.kt` |
| H6 | Side sheet not modal (back handler, a11y tree, scrim) | `ModalSideSheet.kt` |
| H7 | Progress rows announce no role or status | `SessionProgressContent.kt` |
| H8 | Dark exam sheet: answered vs unanswered 1.08:1 | `SessionProgressContent.kt` |
| G1 | Results screen flat — extend sky remnant (~0.30) | `SessionResultsScreen.kt` + shared `SkyRemnant` |
| G2 | Learning/preview flat — sky remnant at 0.11; exam stays flat | `QuestionScreen.kt`, `QuestionPreviewScreen.kt` |
| M1 | Question text always titleLarge regardless of length | `QuestionComponents.kt` |
| M2 | Answer text bodyMedium — 22→14sp cliff | `QuestionComponents.kt` |
| M3 | Hard shear above the floating pill bar | `ScrollableScaffold.kt` |
| M4 | No content width cap (tablet/landscape) | `ScrollableScaffold.kt`, `SessionResultsScreen.kt` |
| M5 | Explanation pops in, can land fully below the fold | `QuestionScreen.kt`, `ExplanationCard.kt` |
| M6 | Explanation card reads as a fifth option | `ExplanationCard.kt` |
| M7 | Explanation images crop (ContentScale.Crop) | `ExplanationBlocks.kt` |
| M8 | Missing explanation renders nothing | `ExplanationCard.kt` |
| M9 | Preview drops explanation for correct & skipped questions | `QuestionPreviewScreen.kt` |
| M10 | Sheet width hard-coded 360dp — full-bleed on 360dp phones | `ModalSideSheet.kt` |
| M11 | Sheet toggle uses the nav-drawer MenuOpen glyph | `QuestionComponents.kt` |
| M12 | Error/empty states swap to stock TopAppBar + ArrowBack | `CenteredContent.kt` |
| M13 | Skips counted as "błędy" in the review CTA | `strings.xml`, `SessionProgressContent.kt` |
| M14 | Score + threshold concatenated, unit-mismatched, sans digits | `SessionResultsScreen.kt` |
| M15 | Ring/progress track 1.23:1 light / 1.57:1 dark | `SessionResultsScreen.kt`, `QuestionComponents.kt` |
| M16 | Pass state invisible under reduced motion | `SessionResultsScreen.kt` |
| M17 | Wrong vs unanswered tiles collapse in dark theme | `SessionProgressContent.kt` |
| M18 | Result rows carry zero tap affordance | `SessionProgressContent.kt` |
| M19 | Ungraded sessions get no qualitative feedback | `SessionResultsScreen.kt` |
| M20 | Destructive "Przerwij" styled as the positive primary pill | `ExamScreen.kt` |
| M21 | Timer pill unlabeled; urgency color-only | `ExamTimerDisplay.kt` |
| M22 | Learning finish dialog says "Zakończyć egzamin?" | `FinishConfirmationDialog.kt` |
| M23 | Timer/time pills hand-rolled twice past TonalChip | `TonalChip.kt` |
| M24 | Mono-numeral convention broken on score line + sheet count | `SessionResultsScreen.kt`, `SessionProgressContent.kt` |
| M25 | Correct-picked vs correct-revealed render identically | `QuestionComponents.kt` |
| M26 | No heading semantics in the session flow | results/preview/sheet titles |
| M27 | Fixed heights/sizes clip at large font scale | CTAs, badge, tiles |
| L1 | Reduced-motion gate skipped across the session flow | sheet, nav, reveal, progress |
| L2 | Letter badge floats mid-block on multi-line options | `QuestionComponents.kt` |
| L3 | Post-reveal irrelevant options keep tappable styling | `QuestionComponents.kt` |
| L4 | Two different reds on a timed-out fail | `SessionResultsScreen.kt` |
| L5 | Question-code placard wastes the exam's first line | `ExamScreen.kt` |
| L6 | Mono colon typesets "02: 54"; "Czas:" label in mono | timer helper |
| L7 | Long paragraphs at bodyMedium default leading | `ExplanationBlocks.kt` |
| L8 | Verdict tile uses filled icons vs app's outlined language | `SessionResultsScreen.kt` |
| L9 | `QuizIcons.kt` is dead code | delete |
| L10 | Sheet close X loses its tonal circle | `ModalSideSheet.kt` |
| L11 | Sheet swipe ignores fling velocity, no affordance | `ModalSideSheet.kt` |
| L12 | PL "1 / 20 odpowiedzi" drops the verb, mislabels total | `values-pl/strings.xml` |
| L13 | Source link distinguished by color alone | `ExplanationBlocks.kt` |
| L14 | Headings float equidistant in block rhythm | `ExplanationBlocks.kt` |
| L15 | Failed exam has no bottom exit action | `SessionResultsScreen.kt` |
| L16 | Exam loading skeleton omits the timer slot | `QuestionLoadingContent.kt` |
| L17 | Results loading is a chromeless spinner with no exit | `SessionResultsScreen.kt`, `CenteredContent.kt` |
| L18 | Fully-answered exam submit is instant and irreversible | `ExamScreen.kt` |
