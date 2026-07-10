# Image Style Spec — v1

**Frozen artifact.** The single visual identity for every illustration in the app, so images stay
consistent no matter who or what produces them. Companion to `style-contract.md` (copy): the text pass
only *requests* images (style contract §8); images are **produced and curated separately** under this
spec (see `content-pipeline.md` → *Consistency at scale*).

## 1. Illustration types

- **Aircraft illustrations** — photorealistic renders of the app's aeroplane (§2–§5).
- **Technical diagrams** — flat schematic figures for charts, airspace, forces, instruments, etc. (§6).
- **Organic illustrations** — natural subjects such as clouds, fog, and weather scenes; clean and
  realistic, with no fixed identity of their own.

Don't mix these styles in one image. (For *how* to produce each type, see the suggestions in §7.)

## 2. The aircraft — one fixed identity

Every illustration that shows an aeroplane depicts the **same** aircraft:

- **Type:** Cessna 152 — high-wing, single piston engine, strut-braced wing, fixed tricycle landing
  gear, two-blade propeller with a spinner.
- **Registration:** **SP-KOK** in **navy blue**, on both sides of the rear fuselage and under the wing.
- **Livery:** predominantly white with **deep navy-blue** trim — a swept navy flash under the
  nose/cowling (edged by a thin lighter-blue pinstripe), a slim navy cheatline running the length of the
  fuselage to the tail, navy wing-tips, and a navy cap at the top of the vertical fin. Small **red**
  details: a navigation light at each wing-tip and a beacon atop the fin.
- **Palette:** white + **deep navy blue** (primary) + a thin lighter-blue pinstripe accent + small red
  nav-light details. Registration is navy, **not black**. No other colours on the airframe.

**References** (in the repo, `docs/content/reference/`):

- `sp-kok.png`, `sp-kok-2.jpg` — **real photographs** of the actual aircraft; the source of truth for
  livery and realism.
- `cessna-152-sp-kok.jpeg` — a CGI 3D render; use for overall shape/angle only, **not** its colours
  (its registration is wrongly black) or its plastic finish.

> The photographs are third-party reference material for **internal style guidance only** — not for
> publication or shipping in the app (generated assets must be original).
> Credit: © Adam Nogly / skrzydla.org.

## 3. Render quality — photorealistic, beyond the reference

Match the realism of the **real photographs** (`sp-kok.png`, `sp-kok-2.jpg`). The CGI render is shape
guidance only and looks slightly plastic — **do not** reproduce that plastic look:

- Real materials: painted aluminium with subtle panel lines, glass canopy with realistic reflections,
  rubber tyres, a metallic spinner and propeller.
- Soft, even studio lighting with a soft contact shadow on the ground.
- **Avoid the "plastic toy / hard-CGI" look:** no oversaturated plastic surfaces, no hard rim glow, no
  cartoon or matte-clay shading.

## 4. Framing & background (identical across all aircraft renders)

- **View:** three-quarter front from the left, aircraft parked on the ground, gear down (the reference
  angle family).
- **Background:** clean, neutral **light-grey seamless studio** backdrop — no scenery, runway, or sky —
  so images sit cleanly on app surfaces in both light and dark themes.
- Aircraft centred, full airframe in frame, generous margin.

## 5. Keeping it consistent at generation

- **Pin the real photos** (`sp-kok.png`, `sp-kok-2.jpg`) as style/subject references (image-to-image,
  IP-Adapter, or a "reference image" input) on **every** aircraft generation.
- Same model version + a fixed seed family + the preamble below.
- **Human review** before an asset enters the library; assets are **reused per concept** (`repCode`),
  not regenerated per question.

**Prompt preamble (aircraft):**

> Photorealistic studio render of a white Cessna 152 high-wing light aircraft with **deep navy-blue**
> trim (swept nose flash, slim fuselage cheatline, navy wing-tips and fin cap) and **navy** registration
> "SP-KOK", parked, three-quarter front view from the left, neutral light-grey seamless background, soft
> studio lighting, soft ground shadow, realistic materials (painted aluminium, reflective glass canopy,
> rubber tyres, metallic propeller spinner). As realistic as a real photograph — not a plastic CGI model.

## 6. Technical diagrams (non-aircraft)

Flat, schematic, minimal: limited palette (the app blue + greys), clean even line weights, labels in the
explanation's language, transparent or neutral background. **Prefer curated/real diagrams** (ULC / EASA /
Wikimedia art in one visual family) over free generation. Not photorealistic.

## 7. Suggested generation approach (guidance, not a requirement)

*How* an asset is produced is up to whoever builds the library — these are recommendations, not rules.
What matters is the visual identity above (§2–§6); the engine is just a means to it. A reasonable default
is to match the tool to the subject:

| Subject | Suggested engine | Why |
|---|---|---|
| **Aircraft** (the SP-KOK render) | a reference-conditioned image model (Gemini "nano banana" / Imagen) with the real photos pinned — or a real 3D model (re-textured CGI asset / Blender) rendered to images | one recurring subject with an exact livery; reference conditioning keeps it on-brand, and a 3D model gives perfectly consistent renders from any angle |
| **Charts & technical diagrams** | code / vector (SVG or a plotting library), or curated real diagrams (ULC / EASA / Wikimedia) | precision, correctness, crisp at any size, and theme-able; generative models tend to get technical detail wrong |
| **Organic illustrations** (clouds, fog, weather) | a generative model under one fixed style (Imagen / nano banana), or curated real photos | forgiving natural subjects where generation excels; for clouds, CC-licensed photos are often the most accurate and cheapest |

Notes:

- All three still flow through the same image-library + resolver — only the engine differs.
- Apply a **human review gate** to anything generative; deterministic vector/curated assets rarely need one.
- The app renders 2D images, so a "3D model" only needs to be *rendered to an image* — keeping an actual
  3D asset is worth it only if you value perfectly consistent any-angle renders.

## Anti-patterns

- A different aircraft type or registration; black (not navy) registration; a livery in other colours.
- Scenery / runway / sky backgrounds, motion blur, or dramatic angles.
- Plastic-CGI, cartoon, sketch, or low-detail rendering for aircraft.
- Photoreal and flat-diagram styles mixed in one image.
