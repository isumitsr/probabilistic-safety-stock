# Technical implementation reference

## 1. Scope

This document records the implemented technical design through notebook 07. It is intended for someone who needs to reproduce, review, or extend the work. It covers data flow, formulas, algorithms, validation rules, generated artifacts, and known technical boundaries.

Implemented notebooks:

1. `01_data_audit_and_eda.ipynb`
2. `02_data_preparation_and_series_selection.ipynb`
3. `03_time_series_eda.ipynb`
4. `04_forecasting.ipynb`
5. `05_uncertainty_and_safety_stock.ipynb`
6. `06_statistical_inference_and_distribution_analysis.ipynb`
7. `07_inventory_policy_simulation_and_final_evaluation.ipynb`

## 2. Execution contract

Run the notebooks from the repository root in numeric order. The notebooks use project-relative path discovery and save generated CSV files under `data/processed/`. Those CSV files are excluded from Git because they are reproducible from the raw data.

Tracked outputs include the executed notebooks, method documentation, result summaries, and report figures.

The raw files are obtained with:

```bash
python scripts/download_data.py
```

The helper downloads the official Kaggle files, validates SHA-256 checksums, and leaves existing valid files unchanged.

## 3. Source data

| File | Key content | Role |
|---|---|---|
| `sales_train_evaluation.csv` | 30,490 series and daily sales through `d_1941` | Primary sales source |
| `sales_train_validation.csv` | Same series through `d_1913` | Structural comparison in notebook 01 |
| `calendar.csv` | M5 day code, date, year-week, weekday, events, SNAP | Calendar enrichment |
| `sell_prices.csv` | Store-item-week selling price | Price enrichment |
| `sample_submission.csv` | Competition submission layout | File-integrity audit only |

The main analytical unit is a product-store series. A `series_key` is created as:

```text
series_key = item_id + "_" + store_id
```

## 4. Time boundaries

The implemented constants are:

```python
TRAIN_END = 1885
VALIDATION_START = 1886
VALIDATION_END = 1913
TEST_START = 1914
TEST_END = 1941
```

| Split | Inclusive days | Number of days |
|---|---|---:|
| Training | `d_1:d_1885` | 1,885 |
| Validation | `d_1886:d_1913` | 28 |
| Final test | `d_1914:d_1941` | 28 |

Notebook 02 reads only identifiers and daily values through `d_1913` into `sales_development`. Every downstream processed table is built from that object. Assertions verify that the maximum development day is 1913.

Notebook 01 reads the evaluation-file header to confirm that the source extends through `d_1941`, but its sales-value audit and initial descriptive charts use only `d_1` through `d_1885`. Notebook 02 then reads development values through `d_1913`. Final-test values from `d_1914` through `d_1941` are first loaded in notebook 07, after the policy rules have been fixed.

## 5. Notebook 01 technical design

### File validation

`scripts/download_data.py` supplies the required filenames and expected checksums. The notebook confirms existence, nonzero size, and checksum validity before analysis.

### Memory handling

Daily sales columns are loaded as `int16`. Identifier columns are loaded as categorical values. This reduces memory use for the wide sales tables without changing integer sales values.

### Sales audit

The audit checks:

- row count and identifier cardinality;
- uniqueness of series IDs;
- total number of series-day observations;
- count of zero and negative values; and
- the daily-column range in validation and evaluation files.

The zero-sales proportion is:

