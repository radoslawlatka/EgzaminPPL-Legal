# P1 — Mistake-driven study: UI design

*Feature slug: `mistake-driven-study` · Designer output · Date: 2026-06-15*
*Builds on: F1 "app-memory" — see `docs/features/app-memory/design.md`*

---

## 0. Scope confirmation

**Epic W — Weighted Quick review: no UI changes whatsoever.**
The mode-selection screen, the Quick review card, and the in-session UI are untouched. The only
change is the question-selection algorithm inside `ShortLearningQuestionStrategy`. From the user's
perspective — visually, behaviourally, and textually — the screen is byte-for-byte identical before
and after Epic W ships. This spec makes no design decisions for Epic W.

**Epic E — "My mistakes" fifth mode card: all design work is here.**
One new card is added below the existing four on the per-category mode-selection screen. No new
screens are introduced. The MistakeReview session and results screens are reused without modification.

---

## 1. Flow diagram

```
Entry point: Categories screen → tap a subject tile
                                     │
                               ModeSelection screen
                               (4 cards, or 5 if Progress flag ON)
                                     │
              ┌──────────────────────┼──────────────────────────────────────────────────────┐
              │                      │                                                        │
         [existing]            [existing]                                         [NEW — Epic E]
         Learning /          Saved / Exam                                     "My mistakes" card
         Quick review                │                                                │
                                     │                                    pool empty → card disabled
                                     │                                    flag off   → card absent
                                     │                                    pool ≥ 1   → card enabled
                                                                                      │
                                                                               tap card (enabled)
                                                                                      │
                                                                          MistakeReview session
                                                                     (existing SessionScreen reused,
                                                                      strategy sourced from pool
                                                                      not a single resultId)
                                                                                      │
                                                                          answer questions
                                                                                      │
                                                                     SessionResultsScreen (existing)
                                                                                      │
                                    ┌─────────────────────────────────────────────────┤
                                    │                                                  │
                          "Review N mistakes"                                     Close / Retake
                          (per-session CTA,                               (back to ModeSelection)
                           existing, unchanged)
                                    │
                          [another MistakeReview
                           sourced from that session's
                           resultId — existing behaviour,
                           untouched per OQ-5]
```

**Interruption behaviour:** The MistakeReview session state is not persisted across process death
(same as the current MistakeReview launched from results — no resume dialog is shown for
MistakeReview). The pool itself is durably held in the F1 log; the card count re-resolves fresh
on the next ModeSelection open.

**Abandon points:** Back from ModeSelection returns to Categories. Back from an in-progress
MistakeReview session (same as Learning) prompts the existing "unfinished session" dialog only if
the session is a mode that supports resume — MistakeReview does not support resume today
(`getSessionUseCase` is only invoked for `LearningMode.Learning` and `LearningMode.ShortLearning`
in `ModeSelectionViewModel`), so back simply exits the session without a dialog.

---

## 2. The fifth "My mistakes" card — all five states

### 2.1 State: Hidden (Progress flag OFF, which is the default)

The card is **absent**. The ModeSelection screen renders exactly four cards — Learning, Quick
review, Exam, Saved — and nothing else. No placeholder, no collapsed state, no hint that a fifth
card exists. The `learningModes(...)` list in `ModeSelectionScreen.kt` simply does not include the
`MistakePool` descriptor. The screen is byte-for-byte the pre-feature four-card layout.

This is the state for every user until the Progress flag is enabled. It matches NFR-2 (flag-off is
a true no-op) and E3.3.

### 2.2 State: Loading (Progress flag ON, pool count not yet resolved)

The ModeSelection screen is already in `ModeSelectionUiState.Loading`, which renders `LoadingContent`
(a `CircularProgressIndicator` centred in the content area). The existing loading behaviour covers
the whole screen, including the as-yet-unresolved fifth card count. No additional skeleton shape is
needed for the card itself because the whole content area is replaced by the loading indicator until
the `ModeSelectionUiState.Content` state arrives.

