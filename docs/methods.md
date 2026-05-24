# Methods

This document walks through every analytical technique used in the project, what it produces, and why we chose it. It exists separately from the report because the report is written for a community-services audience and elides most of the methodological detail. Readers who want to know what the pipeline actually does should read this.

The pipeline divides cleanly into three layers:

1. **Data engineering** — turning a ragged multi-instrument survey into an analyzable single-row-per-respondent dataset
2. **Free-text processing** — canonicalizing open-text responses against a fixed taxonomy using a hybrid NER + zero-shot classification pipeline
3. **Statistical and ML analysis** — chi-square, logistic regression, decision trees, PCA, and K-means, with explicit handling of class imbalance, effect sizes, and instrument-driven missingness

---

## Layer 1 — Data engineering

### The raw input

A single Excel workbook (`UWSM Community Data 4-1-2026.xlsx`) with 1,889 respondent-level rows indexed by a `Unique ID` field. Each row is one survey response. The complication is that responses come from **eight different survey instruments**, each asking a different subset of questions. The instruments and the analytic groupings we used:

| Group | Instruments | n | Coverage |
|---|---|---|---|
| Abbreviated | `3Q`, `Convo CheckIn`, `GiveAtWork`, `United We Can` | 985 | 3 questions (ALICE, location, biggest challenges) |
| Full | `Full Survey` | 639 | 8 questions including demographics and feel-heard |
| ALICE Voices | `Community Impact` | 78 | All 12 questions including hardest expenses, employment, supports |

This is the kind of input where the right move is to treat the schema as a multi-instrument problem from the start, not to imputate coverage across instruments.

### Schema harmonization

We retain one row per respondent indexed by `Unique ID` and append harmonized canonical fields next to the raw columns. The column names from the Excel export are standardized to short snake_case identifiers (`biggest_challenge`, `feel_heard`, `live_work_sm`, etc.) in `notebooks/01_cleaning_processing.ipynb`.

For each column we declared a **column config** specifying its type — `single` (one-to-one mapped), `multi` (semicolon-separated multi-select), or `mixed` (multi-select with a free-text write-in component) — together with its canonical mapping dictionary and, for `mixed` columns, the candidate-label set used by the classifier. This config-driven design lets one pass over the data handle all 14 instrumented columns uniformly:

```python
column_configs = {
    "biggest_challenge": {"type": "mixed", "mapping": ..., "categories": ...},
    "live_work_sm":      {"type": "multi", "mapping": ...},
    "feel_heard":        {"type": "single", "mapping": ...},
    # ... 11 more columns
}
```

### One-hot encoding multi-select fields

Multi-select fields (biggest challenges, race/ethnicity, how-involvement) are split on `;`, each token is resolved against its canonical mapping, and the resulting cleaned strings are one-hot encoded with `pandas.Series.str.get_dummies(sep=";")`. The biggest-challenge field, after merging checkbox responses with classified write-ins, produces a 24-column binary feature matrix used downstream in every model. We deliberately treat absent challenge strings as all-zero indicator rows rather than as missing data, because the survey design ("check all that apply") makes the absence of a checkbox indistinguishable from a "no" response.

### ALICE derivation

The ALICE threshold is derived from the survey's $400-emergency proxy item following the codebook: `Yes` is coded as `Above ALICE`, `No` and `Maybe, with difficulty` are both coded as `Below ALICE`. Respondents who did not answer the proxy item are excluded from any analysis conditioned on ALICE status, but retained in analyses that don't require it. This matters because the analytic sample size moves between models depending on which fields each requires.

### Geography

The direct-selection location field is treated as authoritative for assigning respondents to Cumberland County, York County, or both. ZIP code is retained as a secondary identifier and used only as a spot-check; we built a manual ZIP-to-county dictionary spanning all York and Cumberland ZIPs but never let ZIP override the direct selection. The Southern Maine analytic sample (n=1,702) is defined by excluding rows explicitly coded as `Outside Southern Maine`.

### Missingness handling

Missingness is handled on an analysis-specific basis, never globally:

- For single-response variables, analyses use complete cases on the variables entering that analysis. No imputation.
- For the derived challenge indicators, missing or empty challenge strings are converted to all-zero indicator rows after one-hot encoding.
- Every reported subgroup analysis is restricted to the survey types that asked the relevant question. We never average across instruments that didn't ask the same question.

---

## Layer 2 — Free-text processing

