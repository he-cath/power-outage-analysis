
# Analyzing Power Outages

**By Catherine He**

---

# Introduction

In this project, I examined a dataset of major power outages across the United States from January 2000 to July 2016. These major outages are defined by the Department of Energy as events that impacted at least 50,000 customers or caused an unplanned energy demand loss of at least 300 MegaWatts. The dataset was compiled by Sayanti Mukherjee et al. and published by Purdue University's Laboratory for Advancing Sustainable Critical Infrastructure.

**Central question:** *What factors are associated with the cause of a major power outage, and can we predict the cause from information available at the time the outage begins?*

Understanding what drives outages — severe weather, intentional attacks, equipment failure — matters for grid operators and policymakers. If the cause can be anticipated from contextual signals (region, time of year, climate conditions), utilities can pre-position crews and resources more effectively.

<div class="stat-row">
  <div class="stat-card"><div class="val">1,534</div><div class="lbl">Total outage events</div></div>
  <div class="stat-card"><div class="val">57</div><div class="lbl">Columns in raw dataset</div></div>
  <div class="stat-card"><div class="val">2000–2016</div><div class="lbl">Years covered</div></div>
  <div class="stat-card"><div class="val">7</div><div class="lbl">Cause categories</div></div>
</div>

The columns most relevant to this investigation are:

| Column | Description |
|---|---|
| `CAUSE.CATEGORY` | High-level cause of the outage (target variable) |
| `CLIMATE.REGION` | U.S. climate region where the outage occurred (9 regions) |
| `NERC.REGION` | North American Electric Reliability Corporation region |
| `MONTH` | Month in which the outage started (1–12) |
| `YEAR` | Year in which the outage started |
| `ANOMALY.LEVEL` | Oceanic El Niño/La Niña (ONI) index for the season |
| `CLIMATE.CATEGORY` | warm / normal / cold classification for that period |
| `OUTAGE.DURATION` | Duration of the outage event (minutes) |
| `CUSTOMERS.AFFECTED` | Number of customers impacted by the outage |
| `POSTAL.CODE` | Two-letter state abbreviation (used to derive North/South label) |

---

# Data Cleaning and Exploratory Data Analysis

## Cleaning

1. The raw Excel file has an unusual structure: row 0 is blank, row 1 contains column names, row 2 contains units, and actual data begins at row 3. Column names were extracted from row 1 and data sliced from row 3 onward.
2. Many numeric columns — including `OUTAGE.DURATION`, `CUSTOMERS.AFFECTED`, and `ANOMALY.LEVEL` — were stored as `object` dtype. `pd.to_numeric(..., errors='coerce')` converts them to floats, turning unparseable values into `NaN`.
3. The outage start and restoration timestamps were each split across two columns (date and time). These were combined into single `OUTAGE.START` and `OUTAGE.RESTORATION` datetime columns, and the originals dropped.
4. A binary `IF_SOUTH` column was added based on postal codes (1 for Southern states, 0 otherwise) to support the hypothesis test.
5. Missing values in `CUSTOMERS.AFFECTED` (~443 rows) and `DEMAND.LOSS.MW` (~705 rows) were left as `NaN` rather than imputed, since filling them could distort distributions in analyses that rely on these columns.

The first few rows of the cleaned DataFrame (selected columns):

| U.S._STATE | NERC.REGION | CAUSE.CATEGORY | OUTAGE.DURATION | CUSTOMERS.AFFECTED | OUTAGE.START |
|---|---|---|---|---|---|
| Minnesota | MRO | severe weather | 3060 | 70000 | 2011-07-01 17:00:00 |
| Minnesota | MRO | intentional attack | 1 | NaN | 2014-05-11 18:38:00 |
| Minnesota | MRO | severe weather | 3000 | 70000 | 2010-10-26 20:00:00 |
| Minnesota | MRO | severe weather | 2550 | 68200 | 2012-06-19 04:30:00 |
| Minnesota | MRO | severe weather | 1740 | 250000 | 2015-07-18 02:00:00 |

