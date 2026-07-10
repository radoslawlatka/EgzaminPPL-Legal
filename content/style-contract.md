# Explanation Style Contract — v2

**Frozen artifact.** This is the shared conditioning injected into *every* explanation-generation
request, in parallel or not. Consistency across the corpus comes from this spec being identical for
all calls — not from the order they run in.

**Freeze discipline:** treat this file as versioned. Bumping the version (e.g. v1 → v2) means past
explanations were written against a different spec; re-run the editor/normalization pass (see
`content-pipeline.md` → *Consistency at scale*) so the corpus stays uniform. Record the version used
in your generation run.

> **v2 changelog:** an explanation must now *explain the why*, not just restate the fact (§1, §2); the
> default is *right-sized*, not capped at one short paragraph (§3); a quantitative concept states the
> **general formula with a legend first** and a worked example only after (§5); sources are **expected**
> for regulatory/authority-defined facts (§7); images are tied to **visual/spatial concepts** and are
> requested for the visual subjects, not suppressed to near-zero (§8). A per-subject **expert profile**
> (persona + canonical sources + whether the subject is visual) is supplied with each request — write as
> that subject's expert.

Companions: golden exemplars (`golden-examples.md`) and the term glossary (`glossary-pl-en.csv`).
Both are part of the same frozen set and must be supplied alongside this contract.

---

## 1. What an explanation is

A short, self-contained description of the **concept** a question tests. It is reused across rephrased
questions and across surfaces (after a wrong answer, after a correct answer, on review screens), so it
must stand on its own.

**Three requirements sit above every other rule in this document: an explanation must be *technically
correct*, *clear*, and must *explain why* — not merely restate the fact.**

- **Correct** is non-negotiable — every fact, number, unit, and rule must be right, because this content
  supports real flight-exam preparation. Write as the subject's expert, from verified official sources.
- **Clear** means plain wording and a clean structure a learner grasps on the first read.
- **Explains why** means it gives the *reason or mechanism* behind the fact — the physical cause, the
  regulatory purpose, the governing formula, or a concrete example. A learner who reads only the rule has
  learned nothing they can transfer to a reworded question.

Use plain language, but say **as much as the concept genuinely needs** to make the *why* land — no more,
no less. When clarity and accuracy pull apart, keep the accuracy and add the few words needed to stay
clear — **never simplify to the point of being wrong, and never shrink an explanation down to a
paraphrase of the correct answer.**

## 2. Voice (non-negotiable)

- Explain the **concept/topic itself** — never "why the answer is right" or "why you were wrong". The
  *why* you give is the concept's own cause or purpose, not a comment on the options.
- **Give the reason, not just the rule.** Explain the mechanism or purpose behind the fact: *why* it is
  so. A sentence that only rephrases the correct option in other words **is not an explanation** — it is
  the single most common failure to avoid. If an authoritative source settles the point, state it and
  cite it; if none is at hand, give the sound physical or operational reasoning.
- **Never address the reader.** No "you", "your", "remember", "note that". Write in the third person /
  impersonal.
- **Concise, not minimal.** Cut filler and throat-clearing — but include the reasoning, a formula, or an
  example when they make the concept land. Length follows substance: a self-evident fact may need one
  sentence; a mechanism or a calculation needs the room to show it. Never pad; never starve.
- **Simple, everyday language.** Use the plainest wording that is still correct — short sentences, common
  words, one idea per sentence. Reach for a technical term when it *is* the concept, and keep it
  consistent with the glossary.
- **Neutral and plain.** Standard aviation register; no marketing tone, no exclamation, no rhetorical
  questions.
- **Standalone.** Do not reference "this question", "the options", "answer B", or the exam.

## 3. Length and structure

- **Right-sized, not fixed-length.** Most explanations are a short paragraph or two. A self-evident fact
  may be one sentence; a concept that needs a reason, a formula, and an example will be longer. Let the
  *why* set the length — there is **no word cap**, and there is no minimum that justifies padding.
