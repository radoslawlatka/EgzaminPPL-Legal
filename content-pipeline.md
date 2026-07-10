# Content Delivery Pipeline — Runbook

End-to-end process for getting a license's content live: from the raw ULC PDF, through
the explanation-authoring work done by content-ops (an **AI agent** for the first run),
right up to the Firestore uploads and go-live.

Two independent tracks feed Firestore, joined by a per-category **code → key map**. **Both QUESTIONS and
EXPLANATIONS are now license-agnostic** (storage dedup) — a question code identifies the same question in every
license (verified — see `docs/ulc-reference/shared-code-report.md`), so a question is stored **once per code** and
shared across PPL(A)/PPL(H)/SPL/BPL, as are explanations. The app serves questions from a shared top-level store
`/questions/{categoryId}/langs/{lang}/chunks` (the deduped union of all licenses' codes) **intersected** with a
per-license membership doc `/licenses/{lic}/categories/{cat}/membership/map` (`codesByLang`, in membership order).
The per-license question-chunk upload (`feed_generator.py`) is **no longer** the serving source for questions —
the shared store + membership maps are built and uploaded by the new `question_feed.py` (with `question_blocks.py`
pure transforms). All of this is keyed off the **union** question bank (`ulc_all.csv`) produced by
`validate_shared_codes.py`.

```
ULC PDFs ─┬─▶ question_feed.py  (shared store + membership) ─▶ Firestore: QUESTIONS
          │     (PL union + EN union, deduped)                /questions/{cat}/langs/{lang}/chunks
          │                                                   /licenses/{lic}/categories/{cat}/membership/map
          │
          ├─▶ feed_generator.py  (per license) ─▶ Firestore: license doc + per-(license,category) METADATA
          │     (code · supportedLangs · langDetails · examQuestionCount · examPassCount · …)
          │
          └─▶ validate_shared_codes.py ─▶ ulc_all.csv  (union of all licenses' codes, deduped)
                 │                            (NUMER · PYTANIE · ODP1=correct · ODP2–4)
                 ▼
        CONTENT-OPS  (AI agent, run #1)
        reads each code's question + answers,
        writes ONE explanation per code  (PL + EN)
                 │
                 ▼
        content.csv   (Codes · Explanation (PL) · Explanation (EN))
                 │
                 ▼
        explanation_feed.py  --questions-csv ulc_all.csv      (no --license)
          validate codes · compile blocks · upload · emit coverage
                 │
                 ▼
   Firestore: EXPLANATIONS + code→key MAP   (license-agnostic, top-level)
     /explanations/{repCode}/langs/{lang}
     /explanationMaps/{categoryId}
                 │
                 ▼
        coverage.csv ── review % ── iterate
                 │
                 ▼
   (later) semantic dedup pass  →  flip FeatureFlag.Explanation (Remote Config)
```

The two tracks are fully decoupled: re-authoring explanations never touches question chunks,
and re-uploading questions never touches explanations. One explanation upload covers every license.

---

## 0. Prerequisites (once)

- **Python** — every script auto-bootstraps a `.venv` in `scripts/` on first run and installs
  `scripts/requirements.txt` (`firebase-admin`, `pdfplumber`, `google-genai`). The venv ships
  the macOS system Python (3.9), so script code uses `from __future__ import annotations`.
- **Service account** — `service-account.json` at the **repo root** (scripts default to
  `../service-account.json`; override with `--service-account`). Never commit it.
- **License id** (questions only) — one of `ppl_a`, `ppl_h`, `spl`, `bpl`. Supported languages:
  `ppl_a` / `ppl_h` → `pl`, `en`; `spl` / `bpl` → `pl` only. Explanations are **license-agnostic** and
  take no license id.
- **Union question bank** (explanations) — run `python validate_shared_codes.py` once to parse all four
  Polish PDFs, verify cross-license code consistency, and emit `docs/ulc-reference/ulc_all.csv` (the
  deduped union of every license's codes). Part B uses this single CSV instead of a per-license one.

Run everything from the `scripts/` directory.

---

## Part A — Questions (already automated)

> **Serving store vs. metadata.** The app serves questions from the shared `/questions` store +
> per-license membership maps written by `question_feed.py` (Step 1b). `feed_generator.py` (Step 1)
> still owns the per-license **license doc** and per-(license,category) **metadata** doc; its old
> per-(license,lang) question-chunk upload is **superseded** for serving questions.

### Step 1 — Upload license + metadata docs

`feed_generator.py` does PDF → CSV → parse → upload in one pass (interactive `y/N` prompts; add `-y`
to skip them). Its currently-owned outputs are the per-license **license doc** (`code`, `supportedLangs`,
`langDetails`) and the per-(license,category) **metadata** doc (`examQuestionCount`, `examPassCount`,
`examDurationSeconds`, `questionCountByLang`). Its per-(license,lang) question-chunk upload still runs but
is **no longer the serving source** for questions (see Step 1b).

```bash
python feed_generator.py ../docs/ulc-reference/ppla.pdf   # license + lang derived from the filename
# add: --service-account /path/to/service-account.json    (if not at repo root)
#      -y                                                   (non-interactive)
#      --license / --lang                                   (override only for off-convention filenames)
```

What it does:
1. **PDF → CSV** via `ulc_pdf_to_csv_converter.py`, writing `ppla.csv` **next to the PDF**.
   Columns (`;`-delimited): `L.p. · NUMER · PYTANIE · ODP1 · ODP2 · ODP3 · ODP4`.
2. **Parse** — resolves each `NUMER` prefix to a category (see table below), one question per code.
   **`ODP1` is always the correct answer.** The translated (EN) banks list some codes twice — a verbatim
   repeat, or a *different* question mis-sharing the code — so each code is **deduped to one question**,
   keeping the occurrence whose `ODP1` matches the canonical `pl` bank (the sibling `pl` CSV, resolved
   automatically). `code` always stays equal to `NUMER`.
3. **Upload** — writes the per-license license doc + per-(license,category) metadata doc, and replaces the
   per-`(license, lang)` chunks under `/licenses/{lic}/categories/{cat}/langs/{lang}/chunks/{chunk}` (50
   questions per chunk). **Those per-license chunks are superseded for serving questions** — the app reads
   the shared `/questions` store instead (Step 1b).

> Part B does **not** use this per-license CSV — it works off the union `ulc_all.csv` from
> `validate_shared_codes.py` (which already parses every license's PDF).

For English questions (where the bank exists), re-run on the `_eng` PDF — e.g.
`python feed_generator.py ../docs/ulc-reference/ppla_eng.pdf` (the `en` lang is derived from the
filename). Other languages stay untouched.

### Step 1b — Upload the shared question store + membership

`question_feed.py` (with `question_blocks.py` pure transforms) builds the **license-agnostic** serving
store the app reads. It assembles the **Polish union** from all four PL banks plus an **English union**
from `ppla_eng` / `pplh_eng` (EN deduped against the canonical Polish answers), runs validation gates
(membership langs == `supportedLangs`; every membership code resolves; per-(license,category,lang)
parity; per-license distinct-code totals `ppl_a=2053`, `ppl_h=1619`, `spl=1354`, `bpl=903`), writes a
coverage/parity report, then performs a `y/N`-gated upload of:

- shared content → `/questions/{categoryId}/langs/{lang}/chunks/{chunk}` = `{ index, count, questions }`
  (50/chunk; one canonical reading per code, chosen by `LICENSE_PRIORITY`).
- per-license membership → `/licenses/{lic}/categories/{cat}/membership/map` =
  `{ schemaVersion, codesByLang: { pl: [...], en: [...] } }` — which codes each license includes, per
  language, in membership order. SPL and BPL are Polish-only, so their membership carries a `pl` key
  **only** (no `en`) → they get no English bank, by construction.

The app reconstructs a license's bank as (shared content for lang) ∩ (that license's membership), in
membership order.

> **Not yet run / not yet deployed.** `question_feed.py` has **not** been run, and the `firestore.rules`
> blocks for `/questions` + membership reads are **not** yet deployed. This is the implemented/current
> approach — the shared store is not populated/live data.

### Step 2 — Verify

`question_feed.py` prints a coverage/parity report (per-license distinct-code totals + per-(license,
category,lang) parity). Spot-check in the Firebase console that the shared content at
`/questions/{cat}/langs/pl/chunks` and the per-license `/licenses/ppl_a/categories/*/membership/map`
(`codesByLang`) look right. (`feed_generator.py` separately prints a per-category question count and
chunk count.)

---

## Part B — Explanations

### Step 3 — Prepare the authoring input

The AI agent's input is the **union `ulc_all.csv`** (from `validate_shared_codes.py`) — one row per
distinct code across all licenses, carrying everything an author needs per question:

| Field | Source | Use |
|---|---|---|
| `NUMER` | CSV | the code; becomes the `Codes` cell (one per row) |
| `PYTANIE` | CSV | the question to explain |
| `ODP1` | CSV | the **correct** answer |
| `ODP2`–`ODP4` | CSV | distractors — context for what the question is testing |
| category | derived from `NUMER` prefix | scope / assign work by subject |

No extra tooling is required for the first run. (A human content-ops team would instead get a
spreadsheet "worksheet" generated from this CSV — that exporter is a later addition; see *Open tooling*.)

Work **one category at a time** so you can ship and measure coverage incrementally.

### Step 4 — Author explanations (AI agent, run #1)

**Policy for the first run: one explanation per code. No grouping.** Semantic reuse ("rephrased
question, same meaning") is real but is a judgment best made *after* generation — see *Step 7*. Do not
cluster questions up front; mis-grouping attaches a wrong explanation to a question and is hard to QA.

For each unique `NUMER`, the agent writes an explanation **per requested language** following the
authoring contract below, and appends a row to `content.csv`.

**Output file — `content.csv`** (UTF-8; the ingester reads `utf-8-sig`). Header and columns
(matched by substring, so exact casing is flexible):

```csv
Codes,Explanation (PL),Explanation (EN)
PL010-0001,"Pierwszeństwo ma statek powietrzny po prawej stronie...","Right-of-way goes to the aircraft on the right..."
PL010-0002,"...","..."
```

- **`Codes`** — the single question code for this row (first run = one code per row).
- **`Explanation (PL)`** — required.
- **`Explanation (EN)`** — authored for the whole bank (the runner returns it in the same call). The app
  only surfaces it where a license supports `en` (`ppl_a` / `ppl_h`). May be filled in a later pass
  (coverage-first per language is fine).
- Leave the explanation blank to mean "not authored yet" — that code simply stays uncovered.

#### Authoring contract (what the agent must produce)

Each explanation is **Markdown in a small GFM subset**, compiled to typed blocks at ingestion. Supported
blocks:

| Block | Syntax |
|---|---|
| Paragraph | plain text; inline `**bold**`, `*italic*`, `<sub>…</sub>`, `<sup>…</sup>` |
| Heading | `## Title` |
| Key takeaway | `> one or more lines` |
| Formula | fenced ` ```formula ` … ` ``` ` with a `alt: spoken description` line (**alt mandatory**) |
| Image (pending) | fenced ` ```image ` … `key:` / `alt:` / `depicts:` … ` ``` ` — **no URL**; alt mandatory |
| Video | fenced ` ```video ` … `https://…` … ` ``` ` |
| Sources | `## Sources` then `- [label](https://url) — optional note` items |

Rules:
- **Formulas are plain Unicode** (rendered single-line in a mono face), **not LaTeX** — e.g.
  `Δh = (1013 − 1003) hPa × 27 ft/hPa = 270 ft`. Every formula needs a spoken `alt:`.
- **Video must be `https://`** download/CDN URLs (`gs://` will not load).
- **Images are judicious and URL-less.** Most explanations need none; add one only when a picture truly
  helps. Request it in place as a fenced ` ```image ` block carrying `key` / `alt` / `depicts` and **no
  URL** (style contract §8). It is stored as a **pending** image block; the app **hides it until a URL is
  backfilled** later, keyed by `key` — no generator re-run. See *Consistency at scale*.

#### Editorial voice (non-negotiable)

- **Technically correct first, then as simple as possible.** Every fact, number, unit, and rule must be
  accurate (this content supports real exam prep); within that, use the simplest language a learner can
  grasp on the first read — short sentences, plain words, minimal jargon. Never simplify into being wrong.
- Explain the **concept/topic itself**, not "why you were wrong" or "why this answer is right".
- **Never address the reader** — no "you" / "your".
- As **brief** as possible; neutral, plain wording; no formal filler.
- **Must stand alone / be reusable** — it may later be shown after a correct answer, on a review
  screen, or elsewhere, so it must not reference the question or the answer options.

#### Conditioning every request (consistency)

Supply the **three frozen artifacts** in `docs/content/` on **every** generation request, plus a pinned
model version and low temperature (≈ 0.2). They are what keeps a parallel run stylistically uniform —
see *Consistency at scale* below.

- `style-contract.md` — voice, structure, formatting, decoding (use as the system instruction).
- `golden-examples.md` — few-shot exemplars (the strongest consistency lever).
- `glossary-pl-en.csv` — controlled vocabulary and PL↔EN term pairs.

> Full prose reference with more worked examples: `content-authoring-guide.md` (design scratches).

#### Per-subject model routing (required)

The default model is **`gemini-3.1-pro-preview`**, but two subjects **must** run on **`gemini-2.5-pro`**:

| Subject | Codes (union) | Model | Why |
|---|---|---|---|
| `air_law` | 801 | **`gemini-2.5-pro`** | Largest subject; stable GA model, no diagrams needed |
| `human_performance` | 313 | **`gemini-2.5-pro`** | Second-largest text subject; aeromedical prose |
| *all other subjects* | 1373 | `gemini-3.1-pro-preview` | Default (incl. the four visual subjects) |

The reason is throughput, not just model fit: the request cap is **per model** (the preview model is limited
to ~250 requests/day — see *Quota* below), so pinning the two biggest text subjects to a second model gives
them their **own daily-quota bucket** and roughly doubles synchronous throughput. `gemini-2.5-pro` is a
stable, non-preview model that handles regulatory and human-factors prose well and needs no image
generation.

This routing is **automatic** — the runner applies it on both the synchronous and Batch paths (a Batch job
is bound to one model, so `batch-run` partitions the work and runs one job-pair per model). It is defined
once in `eg.SUBJECT_MODEL_OVERRIDES` (`explanation_gen.py`). `plan` prints the per-subject model column so
you can confirm it; pass `--no-subject-model-routing` to pin the whole run to `--model` (testing only).

#### Running it — the agent runner

`explanation_runner.py` automates this step: it applies the three artifacts as conditioning on every
call, generates all requested languages per code in one request, validates each draft against the block
compiler, and writes an **append-only result store** (`explanation_results/`) so runs are
resumable — re-run any time and only missing codes are generated.

> **License-agnostic, both languages from one call.** The runner works on the **union** bank
> (`ulc_all.csv`) and produces one explanation per code, reused across every license — no `--license`.
> It returns `pl` *and* `en` in a single request (the large shared prompt is billed once, not per
> language); the app reads whichever languages a license supports. So there is **no second EN pass**: the
> EN question bank (`ppla_eng.pdf` / `pplh_eng.pdf`) has the same `NUMER` codes and is redundant for
> explanations. The EN *question text* is still uploaded separately by running `feed_generator.py` on
> `ppla_eng.pdf` / `pplh_eng.pdf` (the `en` lang is derived from the filename) — that's unrelated. Pass
> `--langs pl` to skip EN.

Two paths:

**Iteration / small batches — online:**

```bash
python explanation_runner.py plan     --questions-csv ../docs/ulc-reference/ulc_all.csv
python explanation_runner.py generate --questions-csv ../docs/ulc-reference/ulc_all.csv --category air_law
python explanation_runner.py assemble --questions-csv ../docs/ulc-reference/ulc_all.csv
# generate needs a Gemini key: --api-key, or env GEMINI_API_KEY (Google AI Studio); or --vertex
```

**The ~2500-code bulk run — Batch API (≈50% cheaper, no rate ceiling):**

```bash
python explanation_runner.py build-batch   --questions-csv ../docs/ulc-reference/ulc_all.csv
#   → one JSONL per model (batch__gemini-2.5-pro.jsonl, batch__gemini-3.1-pro-preview.jsonl);
#   submit EACH on its stated model (see "Per-subject model routing" above); download the results JSONL
python explanation_runner.py collect-batch results.jsonl --questions-csv ../docs/ulc-reference/ulc_all.csv
python explanation_runner.py assemble       --questions-csv ../docs/ulc-reference/ulc_all.csv
```

> Fully automated alternative: `batch-run` submits, polls, and collects both rounds for you, partitioning
> by model and applying per-subject caching — no manual JSONL handling.

`assemble` writes `content.csv` (→ Step 5) plus `images_needed.csv` — the structured image
requests, for the image-library step.

> SDK: the runner uses **`google-genai`** (the current Gemini SDK). `plan`, `build-batch`,
> `collect-batch`, and `assemble` run **offline** (no key); only online `generate` needs one. The actual
> batch **submission** is a manual step — verify the JSONL against your SDK/Vertex version first.

### Step 5 — Validate + upload explanations

```bash
python explanation_feed.py content.csv \
    --questions-csv ../docs/ulc-reference/ulc_all.csv
# add -y to skip the upload confirmation
```

What it does:
1. **Validate** every code against the union question bank. **Fails loudly** on an unknown code (typo) or
   a code used on two rows — nothing uploads until the sheet is clean.
2. **Compile** each cell's Markdown into the typed block list; **lint** missing formula/image `alt` and
   non-`https` media (warnings, not fatal).
3. **Upload** (license-agnostic, top-level — one upload covers every license):
   - explanation docs → `/explanations/{repCode}/langs/{lang}` = `{ schemaVersion, blocks }`
   - per-category map → `/explanationMaps/{categoryId}` =
     `{ schemaVersion, entries: { code: repCode } }` — **replaced wholesale** (the sheet is the source of
     truth). Question chunks are untouched.
4. **Emit** `coverage.csv` (`code · category · hasExplanation · repCode · langs`).

### Step 6 — Review coverage, iterate

Open `coverage.csv`, check the covered/total percentage the script prints, find the `hasExplanation = no`
rows, author the next batch (Step 4), re-run Step 5. The map is the app's zero-read "has-explanation?"
index, so uncovered codes cost nothing at runtime.

### Step 7 — (Later, optional) Semantic dedup pass

Once a license is well covered, collapse genuinely-equivalent explanations:

1. Embed the **generated explanation texts** (not the questions) and surface near-duplicates.
2. Have the agent confirm "these two say the same thing."
3. For each confirmed cluster, repoint the others' map entries to one canonical `repCode` and delete the
   redundant docs.

This is a **pure data migration** — no app change — enabled by the `code → repCode` indirection that is
already in place. It is self-correcting: truly-equivalent questions produce near-identical concept
explanations (they merge); look-alike questions that need different explanations produce different texts
(they stay apart). Skip it entirely if the redundancy doesn't bother you — the docs are tiny, read
on-demand, and offline-cached.

---

## Consistency at scale

Generating ~1500–2000 explanations as independent (parallel) requests does **not** by itself produce a
uniform corpus — and going sequential would not fix it either (a long run does not remember its earlier
outputs). Uniformity comes from giving **every** request the *same* conditioning, then sweeping the
residue with an editorial pass.

### Copy (text)

Condition every request on the **three frozen artifacts** in `docs/content/`, plus pinned decoding:

- `style-contract.md` (system instruction) · `golden-examples.md` (few-shot) · `glossary-pl-en.csv`
  (terms) — supplied identically on every call.
- Pinned model version, low temperature (≈ 0.2), fixed `top_p`. Note that `air_law` and
  `human_performance` run on a **different pinned model** (`gemini-2.5-pro` — see *Per-subject model
  routing* above); each subject is still internally uniform (one model, same conditioning), and the UI
  never places two subjects side by side, so cross-subject model drift is immaterial. The same frozen
  artifacts condition every model, so the contract holds regardless.

Two safety nets over the whole corpus (both can be batched):

1. **Editor / normalization pass** — a second pass over each draft against the contract + glossary,
   checking **technical accuracy and plain-language clarity** (this is the "edit so it's grammatically
   correct and easy to understand" step from the brief).
2. **Outlier audit** — embed all explanations, flag length/structure/tone outliers, re-edit only those.

**Freeze discipline:** the artifacts are versioned. Bumping a version means earlier explanations were
written against a different spec — re-run the editor pass so old and new agree, and record the style
version used with each run.

### Images

A different problem: image models are far less stable across independent calls than text models, and the
text pass cannot mint real URLs. So **decouple images from the text run** — author the *intent* now, attach
the *picture* later:

1. **Judicious + URL-less.** The text pass adds an image **only when it genuinely helps** (most
   explanations get none) and requests it **in place** as a fenced ` ```image ` block with `key` · `alt` ·
   `depicts` and **no URL** (style contract §8). This compiles to a **pending image block** that is stored
   in Firestore alongside the text — so the agent's image *decisions* are captured even before any picture
   exists, and never need re-running.
2. **The app hides pending images.** A block with no (or blank) `url` is dropped at decode and renders
   nothing — a pending image can never break a layout or show a broken-image slot.
3. **Backfill gradually, keyed by `key`.** Build a **small shared asset library** under **one fixed visual
   style**, human-reviewed. Prefer **curated/templated** diagrams (SVG, or real ULC/EASA/Wikimedia art in
   one visual family) over free text-to-image for technical content — text-to-image cannot be trusted for
   exact labels, tables, or schematics. The identity is specified in `docs/content/image-style.md` —
   notably, any aeroplane is the **photorealistic Cessna 152 _SP-KOK_** (references:
   `docs/content/reference/sp-kok.png` + `sp-kok-2.jpg`).
4. Assets attach at the **concept (`repCode`) level and are reused** — dozens of concepts, not 2000
   questions, so curation is tractable and consistency comes from *reuse*. A later **resolver** sets the
   `url` on each pending block once a matching asset exists (`images_needed.csv` lists what is
   wanted, `key` · `alt` · `depicts`); the moment a `url` lands, the image lights up in the app.

---

## Part C — Go live

The feature ships dark behind `FeatureFlag.Explanation`. Going live is operational, not code:

1. (One-time, prod) Retire the legacy per-category explanations path in `firestore.rules` and
   `firebase deploy` the rules.
2. Flip `FeatureFlag.Explanation` via **Remote Config**, staged; watch reads and Crashlytics.

---

## Reference

### Question-code prefix → category

| Prefixes | Category |
|---|---|
| PL010, PL100, PL099, PL102 | `air_law` |
| PL020, PL021, PL024, PL025 | `aircraft_general_knowledge` |
| PL030 | `flight_planning` |
| PL040 | `human_performance` |
| PL050 | `meteorology` |
| PL060 | `navigation` |
| PL070 | `operational_procedures` |
| PL080 | `principles_of_flight` |
| PL090 | `communications` |

### Firestore shapes written

| Path | Written by | Content |
|---|---|---|
| `/questions/{categoryId}/langs/{lang}/chunks/{chunk}` | `question_feed.py` | 50 questions/chunk — shared deduped union (license-agnostic); the app's serving store |
| `/licenses/{lic}/categories/{cat}/membership/map` | `question_feed.py` | `{ schemaVersion, codesByLang }` — which codes a license includes, per language (SPL/BPL: `pl` only) |
| `/licenses/{lic}` (license doc) + `/licenses/{lic}/categories/{cat}` (metadata) | `feed_generator.py` | license `code`/`supportedLangs`/`langDetails`; per-(license,category) `examQuestionCount`/`examPassCount`/`examDurationSeconds`/`questionCountByLang` |
| `/explanations/{repCode}/langs/{lang}` | `explanation_feed.py` | `{ schemaVersion, blocks }` (license-agnostic) |
| `/explanationMaps/{categoryId}` | `explanation_feed.py` | `{ schemaVersion, entries }` (license-agnostic) |

### Scripts

| Script | Role |
|---|---|
| `ulc_pdf_to_csv_converter.py` | ULC PDF → `;`-CSV (called by `feed_generator.py`; standalone too) |
| `feed_generator.py` | per-license **license doc** (`code`/`supportedLangs`/`langDetails`) + per-(license,category) **metadata** doc; its per-(license,lang) chunk upload is superseded for serving |
| `question_feed.py` | questions (serving store): builds the shared deduped PL+EN union + per-license membership maps; validation gates + coverage/parity report → Firestore |
| `question_blocks.py` | pure transforms `question_feed.py` builds on (union/dedup/membership; stdlib only) |
| `validate_shared_codes.py` | verify codes are identical across licenses; emit the union `ulc_all.csv` |
| `explanation_feed.py` | explanations: content CSV → validate → compile → Firestore + coverage |
| `explanation_blocks.py` | pure compiler/validator the ingester uses (stdlib only) |
| `explanation_runner.py` | the agent runner — questions → generated explanations → `content.csv` |
| `explanation_gen.py` | pure core of the runner (manifest, prompt, parse, validate, assemble) |
| `batch_status.sh` | list recent Gemini Batch jobs + state (AI Studio has no UI for these) and the result-store count |
| `run_air_law_chunk.sh` | convenience: drive `generate` for `air_law` one resumable chunk at a time |
| `test_question_blocks.py` | unit tests for `question_blocks.py` (union/dedup/membership transforms) |
| `test_explanation_blocks.py`, `test_explanation_gen.py`, `test_explanation_runner.py` | unit tests (compiler/validator; runner core; batch key-mapping + chunking + model fallback) |

### Frozen content artifacts (`docs/content/`)

| File | Role |
|---|---|
| `style-contract.md` | the editorial + formatting spec; system instruction for every request |
| `golden-examples.md` | few-shot exemplars supplied on every request |
| `glossary-pl-en.csv` | controlled vocabulary and PL↔EN term pairs |
| `image-style.md` | the fixed visual identity for illustrations (incl. the Cessna 152 _SP-KOK_) |
| `reference/sp-kok.png`, `sp-kok-2.jpg`, `cessna-152-sp-kok.jpeg` | aircraft references: real photos (livery + realism) + CGI shape render |

Versioned and supplied identically on every generation call — see *Consistency at scale*.

### Open tooling (not built yet)

- **Worksheet exporter** — ULC CSV → human-friendly per-question spreadsheet. Only needed when a
  *human* content-ops team replaces the agent.
- **Dedup tool** — the Step 7 embedding/merge pass.
- **Editor / normalization pass** — the corpus-wide consistency + clarity pass (Consistency at scale).
- **Image library + resolver** — the curated asset store and the step that injects `![…](https…)` from
  structured image requests (`images_needed.csv` lists what's wanted).

(The **agent runner** that drives Step 4 is built — `explanation_runner.py`, see Step 4.)

### Key conventions

- `ODP1` is always the correct answer.
- `NUMER` is **not** unique; same code ⇒ same explanation (keyed by `code`, deduped via `repCode`).
- Content CSV needs only a `Codes` column plus `Explanation (PL)` / `Explanation (EN)`; columns are
  matched by substring and read as `utf-8-sig`.
- First run = **one explanation per code**, no grouping; group later via the Step 7 dedup pass.
