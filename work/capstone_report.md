# Capstone Report — Search Intelligence: Page Refresh Prioritization

* **Author:** Muhammad Hassan
* **Lane:** Machine Learning / Search Intelligence
* **Repo:** `mhassanbuilds/flyrank-ml-internship`
* **Date:** August 2026

---

## 0. Abstract

This project asks whether a machine-learning ranking model can help prioritize pages for review based on their observed growth or decline signals. The analysis uses the official FlyRank internship dataset containing 30,000 rows and an official feature vector with 52 model features. I compare a transparent Week-4 baseline with a Random Forest model and evaluate the model under both a row-level random split and a client-aware holdout. Under the client-aware holdout, the Random Forest measured **0.65 Precision@20**, compared with **0.15** for the Week-4 baseline. The output is intended as **decision-support** for prioritizing pages for human review, not as a guarantee that a page will improve after a refresh.

---

## 1. Problem framing

### Decision

The decision supported by this project is:

> **Which pages should be prioritized for further review based on their observed signals?**

The goal is not to predict Google's ranking algorithm or guarantee that refreshing a page will cause future growth.

### Unit of analysis

The unit of analysis is a **page-level record** in the FlyRank dataset.

### Output

The model produces a probability/ranking score for the target class. Pages can then be ranked according to the model score.

### Human action

A FlyRank editor or analyst could use the ranking as a prioritization aid:

1. Review the highest-ranked pages.
2. Inspect the page and its historical signals.
3. Decide whether further investigation or a content refresh is appropriate.
4. Make the final decision using human judgment.

### Cost of a wrong call

A false positive can cause an editor to spend time reviewing or refreshing a page that does not need attention.

A false negative can cause a potentially useful page to be missed during prioritization.

Because both errors have a practical cost, the model should be treated as a prioritization tool rather than an automatic decision rule.

### Why ML helps

The dataset contains multiple signals describing page, search-demand, and historical performance characteristics. A model can combine these signals into a single ranking score that may be useful for prioritization.

The purpose of the model is therefore to identify a useful observed ranking signal rather than to establish causality.

---

## 2. Data safety

### Dataset

The analysis uses the official FlyRank internship dataset and the official feature-preparation pipeline.

The processed feature vector contained:

* **30,000 rows**
* **52 model features**

The target used for the modeling experiment was:

`is_declining_label`

The target contained:

* **16,262 positive rows**
* **0.5421 positive rate**

This means approximately **54.21%** of the records belonged to the positive class in the modeling dataset.

### Excluded information

The modeling pipeline deliberately excludes target-related information from the feature matrix.

Fields such as target-derived or trend-derived information should not be used as predictors because they can directly reveal the outcome being predicted.

Pseudonymous client identifiers are used for grouping in the client-aware validation design, not as predictive features.

### Leakage risks considered

The main leakage risk identified in the final feature set is **temporal leakage**.

Historical performance features include signals such as:

* impressions
* clicks
* CTR
* average position
* pageviews
* sessions
* users
* engagement
* scroll activity
* traffic

These variables are not automatically leakage simply because they are predictive. However, their **measurement windows must be checked** to ensure that the information would have been available before the outcome period used to construct the target.

Search-demand variables such as:

* search volume
* competition
* CPC

also require a timing check to confirm that they represent information available at prediction time.

Page/content attributes such as word count and content age do not show an obvious leakage risk from their feature definitions alone.

### Leakage audit result

The final feature-name audit did not identify an obvious direct target feature.

However, historical performance variables remain the main area requiring confirmation of their measurement windows.

Therefore, the audit conclusion is:

> **No obvious direct target leakage was identified from the final feature names, but temporal availability of historical performance and search-demand features should be confirmed before making stronger generalization claims.**

No client-identifying information is intentionally included in this report.

---

## 3. Baseline

The Week-4 baseline provides a transparent comparison against the machine-learning model.

The baseline is useful because it provides a simple reference point for evaluating whether the Random Forest adds measurable ranking performance.

The baseline Precision@K measurements were:

| Metric        | Week-4 Baseline |
| ------------- | --------------: |
| Precision@20  |        **0.15** |
| Precision@50  |        **0.24** |
| Precision@100 |        **0.36** |

The model is therefore compared against an established baseline rather than evaluated in isolation.

