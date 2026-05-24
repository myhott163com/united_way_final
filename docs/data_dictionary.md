# Data Dictionary

Schema reference for `processed_data_5.csv` — the output of `notebooks/01_cleaning_processing.ipynb` and the input to `notebooks/02_analysis.ipynb`.

The cleaning pipeline retains every raw column from the Excel export and appends harmonized fields alongside them. Suffix conventions:

- `<name>` — raw column from the Excel export
- `<name>_clean` — canonicalized version (post-mapping, post-classification)
- `<name>_<category>` — one-hot indicator column for one category of a multi-select field

---

## Identifier and survey metadata

| Column | Type | Description |
|---|---|---|
| `uid` | str | Respondent unique ID; one row per respondent |
| `survey_type` | str | One of 8 instruments: `Full Survey`, `3Q`, `Convo CheckIn`, `Community Impact`, `CASH`, `United We Can`, `GiveAtWork`, `QHC` |
| `start_time` | datetime | Survey submission timestamp |

---

## Geography

| Column | Type | Description |
|---|---|---|
| `live_work_sm` | str | Raw geography selection |
| `live_work_sm_clean` | str | One of `Cumberland`, `York`, `Cumberland & York`, `Outside Southern Maine` |
| `zip_code` | int | Raw ZIP (missing for some instruments) |
| `zip_code_full` | str | ZIP padded to 5 digits |
| `county` | str | County resolved from ZIP — used as a spot-check, not the authoritative geography variable |

The Southern Maine analytic sample is defined as `live_work_sm_clean != "Outside Southern Maine"` and contains n = 1,702 respondents.

---

## ALICE status

Derived from the `$400 emergency` proxy item per the codebook.

| Column | Type | Description |
|---|---|---|
| `alice` | str | Raw response (`Yes`, `No`, `Maybe, with difficulty`) |
| `alice_clean` | str | `Above ALICE` (Yes) or `Below ALICE` (No, Maybe) |

n = 1,686 with valid ALICE status. 30% Below ALICE, 70% Above ALICE.

---

## Biggest challenges (24 binary indicators)

A multi-select checkbox field with 14 predefined options and an optional free-text write-in. The write-ins are classified against 10 additional canonical categories with zero-shot classification at a 0.30 confidence threshold, then merged with the checkbox responses and one-hot encoded.

| Column | Type | Description |
|---|---|---|
| `biggest_challenge` | str | Raw semicolon-separated multi-select string |
| `biggest_challenge_clean` | str | Canonicalized semicolon-separated string |
| `biggest_challenge_<category>` | 0/1 | One-hot indicator |

**Checkbox categories (14):** Housing Cost, Cost of Basics, Food Access, Healthcare Cost, Mental Health, Behavioral Health, Affordable Child Care Access, Affordable Elder Care Access, Transportation, Low Wages, Job Instability, Social Isolation, Education Access, Climate Impacts.

**Write-in canonical categories (10):** Affordable Housing Availability, Language Access, Domestic Violence, Rising Taxes, Political Division, Immigration Enforcement, Welfare Dependence, Lack of in-Home Support, Gas Costs, Rent Increases. Plus `Other` for sub-threshold responses.

---

## Civic engagement

| Column | Type | Description |
|---|---|---|
| `feel_heard` | str | Raw response (`Yes`, `No`, `Maybe`) |
| `feel_heard_clean` | str | Collapsed to binary `Heard` / `Unheard` per the cleaning script |
| `want_involvement` | str | Raw response |
| `want_involvement_clean` | str | Cleaned binary `Yes` / `No` |
| `how_involvement` | str | Multi-select of engagement types |
| `how_involvement_<type>` | 0/1 | One-hot indicators across six types: Volunteer, Speak Up, Join Group, Share Story, Donate, Connect |

---

## Community-identity write-in

The free-text "which community feels unheard?" field, classified through a two-stage pipeline (spaCy NER pre-filter + BART zero-shot fallback at 0.25 threshold).

| Column | Type | Description |
|---|---|---|
| `community` | str | Raw write-in response |
| `community_fine` | str | Fine-grained classification — town entities are pinned to `Town of Residence`; everything else gets its zero-shot label or `Other` |
| `community_super` | str | Super-category collapse used for the donut chart |
| `community_<category>` | 0/1 | One-hot indicators across eleven community categories |

---

## Demographics (Full Survey + ALICE Voices only)

| Column | Type | Description |
|---|---|---|
| `age_group` | str | Raw age band |
| `age_group_clean` | str | Cleaned to `Under 24`, `25-64`, `Over 65+` |
| `race_ethnicity` | str | Multi-select; raw |
| `race_ethnicity_clean` | str | Cleaned |
| `race_ethnicity_<group>` | 0/1 | One-hot indicators |
| `relation_uwsm` | str | UWSM relationship status |

---

## ALICE Voices-only fields (n = 78)

Asked only on the `Community Impact` instrument.

| Column | Type | Description |
|---|---|---|
| `hardest_expenses` | str | Multi-select |
| `hardest_expenses_<item>` | 0/1 | One-hot indicators |
| `employed` | str | Employment status |
| `employment_type` | str | Employment category |
| `roadblock_expenses` | str | Multi-select with write-in |
| `roadblock_expenses_<item>` | 0/1 | One-hot indicators |
| `helpful_supports` | str | Multi-select with write-in |
| `helpful_supports_<item>` | 0/1 | One-hot indicators |

---

## A note on missingness

Question coverage varies by instrument. Anything outside the three common-core fields (ALICE status, location, biggest challenges) is missing for the 58% of respondents who took an abbreviated instrument. Per-field missing rates in the analytic sample (from Table 3 of the report):

| Field | Missing |
|---|---|
| Community write-in | 84% |
| Race / ethnicity | 77% |
| UWSM relationship | 70% |
| Interest in involvement | 70% |
| Age group | 59% |
| Feel heard | 49% |
| Top challenges | 19% |
| ALICE proxy ($400) | 0.9% |

Missing here is structural — the question wasn't asked — not nonresponse. Every subgroup analysis in the report is restricted to the survey types that asked the relevant question.
