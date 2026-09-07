# Capstone Report — Ranking Signal Analysis

- **Author:** Somnath
- **Lane:** Ranking Signal Analysis
- **Repo:** https://github.com/somnathsutra/ML-01-ASSIGNMENT
- **Date:** 2026-09-07

## Abstract

This project asks which observable search signals are useful for prioritizing content review. The analysis uses a March 2026 partition of the pseudonymized FlyRank warehouse, with daily Search Console observations aggregated to content-item level. Five first-half signals are measured at the March 15 decision moment, and a second-half impression decline is used as an observed movement proxy. A transparent baseline scores pages using impression volume and position-aware low CTR, then assigns one reason code and action label. A Random Forest improves precision at the top of the review queue on held-out clients, while the result remains decision-support rather than a causal ranking claim.

## 1. Problem framing

An editor or SEO analyst needs to decide which content items deserve diagnostic review first. In this lane, one row in the source table is one pseudonymized content item for one client on one report date; the baseline aggregates those daily rows into one content-level review record for March 2026.

The output is a ranked review queue with a transparent score, one reason code, and one action label. A human may inspect search demand, title/snippet fit, query mix, and measurement quality before changing content. A wrong call costs limited editorial time or may cause an unnecessary change, so the queue is a prioritization aid and never an automatic publishing action.

A signal-analysis approach fits because the question is about interpretable associations between search signals and review opportunities. A fixed rule is useful as a baseline, but it cannot establish that a signal causes a ranking change.

## 2. Data safety

The analysis uses the `fact_content_daily_performance` March 2026 partition from the gated, pseudonymized FlyRank warehouse. The March slice contains 9,841,378 daily rows and 331,437 distinct content items. The source grain check found zero duplicate `(client_hash_id, content_hash_id, report_date)` keys. The availability check retained 413,966 rows where `ga4_data_available IS TRUE`.

The decision moment is March 15. Features use only March 1–15 Search Console observations. The later March 16–31 window is used only for the observed decline proxy in the leakage notebook; content without a complete second-half window is excluded from that validation frame.

The five decision-time fields are:

- `first_half_impressions`
- `first_half_clicks`
- `first_half_ctr`
- `first_half_avg_position`
- `first_half_active_days`

`client_hash_id` and `content_hash_id` are context for grouping and auditing only. They are never score or model features. Second-half outcome fields, `trend_direction`, `trend_pct`, product flags, raw queries, URLs, client names, and tokens are excluded. Average position equal to zero is treated as no position data, not as rank zero.

No client-identifying details or raw queries appear in `work/`.

## 3. Baseline

The baseline rule is deliberately readable:

1. Add volume points: 0 below 50 impressions, 1 at 50+, 3 at 200+, and 4 at 1,000+.
2. Add CTR points when first-half CTR is below a position-aware threshold: 1% for positions 1–10, 0.5% for positions 11–20, and 0.25% beyond position 20. Rows without position data receive no CTR points.
3. Rank by the sum, then break ties by higher impressions and lower CTR.

The rule emits one reason code: `ctr_fix_candidate`, `visibility_opportunity`, or `low_evidence`. It emits one action label: review the title and search snippet, review visibility and content fit, or monitor and collect more evidence.

The queue contains 151,981 content-level records and is written by the notebook to `work/outputs/baseline_action_score.csv`. The output is intentionally regenerated rather than committed as a dataset.

The two signal checks produced these directional conclusions:

| Signal | Verdict | Evidence |
|---|---|---|
| First-half impression volume | CONFIRMED | Volume buckets were printed with counts; the 1,000+ bucket had 26,988 rows and higher median CTR than the 0–49 bucket. |
| CTR relative to position | MIXED | Position buckets were printed with counts and low-CTR rates; the rule is a diagnostic screen, not evidence that changing CTR causes better position. |

The top ten all received individual action, rationale, and failure-condition notes. One weak pick had 83,772 impressions but zero clicks, demonstrating that high visibility alone can still produce a misleading queue entry.

## 4. Model / analysis

The target proxy is `declined_second_half = 1` when second-half impressions per day are below 80% of first-half impressions per day. A Random Forest classifier ranks content by its predicted probability of that proxy outcome. It uses 150 trees, maximum depth 8, minimum leaf size 20, and `random_state=42`.

