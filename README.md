# Major U.S. Power Outages: Causes, Patterns, and Prediction

## Overview

This project investigates major power outages across the United States from 2000 to 2016. The central question is: **can we predict the cause of a major power outage from information available at the time it begins?**

The analysis covers the full data science lifecycle — from data cleaning and exploratory analysis, through missingness assessment and hypothesis testing, to building and evaluating a predictive model.

---

## Dataset

- **Source:** Purdue University's Laboratory for Advancing Sustainable Critical Infrastructure  
- **Original file:** `outage.xlsx`  
- **Rows:** 1,534 (each row = one major outage event)  
- **Columns:** 57 (geographic, climatic, economic, and outage characteristics)

Major outages are defined by the Department of Energy as events impacting at least 50,000 customers or causing an unplanned energy demand loss of at least 300 MW.

---

## Analysis Summary

### Data Cleaning and EDA
- Converted numeric columns stored as `object` dtype using `pd.to_numeric`
- Combined split date/time columns into proper `datetime` objects
- Added a binary `IF_SOUTH` column based on state postal codes
- Left missing values in `CUSTOMERS.AFFECTED` and `DEMAND.LOSS.MW` as `NaN` to avoid distorting distributions
- Univariate analysis revealed severe weather dominates (~50% of outages) and durations are strongly right-skewed
- Bivariate analysis showed cause composition varies significantly by climate region and that fuel supply emergencies have by far the longest durations

### Assessment of Missingness
- `CAUSE.CATEGORY.DETAIL` is argued to be **MNAR** — utilities are less likely to report granular details when the cause is ambiguous or under investigation
- Permutation tests show `CUSTOMERS.AFFECTED` missingness depends on both `CAUSE.CATEGORY` (p = 0.0) and `YEAR` (p = 0.0)

### Hypothesis Testing
- **Question:** Are outages equally distributed between Northern and Southern states?
- **Test:** Chi-squared test — χ² = 26.60, p = 2.5 × 10⁻⁷
- **Conclusion:** Reject the null hypothesis. Northern states experience significantly more major outages, consistent with the geographic concentration of severe weather events and aging infrastructure.

### Prediction Problem
- **Task:** Multiclass classification — predict `CAUSE.CATEGORY` (7 classes) from features available at the start of an outage
- **Metric:** Weighted F1-score, chosen due to a 16.6× class imbalance between the largest and smallest classes

### Baseline Model
- **Model:** Decision Tree Classifier with default hyperparameters in a `sklearn` Pipeline
- **Features:** `CLIMATE.REGION`, `NERC.REGION`, `CLIMATE.CATEGORY` (OneHotEncoded), `MONTH`, `IF_SOUTH`, `ANOMALY.LEVEL` (passthrough)
- **Test F1:** 0.601 — reasonable starting point but the model overfits badly (train F1 = 0.867)

### Final Model
- **Model:** Random Forest Classifier with `class_weight='balanced'`
- **New features:** `IS_SUMMER` (binary flag for Jun/Jul/Aug capturing peak storm season non-linearly) and `LOG_ANOMALY` (signed log-transform of anomaly level to compress extreme climate values)
- **Hyperparameter tuning:** `GridSearchCV` over `n_estimators` and `max_depth` → best: 200 trees, max depth 10
- **Test F1:** 0.641 — improves over baseline by +0.040, with narrowed train/test gap showing better generalization

### Fairness Analysis
- **Groups:** Northern vs. Southern outages
- **Result:** p = 0.0100 < 0.05 — the model performs significantly better on Northern outages (F1 = 0.688) than Southern outages (F1 = 0.563), likely because Southern states have a more varied cause distribution that is harder to classify

---

## Key Results

| Model | Test Accuracy | Test Weighted F1 |
|---|---|---|
| Baseline (Decision Tree) | 0.609 | 0.601 |
| Final (Random Forest, tuned) | 0.635 | **0.641** |

The largest per-class gains in the final model are on minority classes: system operability disruption (+0.12), equipment failure (+0.09), and islanding (+0.08).
