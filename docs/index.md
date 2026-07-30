# Refresh or Retire: What Actually Predicts Content Decay

**Muzammil Zulfiqar** — [github.com/muzammil-12345](https://github.com/muzammil-12345)
FlyRank ML Internship Capstone, Search Intelligence Track · July 31, 2026

---

## Abstract

Content teams need a repeatable way to decide which pages deserve a refresh before traffic disappears. This project builds and validates a scoring model for that decision using the FlyRank internship warehouse (~520,000 content records, 84 clients). A page is labeled a refresh candidate when its clicks drop more than 20% between two recent windows and the page is older than 90 days. After identifying and removing a label-leakage issue, three models — Logistic Regression, Decision Tree, and Random Forest — were trained on content attributes never used to construct the label. Random Forest performed best at a Precision@50 of 0.54, versus 0.52 for both other models. A follow-up ablation showed that nearly all of this predictive power traces to one signal: whether a page had any search visibility in the last 90 days. Attributes like word count, backlinks, and keyword competition carried almost no independent signal.

## Introduction

FlyRank manages content libraries running into the tens of thousands of pages per client. Deciding which pages to refresh, leave alone, or retire is currently manual and judgment-heavy. This work supports a simple, high-value decision: given a page's current signals, should it be flagged for review this week. A model that reliably surfaces the right pages — even imperfectly — saves review time and catches decay earlier than manual triage at scale.

## Data

This project uses FlyRank internship warehouse release `v20260703`: `dim_clients`, `dim_content`, and `fact_content_query_90d`. The content and query tables cover ~520,000 content-level records after aggregating to one row per client-content pair. No client names, domains, URLs, or credentials appear anywhere in this project — all identifiers are the anonymized hashes provided in the warehouse. A planned daily-performance-history table was not present in this release, so the trend calculation uses the last-30-day and prior-30-day windows already computed in `fact_content_query_90d`.

## Methodology

**Label.** `needs_refresh = 1` when the click trend between the last-30-day and prior-30-day windows drops more than 20%, and the page is older than 90 days (the age gate excludes pages too new to have earned traffic). 3.04% of pages met this bar (15,789 of 519,606).

**Leakage check.** The first modeling pass included the trend percentage and content age directly as features — the same values used to build the label. Every model, including the baseline rule, scored a Precision@50 of exactly 1.00, meaning the model was restating its own label rather than learning anything. The fix: drop trend- and age-derived columns from the feature set, and use only genuinely independent signals — search demand, keyword competition, backlink count, word count, and search performance (impressions, click-through rate, average position).

**Baseline.** A hand-written rule scores each page on traffic trend and age (70/30 weighted). It is included only as a sanity check that the label and scoring direction agree; since it reuses the label's own input, it is not a fair comparison to the models.

**Validation.** All 84 clients were split into train/test via a client-grouped holdout — 67 clients train, 17 test, zero overlap confirmed. This prevents the model from learning client-specific quirks and getting credit for "predicting" pages from a client it already saw. Precision@50 was the primary metric, since in practice a content team acts on a fixed-size weekly shortlist rather than a probability threshold.

**Models.** Logistic Regression, Decision Tree (max depth 6), and Random Forest (200 trees, max depth 8), all class-weighted for the ~3% positive rate.

## Results

| Model | Precision@50 | Feature set |
|---|---|---|
| Baseline (trend rule) | 1.00 | Same signal as the label — not a fair comparison |
| Logistic Regression | 0.52 | Independent content and search signals only |
| Decision Tree | 0.52 | Independent content and search signals only |
| Random Forest | **0.54** | Independent content and search signals only |

![Model Precision@50 comparison, independent signals only](images/precision_comparison.png)
*Random Forest is the strongest honest model on this split, roughly 18x the label's base rate of 3%.*

To understand why Random Forest led, a follow-up Decision Tree was fit on an adjusted feature set that replaced continuous click-through rate with an explicit `has_90d_visibility` flag. Feature importances showed this single signal — whether a page had any search visibility at all in the last 90 days — accounted for over 99.9% of the model's decisions.

![Feature importance once visibility is isolated as its own flag](images/feature_importance.png)
*Visibility alone accounts for essentially all of the signal; content attributes contribute almost nothing on their own.*

This aligns with a separate observation: 92% of pages had zero or missing click-through rate in the 90-day window, so the models were effectively learning a near-binary split — visible or not — rather than a fine-grained score. Search demand, competition, backlinks, and word count each individually contributed less than 0.02% of importance in this isolated view.

## Limitations

These findings are directional and decision-support only, not causal claims about why any individual page is losing traffic. The dataset is a single snapshot with a fixed 90-day query window, not a rolling time series, so this measures association with a recent trend rather than a validated forecast. Content attributes showed almost no independent predictive value here — a useful finding, but not evidence that they never matter, only that they add little on top of a visibility signal in this data. The CTR-based visibility signal is heavily zero-inflated (92% zero or missing), so results should be read as "visible vs. not visible" rather than a smooth ranking within the visible group. This is intern capstone work on a training warehouse, not a production system.

## Recommendations

1. **Treat zero visibility as the primary triage filter.** Separate pages with zero search clicks or impressions in the last 90 days before looking at any other metric — this signal carries nearly all of the model's predictive power.
2. **Don't rely on content attributes alone.** Word count, backlinks, and keyword competition are weak standalone predictors here; they may still matter as supporting context, but shouldn't drive the ranked list by themselves.
3. **Use Precision@50-style shortlists, not full rankings.** Given the low positive rate (~3%) and sparse CTR signal, a fixed weekly shortlist is more honest and actionable than a full confidence-ranked list.
4. **Keep a human in the loop on retire decisions.** This model flags candidates for review — it should not make automatic retire calls, since a false positive there is far more costly than a missed refresh.

## Reproducibility

All notebooks — weekly assignments and the capstone feature engineering/modeling notebook — are in the `work/` and `notebooks/` folders of the repository. Random Forest and Decision Tree use a fixed random seed (`random_state=42`), so reported numbers reproduce exactly on a fresh run.

**Repository:** [github.com/muzammil-12345/flyrank-ml-internship](https://github.com/muzammil-12345/flyrank-ml-internship)

## Acknowledgments

Built on the FlyRank ML Internship dataset. [flyrank.ai](https://flyrank.ai)