### Model comparison

The following figure summarizes the measured ranking performance.

![Model comparison](figures/model_comparison.png)

*Figure 1. Model comparison using Precision@K. The Random Forest measured higher Precision@K than the Week-4 baseline under the client-aware evaluation setup.*

The comparison should be interpreted as a measured ranking-performance difference on this dataset and validation setup.

It does not establish that the model will produce the same improvement on future data.

---

## 4. Model / analysis

### Model

The final modeling experiment uses a **Random Forest classifier**.

Random Forest was selected because it can combine multiple numeric and categorical signals and capture non-linear relationships between features without requiring a simple linear relationship between each feature and the target.

### Feature matrix

The official FlyRank modeling utility was used to construct the feature matrix.

The resulting matrix contained:

> **30,000 rows × 52 features**

The feature set includes page/content attributes, search-demand variables, and historical performance variables.

The model does not use the target itself or explicitly target-derived trend fields as predictors.

### Target

The target is:

`is_declining_label`

In this experiment, the target represents the classification outcome supplied by the official FlyRank modeling pipeline.

The target contains:

* **16,262 positive observations**
* **54.21% positive rate**

The positive rate is important because Precision@K should not be interpreted without considering the underlying class distribution.

### Modeling approach

The Random Forest was trained using the official FlyRank modeling utilities.

Two validation designs were examined:

1. **Row-level random split**
2. **Client-aware holdout**

The second design is treated as the more conservative evaluation because records from held-out clients are not used for model training.

---

## 5. Evaluation

### Validation design

The first experiment used a row-level random split with:

* 80% training data
* 20% test data
* `random_state=42`
* stratification by the target

This setup can place records from the same client in both training and test sets.

The second experiment used the official **client-aware holdout**.

The client-aware split produced:

* **27,675 training rows**
* **2,325 test rows**

The client-aware design prevents the model from training on clients that appear in the test set.

This provides a more conservative assessment of how the ranking signal transfers to held-out clients.

### Before vs. after validation

The measured results were:

| Validation design      | Precision@20 | Precision@50 | Precision@100 |
| ---------------------- | -----------: | -----------: | ------------: |
| Row-level random split |     **0.95** |     **0.90** |      **0.90** |
| Client-aware holdout   |     **0.65** |     **0.74** |      **0.72** |

The difference demonstrates that measured performance is sensitive to the validation design.

![Validation sensitivity](figures/validation_sensitivity.png)

*Figure 2. Validation sensitivity of the Random Forest. Precision@K is substantially lower under the client-aware holdout than under the row-level random split.*

The row-level random split produced a Precision@20 of **0.95**, while the client-aware holdout produced **0.65**.

This difference does not by itself prove leakage.

It does show that the row-level random split can give a substantially more optimistic measurement when records from the same clients can appear in both training and test sets.

For stronger generalization claims, the client-aware result is therefore the more useful reference point in this experiment.

### Model vs. baseline on the client-aware setup

Under the client-aware evaluation:

| Method          | Precision@20 |
| --------------- | -----------: |
| Week-4 baseline |     **0.15** |
| Random Forest   |     **0.65** |

The Random Forest measured a higher Precision@20 than the baseline.

The absolute difference is:

**0.65 − 0.15 = 0.50**

This is a measured improvement of **0.50 Precision@20 points** over the baseline under the reported setup.

However, this should not be interpreted as proof of future refresh success.

### Base rate

The target positive rate was:

> **54.21%**

This is important context for interpreting Precision@K.

The model's Precision@20 of 0.65 is above the overall positive-class rate of 0.5421, indicating that the ranking concentrated more positive examples near the top of the ranked list in this evaluation.

However, Precision@K alone does not establish causal value or future business impact.

### Error analysis

I also inspected incorrect predictions from the client-aware test set.

The analysis focused on high-confidence disagreements between the predicted class and the observed label.

These examples show that a high model score does not guarantee that an individual page belongs to the target class.

This supports using the model as **decision-support for prioritization** rather than as an automatic decision rule.

The observed errors are useful for understanding model limitations, but the inspected examples alone are not sufficient to claim a general error pattern.

---

## 6. Interpretation

The model's main output is a ranking signal that combines multiple observed page and performance characteristics.

The most important methodological finding from the validation audit is not simply the high Random Forest score under the row-level split.

