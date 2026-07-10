# F1 — "Give the App a Memory" — UX/UI Design

*Feature slug: `app-memory` · Designer: UX/UI · Status: design · Date: 2026-06-15*
*Scope: Epic C (home-tile progress signal) + Epic D (feature-flag gate). Epics A and B are
invisible infrastructure — this document specifies only what the user sees.*

---

## 0. Design constraints and decisions log

**Read every section before implementing.**

| # | Constraint / decision | Source |
|---|---|---|
| DC-1 | Flag-off state must be byte-for-byte identical to today — no residual layout change | req D1.1 / OQ-10 |
| DC-2 | Signal is mastery-led ("% correct on most recent attempt") — coverage available as secondary | req OQ-4 |
| DC-3 | No red shaming — a zero or low score must not feel punitive | audit §1 |
| DC-4 | Skeleton loaders must match content silhouette at the correct font scale | audit §1 / req C1.5 |
| DC-5 | Reduced-motion: use `rememberReducedMotion()` — snap, do not animate | existing pattern |
| DC-6 | Numeric / instrument values set in `LocalAppFonts.current.numeric` (B612 Mono) | existing `TonalChip` pattern |
| DC-7 | All tokens must come from the existing `MaterialTheme.colorScheme` or `LocalQuizColors` | CLAUDE.md / audit §1 |
| DC-8 | 48×48 dp minimum touch target on all interactive elements | HIG / Material 3 |
| DC-9 | Layout must survive 1.5× font scale without clipping | audit §1 |
| DC-10 | Tile is a progress surface — the raw question count leaves the tile (recommended, not enforced in F1) | audit §3.F3 |

---

## 1. Flow diagram

```
Entry points
  ├── App cold-start → Categories screen (flag off)  → [Baseline — no change]
  └── App cold-start → Categories screen (flag on)   → [F1 flow below]

F1 flow (flag on)
────────────────────────────────────────────────────────────────────────────────
[App launch]
  │
  ├─ CategoriesViewModel emits Loading
  │     → Grid renders: shimmer skeleton tiles (NO progress numbers)
  │
  ├─ Categories + progress data both resolve
  │     → Grid renders: Content tiles with TileProgressBar
  │         ├─ never-practised tile  → empty-state treatment (no bar, "—" label)
  │         └─ practised tile        → filled bar + "62%" mastery label
  │
  ├─ User taps a tile → mode selection → session → results
  │
  └─ User returns to home (back-stack pop)
        → CategoriesViewModel re-emits Content (progress already updated via
          Flow from AnswerEventRepository — no restart needed)
            → Tile re-renders with updated mastery
              ├─ reduced-motion ON  → instant swap (no animation)
              └─ reduced-motion OFF → animateFloatAsState on bar width (350 ms,
                                      FastOutSlowIn, same spec as entrance)

[Licence switch — user taps licence in top bar]
  → changeLicenseUseCase fires
  → CategoriesViewModel emits Loading (or Content with new licence immediately)
  → Tiles re-render to new licence's progress
     ├─ reduced-motion ON  → instant
     └─ reduced-motion OFF → each tile fades through a 1-frame skeleton then
                             re-renders (the existing entrance choreography
                             covers this; no extra animation needed)

[Feature flag toggled at runtime (debug or remote config)]
  → CategoriesViewModel observes FeatureFlag.Progress via featureFlagService.observe()
  → flag OFF: progress state set to null on all tiles (falls back to baseline layout)
  → flag ON:  progress state loads and appears
  → No restart required (D1.4)

Success exit: user sees mastery bar on each practised subject.
Abandon: user closes app — no data loss (answer events already persisted by Epic A).
Interruption: OS kill — on relaunch, progress reloads from DataStore; skeleton shows
              during the read, never shows stale numbers (C1.5).
```

---

## 2. Screen specifications

### Screen: Home / Categories (F1 augmentation)

**Purpose:** The user sees at a glance which subjects they have practised and how well they are
doing, without navigating anywhere.