- Add structure when it earns its place, in this order of preference:
  1. a **formula** block (with its legend) when the concept is quantitative;
  2. a **concrete, real-world example** woven into the prose ("for instance, …") when it makes the
     concept easier to grasp;
  3. a single **key takeaway** (`>`) when there is one rule worth isolating;
  4. a **heading** (`##`) only when the explanation has two or more distinct parts;
  5. **Sources** — see §7 (expected for regulatory/authority-defined facts).
- Never use a heading for a single-paragraph explanation. Never stack multiple takeaways.

## 4. Markdown blocks (the only allowed syntax)

| Block | Syntax | Use |
|---|---|---|
| Paragraph | plain text | the concept |
| Bold / italic | `**term**`, `*word*` | bold the concept's **key term(s)** — including the central term the topic names — to anchor the reader; use readily, but never bold whole sentences |
| Sub / sup | `<sub>x</sub>`, `<sup>2</sup>` | indices and exponents |
| Heading | `## Title` | only to split a multi-part explanation |
| Key takeaway | `> one line` | one isolated rule |
| Formula | ` ```formula ` … `alt: …` … ` ``` ` | quantitative relationships (general form first, see §5) |
| Image | ` ```image ` … `key:` / `alt:` / `depicts:` … ` ``` ` | one picture for a **visual/spatial** concept (see §8) |
| Sources | `## Sources` then `- [label](url) — note` | authoritative references |

Not allowed: LaTeX, tables, raw HTML other than `<sub>`/`<sup>`, nested lists, blockquotes for anything
but a takeaway.

## 5. Formulas

- **Unicode only, single line** (rendered in a mono face). No LaTeX, no stacked fractions.
- Use proper symbols: `×` (not `x`) for multiply, `−` (U+2212) for minus, `·` for a dot product,
  `½`, `°`, `ρ`, `Δ`, exponents via `<sup>` or Unicode (`V²`), indices via `<sub>` (`C<sub>L</sub>`).
- Keep units in the expression with a normal space (`27 ft/hPa`, `1013 hPa`).
- **Every formula needs an `alt:` line** — a spoken-word reading in the **same language** as the
  explanation (a screen reader reads the alt, not the symbols).
- **A quantitative concept states the general formula first.** Give the relationship in *symbols* as the
  `formula` block, then a one-line **legend** defining each symbol **in order of appearance**, with an
  em-dash between symbol and meaning and `<sub>` for indices:
  `where: ρ — air density, V — true airspeed, S — wing area, C<sub>L</sub> — lift coefficient.`
  (PL: `gdzie: …`). Use the glossary terms. The general law is the **reusable** concept — these
  explanations are shared across reworded questions, so a bare arithmetic line tied to one question's
  numbers is *wrong* as the explanation.
- **A worked example is optional and comes *after* the general formula**, never instead of it. When the
  topic is a calculation, show the symbolic formula + legend, then (optionally) one worked line that
  substitutes representative numbers. **Never lead with `55 kg × 2,3 m = 126,5 kgm` and no general form.**

## 6. Numbers, units, language parity

- **PL and EN are the same explanation** — same structure, same blocks, same formulas, faithful
  translation. Generate both for licenses that support `en` (`ppl_a`, `ppl_h`); `pl` only for `spl`, `bpl`.
- **Decimal separator:** Polish uses a **comma** (`1,5`), English a **point** (`1.5`). Apply per language.
- **Unit symbols are identical in both languages** and keep their standard form: `ft`, `kt`, `hPa`,
  `NM`, `°C`, `m`, `kg`. Don't translate or pluralise symbols.
- **Q-codes and ICAO abbreviations stay untranslated:** `QNH`, `QFE`, `IAS`, `TAS`, `METAR`, `ATC`.
- Use the **glossary** (`glossary-pl-en.csv`) for every domain term — it is the single source of truth
  for PL↔EN term pairs. Do not invent alternative translations.

## 7. Sources