$$
\text{zero share}=\frac{\#\{y_t=0\}}{T}.
$$

### Calendar audit

The notebook verifies unique `d` codes, unique dates, daily continuity, and the expected first and last dates. Missing event fields are treated as the absence of a labeled event, not as missing calendar rows.

### Price audit

The proposed price key is:

```text
(store_id, item_id, wm_yr_wk)
```

The notebook checks duplicate keys, missing stored prices, and non-positive prices. It also compares the store and item identifier domains with the sales files.

## 6. Notebook 02 technical design

### Training-only selection measures

For series values $y_1,\ldots,y_T$, where $T=1885$:

**Mean daily sales**

$$
\bar y=\frac{1}{T}\sum_{t=1}^{T}y_t.
$$

**Sample standard deviation**

$$
s=\sqrt{\frac{1}{T-1}\sum_{t=1}^{T}(y_t-\bar y)^2}.
$$

**Coefficient of variation**

$$
CV=\frac{s}{\bar y},
$$

calculated only when $\bar y>0$.

**Zero-sales share**

$$
z=\frac{1}{T}\sum_{t=1}^{T}I(y_t=0).
$$

**Recent nonzero days** counts positive observations among the final 365 training days.

**Lag-7 autocorrelation** is the correlation between $(y_1,\ldots,y_{T-7})$ and $(y_8,\ldots,y_T)$. It is calculated in batches of 1,000 rows to avoid unnecessary full-size array copies.

### Selection plan

The project defines 12 store-category-profile slots in advance. Three series are selected for each profile.

| Profile | Eligibility | Sort order |
|---|---|---|
| Stable | mean >= 1; zero share <= 0.25; recent nonzero days >= 180 | lowest CV, then lowest zero share, then item ID |
| Weekly pattern | mean >= 0.5; zero share <= 0.40; recent nonzero days >= 100 | highest lag-7 correlation, then lowest zero share, then item ID |
| Intermittent | zero share in [0.65, 0.90]; recent nonzero days in [20, 150] | highest zero share, then highest CV, then item ID |
| Variable | mean >= 0.5; zero share <= 0.60; recent nonzero days >= 60 | highest CV, then highest zero share, then item ID |

Assertions require:

- 12 selected series;
- no duplicate item-store pair;
- 10 stores;
- 3 states;
- 3 categories; and
- 3 series in every profile.

### Wide-to-long transformation

`pandas.melt` converts 12 wide rows into 22,956 series-day rows. `day_number` is parsed from the `d_*` code and `split` is assigned with a boundary at 1885.

### Calendar join

Join type: left, many-to-one.

Join key:

```text
d
```

Selected fields include date, year-week, weekday, month, year, the primary event fields, and state SNAP indicators.

### Price join

The price table is first filtered to the selected item-store pairs. The long sales data are then joined using:

```text
(store_id, item_id, wm_yr_wk)
```

Join type: left, many-to-one.

No automatic price imputation is performed.

### Saved tables

`selected_series_metadata.csv` has 12 rows and these fields:

```text
series_key, profile, id, item_id, dept_id, cat_id, store_id, state_id,
mean_daily_sales, zero_sales_share, coefficient_of_variation,
recent_nonzero_days, lag7_autocorrelation
```

`selected_series_development.csv` has 22,956 rows and contains identifiers, date fields, split, sales, event fields, SNAP fields, and selling price.

## 7. Notebook 03 technical design

Notebook 03 filters the development table to `split == "train"` before drawing conclusions.

### Profile summary

It reports the number of selected series and medians of daily mean, zero share, CV, recent nonzero days, and lag-7 correlation by profile. Medians reduce the influence of one extreme selected series.

### Recent-history plot

The latest 365 training observations are plotted for every series. The smooth line is a seven-observation rolling arithmetic mean.

### Normalized weekday pattern

Each daily observation is normalized as:

$$
r_{it}=\frac{y_{it}}{\bar y_i},
$$

where $\bar y_i$ is the training mean of series $i$. The normalized values are then averaged by profile and weekday.

This gives every selected series a comparable scale. It is descriptive and does not adjust for prices, events, or other confounders.

### Autocorrelation table

Autocorrelations are calculated separately at lags 1, 7, 14, and 28, then summarized by profile. A positive lag-7 value supports weekly dependence but is not a model-performance measure.

### Seven-day totals

The first 1,883 training days form 269 complete non-overlapping weeks. The remaining two training days are excluded from this particular weekly distribution so every total contains exactly seven observations.

### Price completeness

The notebook reports missing-price rows and positive-sales rows without price by profile. It preserves missing values for later explicit treatment.

## 8. Notebook 04 technical design

### Input matrices

The long data are pivoted into:

- `training_wide`: 12 x 1,885;
- `validation_wide`: 12 x 28.

Completeness assertions reject missing sales cells.

### Forecast algorithms

Let $h=1,\ldots,28$ index the validation horizon.

**Last value**

$$
\hat y_{T+h}=y_T.
$$

**Seven-day seasonal naive**

$$
\hat y_{T+h}=y_{T-7+((h-1)\bmod 7)+1}.
$$

Operationally, the final seven training values are repeated four times.

**28-day mean**

$$
\hat y_{T+h}=\frac{1}{28}\sum_{j=0}^{27}y_{T-j}.
$$

Forecasts are floored at zero. All three current methods are already nonnegative for nonnegative sales, so the floor is also a safeguard for consistency with later extensions.

### Error definition

$$
e_t=y_t-\hat y_t.
$$

A positive error means under-forecasting.

### Metrics

**Mean absolute error**

$$
MAE=\frac{1}{n}\sum_{t=1}^{n}|e_t|.
$$

**Root mean squared error**

$$
RMSE=\sqrt{\frac{1}{n}\sum_{t=1}^{n}e_t^2}.
$$

**Weighted absolute percentage error**

$$
WAPE=100\times\frac{\sum_t|e_t|}{\sum_t y_t}.
$$

WAPE is stored as missing when $\sum_t y_t=0$ at the per-series level.

### Model-selection rule

Models are ordered by per-series MAE. Exact ties use this fixed priority:

1. `seasonal_naive_7`
2. `mean_28`
3. `last_value`

This rule produces one selected model for every series.

### Saved tables

| File | Rows | Grain |
|---|---:|---|
| `validation_point_forecasts.csv` | 1,008 | series x model x validation day |
| `forecast_validation_metrics.csv` | 36 | series x model |
| `forecast_model_selection.csv` | 12 | series |

## 9. Notebook 05 technical design

### Shared forecasting helper

`baseline_forecast(history, model, horizon)` implements the three forecast rules and rejects unknown model names. The same compact helper is used where the forecast rules must be repeated for calibration, validation, protection-period forecasting, and final simulation.

### Planning constants

```python
LEAD_TIME_DAYS = 7
SERVICE_TARGETS = [0.80, 0.90, 0.95]
```

The targets are forecast quantiles used as candidate planning levels. Until the simulation is run, they should not be called achieved service levels.

### Calibration origins

Each series uses 52 weekly origins from the latest training year. For origin $o$, only observations before $o$ are supplied to the forecasting rule. The following seven observations are summed as the actual lead-time sales.

The calibration table has:

```text
12 series x 52 origins = 624 rows
```

For each series, empirical error quantiles are calculated at 0.05, 0.10, 0.50, 0.80, 0.90, and 0.95.

### Prediction intervals

For point forecast $\hat D$ of seven-day demand and empirical error quantile $Q_e(p)$:

$$
Q_D(p)=\max(0,\hat D+Q_e(p)).
$$

The central 80% interval is:

$$
[\max(0,\hat D+Q_e(0.10)),\ \max(L_{80},\hat D+Q_e(0.90))].
$$

The central 90% interval uses the 0.05 and 0.95 error quantiles. The upper bound is forced to be at least the lower bound after nonnegative truncation.

Coverage is the mean of the Boolean indicator that the actual seven-day total lies inside the interval. Mean width is the average upper bound minus lower bound.

### Rolling validation

The 28-day validation period is divided into four seven-day cycles. For validation week $k$, the history includes training plus any validation observations revealed before that week's origin. This differs intentionally from notebook 04's fixed-origin comparison.

There are:

```text
12 series x 4 validation weeks = 48 weekly outcomes
```

### Pinball loss

For target quantile $q$, actual $y$, prediction $\hat y_q$, and difference $u=y-\hat y_q$:

$$
L_q(u)=\max(qu,(q-1)u).
$$

When $q$ is high, under-prediction receives greater weight than over-prediction. Pinball loss is a proper scoring rule for a quantile forecast.

### Safety stock

After validation, the selected forecasting rule receives all development observations through `d_1913` and predicts the next seven-day total. For each target $q$:

$$
\text{safety stock}_q=\max(0,Q_D(q)-\hat D),
$$

$$
\text{order-up-to level}_q=Q_D(q).
$$

The output has 36 rows: 12 series by three target quantiles.

### Saved tables

| File | Rows | Grain |
|---|---:|---|
| `weekly_uncertainty_calibration.csv` | 624 | series x training calibration origin |
| `validation_weekly_probabilistic_forecasts.csv` | 48 | series x validation week |
| `safety_stock_levels.csv` | 36 | series x target quantile |

## 10. Notebook 06 technical design

### Distribution diagnostics

Training observations form 269 complete seven-day totals per series. The notebook calculates mean, variance, variance-to-mean ratio, zero-week share, skewness, and empirical 80th, 90th, and 95th percentiles.

Normal, Poisson, and negative-binomial quantiles are calculated for comparison. Method-of-moments negative-binomial parameters are:

$$
r=\frac{\mu^2}{\sigma^2-\mu}, \qquad p=\frac{r}{r+\mu},
$$

when $\sigma^2>\mu$. Every selected series satisfies this overdispersion condition.

### Cluster bootstrap

The resampling unit is a complete product-store series. Each replicate samples 12 series with replacement and retains the four validation weeks belonging to each selected cluster. The project uses 5,000 replicates and percentile intervals.

For forecast model $m$, each replicate calculates mean validation MAE. Paired distributions are then formed for:

$$
MAE_{seasonal}-MAE_{mean28}
$$

and

$$
MAE_{last}-MAE_{mean28}.
$$

Coverage intervals use the same cluster design. Safety-stock quantile uncertainty uses 2,000 within-series bootstrap samples of the 52 calibration errors.

### Fourteen-day protection period

For weekly review $R=7$ and lead time $L=7$:

$$
P=R+L=14.
$$

The selected baseline predicts each 14-day training window. Empirical error quantiles at 0.80, 0.90, and 0.95 are clipped at zero to form the probabilistic protection buffers.

## 11. Notebook 07 technical design

### Matched policy buffers

For target $q$, series $i$ receives probabilistic buffer $b_{iq}$. The fixed buffer is:

$$
\bar b_q=\frac{1}{12}\sum_{i=1}^{12}b_{iq}.
$$

This gives both policies the same mean protection buffer before rounding.

### Periodic-review sequence

At review days 0, 7, 14, and 21:

1. scheduled orders arrive;
2. the selected model forecasts 14-day demand;
3. inventory position equals on-hand plus outstanding orders;
4. order-up-to level equals forecast plus policy buffer; and
5. a nonnegative order is scheduled to arrive after seven days.

Daily fulfillment is:

$$
fulfilled_t=\min(onhand_t,demand_t),
$$

$$
unmet_t=demand_t-fulfilled_t.
$$

Unmet demand is treated as lost sales.

### Paired inference

The final comparison uses 5,000 cluster-bootstrap replicates of complete series. Differences are always probabilistic minus fixed. Exact sign-flip tests enumerate all $2^{12}=4096$ sign patterns for per-series unmet units, stockout-day rate, and average ending inventory.

### Saved outputs

| File | Grain |
|---|---|
| `final_inventory_simulation_daily.csv` | series x target x policy x final-test day |
| `final_inventory_review_orders.csv` | series x target x policy x review |
| `final_policy_metrics.csv` | target x policy |
| `final_policy_metrics_by_profile.csv` | target x profile x policy |
| `final_policy_comparison_inference.csv` | target x comparison metric |

## 12. Data lineage

```text
raw M5 files
    -> notebook 01: training-only audit and initial EDA
    -> notebook 02: selection, split, and validated joins
    -> notebook 03: training-only time-series EDA
    -> notebook 04: validation point forecasts
    -> notebook 05: seven-day empirical uncertainty
    -> notebook 06: bootstrap inference and 14-day protection calibration
    -> notebook 07: fixed policies, then one final-test evaluation
```

## 13. Validation safeguards

The implementation checks:

- verified raw-file checksums;
- chronological and non-overlapping periods;
- training-only selection and inference inputs;
- selected-series store, state, category, and profile coverage;
- many-to-one calendar and price joins;
- expected rows in forecast, bootstrap, calibration, and simulation tables;
- nonnegative forecasts, inventory, fulfillment, and unmet units;
- fulfilled plus unmet units equal recorded demand;
- equal average protection buffers across policies; and
- exactly 12 selected series and 28 final-test days.

All notebooks 01 through 07 have been executed from top to bottom without error outputs, standard-error warnings, or unexecuted code cells.

## 14. Limitations

1. The 12 series are purposively selected and do not support population inference for all M5 products.
2. Recorded sales may be lower than unconstrained demand.
3. Inventory availability and Walmart replenishment decisions are not observed.
4. The lead time, review interval, lost-sales rule, and initial inventory are project assumptions.
5. Tail quantiles use only 51 or 52 training origins per series.
6. The final test contains four weekly reviews, producing wide policy-difference intervals.
7. Price, event, and SNAP variables are not used in the forecast baselines.
8. The final result differs across demand profiles and service metrics.

## 15. Course boundary

The implementation deliberately uses methods appropriate for AAI 500:

- descriptive statistics and exploratory distributions;
- variation, correlation, and time dependence;
- chronological validation and error metrics;
- empirical probability distributions and quantiles;
- asymmetric quantile loss;
- transparent statistical forecasting baselines; and
- cluster bootstrap confidence intervals and paired sign-flip tests;
- normal, Poisson, negative-binomial, and empirical distribution checks; and
- simulation-based inference under explicit assumptions.

No neural networks, advanced optimization systems, or opaque automated-modeling frameworks are used. Linear regression was not forced into the workflow because the empirical uncertainty and policy comparison answer the stated research question directly.