It is the difference between the two validation designs.

### Key observation

The Random Forest measured:

* **0.95 Precision@20** under the row-level random split
* **0.65 Precision@20** under the client-aware holdout

This shows that validation design has a substantial effect on the measured result.

The client-aware result should therefore be emphasized when discussing performance on unseen clients.

### Leakage interpretation

The feature audit did not identify an obvious direct target-related feature in the final model feature names.

However, several historical performance variables require measurement-window confirmation.

This means the appropriate conclusion is not:

> "There is definitely no leakage."

Instead, the more accurate conclusion is:

> **No obvious direct target leakage was identified from the feature names, while temporal availability of historical variables remains an important validation check.**

### Surprising result

The largest surprise was the sensitivity of the model's measured performance to the validation design.

The row-level split produced a much stronger result than the client-aware holdout.

This demonstrates why a strong metric under a convenient random split should not automatically be treated as evidence of strong real-world generalization.

---

## 7. Recommendation

The Random Forest can be used as a **prioritization aid** for human review.

### Recommended workflow

A practical workflow would be:

1. Generate model scores for eligible pages.
2. Rank pages by the model score.
3. Start human review with the highest-ranked pages.
4. Inspect page content and supporting search/performance signals.
5. Decide whether a refresh or further investigation is appropriate.
6. Track future outcomes separately rather than assuming that a high score guarantees improvement.

### Recommended use

The model is most appropriate for:

> **Prioritizing which pages deserve human attention first.**

It should not be used as:

* an automatic refresh decision
* a guarantee of future traffic growth
* a causal estimate of refresh impact
* a prediction of Google's ranking algorithm

### Confidence

Confidence in the measured ranking signal is **moderate within the tested dataset and validation setup**.

The Random Forest clearly measured higher Precision@K than the baseline under the client-aware evaluation.

However, the large difference between row-level and client-aware performance shows that validation design matters substantially.

Additional validation on future or independently held-out data would be needed before making a stronger deployment or generalization claim.

### Practical takeaway

The safest operational interpretation is:

> **Use the model to help decide where to look first, then let a human make the final decision.**

---

## 8. Reproducibility

The analysis was built using the official FlyRank repository and the project's `work/` directory.

### Repository

`mhassanbuilds/flyrank-ml-internship`

The official reference pipeline was kept separate from the work-specific analysis.

### Main notebook

The ML-09 validation audit is located at:

`work/notebooks/w06_validation_audit.ipynb`

### Official preparation steps

The experiment uses the official FlyRank scripts to prepare the feature vector and baseline:

```bash
python scripts/01_prepare_features.py
python scripts/02_baseline_score.py
```

The official modeling utilities are loaded from:

```text
scripts/03_train_model.py
```

### Validation

The ML-09 notebook evaluates the Random Forest under:

1. A row-level random split using `random_state=42`
2. The official client-aware holdout

The row-level split uses an 80/20 split and stratification by the target.

The client-aware evaluation produced:

* **27,675 training rows**
* **2,325 test rows**

### Reproducibility notes

The notebook records the validation methodology and measured results.

The official reference pipeline remains unchanged.

No datasets are committed to the repository.

The repository's `work/` directory contains the analysis notebooks, figures, and report rather than raw dataset files.

### Key measured results

For quick verification:

| Experiment                 | Precision@20 | Precision@50 | Precision@100 |
| -------------------------- | -----------: | -----------: | ------------: |
| Row-level Random Forest    |         0.95 |         0.90 |          0.90 |
| Client-aware Random Forest |         0.65 |         0.74 |          0.72 |
| Week-4 baseline            |         0.15 |         0.24 |          0.36 |

These are the measurements used throughout this report.

---

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset.

Data and research context are provided by [FlyRank](https://flyrank.ai).

---

## Claims checklist

This report follows the project's public-safety language requirements:

* **Observed** — used when describing observed relationships or results.
* **Measured** — used when describing experimental metrics.
* **Directional** — used for the measured improvement over the baseline.
* **Decision-support** — used to describe the intended operational use.

The report does not make causal claims about refreshes.

The report does not claim to predict Google's ranking algorithm.

The report does not guarantee future performance.

The model is presented as a ranking and prioritization aid rather than an automatic decision system.

