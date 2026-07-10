# ULC reference PDFs

Snapshot of the official **ULC** (Urząd Lotnictwa Cywilnego) source material the content pipeline is
built from. These are the raw inputs: the question-serving pipeline (`scripts/question_feed.py`) reads
them from here by default — its `--ref-dir` defaults to this `docs/ulc-reference` directory, where it
converts each of the six question PDFs (`ppla`/`pplh`/`spl`/`bpl` plus `ppla_eng`/`pplh_eng`) to a
sibling CSV.

- **Retrieved:** 2026-06-14
- **Question banks:** <https://ulc.gov.pl/personel-lotniczy/komisja-egzaminacyjna/egzaminy-teoretyczne/przykladowe-pytania-egzaminacyjne>
- **Exam rules (time / question counts):** <https://ulc.gov.pl/_download/lke/czas_liczba_pytan_kbp.pdf>

## Files

| File | Licence | Lang | Auto-derived as |
|---|---|---|---|
| `ppla.pdf` | PPL(A) — Private Pilot (Aeroplane) | PL | `ppl_a`, `pl` |
| `ppla_eng.pdf` | PPL(A) — Private Pilot (Aeroplane) | EN | `ppl_a`, `en` |
| `pplh.pdf` | PPL(H) — Private Pilot (Helicopter) | PL | `ppl_h`, `pl` |
| `pplh_eng.pdf` | PPL(H) — Private Pilot (Helicopter) | EN | `ppl_h`, `en` |
| `spl.pdf` | SPL — Sailplane Pilot | PL | `spl`, `pl` |
| `bpl.pdf` | BPL — Balloon Pilot | PL | `bpl`, `pl` |
| `czas_liczba_pytan_kbp.pdf` | Exam rules: per-subject question count, pass mark, and **time limit** | PL | — |

The serving question banks are built and uploaded by `scripts/question_feed.py` into the shared,
license-agnostic `/questions` store plus per-license membership maps. `feed_generator.py` no longer
uploads serving question chunks (that path is superseded); it now owns only the per-license license doc
(`code`, `supportedLangs`, `langDetails`) and the per-(license, category) metadata doc. The convention
of deriving license and language straight from these filenames (the `_eng` suffix selects English, the
base stem selects the license) is shared by `question_feed.py` and `validate_shared_codes.py`. SPL and
BPL are Polish-only (matches `LICENSE_DEFINITIONS` `supportedLangs`); PPL(A)/PPL(H) have PL + EN.

## Notes

- The rules PDF (`czas_liczba_pytan_kbp.pdf`) is the authority for `LICENSE_EXAM_DURATIONS` in
  `scripts/feed_generator.py` — several entries there are still placeholders marked
  `# TODO: verify against ULC PDF`.
- These are a point-in-time snapshot; ULC updates the banks periodically. Re-download and replace when
  refreshing content, and note the new retrieval date above.
- The site sits behind Imperva/Incapsula bot protection, so the PDFs were downloaded manually rather than
  scripted.