## Exploratory Data Analysis

### Univariate Analysis

Severe weather dominates as the cause of major outages (~763 events), followed by intentional attacks (~418). Together these two categories account for roughly 77% of all records. Rarer categories like islanding and fuel supply emergencies appear far less frequently, creating class imbalance to address in modeling.

<iframe src="assets/cause_categories.html" width="800" height="500" frameborder="0"></iframe>

The distribution of outage duration is strongly right-skewed: most outages resolve within a few hours (under ~2,000 minutes), but a long tail extends to tens of thousands of minutes. This skew motivates log-transforming duration as a feature in modeling.

<iframe src="assets/duration_dist.html" width="800" height="500" frameborder="0"></iframe>

### Bivariate Analysis

Fuel supply emergencies and public appeal outages tend to run far longer than intentional attacks, which are typically resolved quickly. Severe weather shows the widest spread of any category. This suggests that outage duration provides a signal about likely cause.

<iframe src="assets/duration_cause.html" width="800" height="500" frameborder="0"></iframe>

The heatmap below reveals geographic variation in cause composition. The Northeast and East North Central regions have the highest density of severe weather outages, while the West region has a notably higher share of intentional attacks. This confirms that climate region is an informative predictor.

<iframe src="assets/region_cause_heatmap.html" width="800" height="500" frameborder="0"></iframe>

### Interesting Aggregates

Grouping by cause category shows stark differences. Fuel supply emergencies average over 13,000 minutes (~9 days), while intentional attacks have a median duration of just 56 minutes. Severe weather has the highest median customers affected.

| CAUSE.CATEGORY | Duration mean (min) | Duration median (min) |
|---|---|---|
| equipment failure | 1816.9 | 221.0 |
| fuel supply emergency | 13484.0 | 3960.0 |
| intentional attack | 430.0 | 56.0 |
| islanding | 200.5 | 77.5 |
| public appeal | 1468.4 | 455.0 |
| severe weather | 3884.0 | 2460.0 |
| system operability disruption | 728.9 | 215.0 |

---

# Assessment of Missingness

## MNAR Analysis

The column `CAUSE.CATEGORY.DETAIL` has substantial missingness (~471 missing values). I believe this column is likely **MNAR** (Missing Not At Random). The detail field requires utility companies to supply a granular description of the event — for example a specific storm name or equipment type. Utilities may be less likely to report these details when the cause is ambiguous, contested, or still under investigation, which are precisely the situations associated with more unusual or severe events. The missingness is therefore related to the unobserved value of the detail itself — the defining characteristic of MNAR.

To make this missingness MAR instead, one would want additional data such as post-incident investigation reports, insurance claim records, or NERC event filings that could supply the missing detail independently of the reporting utility's discretion.

## Missingness Dependency

I analyzed the missingness of `CUSTOMERS.AFFECTED` and tested whether it depends on `CAUSE.CATEGORY` and `YEAR`.

### Cause Category — missingness depends on this column

<div class="hyp-box">
  <p><strong>Null hypothesis:</strong> The missingness rate of <code>CUSTOMERS.AFFECTED</code> is the same across all cause categories.</p>
  <p><strong>Alternative hypothesis:</strong> The missingness rate differs across cause categories.</p>
  <p><strong>Test statistic:</strong> Sum of absolute deviations of per-category missingness rate from the overall rate.</p>
  <p><strong>Observed statistic:</strong> 1.7413 &nbsp;|&nbsp; <strong>p-value:</strong> 0.0000 &nbsp;|&nbsp; <strong>Significance level:</strong> α = 0.05</p>
</div>

<iframe src="assets/miss_cause.html" width="800" height="500" frameborder="0"></iframe>

p-value = 0.0 < 0.05. <span class="result-badge badge-reject">Reject H₀</span> The missingness of `CUSTOMERS.AFFECTED` **does depend** on cause category. Fuel supply emergencies and public appeal outages have by far the highest missingness rates, likely because the affected customer count is harder to define or report for these event types.

