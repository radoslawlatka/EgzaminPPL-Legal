# Hero Screens UI Audit — LicenseSelection & Categories (Home)

- **Date:** 2026-07-01
- **Build audited:** branch `refactor/pr37-review-cleanup` @ `85bafd7`, debug build on a Pixel emulator (1080×2400, API 36)
- **Scope:** the two hero routes rendered transparently over the app-lifetime "Night Flight" horizon shader — `AppRoute.LicenseSelection` and `AppRoute.Categories` — plus the boot splash and licence dropdown, in both themes
- **Method:** live screenshots (dark + light) captured on-device, then a 44-agent audit across seven dimensions (animated background, colour, typography, layout/components, iconography, motion, accessibility) plus a competitive benchmark. Every finding was independently re-verified against source and by re-sampling pixels from the screenshots. **30 findings confirmed, 6 rejected.** Quality bar: best-designed apps on Google Play / App Store (Duolingo, Headspace, Flighty, Arc, Apple Sports, Airbnb).
- **Evidence:** live on-device captures (dark + light) at 1080×2400 — the pixel coordinates quoted below refer to those originals

> File/line references are to the audited commit and will drift.

## Verdict

The **craft layer is at the top-app bar** — the shader, the B612 cockpit typography system, the skeleton loaders, the entrance choreography, and the render-loop engineering are Flighty/Headspace-grade. Every measured *text* pairing passes WCAG AA in both themes. What holds the screens back is a set of fixable execution gaps (two high severity) and a missing *product* layer: the home reads as a beautifully finished navigation list rather than "my training status."

## What's already at the bar (keep)

- **Shader authorship:** bowed horizon, two incommensurate sway periods so the drift never metronomes, temporal dither that measurably kills OLED banding, per-theme vignette, and the `underDim` mechanism that quiets the glow under content. The clock pauses when no hero screen is visible and resumes at the same phase; an opaque cover outside the per-screen fade-through means the scene never leaks between opaque screens.
- **Hand-tuned text contrast passes everywhere:** tile titles 13.4:1 dark / 13.8:1 light; count chips 5.3:1 / 6.9:1; on-sky headers 8.4:1 / 5.1:1; dropdown text 6.1–12.0:1.
- **Type system with intent:** B612 on display/headline/title slots, B612 Mono for instrument-style numerics, delivered via `AppFonts` CompositionLocal; the squared-bracket "PPL[A]" wordmark reads as instrument-panel character.
- **Polish copy safe:** no truncation risk (long titles wrap; all four Polish plural forms implemented); all text in sp; skeletons derive line heights from real type metrics so they track font scale.
- **Micro-craft:** press-scale on every tile, once-per-back-stack-entry staggered entrance that respects reduced motion, merged tile semantics for screen readers, invisible keyed scroll anchor so the late-prepended Continue card can't land above the viewport.
- **Correct insets** on both grids (status bar, gesture nav), transparent edge-to-edge system bars with theme-following icon contrast.

## High-severity findings

### H1. Nine category colours are theme-blind stock swatches and fail contrast

`CategoriesScreen.kt:570` hardcodes Material-500/600 hexes used for both the glyph tint and the 16%-alpha circle, identically in both themes; `LicenseVisuals.kt:15` reuses four of them. Measured glyph-vs-circle ratios: **light** — Human Performance `#FB8C00` **1.77:1**, Communications `#FF7043` 2.00:1, Principles of Flight `#00ACC1` 2.00:1, Navigation `#43A047` 2.39:1, Flight Planning `#1E88E5` 2.61:1; **dark** — Operational Procedures `#3949AB` **1.94:1**, Aircraft General Knowledge `#8E24AA` 2.15:1 (visibly muddy next to Human Performance orange at 5.26:1). Nine unrelated saturated hues also read as noise against the tuned dusk palette.

**Fix:** per-theme tonal pairs delivered via CompositionLocal, the pattern `QuizColors` already uses. Dark ≈ tone-80 glyphs (`#BAC3FF` indigo, `#E0B6FF` purple, `#FFB870` orange → 4.3–6.7:1); light ≈ tone-40/700 (`#BF360C`, `#00695C`, `#1976D2` → 3.3–4.6:1). Constrain all nine to a narrow chroma/tone band warmed toward the horizon-amber/periwinkle brand axis; apply the same pairs to `LicenseVisuals`.

### H2. Licence selection → home is a zero-frame hard cut