**Constraint this imposes on the ViewModel:** The `ModeSelectionUiState.Content` must NOT be emitted
until the `mistakeCount` is known. The ViewModel must collect the pool count (analogous to how
`observeFavouriteCountUseCase` is collected today — the favourite count starts at 0, then updates
live) from a new `ObserveMistakeCountUseCase`. To satisfy E3.4 ("never shows a stale or wrong
count"), the Content state should only first be emitted once both the category load AND the initial
pool count resolve. The existing `latestFavouriteCount` pattern is an acceptable model, but note
that it starts at 0 — for the mistake count, a zero initial value is semantically correct (empty
pool) so stale-count risk is zero: a 0 count causes the card to render as disabled, which is a safe
starting state while the real count loads. This means the same "start at 0, update via Flow" pattern
used for favourites is fine for the mistake count too.

### 2.3 State: Populated / Enabled (Progress flag ON, pool count ≥ 1)

This is the primary interactive state.

```
┌────────────────────────────────────────────────────────┐
│  ⚠  My mistakes                         [15 questions] │
│     Questions you got wrong in this category           │
└────────────────────────────────────────────────────────┘
```

- **Icon:** `Icons.Outlined.Warning` (the outlined exclamation-in-triangle from Material Icons).
  Justification below in §3.
- **Accent colour:** `Color(0xFFE65100)` — a deep amber-orange, distinct from:
  - Learning (`Color(0xFF1E88E5)` blue)
  - Quick review (`Color(0xFF00897B)` teal)
  - Exam (`Color(0xFFFB8C00)` bright amber — similar hue but one step warmer/darker than Exam to
    distinguish them; see §3 for the contrast argument)
  - Saved (`Color(0xFFE53935)` red)
  The deep amber reads as "attention / caution" without being alarm-red (which is already Saved's
  colour) and fits the cockpit-placard aesthetic.
- **Title:** `mode_selection_card_mistakes_title` — "My mistakes" / "Moje błędy"
- **Description:** `mode_selection_card_mistakes_description_populated` — a `PluralStringResource`
  so it reads: "15 questions to review" / "15 pytań do powtórzenia"
  (see §4 copy table for all forms).
- **Chip:** `TonalChip` with `pluralStringResource(categories_text_question_count, count, count)`.
  This reuses the existing plural resource shared by all mode cards — same chip component, same
  font, same `surfaceContainerHighest` background, same `labelMedium` + numeric typeface. No new
  chip variant.
- **Enabled:** `true`. `AppTile` renders at full `contentAlpha = 1f`, tap triggers navigation.
- **Interaction:** `onClick` calls `viewModel.onModeSelected(LearningMode.MistakeReview, onModeSelected)`.
  The ViewModel navigates directly (no resume-dialog branch, because MistakeReview has no session
  persistence — `MistakeReview` is not handled in the `Learning`/`ShortLearning` resume-check block).

**Content description for TalkBack/VoiceOver (populated state):**
`"My mistakes. 15 questions to review."`
Delivered as the `AppTile`'s accessibility label, constructed from the title + description strings.
The `AppTile` component currently passes `contentDescription = null` on the icon (decorative) and
the `Surface` gets its label from its child text content. No change needed to `AppTile` itself —
Compose merges the `Text` nodes inside the tile into one announcement. Confirm with
`Modifier.semantics(mergeDescendants = true)` on the `Surface`; this is how the existing tiles
already work.

### 2.4 State: Empty / Disabled (Progress flag ON, pool count = 0)

The card is **shown but disabled**. This is the congratulatory moment: the pool is empty because
the user has answered every previously-wrong question correctly.

```
┌────────────────────────────────────────────────────────┐  (disabled — 50% alpha
│  ⚠  My mistakes                                        │   via AppTile's
│     Questions you answer incorrectly are collected     │   contentAlpha = 0.5f)
│     here                                               │
└────────────────────────────────────────────────────────┘
```

- **No chip** — `chipCount = null`, so the `footer` slot is `null`. A "0 questions" chip would be
  confusing; omitting it matches how the Saved card shows no chip when `favouriteCount = 0`.
- **Description changes** to the hint copy: `mode_selection_card_mistakes_description_empty` —
  "Questions you answer incorrectly are collected here" / "Tu trafią pytania, na które odpowiesz
  błędnie" (see §4 copy table).
- **`enabled = false`** — `AppTile` applies `contentAlpha = 0.5f` to all child text and icon, dims
  the accent colour to `accentColor.copy(alpha = 0.38f)` (same pattern as the Saved card when
  `favouriteCount == 0`), and `Surface(enabled = false)` disables the ripple and touch callback.
- **Accent colour:** same `Color(0xFFE65100)` as populated, dimmed to 38% by the existing
  `AppTile` disabled logic — no special empty-state colour needed.

**Content description for TalkBack/VoiceOver (empty/disabled state):**
`"My mistakes. Disabled. Questions you answer incorrectly are collected here."`
The system appends "dimmed" or "unavailable" on Android TalkBack when `enabled = false`; no
custom semantics override is needed.

**Audit §3 note:** The requirements reference an audit action asking for hint copy on the empty
Saved tile. Inspecting `ModeSelectionScreen.kt`, the Saved card's `description` when empty uses
`mode_selection_card_favourites_description` with a plural count — it reads "0 saved questions",
which is not a genuine hint. The audit action has not been implemented yet. The My-mistakes empty
state defined above (descriptive hint copy, no zero-count chip) sets the right pattern and should
be applied to the Saved card in the same change (add a `mode_selection_card_favourites_description_empty`
string "Bookmark questions to review them here" / "Dodaj pytania do zakładek, by powtórzyć je
później" and switch the Saved card's description to use it when `favouriteCount == 0`). This is a
minor fix bundled with the same PR; it is not a scope expansion.

### 2.5 State: Error (pool count read fails)

**Treated as the empty/disabled state — not a distinct error card.**

The requirements (NFR-5) state that a corrupt/unreadable log is skipped, not fatal. A pool-count
read failure resolves to count = 0, which produces the empty/disabled card. There is no "scary
error UI" on this secondary surface (per the brief). The top-level `ModeSelectionUiState.Error`
(which replaces the whole screen with `ErrorContent`) is reserved for the primary category-load
failure, not for a secondary count resolution failure. This is consistent with how a favourite-count
Flow error would be handled today — the screen does not collapse; only the affected count is stale.

---

## 3. Visual specification

### Placement

The fifth card appears **below the four existing cards**, as the last item in the vertical
`Column` inside `learningModes(...)`. The `Arrangement.spacedBy(12.dp)` spacing between cards is
unchanged; the fifth card uses the same 12 dp gap above it as every other card.

```
┌────────────────────────────────────────────────────────┐
│ ←  [Subject name]                                      │  FloatingTopBar (existing)
│    Choose how you want to study                        │
├────────────────────────────────────────────────────────┤
│                                             16dp pad   │
│  ┌──────────────────────────────────────────────────┐  │
│  │ 📖  Learning                   [240 questions]   │  │  Learning card
│  └──────────────────────────────────────────────────┘  │
│                                              12dp gap  │
│  ┌──────────────────────────────────────────────────┐  │
│  │ ⚡  Quick review               [10 questions]    │  │  Quick review card
│  └──────────────────────────────────────────────────┘  │
│                                              12dp gap  │
│  ┌──────────────────────────────────────────────────┐  │
│  │ ⏱  Exam                [40 questions] [45 min]   │  │  Exam card
│  └──────────────────────────────────────────────────┘  │
│                                              12dp gap  │
│  ┌──────────────────────────────────────────────────┐  │
│  │ 🔖  Saved              [7 saved questions]        │  │  Saved card
│  └──────────────────────────────────────────────────┘  │
│                                              12dp gap  │
│  ┌──────────────────────────────────────────────────┐  │
│  │ ⚠  My mistakes         [15 questions]            │  │  ← NEW (flag ON, pool ≥ 1)
│  │    Questions you got wrong in this category       │  │
│  └──────────────────────────────────────────────────┘  │
│                                              16dp pad  │
└────────────────────────────────────────────────────────┘
```

The screen is already `verticalScroll`, so adding a fifth card on short devices causes natural
scrolling — no layout change needed.

### Icon choice: `Icons.Outlined.Warning`

The icon must be visually distinct from:
- `Icons.AutoMirrored.Outlined.MenuBook` (Learning — open book)
- `Icons.Outlined.Bolt` (Quick review — lightning)
- `Icons.Outlined.Timer` (Exam — stopwatch)
- `Icons.Outlined.BookmarkBorder` (Saved — bookmark)

`Icons.Outlined.Warning` (exclamation triangle) satisfies all constraints:
- It is semantically accurate: a "warning" that these are the user's current weak spots
- It is visually distinct from all four existing icons — no overlap in shape or metaphor
- It belongs to the Material Icons set already imported in the project (no new dependency)
- The outlined variant matches the design convention established by all four existing cards (all use
  `Outlined` variants, never filled)
- At 24dp with `accentColor.copy(alpha = 0.16f)` circle background, it reads as "attention" rather
  than "danger", which is the right register (not alarming, just task-oriented)

Alternative considered: `Icons.Outlined.ErrorOutline` (circle with exclamation). Rejected because
the circle-badge shape is semantically closer to "error state" (which this is not — it is a study
mode) and it could be confused with the app's own `ErrorContent` icon (`CloudOff`). The triangle
warning is a better "things to fix" metaphor.

### Accent colour: `Color(0xFFE65100)` (deep amber-orange)

Exam uses `Color(0xFFFB8C00)`. The My-mistakes card uses `Color(0xFFE65100)` — slightly darker and
more saturated, shifting toward red-orange. On screen they read as clearly different: Exam is
bright warm amber, My-mistakes is deeper and more urgent. Neither conflicts with Saved's
`Color(0xFFE53935)` red, which has a strongly different luminance.

In the disabled state `AppTile` applies `copy(alpha = 0.38f)` to the accent — no additional token
needed.

### Chip styling

`TonalChip` — identical component to all other cards. The chip text uses the existing
`categories_text_question_count` plural: "15 questions" / "15 pytań". No new chip variant, no new
string resource for the chip text.

### Font scaling

`AppTile` uses `Modifier.weight(1f)` on the `Column` containing title and description, and the
`Text` composables do not have `maxLines` set — they wrap naturally. The description string
"Questions you answer incorrectly are collected here" is 47 characters in English and approximately
53 characters in Polish ("Tu trafią pytania, na które odpowiesz błędnie"), which wraps gracefully
at 1.5× text scale on any screen ≥ 360dp wide. This is within the tested range of the other
existing descriptions (Learning: 49 chars EN, "Ucz się wszystkich pytań i odpowiedzi we własnym
tempie" 53 chars PL).

### Reduced motion

The `AppTile` press-scale animation uses `rememberPressScale` (via `PressScale.kt`). The
`TileProgressBar` inside `AppTile` checks `rememberReducedMotion()` and snaps instead of animating.
The My-mistakes card shows no progress bar (no `TileProgress.Populated` — the card has no mastery
fraction concept), so the reduced-motion concern is limited to the press-scale — which already
respects `LocalAnimationSpec` / reduced-motion on the platform level. No additional work needed.

---

## 4. Copy table

All strings are added to `composeApp/src/commonMain/composeResources/values/strings.xml` (EN) and
`composeApp/src/commonMain/composeResources/values-pl/strings.xml` (PL).

| String resource key | EN value | PL value | Notes |
|---|---|---|---|
| `mode_selection_card_mistakes_title` | `My mistakes` | `Moje błędy` | Card title |
| `mode_selection_card_mistakes_description_populated` | (plural — see below) | (plural — see below) | When pool ≥ 1 |
| `mode_selection_card_mistakes_description_empty` | `Questions you answer incorrectly are collected here` | `Tu trafią pytania, na które odpowiesz błędnie` | When pool = 0 (disabled) |

**Plural resource — populated description:**

```xml
<!-- values/strings.xml -->
<plurals name="mode_selection_card_mistakes_description_populated">
    <item quantity="one">%1$d question to review</item>
    <item quantity="other">%1$d questions to review</item>
</plurals>
```

```xml
<!-- values-pl/strings.xml -->
<plurals name="mode_selection_card_mistakes_description_populated">
    <item quantity="one">%1$d pytanie do powtórzenia</item>
    <item quantity="few">%1$d pytania do powtórzenia</item>
    <item quantity="many">%1$d pytań do powtórzenia</item>
    <item quantity="other">%1$d pytań do powtórzenia</item>
</plurals>
```

The Polish plural follows the established pattern in the codebase: one/few/many/other, with the
genitive plural "pytań" for many/other (matching `categories_text_question_count`,
`session_results_button_review_mistakes`, etc.).

**Chip text:** reuses `categories_text_question_count` — no new string.

**Content descriptions (for accessibility — not shown as visible text):**

| State | Announced text (EN) | Notes |
|---|---|---|
| Populated (15 questions) | "My mistakes. 15 questions to review." | Merged from title + description Text nodes inside AppTile |
| Empty/disabled | "My mistakes. Dimmed. Questions you answer incorrectly are collected here." | System appends "Dimmed" for `enabled = false` on Android TalkBack |
| Loading | Whole-screen spinner is announced by the system as "Loading" | No card-specific description during Loading |
| Hidden (flag off) | Card absent — nothing to announce | |

No custom `Modifier.semantics` override is needed for the content description: `AppTile`'s `Surface`
uses `mergeDescendants = true` semantics by default via Material3, which causes TalkBack to announce
all child `Text` content as a single node. The populated description "15 questions to review" already
conveys the count. The chip text ("15 questions") would be merged into the same node — this
duplication is acceptable (it adds emphasis to the count) but if it proves verbose in testing, the
chip can be annotated `Modifier.semantics { invisibleToUser() }`.

---

## 5. Accessibility notes (NFR-7)

Beyond the standard checklist:

**Touch target:** `AppTile` fills `fillMaxWidth()` and has `padding(16.dp)` inside the `Surface`
producing a minimum height well above 48 dp even with a single line of description. The fifth card
inherits this — no additional constraint needed.

**Disabled state contrast:** When `enabled = false`, `AppTile` applies `contentAlpha = 0.5f` and
the accent fades to 38%. At 50% alpha on `MaterialTheme.colorScheme.onSurface` over
`surfaceContainer`, the resulting grey passes the 3:1 large-text threshold for the title
(`titleMedium` = 16sp, which qualifies as large text) but may not pass 4.5:1 for the description
(`bodyMedium` = 14sp). This is an existing behaviour inherited from the Saved card and is a
pre-existing accessibility debt, not introduced by this feature. Flag it as a known issue in the
existing audit tracker.

**Icon is decorative:** The warning icon has `contentDescription = null` (matching all four existing
cards). The icon's meaning is conveyed by the card title. Correct — do not add an icon content
description.

**State change on pool update:** When the user returns from a MistakeReview session to
ModeSelection, the pool count updates via the `observeMistakeCountUseCase` Flow. This is a live
`StateFlow` collect that calls `updateContent { copy(mistakeCount = count) }`, which triggers a
recompose. TalkBack will not re-announce the screen unless the user explicitly focuses the card
again — this is correct behaviour (no intrusive live-region announcement on a background update).

**Traversal order:** The fifth card sits at the bottom of the vertical `Column`, so TalkBack
traversal order is top-to-bottom: Learning → Quick review → Exam → Saved → My mistakes. This is
the natural reading order and requires no custom traversal override.

---

## 6. Distinct-from-Saved clarity

The risk is that users conflate "My mistakes" (auto-populated, cannot be edited) with "Saved"
(manual bookmarks, user-controlled). Three design choices prevent this:

1. **Different icon and colour.** Saved uses a bookmark (`BookmarkBorder`, red). My mistakes uses a
   warning triangle (`Warning`, deep amber-orange). The metaphors are unambiguous: a bookmark is
   something you put there; a warning is something the system flags.

2. **The description states the source.** The populated description "N questions to review" doesn't
   distinguish them, but the empty-state hint does — "Questions you answer incorrectly are collected
   here" is explicit that the system fills this automatically. The Saved empty hint (once added)
   will read "Bookmark questions to review them here" — the contrast between "are collected" (passive,
   automatic) and "Bookmark questions" (imperative, user-action) makes the distinction legible.

3. **Position.** My mistakes is the fifth card, below Saved. Users who open the screen for the first
   time see Saved first and can form a mental model of it before encountering My mistakes. The
   vertical ordering (manual above automatic) matches the conceptual hierarchy.

No additional visual badge, label, or tooltip is needed to distinguish them. The copy alone is
sufficient if kept precise.

---

## 7. ModeDescriptor additions (implementation guidance, not prescriptive)

The existing `learningModes(...)` function in `ModeSelectionScreen.kt` accepts `favouriteCount: Int`
as a parameter. The same pattern should be extended with `mistakeCount: Int` (only passed when the
Progress flag is on and the fifth card should appear; the parameter is absent or unused when the
flag is off, keeping flag-off a true no-op at the composable level).

The descriptor for the fifth card, when included:

```
ModeDescriptor(
    mode = LearningMode.MistakeReview,   // existing mode enum value
    titleRes = Res.string.mode_selection_card_mistakes_title,
    description = when {
        mistakeCount > 0 -> ModeDescription.Plural(
            Res.plurals.mode_selection_card_mistakes_description_populated,
            mistakeCount,
        )
        else -> ModeDescription.Simple(
            Res.string.mode_selection_card_mistakes_description_empty,
        )
    },
    icon = Icons.Outlined.Warning,
    accentColor = Color(0xFFE65100),
    enabled = mistakeCount > 0,
    chipCount = mistakeCount.takeIf { it > 0 },   // null when empty → no chip
)
```

The `onModeSelected` handler in `ModeSelectionViewModel` currently throws for `LearningMode.MistakeReview`:
```kotlin
LearningMode.MistakeReview ->
    error("Mistake review is entered from session results, not mode selection")
```
This guard must be updated to handle the new entry point — navigate directly without a resume check
(MistakeReview has no resume state).

**[Cross-check resolution, 2026-06-15 — authoritative]** To keep a single source of truth and make
illegal states unrepresentable, the count is modelled exactly as the architecture specifies:
`ModeSelectionUiState.Content` gains **`val mistakesPoolCount: Int? = null`**, populated by
`ObserveMistakesPoolCountUseCase(categoryId): Flow<Int?>` (collected like `observeFavouriteCountUseCase`).
Semantics — the one mapping the whole card derives from:

- **`null` → Hidden** — Progress flag OFF (default) **or** a pool-count read failure. Card absent
  (E3.3, NFR-5). A read failure maps to `null`, **not** `0`, so a transient error never falsely tells
  a user with real mistakes that they are clear.
- **`0` → Empty/disabled** — flag ON, no current mistakes. Disabled card + hint copy (E3.2).
- **`≥1` → Populated** — flag ON. Enabled card + count chip (E3.1).

The `learningModes(...)` builder takes `mistakesPoolCount: Int?`; the fifth descriptor is appended
only when it is non-null, with `enabled = count > 0` and `chipCount = count?.takeIf { it > 0 }`. This
replaces the earlier two-field (`showMistakesCard: Boolean` + `mistakeCount: Int`) sketch and keeps
the composable layer free of direct flag-checking.

---

## 8. Open questions

**OQ-D1 — Exam accent colour clash.**
`Color(0xFFE65100)` for My mistakes vs. `Color(0xFFFB8C00)` for Exam are distinguishable at normal
vision but may be difficult for users with red-green colour vision deficiency (deuteranopia). The
recommended fix — different icon shapes and different descriptions — already compensates because
information is never conveyed by colour alone. No action required for this iteration. Flag for the
colour-palette audit.

**OQ-D2 — Should the Saved empty-hint fix ship in the same PR as Epic E?**
Recommended: yes, as a two-line string addition bundled in the same PR. It is the most natural
moment to align the two cards' empty patterns. If the PR scope is deliberately minimal, it can be a
separate follow-up issue.

**OQ-D3 — Results screen: should the "Review N mistakes" CTA from a pool-sourced MistakeReview
session navigate back to mode-selection instead of starting another per-session MistakeReview?**
The requirements (OQ-5) explicitly say the results-screen CTA continues to work as today
(per-session, from that `resultId`). This spec proceeds on that. The nested loop — MistakeReview →
results → "Review mistakes" from that session → another MistakeReview — is short-circuited by the
existing logic in `ResultActions`: MistakeReview sessions do not show "Retake", so the user's only
forward options are "Review N mistakes" (if any remain in that session's result) or Close. The
pool-sourced MistakeReview feeds graduate-on-correct (E2), so the pool shrinks; returning to
ModeSelection and re-entering the My-mistakes card gives the freshest pool. This flow is coherent
without changes.
