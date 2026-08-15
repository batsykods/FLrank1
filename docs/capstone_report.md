# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Sarthak Bartwal
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/batsykods/FLrank1
- **Date:** 2026-08-15

> This report follows the supplied capstone report template. Sections 0–9 are the report/paper structure required for the capstone.

## 0. Abstract

This study asks whether historical search-performance signals available before a prediction date can rank content items that experience meaningful performance decline in the following month. The analysis uses the pseudonymized FlyRank full warehouse, with daily performance data aggregated into monthly content-level observations over 18 monthly partitions from January 2025 through June 2026. Logistic Regression and Random Forest are compared with a transparent historical-deterioration baseline using a chronological holdout, with Precision@50 as the primary ranking metric because the intended output is a limited editorial review queue. On the March–May 2026 test period, Logistic Regression achieved 92% Precision@25, 82% Precision@50, and 80% Precision@100 against a 65.1% test base rate, while its ROC-AUC was 0.488 and the historical baseline had higher PR-AUC. The resulting output is therefore best treated as a decision-support ranking for editorial investigation rather than a generally accurate predictor or a causal explanation of search performance.

## 1. Problem framing

### Decision

The capstone supports the decision:

> Which content items should receive editorial investigation first when review capacity is limited?

### Unit of analysis

One pseudonymized content item at a monthly prediction point.

### Output

The analysis produces a ranked risk score and an ordered review queue. The final recommendation output contains a rank, pseudonymous content/client identifiers, prediction month, risk score, action, reason code, and supporting historical movement fields.

### Human action

The queue supports three operational actions:

1. **Refresh review** — highest-priority items for editorial investigation.
2. **Priority review** — items requiring manual investigation but below the highest queue priority.
3. **Monitor** — items for continued observation rather than immediate intervention.

### Cost of wrong calls

False positives consume limited editorial review capacity. False negatives can leave genuinely declining content lower in the queue.

### Why data/ML helps

Historical search-performance observations provide more information than a manually selected single rule. The model can combine multiple historical signals and produce a consistent ranking. The intended value is not autonomous action; it is concentrating likely future-decline observations near the top of a limited review queue.

## 2. Data safety

### Data used

The analysis uses the **FlyRank full pseudonymized internship warehouse**. The primary source is:

```text
fact_content_daily_performance
```

Daily performance records were aggregated into monthly content/client observations.

The warehouse run discovered **18 monthly Parquet partitions covering January 2025 through June 2026**. The final future-window test period is March 2026 through May 2026.

### Deliberate exclusions

The model excludes:

- `trend_direction`
- `trend_pct`
- other target-derived fields
- pseudonymous identifiers as predictive features

Pseudonymous client/content IDs are used for grouping and anonymous queue construction only.

The query-level 90-day snapshot was excluded because its snapshot timing does not establish that its signals were available at every historical prediction point.

### Leakage risks

The target is defined from the future month, while model features are constructed only from the three preceding monthly observations:

```text
T-3 ─┐
T-2 ─┼──> features at T0 ──> future outcome at T+1
T-1 ─┘
```

This prevents the future target from being used as a model feature.

### Public-safety rule

No client names, domains, URLs, private search queries, credentials, or raw warehouse exports are included in the published analysis.

The repository should be checked before submission to confirm that no client-identifying material appears anywhere in `work/`.

## 3. Baseline

The baseline is a transparent historical-deterioration rule based on negative impression movement, negative click movement, and worsening average position.

It is a fair comparison because it operates on the same prediction task and is evaluated on the same chronological test period as the ML models.

### Final test results

| Model | Precision@10 | Precision@25 | Precision@50 | Precision@100 | PR-AUC | ROC-AUC |
|---|---:|---:|---:|---:|---:|---:|
| Historical rule baseline | 0.80 | 0.60 | 0.58 | 0.58 | 0.685 | 0.538 |
| Logistic Regression | 0.90 | 0.92 | **0.82** | **0.80** | 0.650 | 0.488 |
| Random Forest | 0.50 | 0.64 | 0.60 | 0.61 | 0.655 | 0.505 |

The test-set positive/base rate is **65.1%**.

The baseline remains competitive and has the highest overall PR-AUC (0.685). Therefore, the ML result should not be presented as a general improvement over the baseline. The selected ML model is chosen for the specific top-of-queue decision criterion described below.

## 4. Model / analysis

### Method

The lane is **Refresh / Content Opportunity Scoring**, so the modelling objective is ranking rather than claiming an exact future outcome for every content item.

The target is:

> `future_decline = 1` when next-month impressions are less than 80% of the immediately preceding month's impressions.

Features use the previous three consecutive monthly observations.

### Feature families

The feature construction uses historical search-performance and content signals including:

- impressions
- clicks
- average position
- CTR
- active days / days with impressions
- log-scaled performance measures
- month-over-month movement
- three-month movement
- other non-future content-performance features created by the capstone notebook

The exact feature list should be kept synchronized with the `FEATURES` list in `work/notebooks/capstone.ipynb`.

### Deliberately omitted

The following are intentionally not model features:

- future target information
- `trend_direction`
- `trend_pct`
- target-derived flags
- pseudonymous client/content IDs

### Models

Two ML models were evaluated:

- Logistic Regression
- Random Forest

The selected model is **Logistic Regression**, because the predefined model-selection criterion is Precision@50 and Logistic Regression achieved **0.82** compared with **0.60** for Random Forest.

### Target in one sentence

The task is to rank content observations according to their likelihood of meeting the operational definition of a next-month impression decline.

## 5. Evaluation

### Split

The final future-window experiment uses a **time-aware chronological holdout**.

The test months are:

- March 2026
- April 2026
- May 2026

Earlier eligible observations form the training period.