- **Cite the governing source for any regulatory, procedural, or authority-defined fact.** Air Law,
  Operational Procedures, Communications, and specific limits/definitions should name where the rule
  comes from — use the **canonical sources from the subject's expert profile** supplied with the request
  (e.g. the Polish *Ustawa — Prawo lotnicze*, SERA, EASA Part-FCL / Part-NCO, the relevant ICAO Annex,
  the ULC syllabus). Self-evident physics (lift rising with the square of speed) needs no citation.
- Cite **authoritative** references only: *Prawo lotnicze*, EASA, ICAO, SERA, ULC. Never cite a forum,
  a blog, or an unofficial summary.
- Format: `- [Label](url) — short note`. **Include a URL only if it is a stable, known link** — otherwise
  leave the parentheses empty rather than inventing one (e.g. `- [Prawo lotnicze, art. 2]() — definicja`).
- The marker is fixed: the ingester recognises only `## Sources` (English) — keep that literal heading
  even in Polish; the app renders a localized label. Translate only the link text and the note.

## 8. Images — for visual concepts, requested in place, never minted as a URL

**Match the image to the concept, not to a quota.** A *visual or spatial* concept — a shape, a force or
vector diagram, a curve, a component layout, an instrument, a cloud type — is genuinely easier to grasp
with a picture, so **include exactly one** for it. A purely textual or abstract concept — a rule, a
procedure, a definition, a bare calculation — needs none; never add a decorative image. **The subject
expert profile states whether the subject is a visual one — follow it.** Never more than one image per
explanation.

When an image is warranted, **request it in place** with a fenced ` ```image ` block — and **never write
an image URL** (the text pass would hallucinate it):

````
```image
key: empennage-components
alt: Aircraft empennage — horizontal and vertical stabilizers, elevator, rudder
depicts: the tail assembly parts and their role in stability and control
```
````

- **`key`** — a stable, descriptive slug; use the **same key in PL and EN** (and across questions that
  should share the same picture). **`alt`** — a screen-reader description (mandatory). **`depicts`** — what
  the picture must show, for whoever produces it later. **No `url`.**
- The block is stored as a **pending** image. The app **shows nothing for it until a picture is attached**,
  so a pending image never breaks a layout. URLs are backfilled later, keyed by `key`, with no re-run of
  the generator. See `content-pipeline.md` → *Consistency at scale*.
- **Visual identity is fixed** — describe *what* to show, never the style. Every illustration follows
  `image-style.md`; in particular **any aeroplane is always the photorealistic Cessna 152 _SP-KOK_**
  (references in `docs/content/reference/`: real photos `sp-kok.png`, `sp-kok-2.jpg`).

## 9. Decoding (set on every request)

- **Pinned model version** across the whole run.
- **Low temperature** (≈ 0.2) and fixed `top_p` — high temperature is literally more stylistic variance
  per call.
- Supply this contract as the system instruction, the **subject expert profile** for the question, and
  the golden exemplars as few-shot examples on **every** request.

## 10. Anti-patterns (reject in review)

- **A restatement** — an explanation that only paraphrases the correct answer without giving a reason,
  mechanism, formula, or example. The most common and most important failure to catch.
- **A worked calculation with no general formula or legend** — numbers without the law they instantiate.
- **Anything technically wrong or imprecise** — incorrect facts, numbers, units, or rules, or an
  oversimplification that makes the statement false.
- **An unsourced regulatory/authority-defined claim** where the governing instrument should be named.
- **Needless complexity** — jargon, long clauses, or abstraction where a plain word or a short sentence
  would do. (Concise ≠ minimal: cut filler, keep the reason.)
- "You", "your", "remember", "keep in mind", "as you can see".
- "The correct answer…", "Option B…", "In this question…".
- LaTeX (`\frac`, `$…$`), tables, emoji, exclamation marks.
- Hedging filler: "It is worth noting that", "Generally speaking", "In essence".
- Invented URLs or non-`https` media.
- PL/EN that diverge in structure, numbers, or formulas.