<iframe src="assets/miss_perm.html" width="800" height="500" frameborder="0"></iframe>

### Year — missingness also depends on this column

<div class="hyp-box">
  <p><strong>Null hypothesis:</strong> The distribution of <code>YEAR</code> is the same when <code>CUSTOMERS.AFFECTED</code> is missing vs. not missing.</p>
  <p><strong>Alternative hypothesis:</strong> The distribution of <code>YEAR</code> differs between the two groups.</p>
  <p><strong>Test statistic:</strong> Difference in mean year (missing − non-missing).</p>
  <p><strong>Observed difference:</strong> 1.8541 &nbsp;|&nbsp; <strong>p-value:</strong> 0.0000 &nbsp;|&nbsp; <strong>Significance level:</strong> α = 0.05</p>
</div>

p-value = 0.0 < 0.05. <span class="result-badge badge-reject">Reject H₀</span> The missingness of `CUSTOMERS.AFFECTED` **does depend** on year. Earlier years tend to have higher missingness, suggesting that data collection practices improved over time.

---

# Hypothesis Testing

I tested whether power outages are equally distributed between Northern and Southern states.

<div class="hyp-box">
  <p><strong>Null hypothesis:</strong> The proportion of major power outages occurring in the South equals the proportion occurring in the North (each = 50%).</p>
  <p><strong>Alternative hypothesis:</strong> The proportions are not equal.</p>
  <p><strong>Test statistic:</strong> Chi-squared statistic (1 degree of freedom).</p>
  <p><strong>Significance level:</strong> α = 0.05</p>
</div>

<div class="stat-row">
  <div class="stat-card"><div class="val">666</div><div class="lbl">Southern outages</div></div>
  <div class="stat-card"><div class="val">868</div><div class="lbl">Northern outages</div></div>
  <div class="stat-card"><div class="val">26.60</div><div class="lbl">χ² statistic</div></div>
  <div class="stat-card"><div class="val">2.5 × 10⁻⁷</div><div class="lbl">p-value</div></div>
</div>

<iframe src="assets/hyp_test.html" width="800" height="500" frameborder="0"></iframe>