A chronological split is appropriate because the intended deployment setting is prediction on a later period from historical information. A random row split would mix temporal periods and could make evaluation less representative of deployment.

### Metrics

Precision@K is the primary operational metric because the intended action is reviewing only the highest-ranked items.

The final test base rate is **65.12%**, so Precision@K is interpreted relative to that prevalence.

| Model | Precision@10 | Precision@25 | Precision@50 | Precision@100 | Recall | F1 |
|---|---:|---:|---:|---:|---:|---:|
| Logistic Regression | 0.90 | **0.92** | **0.82** | **0.80** | 0.508 | 0.568 |
| Random Forest | 0.50 | 0.64 | 0.60 | 0.61 | 0.395 | 0.498 |
| Historical rule baseline | 0.80 | 0.60 | 0.58 | 0.58 | — | — |

### Main result

Logistic Regression is the selected ML model for the review-queue objective:

- Precision@10: **90%**
- Precision@25: **92%**
- Precision@50: **82%**
- Precision@100: **80%**

The result is materially above the 65.1% test base rate at the top of the queue.

However, overall discrimination is weak:

- Logistic Regression ROC-AUC: **0.488**
- Logistic Regression PR-AUC: **0.650**
- Historical baseline ROC-AUC: **0.538**
- Historical baseline PR-AUC: **0.685**

Therefore, the strongest evidence is specifically about **top-of-queue concentration**, not general predictive discrimination.

### Threshold sensitivity

| Decline threshold | Test base rate | Precision@50 | Precision@100 |
|---:|---:|---:|---:|
| 10% | 70.3% | 0.90 | 0.86 |
| 20% | 65.1% | 0.82 | 0.80 |
| 30% | 59.2% | 0.78 | 0.75 |

### Error analysis

The notebook includes false-positive and false-negative analysis on the chronological test set.

The final published paper should include the actual error counts and representative rows from the completed notebook output. These counts are not inserted here because they were not present in the supplied result artifacts, and no unsupported numbers should be fabricated.

The intended interpretation is:

- false positives = items prioritised for review that did not meet the future-decline label
- false negatives = labelled future declines that were not strongly prioritised

This analysis directly measures the operational trade-off between consuming review capacity and missing declining content.

## 6. Interpretation

The final evidence supports a narrow interpretation.

The Logistic Regression ranking concentrates labelled future declines near the top of the review queue. At K=25, 23 of the top 25 observations are positive under the operational target definition; at K=50, 41 of the top 50 are positive.

This is useful for a constrained editorial-review workflow.

However, the model does **not** demonstrate strong overall discrimination. Its ROC-AUC is approximately 0.49, while the historical baseline has a higher PR-AUC than either ML model.

This is an important negative result rather than something to hide.

The threshold sensitivity analysis shows that the top-of-queue precision remains relatively high across the tested definitions:

- 10% decline: Precision@50 = 90%
- 20% decline: Precision@50 = 82%
- 30% decline: Precision@50 = 78%

This indicates directional robustness of the queue concentration to the selected decline threshold, although it does not establish generalisation beyond the evaluated period.

Feature interpretation should remain descriptive. Logistic Regression coefficients indicate how the fitted model weights the available historical features; they do not establish that any feature causes future decline.

## 7. Recommendation

The ranked output supports the following editorial workflow.

### Rank 1 — Refresh review

Use for the highest-risk observations with sufficient historical visibility. These items should receive manual investigation of content freshness, search visibility, and recent performance movement.

### Rank 2 — Priority review

Use for moderate/high-risk observations where manual investigation is warranted but the item is below the highest-priority queue.

### Rank 3 — Monitor

Use for lower-risk observations without enough evidence for immediate editorial intervention.

### How an editor would use it

An editor with a fixed review capacity can take the first K items from the ranked queue, inspect the associated historical evidence and reason code, and decide whether a refresh or other editorial action is justified.

The model should **not** automatically trigger a refresh.

### Confidence and limits

Confidence is highest in the narrow claim that the observed ranking concentrated future-decline labels near the top of this test queue.

Confidence is lower for general predictive claims because ROC-AUC is near random and the historical baseline has stronger PR-AUC.

No recommendation should be interpreted as a guarantee that changing content will improve search performance.

## 8. Reproducibility

### Repository

```text
https://github.com/batsykods/FLrank1
```

### Main notebook

```text
work/notebooks/capstone.ipynb
```

### Result artifacts

```text
work/outputs/future_metrics.json
work/outputs/future_model_results.csv
work/outputs/future_threshold_sensitivity.csv
work/outputs/future_ranked_recommendations.csv
```

### Fresh-clone environment

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install duckdb huggingface_hub pyarrow scikit-learn matplotlib pandas numpy
```

Run:

```text
work/notebooks/capstone.ipynb
```

Supply the Hugging Face READ token at runtime. The token must never be committed to the repository.

### Random seed

```text
42
```

### Data-access design

The full warehouse is not committed to the repository. The notebook uses the Hugging Face Hub to discover/download required Parquet partitions and DuckDB for local aggregation, avoiding a repository copy of the warehouse.

### Holdout reproducibility

The notebook contains the cell logic that constructs the monthly panel, future label, chronological holdout, model evaluation, and metrics. The resulting metrics file is:

```text
work/outputs/future_metrics.json
```

The reported values in this document match the supplied final `future_metrics.json`.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support language is used throughout. The **65.12% test base rate** is reported next to Precision@K. AUC and baseline comparisons are reported to prevent top-K precision from being interpreted in isolation. No causal claims are made, the analysis does not claim to predict Google's algorithm, and no client-identifying information is intentionally published. Before final submission, verify that the contents of `work/` contain no client-identifying material and that all reported numbers still match the final notebook run.