**Navigation:** Root destination. Entered on app launch (if licence already selected), or after
licence selection. Back exits the app (system back). Licence switch happens in-place via the top-bar
dropdown.

**This is an augmentation of an existing screen — not a new screen.** Only the `CategoryItem`
composable and the `ShimmerTileCard` change. Everything else (top bar, Continue card, entrance
choreography, grid layout, empty-category state, error state) is unchanged.

---

#### 2.1 Layout — Content state (flag ON, subjects practised)

```
┌──────────────────────────────────────────────────────────────┐
│  Home · PPL(A)  ▾                              [settings ⚙]  │  FloatingTopBar (unchanged)
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Continue learning                                           │  HomeSectionHeader (unchanged)
│  ┌────────────────────────────────────────────────────────┐  │
│  │ ▷  Air Law          Learning                          │  │  ContinueSessionCard (unchanged)
│  │    [12 / 45      ]                                    │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  Categories                                                  │  HomeSectionHeader (unchanged)
│  ┌────────────────────────────────────────────────────────┐  │
│  │ ⚖  Air Law                                            │  │
│  │    ══════════════════════░░░░░░░  [62%]               │  │  <── TileProgressBar (NEW)
│  └────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 🔧  Aircraft Knowledge                                 │  │
│  │    ──────────────────────────────  [—]                 │  │  <── empty state (never practised)
│  └────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 🗺  Flight Planning                                    │  │
│  │    ████████████████████████████░  [91%]               │  │
│  └────────────────────────────────────────────────────────┘  │
│  … (remaining tiles)                                         │
└──────────────────────────────────────────────────────────────┘

Legend:
  ══  filled segment (primary color at 70% opacity on the track)
  ░░  unfilled track (surfaceContainerHighest)
  ──  empty-state track (dashed appearance via alternating segments — see §3)
  [62%]  TonalChip in B612 Mono, labelMedium, surfaceContainerHighest
  [—]    TonalChip showing an em-dash (never-practised; NOT "0%")
```

**What changes inside AppTile for F1:**

The `footer` slot of each `CategoryItem` receives a `TileProgressBar` instead of (or
below) the existing `TonalChip`. The raw question count chip moves to mode selection in a future
iteration; in F1 it is **replaced by the progress chip** when the flag is on. When the flag is off,
the question count chip remains exactly as today.

---

#### 2.2 Tile anatomy — detailed ASCII

The `AppTile` layout (icon circle + content column) is unchanged. The `footer` slot content
changes:

```
Flag OFF (baseline — byte-for-byte today):
┌─────────────────────────────────────────────────┐
│ ◉  Air Law                                      │  icon 40×40dp, CircleShape, accentColor@16%
│    [245 questions  ]                            │  TonalChip — existing, unchanged
└─────────────────────────────────────────────────┘
    ← 14dp →   ← weight(1f) column ────────────→
    padding: 16dp all sides (existing)

Flag ON, empty (never practised):
┌─────────────────────────────────────────────────┐
│ ◉  Air Law                                      │
│    ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌  [—]      │  dashed track + em-dash chip
└─────────────────────────────────────────────────┘
     bar height: 4dp    chip: "—"

Flag ON, populated (mastery = 62%):
┌─────────────────────────────────────────────────┐
│ ◉  Air Law                                      │
│    ████████████████████░░░░░░░░░░░░  [62%]      │  filled track + percentage chip
└─────────────────────────────────────────────────┘
     bar height: 4dp    gap: 8dp    chip: "62%"

Flag ON, error/fallback:
┌─────────────────────────────────────────────────┐
│ ◉  Air Law                                      │
│    ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌  [—]      │  same as empty — no error color
└─────────────────────────────────────────────────┘
     Degrades to the same visual as never-practised. No crash, no red indicator.
```

**The progress row sits below the title, taking the place of the existing `TonalChip` footer.**
Top padding of 8 dp matches the existing chip padding (Modifier.padding(top = 4.dp) in
`CategoryItem` + the chip's own 5 dp internal padding ≈ 9 dp total; the bar row uses 8 dp to keep
optical alignment tight with the single-line title above).

---