The p-value of ~2.5 × 10⁻⁷ is well below α = 0.05. <span class="result-badge badge-reject">Reject H₀</span> The proportions of major outages in the North and South are not equal. Northern states experience a disproportionately higher share, consistent with the concentration of severe weather events (ice storms, nor'easters) and aging infrastructure in older Northern cities.

*Note: this test uses a pre-defined North/South partition based on postal codes. A more refined analysis might use Census regions or NERC reliability boundaries.*

---

# Framing a Prediction Problem

My model predicts the **cause category** of a major power outage. This is a **multiclass classification** problem with 7 possible classes: severe weather, intentional attack, system operability disruption, public appeal, equipment failure, fuel supply emergency, and islanding.

**Response variable:** `CAUSE.CATEGORY`. Identifying the likely cause early determines how crews are dispatched, which agencies are notified, and how communications are handled.

**Features available at time of prediction:** Location (state, NERC region, climate region), time (year, month), and prevailing climate conditions (anomaly level, climate category). Outage duration and total customers affected are *not* available at prediction time and are excluded.

**Evaluation metric:** Weighted F1-score. The class distribution is heavily imbalanced — severe weather and intentional attacks make up ~77% of events (16.6× imbalance ratio). Accuracy would reward predicting "severe weather" always; F1-score penalizes poor recall on minority classes.

---

# Baseline Model

The baseline model is a **Decision Tree Classifier** with default hyperparameters, wrapped in a `sklearn` Pipeline.

## Features

| Feature | Type | Encoding |
|---|---|---|
| `CLIMATE.REGION` | Nominal | OneHotEncoder |
| `NERC.REGION` | Nominal | OneHotEncoder |
| `CLIMATE.CATEGORY` | Nominal/Ordinal (cold/normal/warm) | OneHotEncoder |
| `MONTH` | Quantitative (1–12) | Passthrough |
| `IF_SOUTH` | Binary quantitative | Passthrough |
| `ANOMALY.LEVEL` | Quantitative (continuous) | Passthrough |

## Performance

<table class="perf-table">
  <thead><tr><th>Split</th><th>Accuracy</th><th>Weighted F1</th></tr></thead>
  <tbody>
    <tr><td>Train</td><td>0.868</td><td>0.867</td></tr>
    <tr><td>Test</td><td>0.609</td><td>0.601</td></tr>
  </tbody>
</table>

The large train/test gap indicates the default decision tree severely overfits. There is clear room to improve through regularization, better feature engineering, and a more powerful ensemble method.

---

# Final Model

## Feature Engineering

Two new features were engineered in addition to the baseline features:

- **`IS_SUMMER`** (binary): A flag for June, July, and August. Summer is the peak season for severe storms and high-demand grid stress events. This captures a non-linear seasonal pattern that a raw month integer only partially reflects — the probability of a severe weather outage jumps substantially in summer months rather than increasing linearly.
- **`LOG_ANOMALY`** (quantitative): The signed log-transform of `ANOMALY.LEVEL` (`sign(x) × log(1 + |x|)`). Extreme El Niño / La Niña episodes have a disproportionate effect on severe weather risk. The log transform compresses extreme values while preserving the direction of the anomaly, giving the model a cleaner signal.

## Model and Hyperparameter Tuning

The final classifier is a **Random Forest** with `class_weight='balanced'` to handle class imbalance. Hyperparameters were selected via 5-fold cross-validated `GridSearchCV`:

- `n_estimators`: [100, 200] → best: **200**
- `max_depth`: [5, 10, 20, None] → best: **10**

## Performance

<table class="perf-table">
  <thead><tr><th>Model</th><th>Test Accuracy</th><th>Test Weighted F1</th></tr></thead>
  <tbody>
    <tr><td>Baseline (Decision Tree, default)</td><td>0.609</td><td>0.601</td></tr>
    <tr style="background:#eaf3de;"><td><strong>Final (Random Forest, tuned)</strong></td><td><strong>0.635</strong></td><td><strong>0.641</strong></td></tr>
  </tbody>
</table>

The final model improves test weighted F1 by **+0.040** over the baseline. The train/test gap also narrows (train F1 drops from 0.867 to 0.824 while test F1 rises), indicating better generalization. The biggest gains are on minority classes: system operability disruption (+0.12), equipment failure (+0.09), and islanding (+0.08) — exactly the intended effect of `class_weight='balanced'`.

---

# Fairness Analysis

**Question:** Does the final model perform worse for outages in the South than for outages in the North?

**Groups:** Southern outages (`IF_SOUTH == 1`) vs. Northern outages (`IF_SOUTH == 0`). This grouping is motivated by the hypothesis test above, which showed the two regions have significantly different outage counts and cause distributions.

<div class="hyp-box">
  <p><strong>Null hypothesis:</strong> Our model is fair. Its weighted F1-score for southern and northern outages are roughly the same, and any observed difference is due to random chance.</p>
  <p><strong>Alternative hypothesis:</strong> Our model is unfair. Its weighted F1-score for southern outages is lower than for northern outages.</p>
  <p><strong>Test statistic:</strong> Difference in weighted F1 (North F1 − South F1).</p>
  <p><strong>Observed difference:</strong> 0.1254 &nbsp;|&nbsp; <strong>p-value:</strong> 0.0100 &nbsp;|&nbsp; <strong>Significance level:</strong> α = 0.05</p>
</div>

<iframe src="assets/fairness.html" width="800" height="500" frameborder="0"></iframe>

The p-value of 0.0100 is below α = 0.05. <span class="result-badge badge-reject">Reject H₀</span> The model is **not fair** with respect to North/South geography. It performs significantly better on Northern outages (F1 = 0.688) than Southern outages (F1 = 0.563). This is likely because Northern states are dominated by severe weather and intentional attack outages — the two classes the model predicts best — while Southern states have a more varied cause distribution that is harder to classify.
