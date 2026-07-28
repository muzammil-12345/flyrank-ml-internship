# Refresh or Retire: What Actually Predicts Content Decay

**Muzammil Zulfiqar** (FlyRank ML Internship Capstone, Search Intelligence Track)

---

## Abstract

Content teams need a repeatable way to decide which pages deserve a refresh before traffic disappears entirely. This project builds and validates a scoring model for that decision using the FlyRank internship warehouse, roughly 520,000 content records across 84 clients. A page is labeled a refresh candidate when its clicks dropped more than 20 percent between two recent windows and the page is older than 90 days. After catching and removing a leakage issue where the label's own inputs were also used as model features, three models (Logistic Regression, Decision Tree, Random Forest) were trained on content attributes that were never used to build the label. Random Forest reached a Precision at 50 of 0.62, well ahead of Logistic Regression and Decision Tree, which both scored 0.50. A follow up test showed that nearly all of that predictive power traces back to a single signal, whether a page received any search visibility at all in the last 90 days, while attributes like word count, backlinks, and keyword competition carried almost no independent signal on their own.

## Introduction

FlyRank manages content libraries that can run into the tens of thousands of pages per client. Deciding which pages to refresh, which to leave alone, and which to retire is currently a manual, judgment heavy process. The decision this work supports is simple to state and hard to do well at scale: given a page's current signals, should it be flagged for review this week. A model that reliably surfaces the right pages, even imperfectly, saves real review time and catches decay earlier than a human doing this by hand across thousands of rows.

## Data

This project uses the FlyRank internship warehouse release `v20260703`, specifically three tables: `dim_clients`, `dim_content`, and `fact_content_query_90d`. The content and query tables together cover roughly 520,000 content level records after aggregating from query level rows up to one row per client and content pair. No client names, domains, URLs, or credentials appear anywhere in this project, all identifiers are the anonymized hashes provided in the warehouse (`client_hash_id`, `content_hash_id`). One planned table, daily performance history, was not present in this release, so the trend calculation instead uses the last 30 day and prior 30 day windows already computed inside `fact_content_query_90d`.

## Methodology

**Label definition.** A page is marked `needs_refresh = 1` when its click trend between the last 30 day and prior 30 day windows dropped more than 20 percent, and the page is older than 90 days. The age gate exists so that pages too new to have earned traffic yet are not mistakenly flagged. Roughly 3 percent of pages met this bar (15,576 out of 519,606), which is a realistic base rate for this kind of problem.

**Leakage check, and what it caught.** The first modeling pass included the trend percentage and content age directly as model features, the same values used to build the label. Every model, and even the simple baseline rule, scored a Precision at 50 of exactly 1.00. That is not a good result, it means the model was just restating its own label definition rather than learning anything. The fix was to drop the trend and age derived columns from the feature set entirely, and add genuinely independent signals instead: search demand, keyword competition, backlink count, word count, and search performance metrics (impressions, click through rate, average position) that were never used to construct the label.

**Baseline.** A transparent hand written rule scores each page using its traffic trend and age, weighted 70/30. This baseline is kept in the results only as a sanity check that the label and scoring direction agree, since it still uses the same trend value the label is built from, it is not a fair comparison for the model.

**Validation design.** All 84 clients were split into a train and test group using a client grouped holdout, 67 clients in train and 17 in test, with zero overlap confirmed. This prevents the model from learning client specific quirks and then getting credit for "predicting" pages from a client it already saw in training. Precision at 50 was the primary metric, chosen because in practice a content team will act on a fixed sized weekly shortlist, not a probability threshold, so ranking quality at the top of the list is what matters.

**Models compared.** Logistic Regression, Decision Tree (max depth 6), and Random Forest (200 trees, max depth 8), all with class weighting to account for the roughly 3 percent positive rate.

## Results

| Model | Precision@50 | Feature set |
|---|---|---|
| Baseline (trend rule) | 1.00 | Uses the same signal as the label, not a fair comparison |
| Logistic Regression | 0.50 | Independent content and search signals only |
| Decision Tree | 0.50 | Independent content and search signals only |
| Random Forest | **0.62** | Independent content and search signals only |

![Model Precision@50 comparison, independent signals only](images/precision_comparison.png)
*Random Forest is the strongest honest model on this split, roughly 2.5x the label's base rate of 3 percent.*

Random Forest was the strongest honest model, reaching a Precision at 50 of 0.62 against a base rate of about 3 percent. To understand why, a follow up Decision Tree was fit on an adjusted feature set that replaced continuous click through rate with an explicit `has_90d_visibility` flag. Its feature importances showed that one signal, whether a page had any search visibility at all in the last 90 days, accounted for over 99.9 percent of the model's decisions.

![Feature importance once visibility is isolated as its own flag](images/feature_importance.png)
*Visibility alone accounts for essentially all of the signal; content attributes contribute almost nothing on their own.*

This lines up with a separate observation in the data: 92 percent of pages had zero or missing click through rate in the 90 day window, so the models were really learning a near binary split, did this page get any search visibility at all, rather than a fine grained score. Search demand, competition, backlinks, and word count each individually contributed less than 0.02 percent of the importance in this isolated view.

## Limitations and Honest Framing

These findings are directional and decision support only, not causal claims about why any individual page is losing traffic. The dataset is a single snapshot with a fixed 90 day query window, not a rolling time series, so this project measures association between current signals and a recent trend, not a validated forward looking forecast. Content attributes (word count, backlinks, keyword competition) showed almost no independent predictive value in this dataset, which itself is a useful, if humbling, finding, it does not mean those attributes never matter, only that they added little on top of a visibility signal in this specific data. The CTR based visibility signal is heavily zero inflated (92 percent zero or missing), so results should be read as "visible versus not visible" rather than a smooth ranking within the visible group. This is intern capstone work on a training warehouse, not a production recommendation system.

## Ranked Recommendations

1. **Treat zero visibility as the primary triage filter.** Before looking at any other metric, separate pages that received zero search clicks or impressions in the last 90 days. This single signal carried almost all of the model's predictive power.
2. **Do not rely on content attributes alone to prioritize refreshes.** Word count, backlink count, and keyword competition, taken alone, are weak standalone predictors of refresh need in this dataset. They may still matter as supporting context, but should not drive the ranked list by themselves.
3. **Use Precision@50 style shortlists, not full rankings.** Given the low positive rate (about 3 percent) and the sparsity of the CTR signal, a fixed weekly shortlist of the top ranked pages is a more honest and actionable output than a full confidence ranked list across every page.
4. **Keep a human in the loop on retire decisions specifically.** This model flags candidates for review, it does not and should not make an automatic retire call, since a false positive there is far more costly than a missed refresh.

## Reproducibility

All notebooks, including the weekly assignment notebooks and the capstone feature engineering and modeling notebook, are in the `work/` and `notebooks/` folders of the project repository. Random Forest and Decision Tree models use a fixed random seed (`random_state=42`), so the reported numbers reproduce exactly on a fresh run. The exact deployed URL for this paper is recorded in `submission/paper_url.txt` at the repository root, per the capstone submission format.

Repository: [github.com/muzammil-12345/flyrank-ml-internship](https://github.com/muzammil-12345/flyrank-ml-internship)

## Acknowledgments and Data Credit

Built on the FlyRank ML Internship dataset. [flyrank.ai](https://flyrank.ai)
