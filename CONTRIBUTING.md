# Contributing

This repo is a final project deliverable, not an actively maintained open-source project — but the structure should make it easy for someone (a teammate, a future reviewer, or yourself in six months) to pick it up.

## Where things live

- `notebooks/01_cleaning_processing.ipynb` — raw data → cleaned + classified CSV. Authored by Marzouk. **Note:** the in-notebook output file path expects `data/processed_data_5.csv` to be writable; the cleaning pipeline downloads `facebook/bart-large-mnli` (~1.6 GB) on first run.
- `notebooks/02_analysis.ipynb` — all descriptives, visualizations, statistical tests, and ML. Authored by Hangliang.
- `figures/` — every figure in the report and deck, as PNG. The analysis notebook regenerates these into the working directory; copy or commit them into `figures/` when ready.
- `report/` — final report PDF and deck PDF.
- `docs/` — methods walkthrough, findings summary, data dictionary.
- `data/` — not tracked. See [`data/README.md`](data/README.md).

## Picking up the work

1. Clone, create a virtualenv, `pip install -r requirements.txt`, `python -m spacy download en_core_web_sm`.
2. Drop `UWSM Community Data 4-1-2026.xlsx` and/or `processed_data_5.csv` into `data/`.
3. If only the raw Excel is present, run notebook 01 first to generate the CSV. Otherwise skip directly to notebook 02.
4. Notebook 02 is organized into eight numbered sections (setup, load, descriptives, visualizations, chi-square, predictive modeling, PCA, wrap-up). Cells are intended to be run top-to-bottom in order.

## Coding conventions used

- One-row-per-respondent indexed by `uid`. Don't aggregate at any other level without reason.
- Canonical column names use `snake_case`. Raw columns from the Excel export keep their original headers; harmonized columns get a `_clean` suffix (single-response) or are one-hot encoded with a `<field>_<category>` pattern (multi-select).
- All subgroup analyses are restricted to the survey types that asked the relevant question. Never average a field across instruments that didn't ask it. The pattern is `df_sm[df_sm["<field>"].notna()]`.
- All ML models use `random_state=42`. Cross-validation is stratified 5-fold.
- All figures use the UWSM palette defined at the top of notebook 02. Don't introduce new colours without checking the deck.

## Discrepancies to be aware of

The `feel_heard` field has two reasonable encodings: collapsing `No` and `Maybe` into `Unheard` (the report's preferred coding) versus collapsing `Yes` and `Maybe` (the cleaning script's coding). The figures in this repo and notebook 02 use the cleaning script's coding, which is the one the published deck used. Earlier drafts of the report mixed conventions; the current published PDF should match.

## Asking questions

Contact the authors listed in the README.
