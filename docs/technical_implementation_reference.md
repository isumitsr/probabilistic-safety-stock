# Technical implementation reference

## 1. Scope

This document records the implemented technical design through notebook 05. It is intended for someone who needs to reproduce, review, or extend the work. It covers data flow, formulas, algorithms, validation rules, generated artifacts, and known technical boundaries.

Implemented notebooks:

1. `01_data_audit_and_eda.ipynb`
2. `02_data_preparation_and_series_selection.ipynb`
3. `03_time_series_eda.ipynb`
4. `04_forecasting.ipynb`
5. `05_uncertainty_and_safety_stock.ipynb`

Notebook 06, inventory-policy simulation, is not implemented yet.

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

Notebook 01 is different: it audits the full evaluation file and uses it for two initial descriptive charts. Those charts are not used for series selection or model selection. If the project later adopts a strict rule that no target value from the final period may be visualized before final evaluation, notebook 01's two EDA calculations should be limited to the training period. This does not affect notebooks 02 through 05.

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

`baseline_forecast(history, model, horizon)` implements the three forecast rules and rejects unknown model names. It is the only custom analytical function in notebooks 02 through 05 because the same logic is reused for calibration, validation, and next-period planning.

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

## 10. Data lineage

```text
M5 raw CSV files
    |
    | notebook 01: audit only
    v
sales_train_evaluation + calendar + sell_prices
    |
    | notebook 02: training-only measures, deterministic selection,
    |              long reshape, validated joins
    v
selected_series_metadata.csv
selected_series_development.csv
    |                         |
    | notebook 03             | notebook 04
    | training-only EDA       | validation forecasts and selection
    v                         v
report figures          validation_point_forecasts.csv
                        forecast_validation_metrics.csv
                        forecast_model_selection.csv
                                    |
                                    | notebook 05
                                    v
                        weekly_uncertainty_calibration.csv
                        validation_weekly_probabilistic_forecasts.csv
                        safety_stock_levels.csv
```

## 11. Validation safeguards

The implementation includes assertions or explicit checks for:

- verified raw-file checksums;
- expected chronological boundaries;
- non-overlapping validation and test periods;
- unique selected item-store pairs;
- required store, state, category, and profile coverage;
- expected long-table row counts;
- many-to-one calendar and price joins;
- absence of duplicate series-date rows;
- complete sales matrices for forecasting;
- expected forecast, calibration, validation, and safety-stock row counts;
- nonnegative forecasts and safety stock; and
- maximum processed day equal to `d_1913`.

All notebooks 02 through 05 have been executed from top to bottom with no error outputs, no standard-error warnings, and no unexecuted code cells.

## 12. Current limitations

1. The 12 series are purposively selected, so results cannot be generalized statistically to all 30,490 M5 series.
2. Recorded sales are an imperfect proxy for unconstrained demand.
3. The point-forecast candidates are intentionally simple and do not yet include a regression model with calendar or price predictors.
4. Model selection and uncertainty evaluation both use the same 28-day validation period in different ways. Final claims require the untouched final period.
5. Each series has only 52 weekly calibration errors, so empirical tail quantiles are uncertain.
6. The seven-day lead time and weekly review frequency are assumptions rather than observed Walmart settings.
7. Price and event fields are joined and audited but are not used in the current baseline forecasts.
8. No inventory process has been simulated, so service and inventory performance are not yet known.

## 13. Required interface for notebook 06

The inventory simulation should consume:

- observed evaluation-period sales for the selected series;
- the selected point-forecast model for each series;
- fixed-buffer policy parameters;
- probabilistic safety-stock values or demand quantiles;
- lead time, review interval, and starting-inventory assumptions; and
- a clearly defined lost-sales or backorder rule.

Both policies must use the same demand path and operational assumptions. The output should include at least:

- simulated units demanded and fulfilled;
- simulated unmet units;
- stockout-day rate;
- fill rate or cycle service level;
- average and ending inventory; and
- results by series, demand profile, and policy.

The final test should be opened only after these choices and comparison metrics are fixed.

## 14. Course boundary

The implementation deliberately uses methods appropriate for AAI 500:

- descriptive statistics and exploratory distributions;
- variation, correlation, and time dependence;
- chronological validation and error metrics;
- empirical probability distributions and quantiles;
- asymmetric quantile loss;
- transparent statistical forecasting baselines; and
- simulation-based inference under explicit assumptions.

No neural networks, advanced optimization systems, or opaque automated-modeling frameworks are required for the current research question. A later linear-regression comparison would be appropriate only if it has a clear hypothesis, uses training-only features, and is assessed on the same chronological validation period.
