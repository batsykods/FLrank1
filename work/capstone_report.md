# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Sarthak Bartwal
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/batsykods/FLrank1
- **Date:** 2026-08-15

## 0. Abstract

This study asks whether historical search-performance signals available before a prediction date can rank content items that experience meaningful performance decline in the following month. The analysis uses the pseudonymized FlyRank full warehouse, queried through DuckDB, and constructs monthly historical features without downloading the warehouse into memory. A transparent historical-deterioration baseline is compared with Logistic Regression and Random Forest models using a chronological holdout, with Precision@50 as the primary ranking metric. **Insert the freshly measured headline result from `future_metrics.json` here; do not use results from an earlier starter-data experiment.** The resulting score is a decision-support queue for editorial review, not a causal claim about refresh impact or Google's ranking system.

## 1. Problem framing

The decision is which content items should receive editorial investigation first when review capacity is limited.

Unit of analysis: one pseudonymized content item at a monthly prediction point.

Output: a risk score and ranked review queue.

Human action: refresh review, priority review, or monitor.

False positives consume limited editorial review capacity. False negatives can leave genuinely declining content lower in the queue. ML is useful if it concentrates future declines near the top of the queue better than a transparent baseline.

## 2. Data safety

The capstone uses the FlyRank full pseudonymized warehouse. The daily performance fact table is aggregated inside DuckDB to monthly content/client observations.

The model deliberately excludes `trend_direction`, `trend_pct`, target-derived flags, and pseudonymous IDs as predictive features. Client/content identifiers are used only for grouping and anonymous queue construction.

The query-level 90-day snapshot is excluded because its snapshot timing does not establish that its signals were available at every historical prediction point.

No client names, domains, URLs, private queries, credentials, or raw exports are published.

## 3. Baseline

The baseline is a transparent historical-deterioration score using negative impression movement, negative click movement, and worsening average position.

**Fresh test result:** replace the placeholders below with the generated values.

- Precision@10: `...`
- Precision@25: `...`
- Precision@50: `...`
- Precision@100: `...`
- PR-AUC: `...`

## 4. Model / analysis

The target is:

> `future_decline = 1` when next-month impressions are less than 80% of the immediately preceding month's impressions.

Features are constructed from the three previous consecutive monthly observations. They include historical impressions, clicks, position, CTR, active days, log-scaled performance, and month-over-month / three-month movement.

Models:
- Logistic Regression
- Random Forest

The selected model is determined by Precision@50 on the chronological test set.

## 5. Evaluation

The latest 20% of eligible prediction months form the test period. Earlier months form training data. This avoids a random row split and reflects deployment on a later period.

The report must state the test base rate next to Precision@K.

**Fresh results:** populate directly from `work/outputs/future_metrics.json`.

Include:
- model-vs-baseline table
- Precision@K chart
- PR-AUC
- test base rate
- false-positive / false-negative analysis
- threshold sensitivity at 10%, 20%, and 30%

## 6. Interpretation

Feature importance is descriptive rather than causal. Explain the strongest historical signal families in plain language.

Report surprises and negative findings. Do not infer that a feature causes decline merely because the model uses it.

## 7. Recommendation

The final ranked queue supports three actions:

1. **Refresh review** — highest-risk items with meaningful historical visibility.
2. **Priority review** — moderate/high risk requiring manual investigation.
3. **Monitor** — lower-risk items without sufficient evidence for immediate intervention.

The score should be treated as prioritization evidence. It does not guarantee that a refresh will improve performance.

## 8. Reproducibility

From a fresh clone:

```bash
python -m venv .venv
# Windows PowerShell
.venv\Scripts\Activate.ps1
pip install duckdb huggingface_hub pyarrow scikit-learn matplotlib pandas numpy
```

Run `work/notebooks/capstone.ipynb` with a Hugging Face READ token supplied through `HF_TOKEN`, Colab Secrets, or an interactive prompt.

Random seed: `42`.

Committed receipts:
- `work/outputs/future_metrics.json`
- `work/outputs/future_model_results.csv`
- `work/outputs/future_threshold_sensitivity.csv`

Do not commit the warehouse or raw datasets.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).

---

**Submission claim check:** use observed / measured / directional / decision-support language. Do not make causal claims, do not claim to predict Google's algorithm, and make sure every number in this report matches a fresh notebook run.