Two survey fields contained free-text write-ins that needed canonicalization before they could enter any downstream model.

### Biggest-challenge write-ins

Beyond the 14 predefined challenge checkboxes, respondents could write in free-text responses. We expanded the canonical taxonomy with 10 additional categories — Affordable Housing Availability, Language Access, Domestic Violence, Rising Taxes, Political Division, Immigration Enforcement, Welfare Dependence, Lack of in-Home Support, Gas Costs, Rent Increases — and an `Other` bucket for responses that don't map to any category.

Each write-in is scored against the full 24-category label set using zero-shot classification with `facebook/bart-large-mnli`. The label with the highest score above a **confidence threshold of 0.30** is assigned; responses below threshold drop into `Other`. The result is one-hot encoded onto the same schema as the checkbox responses, producing a unified 24-column binary feature matrix.

The threshold of 0.30 was chosen empirically as a compromise: 0.10 (the very-low default) produces many false-positives where a write-in is force-matched to a tangentially related category; 0.50 throws away too many genuine matches because BART-MNLI is calibrated for natural-language inference on full sentences, and survey write-ins are often short fragments.

### Community-identity write-ins

The "which community feels unheard?" field is the harder problem — completely open-text, no checkbox prefilter, and respondents often write in their town name as the answer ("our small town gets ignored", "Brunswick", "the people in Wells"). Treating these all as zero-shot classification inputs against demographic labels would force-match them to whatever identity category they share lexical overlap with.

We built a **two-stage hybrid pipeline**:

1. **NER pre-filter.** Each response is passed through `spaCy en_core_web_sm` to detect named entities. Anything tagged `GPE` (geo-political entity) or `LOC` (location) is classified directly as `Town of Residence`. We also maintain a small manually-constructed dictionary of Southern Maine town names because the off-the-shelf spaCy model misses some smaller Maine municipalities.
2. **Zero-shot classifier fallback.** Anything that doesn't hit the NER filter is passed to `facebook/bart-large-mnli` zero-shot against eleven community categories (BIPOC, ALICE, Youth, Older Adults, Recovery Community, Parents, Women, LGBTQ+, etc.) at a 0.25 threshold. Sub-threshold responses are coded as `Other`.

The pipeline achieved an **overall classification rate of 89.5%**. Critically, the NER pre-filter is what surfaced the headline finding that "town of residence" is the single largest unheard community (40.5% of identified communities) — without it, every town name would have been force-matched to whatever demographic label the classifier scored highest, and the geographic signal would have been buried.

### Why this matters as a pattern

The same shape recurs anywhere you're trying to reconcile heterogeneous free-text inputs against a canonical taxonomy — entity normalization, deduplication, record linkage, intake-form processing. The pattern is: a fast rule-based pre-filter for the easy cases (here: spaCy NER + a domain dictionary), followed by a probabilistic classifier for the residual with an explicit confidence threshold and an honest "unclassifiable" bucket. The rule-based stage handles the cases the probabilistic stage would systematically mis-score; the classifier handles the long tail; and the threshold-plus-Other-bucket design lets you report a real classification rate instead of hiding low-confidence predictions in the output.

---

## Layer 3 — Statistical and ML analysis

All analyses are in `notebooks/02_analysis.ipynb`. Random seeds and cross-validation protocols are reported per method.

### Chi-square tests of independence

Pairwise tests across three families of relationships: ALICE × each of the 24 challenges, feel-heard × demographics (ALICE, race, age), and want-involvement × demographics. Computed with `scipy.stats.chi2_contingency` after verifying the expected-cell-count condition on every contingency table.

**We report Cramér's V alongside every p-value.** With n ≈ 1,700, almost any non-trivial difference reaches statistical significance, and reporting only p-values would over-claim. Cramér's V is interpreted with the conventional thresholds (V < 0.1 weak, 0.1 ≤ V < 0.3 moderate, V ≥ 0.3 strong), so the *substantive* size of an effect can be read off the table next to its significance.

The chi-square summary figure ([`slide_chi_square_summary.png`](../figures/slide_chi_square_summary.png)) shows the result for all three relationship families on one panel.

### Logistic regression

A binary logistic regression predicts Below-ALICE status from the 24 binary challenge indicators using `sklearn.linear_model.LogisticRegression` with `class_weight="balanced"` to correct for the 30/70 class imbalance. Performance is evaluated with **stratified 5-fold cross-validation** (`StratifiedKFold(shuffle=True, random_state=42)`), and accuracy, precision, recall, and F1 are reported as mean ± standard deviation across folds.