The feature vector contains only the five first-half fields listed above. It deliberately excludes label-derived fields, later outcome fields, identifiers, and product decisions. The baseline was frozen before fitting the model, and both methods use the same feature frame and client-grouped holdout.

The leakage experiment deliberately added an exact copy of the label. The grouped decision-tree score rose to `1.000` with that leak and fell to `0.604` after the leak was removed. This confirms that the test harness exposes label leakage rather than rewarding it.

## 5. Evaluation

The leakage notebook uses `GroupShuffleSplit` with `test_size=0.25` and `random_state=42`, grouping by client. The grouped test contains 11 clients. The majority-class baseline is `0.607`; the honest decision-tree score is `0.604`. The deliberately leaked score is `1.000` and is not a valid result.

The ML-08 model improves ranking precision on the same held-out rows:

| Method | Precision@50 | Precision@100 |
|---|---:|---:|
| ML-07 transparent baseline | 0.220 | 0.280 |
| ML-08 Random Forest | 0.440 | 0.430 |

The held-out decline-proxy base rate is `0.393`, and the model's ROC AUC is `0.573`. Accuracy is `0.590`, but precision@K is the more relevant metric for this ranking lane. Error analysis found 5,205 false negatives and 748 false positives, so the model improves the top of the queue without being a complete detector.

## 6. Interpretation

The strongest usable finding so far is about review opportunity, not causality. Higher impression volume gives an editor more measurable evidence and more potential upside, so it is reasonable as a prioritization component. CTR relative to position is useful for forming a diagnostic queue, but its mixed bucket result and the zero-click weak pick show why query mix, measurement quality, and page context must be checked by a human.

The model's top importance values were average position (`0.261`), impressions (`0.256`), active days (`0.213`), CTR (`0.196`), and clicks (`0.074`). These are diagnostic clues about the observed slice, not causal explanations. The model's false negatives show that similar first-half performance can still lead to different second-half movement.

## 7. Recommendation

1. **Review high-volume, low-CTR pages first**, beginning with the baseline queue's `ctr_fix_candidate` rows. Check query mix and title/snippet alignment before making any edit.
2. **Use impression volume as an opportunity filter**, not as proof of poor content. High impressions with zero clicks should be treated as a measurement or query-intent investigation.
3. **Keep `low_evidence` rows in monitoring**, because sparse history cannot support a confident intervention.
4. **Use the Random Forest only as a ranking aid.** It beats the frozen baseline at precision@50 and precision@100 on held-out clients, but the modest ROC AUC and many false negatives require human review.

Confidence is directional and moderate for queue prioritization, low for any claim about future improvement, and zero for causal claims. This work does not predict Google's algorithm.

## 8. Reproducibility

From a fresh clone:

```powershell
pip install -r requirements.txt
```

Request access to the FlyRank warehouse and provide a read-only Hugging Face token through the notebook's hidden prompt or the `HF_TOKEN` environment variable. Never paste the token into a cell or commit it.

Run these notebooks in order:

1. `work/notebooks/w01_research_question.ipynb`
2. `work/notebooks/w02_ml_task_framing.ipynb`
3. `work/notebooks/w03_data_contract.ipynb`
4. `work/notebooks/w03_feature_leakage_check.ipynb`
5. `work/notebooks/w04_baseline_score.ipynb`
6. `work/notebooks/w05_model.ipynb`

The warehouse notebooks use the March 2026 partition and DuckDB Parquet queries. The baseline uses `random_state=42` wherever a split is required. The baseline queue is regenerated at `work/outputs/baseline_action_score.csv`; it is ignored by Git because the repository blocks committed datasets.

ML-08 writes the aggregate metric receipt to `work/outputs/model_metrics.json`. The baseline queue remains at `work/outputs/baseline_action_score.csv`; it is regenerated by ML-07 and ignored by Git because the repository blocks committed datasets.

## Data credit

Data source: FlyRank, `internship-warehouse`, pseudonymized internship release. The repository's `DATA_USE.md` and `docs/data-dictionary.md` define the public-safety and measurement rules used here.
