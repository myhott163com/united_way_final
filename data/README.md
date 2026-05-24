# `data/`

Survey data files live here. **None of them are tracked in this repo** (see [`.gitignore`](../.gitignore)) because the dataset contains community-survey responses we don't have permission to redistribute.

## Expected files

| File | Source | Used by |
|---|---|---|
| `UWSM Community Data 4-1-2026.xlsx` | UWSM, raw export | `notebooks/01_cleaning_processing.ipynb` |
| `processed_data_5.csv` | Output of notebook 01 | `notebooks/02_analysis.ipynb` |

## To run the analysis

If you have only the raw Excel file:

```bash
cp /path/to/UWSM\ Community\ Data\ 4-1-2026.xlsx data/
jupyter notebook notebooks/01_cleaning_processing.ipynb
# Wait for the cleaning pipeline to write processed_data_5.csv into ./data/
jupyter notebook notebooks/02_analysis.ipynb
```

If you have `processed_data_5.csv` already (from a teammate, a prior run, or our shared drive):

```bash
cp /path/to/processed_data_5.csv data/
jupyter notebook notebooks/02_analysis.ipynb
```

## How to get the data

Contact the project authors (see [README](../README.md#authors)). The data is shared with United Way of Southern Maine and is not for public release.

## What's in the raw file

A single Excel workbook with three sheets:

- **Assignment** — project brief
- **Data** — 1,889 respondent-level records with 19 columns spanning 8 survey instruments
- **Codebook** — question text, response options, and canonical recodings

See [`docs/data_dictionary.md`](../docs/data_dictionary.md) for the column-by-column schema.