Odds ratios are taken from a single final fit on the full ALICE-valid sample (n=1,686). Values > 1 mean citing that challenge increases the odds of being Below ALICE; values < 1 mean the opposite.

**A note on the model's purpose.** The logistic regression is not a deployment-grade classifier — its cross-validated F1 of 0.466 against a 30/70 split means it doesn't reliably identify ALICE status from challenge profiles alone. We didn't fit it to deploy it; we fit it as a **construct validation** exercise. If the survey genuinely captures the experience that defines economic hardship, then the pattern of challenges a respondent reports should be systematically linked to their ALICE status. The fact that the model is significantly better than chance, and that the challenges that predict Below-ALICE status (food access, transportation, basics) are precisely the items that poverty research independently identifies as markers of material deprivation, is evidence the instrument is measuring something real.

### Decision tree

A `DecisionTreeClassifier` is fit on the same 24-feature challenge matrix with `max_depth=4`, `min_samples_leaf=10`, and `class_weight="balanced"`. The depth and leaf constraints keep the tree interpretable and prevent overfitting on a feature set of binary indicators. Evaluation uses the same stratified 5-fold protocol as the logistic regression for direct comparability.

Feature importances are the normalized mean decrease in Gini impurity from a single final fit on the full sample. The decision tree and the logistic regression converge on the same diagnostic features (food access, child care, elder care, cost of basics) via methodologically independent routes, which strengthens confidence that the two-tier survival/infrastructure structure is genuine and not an artifact of any single modeling choice.

### Principal component analysis

PCA is applied to the 24 binary challenge indicators after standardization with `StandardScaler`. We retain two components for visualization. The result is informative as a *negative* finding: the first two components together explain only 16.2% of variance, which means challenge citations don't reduce cleanly to a small number of underlying axes. PC1 is interpretable as a "severity of everyday material hardship" axis (high loadings on housing, basics, healthcare, mental health) and PC2 as a "systemic / marginal issue" axis (in-home support, affordable housing, domestic violence, political division), but neither separates Below-ALICE from Above-ALICE respondents on its own. That result is fully consistent with the modest classifier performance: challenges do contain ALICE-related signal, but it's distributed across many features rather than concentrated in one direction.

### K-means clustering

Respondents with at least one cited challenge are segmented with `sklearn.cluster.KMeans` on the 24-feature challenge matrix, with `n_clusters=4`, `n_init=10`, and `random_state=42`. The choice of k was made after inspecting the inertia curve for k ∈ {2, …, 8}. **The clustering is seed-sensitive** — different random states produce meaningfully different cluster assignments — and we flag this explicitly. We use the four-cluster solution as exploratory segmentation, not as a stable taxonomy. A proper segmentation analysis would average across many random seeds and report silhouette scores; that's flagged in the limitations section of the report.

### Significance reporting

Throughout the analysis, significance is indicated with `*` p < 0.05, `**` p < 0.01, `***` p < 0.001. Effect sizes are reported alongside every p-value because the large sample size makes substantively trivial differences statistically significant.

---

## What's deliberately out of scope

- **Causal claims.** Every finding in the report is associational. Chi-square and logistic regression establish correlation, not causation.
- **Race/ethnicity inference.** The BIPOC subsample is n=30 after geographic filtering, which is too small for reliable inference. We report race comparisons in the appendix as directional only.
- **Imputation across instruments.** We never average a question across instruments that didn't ask it. The Abbreviated survey didn't ask race; respondents who took it are simply excluded from race-conditioned analyses.
- **Deployment of the ALICE classifier.** The logistic regression was fit for construct validation, not for assigning ALICE labels to new respondents. The data-collection instrument provides the ALICE proxy directly; the classifier is downstream of it, not a substitute for it.

## Reproducibility

| Item | Value |
|---|---|
| Random seed (cross-validation, K-means, models) | `42` |
| Cross-validation protocol | `StratifiedKFold(n_splits=5, shuffle=True, random_state=42)` |
| Zero-shot model | `facebook/bart-large-mnli` |
| Zero-shot confidence thresholds | 0.30 (challenges), 0.25 (community) |
| spaCy model | `en_core_web_sm` |
| Class-balancing | `class_weight="balanced"` (LR and DT) |
| Effect-size measure | Cramér's V with conventional thresholds |