### 2.3 All five states — home grid

#### State 1 — Loading (skeleton)

When `CategoriesUiState.Loading` is emitted:

- The existing `ShimmerTileCard` is rendered for each of the 9 category slots.
- **F1 change:** when the flag is ON, the shimmer tile also carries a shimmer bar placeholder in
  the footer slot: a 4 dp high pill the full width of the content column, animated with
  `rememberShimmerBrush()`, clipped to `RoundedCornerShape(2.dp)`.
- When the flag is OFF: `ShimmerTileCard` is unchanged — no footer bar.
- The shimmer tile must match the real tile silhouette at the device's current font scale
  (existing `shimmerLineHeight` pattern). Because the bar is 4 dp fixed-height (not font-scale
  dependent), no sp-to-dp conversion is needed for the bar placeholder.
- Grid is **not scrollable** during loading (existing `userScrollEnabled = content != null`).

```
Loading tile (flag ON):
┌─────────────────────────────────────────────────┐
│ ◯  ████████████  (shimmer)                      │  icon circle + title line
│    ▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭  (shimmer) │  4dp bar placeholder (NEW)
└─────────────────────────────────────────────────┘

Loading tile (flag OFF) — unchanged from today:
┌─────────────────────────────────────────────────┐
│ ◯  ████████████  (shimmer)                      │
│    ▭▭▭▭▭▭▭  (shimmer chip)                      │  existing chip placeholder
└─────────────────────────────────────────────────┘
```

#### State 2 — Empty / never-practised tile

