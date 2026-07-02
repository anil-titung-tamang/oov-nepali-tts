# OOV Benchmark & Bigram-Gap Dataset Pipeline

Benchmark-driven pipeline for improving out-of-vocabulary (OOV) coverage in
a custom-trained **Nepali TTS voice** — mine the bigrams the model doesn't
know, build a category-balanced benchmark of OOV vs. IV words, collect
volunteer recordings targeting exactly those gaps, and measure the
intelligibility improvement from finetuning on that targeted data.

> **Status:** Draft / preliminary. Step 8 (finetuning on volunteer
> recordings) has not been run yet in this repo's current results — see
> [Known Issues](#known-issues--next-steps) below before citing any numbers.

---

## Why this exists

TTS voices trained on limited corpora tend to mispronounce or degrade on
words containing letter/sound combinations (**bigrams**) rarely or never
seen during training — commonly proper nouns, abbreviations, brand names,
code-mixed terms, and government/organization names. These are exactly the
words most likely to appear in real deployment (news, navigation, official
documents), and exactly where models are weakest.

This pipeline makes that weakness **measurable** (via a bigram-coverage
benchmark and an Intelligibility Error Rate metric) and **improvable** (by
generating a volunteer recording set that specifically targets the gaps the
benchmark finds — see [Step 8 note](#step-8--finetuning-bring-your-own-methodology)).

---

## Pipeline overview

| Step | Notebook | Purpose |
|------|----------|---------|
| 1 | `notebooks/step1_vocabulary_extraction.ipynb` | Build the combined in-vocabulary (IV) bigram/word set from known TTS training datasets |
| 2 | `notebooks/step2_corpus_mining.ipynb` | Mine a large external corpus for candidate words (Devanagari-filtered) |
| 3 | `notebooks/step3_oov_detection.ipynb` | Split candidates into OOV vs. IV; greedily select words maximizing missing-bigram coverage |
| 4 | `notebooks/step4_categorization.ipynb` | Auto-classify + manually supplement OOV/IV words into 7 categories (ABBR, BRAND, CM, CMPY, GOVT, PROP, NAV) |
| 5 | `notebooks/step5_utterance_generation.ipynb` | Group words into 5-word recording utterances; generate the recording script |
| 6 | `notebooks/step6_postprocess_recordings.ipynb` | Trim, normalize, and QC volunteer recordings |
| 7 | `notebooks/step7_synthesis_and_evaluation.ipynb` | Synthesize utterances, collect intelligibility ratings, compute IER (also handles Step 9 reporting) |
| 8 | `notebooks/step8_finetune_README.md` | **Finetune the target model on Step 6 recordings** — methodology is implementation-specific, see below |
| 9 | *(folded into Step 7 notebook, cells 21–29)* | Aggregate figures, final report, final results |

Full methodology, per-step inputs/outputs, metrics, and known issues are in
[`docs/OOV_Benchmark_Documentation.docx`](docs/OOV_Benchmark_Documentation.docx).

---

## Repo structure

```
oov-benchmark-nepali-tts/
├── README.md
├── requirements.txt
├── .gitignore
├── docs/
│   └── OOV_Benchmark_Documentation.docx   # full methodology / SOP / results / known issues
├── notebooks/
│   ├── step1_vocabulary_extraction.ipynb
│   ├── step2_corpus_mining.ipynb
│   ├── step3_oov_detection.ipynb
│   ├── step4_categorization.ipynb
│   ├── step5_utterance_generation.ipynb
│   ├── step6_postprocess_recordings.ipynb
│   ├── step7_synthesis_and_evaluation.ipynb
│   └── step8_finetune_README.md
├── data/
│   ├── raw/            # gitignored — place raw corpora here locally
│   ├── recordings/      # gitignored — place raw volunteer .wav files here locally
│   └── processed/       # committed — IV sets, OOV selections, category CSVs, recording script, evaluation_sentences.csv
└── results/
    ├── ier_results.csv
    ├── final_report.md
    └── figures/
```

---

## Setup

```bash
git clone https://github.com/<your-username>/oov-benchmark-nepali-tts.git
cd oov-benchmark-nepali-tts

python3 -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

Raw corpora and raw/processed recordings are **not** committed to this repo
(see `.gitignore`) — either regenerate them by running Steps 1–2 against
your own datasets, or fetch them from wherever your team stores large
audio/corpus assets, and place them under `data/raw/` and
`data/recordings/` respectively.

---

## Running the pipeline

Run notebooks in order, 1 → 9. Each notebook's expected inputs/outputs are
documented at the top of the notebook and in `docs/OOV_Benchmark_Documentation.docx`.

1. **Steps 1–3** — run once per training-vocabulary / corpus combination to produce the OOV-selected word list.
2. **Step 4–5** — run once to produce the category-balanced benchmark and the recording script.
3. Distribute `data/processed/recording_script.txt` to volunteers; collect recordings into `data/recordings/speaker_XX/{OOV,IV}/`.
4. **Step 6** — QC the recordings before using them anywhere downstream.
5. **Step 8** — finetune the baseline checkpoint (`milanakdj/indic-parler-tts-nepali-finetuned-dgx-v2.1-cosine`) on the QC'd Step 6 recordings (see note below).
6. **Step 7** — run **twice**: once against the baseline checkpoint (enable the baseline-synthesis cell — this reproduces the results currently in `results/`) and once against the new Step 8 finetuned checkpoint, so the final numbers are a true baseline-vs-finetuned comparison, not a single-model snapshot.

---

## Step 8 — Finetuning (bring your own methodology)

Step 8 is intentionally **not** a single fixed notebook. Finetuning
methodology differs by team, base architecture, and framework — someone
finetuning Parler-TTS will set this up very differently than someone
finetuning VITS, XTTS, or an in-house model. Rather than prescribe one way,
`notebooks/step8_finetune_README.md` documents the **input/output
contract** so you can plug in your own training code:

- **Input:** `milanakdj/indic-parler-tts-nepali-finetuned-dgx-v2.1-cosine` (the current baseline checkpoint — see [Results](#current-results-baseline-preliminary)) plus the QC-passed recordings from Step 6 and `evaluation_sentences.csv` for transcript↔audio mapping.
- **Output:** a *new* finetuned checkpoint (local path or HF model ID), referenced from exactly one place in the repo (avoid the kind of version-string drift documented in `docs/` Known Issues).

**Why this step matters more than it might look:** the checkpoint currently
evaluated in `results/` has never seen this project's own recordings — it
is effectively the **baseline**, not a finished result. The Step 6
recordings aren't generic speech data either: they were purpose-built by
Steps 1–5 to cover the exact bigrams and word categories (ABBR, PROP, CM,
GOVT, etc.) this benchmark identified as the baseline model's weakest
spots. Finetuning the baseline on this OOV-targeted dataset should
therefore produce a disproportionately larger intelligibility gain per
recorded minute than the same amount of generic training data would — and
the resulting model, evaluated head-to-head against the baseline above, is
the actual outcome this project is working toward. That comparison is the
highest-leverage step in the whole pipeline and hasn't been run yet.

---

## Current results (baseline, preliminary)

| Type | IER |
|------|-----|
| OOV | 22.6% |
| IV | 18.1% |
| Gap | +4.5 pp |

These numbers come from evaluating `milanakdj/indic-parler-tts-nepali-finetuned-dgx-v2.1-cosine`
— a checkpoint **not** finetuned on this project's own Step 6 recordings.
That makes this the **baseline** going forward, not a finished result. The
model that actually matters — a version of this checkpoint finetuned on the
OOV-targeted volunteer recordings — doesn't exist yet. Full breakdown in
`results/ier_results.csv` and `docs/OOV_Benchmark_Documentation.docx`.

---

## Known issues & next steps

- **Step 8 not yet run — this is the key next step.** The checkpoint evaluated in Step 7 was not finetuned on this project's own Step 6 recordings, so it is, in effect, the *baseline* for this project — not the endpoint. The model that actually matters is the one not yet created: this checkpoint finetuned on the OOV-targeted volunteer recordings from Step 6. Until that model exists and is evaluated, this project hasn't yet measured its own contribution. See the [Step 8 note](#step-8--finetuning-bring-your-own-methodology) above.
- **No baseline-vs-finetuned comparison.** Cell 6b (baseline synthesis) in Step 7 was left optional/commented out, so the current run only evaluates the baseline checkpoint above. Re-running Step 7 with the future Step 8 output against this same baseline is what will actually demonstrate whether the OOV-targeted recordings improve intelligibility.
- **Single rater** — category-level results (especially anywhere OOV scores *better* than IV) are more likely rater noise than a real effect; add ≥2 more raters before treating results as final.
- **Checkpoint version inconsistency** — reconcile `v2.0-cosine` (referenced in `summary.json`) vs `v2.1-cosine` (referenced in the Step 7 notebook header), so both the baseline and future finetuned-model results are attributed to the correct starting checkpoint.
- **Dependency pinning** — `transformers` version differs between notebook header and verify cell in Step 7; fixed in `requirements.txt` here, keep it that way.
- **Missing raw artifacts** — `step4_categorized_placeholder`, `step7_results/`, `ratings/`, and `synthesized_uploads/` were uploaded empty; raw per-rating and per-utterance audio was not available for this documentation pass.

Full detail on all of the above is in `docs/OOV_Benchmark_Documentation.docx`, Section 7.

---


## Contributing / Contact

Issues and PRs welcome — especially additional Step 8 finetuning notebooks
for other architectures/frameworks, additional rater data, or corpus
extensions for other languages/varieties.