`App.kt:57` wraps the NavHost in `key(state.startRoute)`; selecting a licence flips the start route synchronously, which destroys and recreates the NavHost. Navigation-compose doesn't animate an initial destination and `AppFrame`'s crossfade is deliberately keyed on state *class*, so the greeting vanishes in one frame on the app's most emotional first-run action. Every other screen change gets a 300ms fade-through.

**Fix:** remember the initial start route for `startDestination`, drop `key()`, and react to the in-session switch with `navController.navigate(AppRoute.Categories) { popUpTo(AppRoute.LicenseSelection) { inclusive = true } }`. Both routes are transparent over the same horizon, so the existing fade-through plays over a continuous sky and the home entrance stagger becomes the reward for the choice.

## Medium findings

### Background & performance

- **The signature glow band is ~90% occluded.** `horizonY = 0.34` lands exactly under the tile rows on both layouts; the band survives as one ~30px sliver in a single inter-tile gap, screen-fixed while tiles scroll through it — it reads as a smudge, not a scene. *Fix:* make `horizonY` a uniform and settle it per route (~0.13 on home, above the first tile; ~0.25 on LicenseSelection; 0.34 for boot), animated ~700ms FastOutSlowIn — the horizon "settling as you land on a screen" is on-brand.
- **Full-screen redraw at up to 120Hz** for motion imperceptible above ~30fps (fastest scene motion: shimmer at `t*1.5` where `t = iTime*0.10`; pitch periods ~28s/75s), plus a new `ShaderBrush` allocated per frame (and a new Skia shader object per vsync on iOS). *Fix:* quantize the clock to ≥33ms steps inside the frame callback (dither keeps working) and hoist the Android brush into `remember {}` — 2–4× GPU cost reduction on the screen users idle on, no visible change.
- **The Android < 13 gradient fallback breaks the design intent:** `glowCore`/`glowWarm` as full-strength, full-width stops (~3× the shader's rest intensity) with no vignette — on exactly the devices that can least afford it. *Fix:* pre-mix the band stops to rest intensity (`lerp(skyHorizon, glowCore, 0.35)`), draw the band as a radial gradient for horizontal falloff, add one radial scrim rect scaled by `palette.vignette`.

### Layout & scroll

- **Tiles guillotine at an invisible line under the transparent top bar.** Bar and grid are Column siblings; the grid clips at its own top bound (8dp padding), the bar draws no surface or scrim, and there's no scroll-under response. Mid-drag frames shear cards flat against the sky. *Fix:* overlay the bar in a Box, add bar height to the grid's top contentPadding, and mask the grid's top edge with a ~24dp `DstIn` alpha fade so tiles dissolve into the sky as they pass under. Apply to both hero screens.
- **LicenseSelection leaves 37% of the screen dead** below the four rows (block hangs from a fixed 72dp top pad; measured last-row bottom y=1516/2400). *Fix:* centre the static block with weighted spacers (≈1f above / 1.4f below) and anchor a reassurance caption at the bottom inset ("You can change this later in Settings" / "Możesz to później zmienić w ustawieniach").
- **Three left edges in the first 150dp of home:** wordmark ≈34dp, section header ≈21dp, cards 16dp. *Fix:* drop the title Box horizontal padding in `FloatingTopBar` from 12dp to 0 — wordmark lands at 20dp, matching the header column; two intentional edges remain.

### Typography

- **No weight contrast anywhere — by accident.** B612 ships only Regular and Bold; every M3 Medium-weight slot (tile titles `titleMedium`, headers `titleSmall`, wordmark `titleLarge`) silently resolves to Regular (Compose only fake-bolds ≥W600). *Fix:* pin deliberately in `appTypography` — `titleMedium` → Bold for tile titles; explicitly `Normal` elsewhere; optionally bold only the accent span of the wordmark for a two-tone, two-weight lockup.
- **Same string, two typefaces, 24dp apart:** the B612 anchor "PPL[A]" (squared parens) sits directly above the dropdown's Roboto "PPL(A)" (round parens) because `SelectorDropdown` titles use `bodyLarge`. *Fix:* style dropdown option titles with the heading face (`bodyLarge.copy(fontFamily = LocalAppFonts.current.heading)` or `titleMedium`).

### Light theme (dark passes all of these)

- **Top-bar accent "PPL[A]" fails AA over the day sky:** primary `#4A5AB8` straight on the shader, measured **3.14:1** (22sp regular is below the large-text threshold, so 4.5:1 applies) — and it's the tappable licence-switch affordance. *Fix:* on-sky variant ≈`#2F3E9E` (4.7:1 worst-case) or back the title with the same `surfaceContainer` pill the actions get.
- **Cards dissolve into the ground** below the horizon (1.03–1.11:1 vs gaps/ground). *Fix:* 1dp `outlineVariant` border, or 1dp shadow, or deepen `ground` to ~`#EDEFF4` with white cards.
- **Warm glow band reads as a beige stain** between the cool indigo cards. *Fix:* day-tune the band the way vignette already is — ~50% amplitude, pull `glowWarm` toward `#F0D3BC`.
- **Vignette darkens whole edges, not corners** (`uv.x*uv.y*(1-uv.x)*(1-uv.y)` is zero along all edges): top edge centre darkened to 80%, grey arcs on the white ground at the bottom. *Fix:* centre-distance falloff (`1 - smoothstep(0.55, 0.95, length((uv-0.5)*float2(1.0,1.3)))`) or gate the vignette to the sky in light mode. *(Severity: low.)*
- **Mastery-bar fill 2.73:1** vs its track (`primary @ 0.70f`), below the 3:1 non-text minimum; appears for every returning user. *Fix:* theme-aware alpha — keep 0.70f dark, ≥0.85f (or 1f) light. *(Severity: low.)*

### Motion

- **Predictive-back onto home previews a flat surface, then the sky pops in after commit** (the opaque horizon cover is keyed to the *committed* top entry; the code comment accepts the trade-off). *Fix:* gate the cover on visible entries for the pop direction — `coverHorizon = topEntry not hero && visibleEntries.none { it.isHeroRoute() }`; the push direction stays covered by each opaque screen's own background. The horizon becomes part of the predictive-back preview — the payoff this architecture was built for.
- **Skeleton → content handoff drops to a blank grid for up to ~0.5s** on slow loads: placeholders are removed the same frame content arrives while real tiles compose at alpha 0 with up to 440ms stagger delay. *Fix:* 150ms crossfade on the swap and cap the entrance delay (`(index*40).coerceAtMost(240)`) when arriving from a visible skeleton; keep the full cascade exclusive to the prefetched cold-start.
- **Boot "engine start" is under-choreographed in the foreground:** the glow surge fires at first composition (mostly playing under the home crossfade on cached boots), the paper plane exits via a plain fade, and LicenseSelection has no entrance stagger. *Fix:* (1) take-off exit for the glyph (`slideOutVertically` ~ -screen/12); (2) extract `entranceProgress`/`Modifier.entrance` into `ui/components` and apply to the greeting + 4 rows; (3) expose `pulse()` on `HorizonSceneState` and fire it when the app state becomes Ready. *(Severity: low.)*
- **Micro-interactions are serviceable, not top-app:** symmetric 120ms press tween (sluggish grab, no springy settle), chevron never rotates while its menu is open, licence switch cut-swaps the grid. *Fix:* asymmetric press (60ms linear in, `spring(0.75f, 400f)` out), `rotationZ` 0→180° tween(200) on expand, `AnimatedContent` keyed on `selectedLicense.id` with the existing fade-through spec. *(Severity: low.)*

### Accessibility

- **Dropdown rows have no selection semantics or role** — TalkBack hears four identical rows; the check icon is decorative. *Fix:* `Modifier.selectable(selected = …, role = Role.RadioButton, onClick = …)`.
- **Licence-switcher title has no role, action label, or expanded state** ("Egzamin PPL(A). Double-tap to activate" — activates what?). The localized "Change licence"/"Zmień licencję" string already exists. *Fix:* `semantics { role = Role.DropdownList; onClick(label = …) }` + expand/collapse state.
- **Zero `heading()` semantics on either screen** — no rotor/heading navigation; screen-reader users must swipe linearly through every tile. *Fix:* `Modifier.semantics { heading() }` on `HomeSectionHeader` and the greeting. Two lines.
- **Shimmer ignores reduced motion** (the only ambient animation not gated by `rememberReducedMotion()`), and the loading state is silent to screen readers. *Fix:* static brush under reduced motion; one semantics node with a localized "Loading" description + polite live region.

## Low findings (remaining)

- **The brand paper-dart never reappears in product iconography:** PPL(A) — the flagship row — uses stock `Icons.Outlined.FlightTakeoff` and Principles of Flight uses stock `Flight`, while PPL(H)/SPL/BPL already have bespoke glyphs. *Fix:* draw a custom 24dp PPL(A) glyph from the dart's geometry as the anchor of the set.
- **Glider glyph is optically off-centre and its wing collapses at 24dp** (ink bbox centre y=9.25 vs 12; the wing is a sliver triangle that degenerates to a hairline). *Fix:* translate down ~2.5–3 viewport units; redraw the wing as a parallel-edged 2-unit form. *(Verified medium by the skeptic; shipping as-is is visible on the first-run screen.)*
- **Two licence-switch affordances with different anatomy:** the home `SelectorDropdown` is text-only while Settings' `LicensePickerBottomSheet` carries the 40dp tinted icon chips users learn on first-run. *Fix:* add an optional leading-icon slot to `SelectorOption` (32dp circle, same `accentColor @ 16%` recipe), pass `license.visuals()`, and collapse the two pickers onto one implementation so they can't drift.
- **Wordmark has no wrap guard at accessibility font scales** (wraps at 200%, splitting brand from licence code). *Fix:* `maxLines = 1` + ellipsis as a floor, or Compose 1.8 `TextAutoSize.StepBased(minFontSize = 16.sp, maxFontSize = 22.sp)`.

## Rejected findings (adversarially killed — don't re-raise without new evidence)

| Claim | Why rejected |
|---|---|
| Predictive-back preview issue (background dimension duplicate) | Same finding as the confirmed motion one; duplicate filed from a second dimension |
| Count-chip mono outweighs tile titles | Documented intent (`TonalChip` KDoc: instrument-face counts); works in practice |
| "Hi!" greeting undersized for a ~55% empty screen | Pixel measurement showed the void is 37%, not 55%; size hierarchy is correct |
| Home is "nine identical rows, no hierarchy" | Overstated: glyph + hue + title wrap differentiate; the proposed fix (varying tile anatomy) judged unsound — but see Benchmark #5 for the verified version of the underlying instinct |
| Licence glyphs mix three visual languages | Key claim ("chunky filled helicopter") failed pixel verification — the set renders as consistent outlines |
| Hero text lacks a guaranteed contrast floor over the animated shader | Mathematically impossible for the shader to reach failing luminance under the text regions; text sits on opaque surfaces or the calm upper sky |

## Competitive benchmark — the product-layer gap

One line: the craft is at the bar; what's missing is the product layer of a great home — an identity moment, user-state content, and depth. Priority-ordered, respecting the quiet cockpit identity (no confetti):

1. **Home leads with inventory, not the pilot's state (L).** "608 questions" is the same for every user forever. The killer stat exists in the product model: ULC pass/fail is per-subject, so weakest-subject readiness *is* the user's status. A full-width "flight deck" status card between the top bar and the grid — B612 Mono numerics, overall readiness (min-of-subjects framing), weakest subject with one-tap CTA, optional days-to-exam. Flighty's status block rendered as a cockpit instrument. This is the single largest gap vs. every benchmark app, and the strategic piece of the home redesign.
2. **Depth/scene presence (M).** Tiles at ~92–94% alpha on hero routes only, a 0–24dp scroll-driven parallax on the horizon, and the scroll-aware top-bar treatment (also a confirmed finding). Makes the dusk scene feel inhabited rather than wallpapered.
3. **First-run theater (S–M).** Entrance stagger (system already exists), licence glyphs at 32–40dp in a 56dp circle (the bespoke icons *are* the identity — currently 24dp), optionally the splash plane handing off to the screen. The empty bottom half is fine once the top half has a moment.
4. **Voice (S).** Every hero string is generic UI-speak in an app whose background depicts a night flight. A restrained pilot register: time-aware greeting mirroring the scene ("Good evening" over the night shader / "Dzień dobry" over Day VFR), "Your subjects", resume-flight-plan framing. One pass over ~10 strings in both locales; verify Polish lengths.
5. **Exactly one loud tile (M).** A single "recommended" treatment derived from data already on screen (lowest mastery among started subjects, else first unstarted): container tinted ~8% accent or a "weakest subject" reason chip. One tile may be loud — that restraint keeps it Flighty rather than Duolingo.

## Suggested execution order

1. **Day one:** H1 (tonal colour pairs), H2 (kill the hard cut), the four a11y fixes — most of it mechanical.
2. **Light-theme pass:** on-sky accent colour, card separation, band tuning, vignette falloff, mastery-fill alpha.
3. **Scroll & scene staging:** top-bar overlay + edge fade, per-route `horizonY`, predictive-back cover gating, shader clock quantization, fallback fidelity.
4. **First-run + micro-motion polish:** LicenseSelection centring/stagger/caption, boot choreography, press spring, chevron, picker unification, PPL(A) glyph, glider glyph.
5. **Product layer (with the home-redesign effort):** flight-deck status card, depth/parallax, voice pass, recommended tile.

---

*Full per-finding evidence (measured ratios, pixel samples, verifier rationales) from the audit run is archived outside the repo; the findings above are the complete confirmed set.*