A tile whose subject has never been answered (coverage = 0, mastery = 0) in the current
licence. Stats are language-agnostic (PR #33): the answer-event log is keyed by
`(licenseId, categoryId)` only, so a question answered in Polish or English folds to one
mastery entry and progress survives a language switch — scoping is per licence, not per language:

- The bar track is rendered as a **dashed line** using `PathEffect.dashPathEffect` on a `Canvas`
  drawn with `MaterialTheme.colorScheme.outlineVariant`. This visually signals "nothing here yet"
  without looking like a filled 0% bar or a broken UI.
- The chip shows an **em-dash ("—")** in B612 Mono (`labelMedium`, `onSurfaceVariant`).
- No "0%" text anywhere — a zero rendered as a percentage reads as failure (DC-3).
- The tile is still fully tappable — the empty state is an invitation to start, not a disability.

**Why dashed and not hidden?** Hiding the bar entirely would make the tile silhouette different from
a practised tile on first render, causing a layout jump. Showing a dashed track keeps the tile
height stable while communicating "not yet started" without a number.

#### State 3 — Populated (practised tile)

A tile where `masteredCount > 0` and `totalQuestions > 0`:

- A filled LinearProgressIndicator from `masteredCount / totalQuestions.toFloat()`.
- Chip shows the rounded integer percentage, e.g. "62%", in B612 Mono.
- The fill colour is `MaterialTheme.colorScheme.primary` at alpha 0.7 (see §4 for contrast check).
  The track is `MaterialTheme.colorScheme.surfaceContainerHighest`.
- The bar width animates when the value changes (after returning from a session):
  - Reduced motion OFF: `animateFloatAsState(durationMillis = 350, FastOutSlowIn)` — consistent
    with the existing entrance animation spec.
  - Reduced motion ON: immediate (no animation). Use `rememberReducedMotion()`.
- Coverage is NOT displayed separately in F1. Mastery alone is sufficient for the MVP proof.
  Coverage data is captured by the domain layer (A2) and available for future use.

#### State 4 — Error / data corruption fallback

If the progress read fails for a given tile (corrupt DataStore entry, read exception):

- The tile renders identically to the **empty / never-practised state** (dashed track, "—" chip).
- No error colour, no error icon, no toast.
- The failure is logged silently (consistent with NFR-5 and existing `deserialize-and-log-on-failure`
  behaviour).
- Rationale: a practising user losing one tile's progress indicator is a minor degradation, not a
  blocking error. Surfacing it as an error would be disproportionate and risk the "no red shaming"
  principle.

#### State 5 — Flag-off baseline

When `FeatureFlag.Progress` is **off**:

- `CategoryItem` renders exactly as it does today: icon circle + title + `TonalChip` with the
  question count.
- `ShimmerTileCard` renders exactly as it does today: icon circle + title shimmer + chip shimmer.
- Zero code divergence at the callsite: the `TileProgressBar` composable is simply not called
  (replaced by the existing `TonalChip` footer).
- The ViewModel does not request progress data when the flag is off (data layer also gated — OQ-10).
- **The grid must be pixel-identical to the pre-F1 build.** QA can verify by diffing screenshots
  with the flag off against the last release screenshot.

---

### 2.4 Dynamics

**After a session (C1.3):**

The `CategoriesViewModel` observes a `Flow` from `AnswerEventRepository` (or a derived
`SubjectProgressRepository`). When new answer events are written, the flow emits updated mastery
values and the grid re-renders without a restart. The `combine()` pattern already in the ViewModel
absorbs an additional upstream flow cleanly.

**Licence switch (C1.4):**

`changeLicenseUseCase` triggers a new emission from `ObserveCategoriesUseCase` and the progress
flow is re-queried for the new licence. The ViewModel handles this by switching the scope passed
to `flatMapLatest` — the same pattern as today. From the UI perspective, the grid goes through a
brief Loading skeleton (or Content re-render if progress is in memory), then each tile shows the
new licence's progress. No special transition is needed beyond the existing entrance choreography.

**Reduced motion:**

- Bar fill animation: guarded by `rememberReducedMotion()` — `if (reducedMotion) progress else
  animatedProgress`.
- Entrance choreography: already guarded by `rememberReducedMotion()` — no change needed.

---

## 3. Component specifications

### Component: TileProgressBar

**Purpose:** Renders the mastery progress signal inside an `AppTile`'s footer slot. Stateless;
receives a sealed `TileProgressState` and emits nothing (it is display-only — the tap target is
the entire `AppTile`).

**Anatomy:**

```
Row(verticalAlignment = CenterVertically, modifier = Modifier.fillMaxWidth()) {
  ├── LinearProgressBar (custom canvas draw — see below)  [weight(1f)]
  └── Spacer(8.dp)
  └── TonalChip(text = label)
}
```

**Bar rendering detail:**

The bar is a `Canvas` drawn at `fillMaxWidth()`, `height = 4.dp`. This replaces Material3's
`LinearProgressIndicator`, which has a mandatory 4 dp corner radius and a minimum width contract
that makes it awkward to compose. A raw `Canvas` draw takes 3 lines and gives full control:

```
// Pseudo-code for illustration — not production code
drawRoundRect(
  color = trackColor,   // surfaceContainerHighest
  cornerRadius = CornerRadius(2.dp.toPx()),
  size = Size(canvasWidth, 4.dp.toPx()),
)
if (state is Populated) {
  drawRoundRect(
    color = fillColor,  // primary.copy(alpha = 0.70f)
    cornerRadius = CornerRadius(2.dp.toPx()),
    size = Size(canvasWidth * animatedFraction, 4.dp.toPx()),
  )
} else {
  // Empty or Error: draw dashed segments over the track
  drawLine(
    color = outlineVariant,
    pathEffect = PathEffect.dashPathEffect(floatArrayOf(6.dp.toPx(), 4.dp.toPx())),
    strokeWidth = 4.dp.toPx(),
    start = Offset(0f, 2.dp.toPx()),
    end = Offset(canvasWidth, 2.dp.toPx()),
  )
}
```

**Why not `LinearProgressIndicator`?** The existing `TonalChip` establishes that the app uses raw
Material shapes for metadata pills rather than adopting `AssistChip` or `FilterChip`. The same
reasoning applies here: `LinearProgressIndicator` carries an indeterminate animation, a minimum
width, and a color contract that are all awkward in a 4 dp inline bar. The raw canvas draw is 10
lines, zero state, and gives the exact dashed-line empty state without a custom `drawBehind`.

**Variants:**

| Variant | `TileProgressState` | Bar | Chip label |
|---|---|---|---|
| Empty | `TileProgressState.Empty` | Dashed, `outlineVariant` | `"—"` (em-dash) |
| Populated | `TileProgressState.Populated(fraction)` | Filled, `primary@70%` on `surfaceContainerHighest` | `"${(fraction * 100).roundToInt()}%"` |
| Error | `TileProgressState.Error` | Same as Empty | `"—"` |
| Loading | Not a `TileProgressState` — caller renders `ShimmerTileCard` instead | — | — |

Note: Loading is NOT a variant of this composable. The caller (`CategoriesGrid`) renders
`ShimmerTileCard` for the entire tile while loading; `TileProgressBar` is only instantiated when
`CategoriesUiState.Content` is present.

**States (within Populated):**

- `fraction = 0.0` — this state must not be reached. A subject with `masteredCount = 0` but
  `totalSeen > 0` (you've tried every question and gotten them all wrong) returns `Populated(0.0f)`.
  The bar would render as empty. This is intentional: a zero-mastery bar (fully unfilled track, no
  fill) is visually identical to the empty state. The chip however would show "0%" — this is the
  one exception where the percentage is shown, because the user HAS practised (they've just not
  gotten anything right yet). The chip colour remains `onSurfaceVariant`, not red. The bar remains
  the neutral track only (no red fill). This treats the situation as "keep going" rather than
  "failed."
  
  Implementation note for the ViewModel: produce `TileProgressState.Populated(0f)` when
  `totalSeen > 0 && masteredCount == 0` so the screen renders "0%" rather than "—". Reserve
  `TileProgressState.Empty` for the true zero: `totalSeen == 0` (never practised).

**API:**

```kotlin
// Sealed state — defined in the UI layer, populated by CategoriesViewModel
sealed interface TileProgressState {
    data object Empty : TileProgressState        // never practised: show "—" + dashed track
    data class Populated(                        // at least one question seen
        val mastery: Float,                      // 0f..1f — masteredCount / totalQuestions
        val masteredCount: Int,                  // for content description
        val totalQuestions: Int,                 // for content description
    ) : TileProgressState
    data object Error : TileProgressState        // read failed — degrades to Empty visually
}

// The composable
@Composable
fun TileProgressBar(
    state: TileProgressState,
    modifier: Modifier = Modifier,
)
// No event callback: the bar is display-only.
// Caller wraps it in AppTile's footer slot — the tap target is AppTile, not this.
```

**Content rules:**

- Chip label max length: 4 characters ("100%"). Never truncates.
- At 1.5× font scale: `TonalChip` text grows; the `weight(1f)` bar shrinks to fill remaining
  space. Minimum bar width should be at least 48 dp — enforce with `Modifier.widthIn(min = 48.dp)`
  on the bar canvas.
- Long category names (German locale): title wraps to a second line in `titleMedium` (the
  existing `AppTile` allows this). The footer row is on a separate line, so wrapping the title
  does not affect the bar layout.

**Tokens consumed:**

| Token | Usage |
|---|---|
| `MaterialTheme.colorScheme.primary` @ `alpha = 0.70f` | Bar fill (populated) |
| `MaterialTheme.colorScheme.surfaceContainerHighest` | Bar track background |
| `MaterialTheme.colorScheme.outlineVariant` | Dashed line (empty / error) |
| `MaterialTheme.colorScheme.surfaceContainerHighest` | `TonalChip` background (existing) |
| `MaterialTheme.colorScheme.onSurfaceVariant` | Chip label text (existing) |
| `LocalAppFonts.current.numeric` | Chip label font family (existing `TonalChip` pattern) |
| `MaterialTheme.typography.labelMedium` | Chip label text style (existing `TonalChip`) |
| `MaterialTheme.shapes.extraSmall` (8 dp corner) — or raw `2dp` radius on canvas | Bar end caps |

No new tokens are introduced. All values are existing theme tokens.

---

### Component: ShimmerTileCard (modified for flag-on)

The existing `ShimmerTileCard` private composable gains a `showProgressBar: Boolean` parameter
(default `false`). When `true`, the chip placeholder is replaced by a full-width 4 dp shimmer bar:

```
// Existing chip placeholder (flag OFF / default):
Box(modifier = Modifier.width(72.dp).height(chipHeight).clip(CircleShape).background(brush))

// New bar placeholder (flag ON):
Box(modifier = Modifier.fillMaxWidth().height(4.dp).clip(RoundedCornerShape(2.dp)).background(brush))
```

The `CategoriesGrid` passes `showProgressBar = progressEnabled` (where `progressEnabled` is the
observed feature-flag boolean from the ViewModel).

This keeps the shimmer tile height consistent with the real tile in both flag states, preventing
layout jumps during the skeleton → content transition.

---

## 4. Design tokens

No new tokens are required. All values are existing theme tokens. The table below is a reference
summary for the developer.

| Purpose | Token | Light value (ref) | Dark value (ref) |
|---|---|---|---|
| Bar fill | `colorScheme.primary @ 0.70f` | `#4A5AB8` @ 70% | `#AEB9FF` @ 70% |
| Bar track | `colorScheme.surfaceContainerHighest` | `#DCDFE C` | `#323337` |
| Dashed line (empty) | `colorScheme.outlineVariant` | `#C5C6D6` | `#3A3B42` |
| Chip background | `colorScheme.surfaceContainerHighest` | (same as track) | (same as track) |
| Chip text | `colorScheme.onSurfaceVariant` | `#45475A` | `#A6A8B0` |
| Chip font | `LocalAppFonts.current.numeric` (B612 Mono) | — | — |

**Contrast check (bar fill against tile background):**

- Light: primary (`#4A5AB8`) at 70% over `surfaceContainer` (`#E8EBF7`). The effective colour is
  approximately `#7E89CE`. Against `#E8EBF7`, contrast ratio is approximately 3.8:1. This clears
  the 3:1 threshold for non-text UI components (WCAG 2.1 §1.4.11). The bar is 4 dp tall — it is
  not text, so 3:1 applies.
- Dark: primary (`#AEB9FF`) at 70% over `surfaceContainer` (`#1E1F22`). Effective colour
  approximately `#7A84CC`. Against `#1E1F22`, contrast ratio is approximately 6.1:1 — well above
  3:1.
- Both themes pass. No new token is needed.

---

## 5. ViewModel changes (design-level specification for Epic C)

The `CategoriesViewModel` and `CategoriesUiState` require the following additions. This is a
design-level specification; implementation details belong to the architect.

### 5.1 New state field

`CategoriesUiState.Content` gains a map from category id to progress state:

```kotlin
// Added to CategoriesUiState.Content:
val tileProgress: Map<String, TileProgressState> = emptyMap()
// key = Category.id
// absent key = TileProgressState.Empty (safe default)
```

`CategoriesUiState.Loading` does not carry progress (skeleton shows for everything).

### 5.2 New upstream flow

The ViewModel adds a new upstream: `observeSubjectProgressUseCase(licenceId)` which
returns `Flow<Map<CategoryId, SubjectProgress>>` (domain type from Epic A/A2). Stats are
language-agnostic (PR #33): the use case dropped its language/`languageProvider` scoping and the
underlying `AnswerLogRepository.observeSubjectProgress(licenseId)` takes only `licenseId`. This is combined
with the existing `combine()` call. When the flag is off, this flow is replaced with
`flowOf(emptyMap())` so zero data work is done.

### 5.3 Flag observation

```kotlin
// Inside CategoriesViewModel.uiState combine():
featureFlagService.observe(FeatureFlag.Progress)
// When false: tileProgress = emptyMap() and shimmer shows no bar placeholder
// When true:  tileProgress populated from SubjectProgress read
```

The flag is a live upstream, so toggling it at runtime (D1.4) causes a re-emission without
restart.

### 5.4 Mapping to TileProgressState

```kotlin
fun SubjectProgress?.toTileProgressState(): TileProgressState = when {
    this == null               -> TileProgressState.Error
    totalSeen == 0             -> TileProgressState.Empty
    else                       -> TileProgressState.Populated(
        mastery        = masteredCount.toFloat() / totalQuestions.toFloat(),
        masteredCount  = masteredCount,
        totalQuestions = totalQuestions,
    )
}
```

---

## 6. Accessibility

### 6.1 Content description for the progress bar

The `TileProgressBar` is a sub-element of `AppTile`. The outer `AppTile` `Surface` is the
semantic node. The content description for the whole tile must include the progress information
so screen-reader users get it in one focus event.

**Strategy:** The `CategoryItem` composable passes a `semanticDescription` to `AppTile` (or uses
`Modifier.semantics { contentDescription = "..." }` on the tile). This description is constructed
from the tile's progress state:

```
Empty:         "Air Law. No practice recorded."
Populated 62%: "Air Law. 62% mastered. 28 of 45 questions correct."
Error:         "Air Law. Progress unavailable."
Loading:       The shimmer tile carries contentDescription = null (decorative skeleton).
               TalkBack will skip it or announce the tile count via list semantics.
```

The icon inside the 40 dp circle has `contentDescription = null` (it is decorative — the title
already names the subject). This is already the pattern in the existing `CategoryItem`.

### 6.2 Touch targets

- The entire `AppTile` `Surface` is the touch target. Its minimum height with the bar row is
  approximately: 16 dp (top pad) + titleMedium line (~20 dp) + 8 dp top pad + 4 dp bar + 16 dp
  bottom pad = 64 dp. Well above 48 dp at default font scale.
- At 1.5× font scale: titleMedium line grows to ~30 dp → tile height ~74 dp. Still above 48 dp.
- The `TileProgressBar` itself is not interactive and carries no touch target requirement.

### 6.3 Colour-only information

Meaning is never encoded by colour alone (DC-7):
- Empty state: dashed track AND "—" chip — two signals (shape + label).
- Populated state: filled bar AND percentage chip — two signals (fill level + label).
- Error state: same as empty — no distinct error colour (by design, DC-3).
- The bar fill uses the theme's `primary` colour, which already carries semantic weight in the
  design language (interactive accent). Using it for progress is consistent with existing usage
  (e.g. the Continue card's `PlayArrow` icon).

### 6.4 Font scale

- The `TonalChip` inside `TileProgressBar` uses `labelMedium` + `LocalAppFonts.current.numeric`.
  At 1.5× scale, the chip text grows. The chip does not truncate ("100%" is 4 chars).
- The bar is 4 dp tall (fixed, not font-scale dependent). It stays legible at all scales.
- The `weight(1f)` on the bar column ensures the bar shrinks rather than overflows if the chip
  grows unusually wide (very large accessibility font sizes).

### 6.5 TalkBack / VoiceOver traversal order

The progress state is embedded in the tile's `contentDescription`, not as a separate focusable
element. This means TalkBack reads: "[icon — ignored] Air Law. 62% mastered. 28 of 45 questions
correct." in a single focus event. The user does not need to navigate to a separate child element
to hear the progress. This is the correct pattern for sub-tile metadata that is not interactive.

---

## 7. Platform notes (Android / iOS divergence)

This feature has **no platform-specific divergence**. The implementation is 100% in `commonMain`.

- The `Canvas`-based bar draw uses `androidx.compose.ui.graphics.drawscope.DrawScope`, which is
  fully cross-platform in Compose Multiplatform.
- `rememberReducedMotion()` already has `expect`/`actual` implementations for both platforms.
- `PathEffect.dashPathEffect` is available in `commonMain` via `androidx.compose.ui.graphics`.
- No system gestures, no sheets, no back-navigation changes.

---

## 8. Open questions (for the developer / architect)

These are not design decisions but implementation ambiguities raised during spec writing.

| # | Question | Recommended answer |
|---|---|---|
| OQ-A | What is the exact `FeatureFlag` enum name and remote-config key for F1? | `FeatureFlag.Progress(key = "progress_enabled", default = false)` — mirrors the `Explanation` flag pattern. _(Decommissioned in PR #23: the flag was removed once the feature shipped; F1 is now permanently on.)_ |
| OQ-B | Does `SubjectProgress` carry `totalQuestions` (bank size) or only `totalSeen`? The mastery denominator should be the full bank size (so 28/45 means 28 of 45 in the bank are correct on last attempt), not just the questions seen. | Mastery denominator = the Polish superset count (`Category.questionCount(Language.PL)`). This makes 91% mean "91% of the full subject is correct on last attempt," which is the honest readiness signal. Stats are language-agnostic (PR #33), so mastery is computed against the Polish superset regardless of the active language — not a language-parameterised bank size. Use `totalQuestions = category.questionCount(Language.PL)` from the already-loaded `Category`. |
| OQ-C | Should coverage (% seen) be shown as a secondary indicator in F1? | No — mastery alone is cleaner for the MVP. Coverage data is captured (req A2) and available for a future secondary bar segment if desired. |
| OQ-D | When the flag is first turned ON (no history yet), every tile shows the empty state. Is there any onboarding hint needed? | No. The empty state "—" is self-explanatory: it means "start practising to see progress." No tooltip or empty-state copy is needed on the tile itself. If a future pass adds onboarding, it goes on the mode-selection screen ("Your progress will appear on the home screen after your first session"). |
| OQ-E | What is the update cadence for progress after a session? | The ViewModel observes a `Flow` from the progress repository. Progress should update as soon as the session result is written (C1.3). The flow should emit within 1 session-write cycle — no polling, no manual refresh. |

---

## 9. Acceptance criteria mapping

Each C1 acceptance criterion maps directly to a state or behaviour in this design:

| Req | Criterion | Satisfied by |
|---|---|---|
| C1.1 | Practised tile visually differs from never-practised | Populated state (filled bar + %) vs. Empty state (dashed + "—") |
| C1.2 | Empty state doesn't read as failure, no misleading "0%" | Em-dash chip + dashed track; "0%" only when `totalSeen > 0 && masteredCount == 0` (you've tried) |
| C1.3 | Progress updates after session without restart | ViewModel observes `Flow` from progress repository |
| C1.4 | Licence switch re-renders to new licence's progress | Existing `flatMapLatest` on licence change + progress flow scoped to active licence |
| C1.5 | Loading: skeleton only, no stale/wrong progress | `ShimmerTileCard` replaces entire tile during Loading; `TileProgressBar` not instantiated |
| D1.1 | Flag off = today's grid, byte-for-byte | `TileProgressBar` not called; `ShimmerTileCard` unchanged; `TonalChip` with question count shown |
| D1.2 | Flag on = full F1 behaviour | `TileProgressBar` replaces question count chip |
| D1.3 | Debug screen ForceOn/ForceOff | Existing `FeatureFlagOverrideStore` / `DebugViewModel` pattern, unchanged |
| D1.4 | Flag toggle at runtime, no restart | `featureFlagService.observe()` as upstream in `combine()` |
| D1.5 | Flag first turned on = empty state, no crash | `TileProgressState.Empty` for all tiles when no history — no migration needed |

---

## 10. Future surfaces (out of scope for F1 — noted for continuity)

The following features will build on the progress signal this spec defines. They are listed so the
developer can design the domain API generously now without over-specifying UI that is not in F1.

- **Category hub / mode selection:** the 4-dp bar graduates to a full progress ring here. The
  `TileProgressState.Populated.mastery` fraction is reusable without change.
- **Readiness indicator (home):** once exam-attempt aggregation exists, the tile gains a secondary
  "at exam standard" indicator (e.g. a coloured dot beside the bar, not replacing it).
- **Coverage as secondary bar segment:** the track could split into two segments (mastered in
  solid primary, seen-but-not-mastered in primary@30%). This requires only a `coveredFraction`
  field added to `TileProgressState.Populated`.
- **Question count on mode selection:** the audit recommends moving the raw count off the home
  tile to mode selection. In F1, the count is replaced by progress. In a future pass, mode
  selection prominently shows the count above the mode cards.
