# Implementation walkthrough: current project state

## Why this document exists

This document explains the project as a connected piece of work rather than as seven separate notebooks. It focuses on the questions a reader is likely to ask:

- What problem are we trying to solve?
- Why is each step necessary?
- What does the code do?
- How should the output be read?
- What can we infer from the result?
- What can we not claim from the result?

The statistical analysis, inventory-policy simulation, and final holdout evaluation are complete. The implementation remains intentionally limited to transparent probability, statistics, statistical forecasting, bootstrap inference, and simulation methods appropriate for AAI 500.

## 1. Project objective

Retailers need enough inventory to satisfy demand without holding an unnecessarily large amount of stock. The difficulty is that future demand is uncertain and different products behave differently. A stable, frequently purchased item should not automatically receive the same buffer rule as a product that sells only occasionally.

The project asks:

> Under stated replenishment assumptions, can probabilistic demand forecasts reduce simulated stockouts or inventory held compared with a simple forecast and fixed-buffer policy?

This is a probability and statistics project. The purpose is not to build the most complex forecasting system possible. The purpose is to show how descriptive statistics, time dependence, forecast errors, empirical probability distributions, quantiles, and validation can support an inventory decision.

## 2. What the M5 data can and cannot tell us

The M5 dataset contains daily recorded unit sales, dates, events, SNAP indicators, and weekly selling prices. It does not contain:

- customer demand that could not be fulfilled;
- on-hand inventory;
- actual stockout events;
- supplier lead times;
- Walmart's replenishment rules; or
- holding and shortage costs.

This distinction controls the interpretation of the entire project. A recorded value of zero means no sale was recorded. It does not prove that demand was zero. Any later service level, stockout rate, or inventory level must therefore be described as a simulated outcome under project assumptions, not as a measured Walmart outcome.

## 3. Why the workflow is ordered this way

The notebooks follow a dependency chain:

```text
Raw-file audit
    -> chronological split and series selection
    -> time-series EDA
    -> point-forecast comparison
    -> uncertainty calibration and safety stock
    -> statistical inference and protection-period calibration
    -> inventory-policy simulation and final evaluation
```

Each stage answers a question needed by the next stage. We audit before modeling because corrupted or duplicated data would invalidate later results. We split by time before selecting models because random splitting would leak future information. We study demand behavior before choosing forecast comparisons. We estimate forecast uncertainty before setting probabilistic safety stock, quantify the uncertainty in the conclusions, and then use the protected final period once to evaluate the service-inventory tradeoff.

## 4. Notebook 01: raw-data audit and initial EDA

### Objective

The first notebook checks that the official M5 files are present, readable, internally consistent, and suitable for analysis. It also establishes the broad properties that motivate the rest of the project.

### Why this is necessary

A model can run successfully on incorrect data. Duplicate identifiers could count the same series twice, duplicate price keys could multiply rows during a join, missing dates could break a time series, and negative sales would be structurally inconsistent with the meaning of the target. These problems must be checked before any statistical result is trusted.

### What the notebook does

It verifies file checksums, loads memory-conscious data types, and checks:

- the number of products, stores, categories, departments, and states;
- duplicate product-store series identifiers;
- negative and zero sales observations;
- calendar uniqueness and daily continuity;
- price-key uniqueness and price validity; and
- whether the identifiers required for future joins are available.

It then plots total recorded sales over time and the distribution of zero-sales shares across product-store series.

### Main output and interpretation

The data contain 30,490 product-store series, built from 3,049 products across 10 stores. No duplicate series identifiers or negative sales values were found. The calendar is continuous and the stored price keys are unique.

About 68% of the inspected series-day values are zero. More importantly, the zero-sales share differs substantially across individual series. This means sparsity is not a small or uniform feature of the data. It supports the decision to compare products with different demand profiles rather than report only one dataset-wide average.

The total-sales plot changes over time. This indicates that the data are not exchangeable across dates and supports chronological validation. Randomly assigning dates to training and test sets would allow later demand conditions to influence an earlier forecast.

### Boundary to remember

Notebook 01 reads the evaluation-file header to confirm that the source extends through `d_1941`, but its sales-value audit and charts use only `d_1` through `d_1885`. The selected-series workflow begins in notebook 02, where selection statistics use the same training period and development data stop at `d_1913`. Final-test values from `d_1914` through `d_1941` are first loaded in notebook 07, after the policy rules are fixed.

## 5. Notebook 02: data preparation and series selection

### Objective

The second notebook creates a small, reproducible analysis dataset while protecting the future periods used for model assessment.

### Why chronological splitting is used

Forecasting always moves forward in time. The model should learn from the past and predict a later period. A random split would mix future and past observations and produce an unrealistically easy evaluation.

The project uses:

| Period | Days | Dates | Purpose |
|---|---|---|---|
| Training | `d_1` to `d_1885` | 2011-01-29 to 2016-03-27 | Series selection, EDA, and model fitting |
| Validation | `d_1886` to `d_1913` | 2016-03-28 to 2016-04-24 | Model comparison and uncertainty checks |
| Final test | `d_1914` to `d_1941` | 2016-04-25 to 2016-05-22 | Final evaluation after the complete design is fixed |

Only columns through `d_1913` are loaded into the development dataset. This is stronger than merely filtering later: the downstream table does not contain final-test sales.

### Why only 12 series are selected

The full dataset has 30,490 series. Modeling all of them would make the project much larger without automatically improving the explanation of the statistical ideas. The project instead uses a purposive sample that covers all 10 stores, all three states, all three categories, and four distinct demand profiles.

The sample is designed for contrast rather than population estimation. It helps show how methods behave for different kinds of demand, but it cannot establish average performance across every M5 series.

### How the demand profiles are defined

All selection measures use the training period only:

- Mean daily sales measures volume.
- Zero-sales share measures sparsity.
- Coefficient of variation measures standard deviation relative to the mean.
- Recent nonzero days measure current sales activity.
- Lag-7 autocorrelation measures repeated weekly timing.

The four profiles are:

- **Stable:** reasonably frequent sales and low relative variation.
- **Weekly pattern:** sufficient sales activity and high lag-7 correlation.
- **Intermittent:** a high proportion of zero-sales days but some recent activity.
- **Variable:** sufficient activity with high relative variation.

Each store-category slot is fixed before ranking candidates. The item identifier resolves exact ties. This makes the output reproducible and reduces the opportunity to choose series after seeing favorable validation results.

### Reshaping and joining

The sales file starts in wide format, with one row per product-store series and one column per day. It is reshaped into long format, with one row per series and date. Calendar data are joined using the M5 day code. Selling prices are joined using store, item, and Walmart year-week.

Both joins are declared as many-to-one. Assertions confirm that they do not create duplicate series-date rows or change the expected row count.

### Main output and interpretation

The processed development dataset contains 22,956 rows:

```text
12 series x 1,913 development days = 22,956 rows
```

It contains 22,620 training rows and 336 validation rows. There are no duplicate series-date observations, missing calendar dates, or final-test rows.

There are 2,009 rows without a matched selling price, but none of those rows has positive recorded sales. The missing values remain missing because the data do not establish that every missing price has the same cause. Filling them with a mean or zero would introduce unsupported information.

## 6. Notebook 03: time-series exploratory analysis

### Objective

Notebook 03 checks whether the selected labels correspond to visibly and statistically different demand behavior. It uses training data only, leaving validation data for model comparison.

### Sparsity and variability

The profile map compares zero-sales share with coefficient of variation. Stable series occupy the less sparse and less variable area. Intermittent series have long zero runs, while variable series combine more frequent sales with large changes in magnitude.

The stable group has a median zero-sales share of 3.5% and a median coefficient of variation of 0.578. The intermittent group has a 90% median zero-sales share and a median coefficient of variation of 3.343. The variable group has a median coefficient of variation of 2.655.

This evidence means one forecasting assumption is unlikely to perform equally well for every profile. It does not prove that the labels are permanent characteristics of the products.

### Recent histories

The latest training year is plotted for each series with daily observations and a seven-day rolling mean. The chart connects the summary measures to the actual time paths: stable series sell more continuously, intermittent series have long zero runs, and variable series contain larger spikes.

The rolling mean is used to make the local level easier to see. It is descriptive and is not itself the forecasting model.

### Weekday patterns

Sales are divided by each series' own training mean before being averaged by profile and weekday. This prevents a high-volume series from dominating the group pattern. A value above 1 means that the weekday is above the typical daily level for that series.

All four profiles have higher average relative recorded sales on Saturday. The stable group has the largest Saturday value at 1.381. The result establishes association with weekday, not causation. Price changes, events, store operations, and customer behavior remain possible explanations.

### Autocorrelation

Autocorrelation compares a series with earlier values of itself. Lag 7 measures similarity one week apart; lags 14 and 28 extend that check over two and four weeks.

The weekly-pattern group has median autocorrelations of 0.663 at lag 7, 0.602 at lag 14, and 0.497 at lag 28. This supports including a seven-day seasonal-naive forecast in the baseline comparison. Autocorrelation is supporting evidence, not a guarantee that the seasonal method will win on validation data.

### Seven-day totals

Seven-day sales totals connect daily forecasting to the later replenishment assumption. Median weekly recorded sales are 48 units for stable series, 42 for weekly-pattern series, 6 for variable series, and 0 for intermittent series.

The intermittent median is zero even though its mean is 0.94 units. Occasional nonzero weeks raise the mean while at least half of the weeks remain zero. This is a useful example of why the mean alone can be misleading for skewed or sparse data.

## 7. Notebook 04: point-forecast comparison

### Objective

Notebook 04 establishes an understandable point-forecast benchmark before uncertainty or inventory assumptions are added. Each model uses data through `d_1885` and predicts the same 28 validation days.

### Why simple baselines are appropriate here

A more complicated model is not automatically a better statistical model. Simple baselines reveal how much performance comes from recent level or weekly repetition. They are easy to reproduce and interpret, and they provide a fair reference for any later regression or introductory machine-learning model.

The three methods are:

- **Last value:** repeat the final training observation.
- **Seasonal naive:** repeat the final seven training observations four times.
- **28-day mean:** repeat the mean of the final 28 training days.

The forecasts use a fixed origin. Validation actuals are not fed back into the 28-day forecasts.

### How performance is measured

- MAE gives the average absolute error in sales units.
- RMSE gives more weight to large errors.
- WAPE divides total absolute error by total actual sales and expresses it as a percentage.

MAE is the primary per-series selection metric because it remains in understandable units and is defined even for a validation period with zero total sales. WAPE is left undefined when the denominator is zero.

### Main output

| Model | MAE | RMSE | WAPE |
|---|---:|---:|---:|
| Mean of final 28 days | 3.121 | 5.604 | 29.736% |
| Seasonal naive, 7 days | 3.250 | 5.525 | 30.970% |
| Last value | 4.905 | 9.592 | 46.739% |

The 28-day mean has the lowest overall MAE and WAPE. Seasonal naive has a slightly lower RMSE. This difference is meaningful: the preferred model can depend on whether typical errors or occasional large errors receive more weight.

Lowest per-series MAE selects the 28-day mean for 7 series, seasonal naive for 4, and last value for 1. The result supports series-specific selection rather than declaring one universal winner.

One intermittent series, `HOBBIES_2_110_CA_3`, has zero actual sales throughout validation. Its zero seasonal-naive forecast therefore has zero MAE and RMSE, while WAPE is undefined. This is mathematically correct but should not be interpreted as proof of reliable future forecasting.

## 8. Notebook 05: uncertainty and safety stock

### Objective

A point forecast says what demand level is expected, but inventory decisions also need to account for forecast error. Notebook 05 estimates an empirical error distribution for seven-day demand and converts its upper quantiles into planning buffers.

### Why empirical errors are used

The project does not assume forecast errors are normally distributed. This matters because intermittent and variable sales can be skewed, contain many zeros, and have occasional spikes. Empirical quantiles use the observed historical error distribution directly.

For each series, the selected model makes rolling seven-day forecasts at 52 weekly origins in the latest training year. The calibration error is:

```text
seven-day error = actual seven-day sales - forecast seven-day sales
```

A positive error means the model under-forecasted. Upper positive error quantiles therefore represent the extra units that would have protected against a chosen share of historical under-forecast errors.

### Validation of uncertainty

The validation period is divided into four weekly cycles for each of 12 series, producing 48 outcomes. Unlike the fixed-origin model-selection comparison, each weekly uncertainty forecast may use observations revealed in earlier validation weeks. This mirrors a weekly operational update.

The central 80% interval uses the 10th and 90th error quantiles. The central 90% interval uses the 5th and 95th quantiles.

| Interval | Target coverage | Observed coverage | Mean width |
|---|---:|---:|---:|
| Central 80% | 80.00% | 79.17% | 34.81 units |
| Central 90% | 90.00% | 85.42% | 45.42 units |

The 80% interval is close to its target. The 90% interval undercovers by 4.58 percentage points, so it should not be described as perfectly calibrated. Only 48 validation outcomes are available, making the observed percentages sensitive to a few cases.

Pinball loss is also calculated for the 80th, 90th, and 95th demand quantiles. It applies a larger penalty to under-forecasting as the target quantile rises. The mean losses are 3.500, 2.278, and 1.474 respectively. These values assess each quantile under its own asymmetric loss; they are not directly comparable as if they were three values of the same symmetric error measure.

### From uncertainty to safety stock

The notebook assumes a seven-day lead time and weekly review cycle. These are project assumptions because M5 does not provide Walmart's actual replenishment settings.

For target quantile `q`:

```text
demand quantile(q) = expected seven-day demand + empirical error quantile(q)
safety stock(q) = max(demand quantile(q) - expected demand, 0)
order-up-to level(q) = demand quantile(q)
```

The buffer is floored at zero because negative safety stock would not be meaningful in this planning rule.

Across the 12 series, mean safety stock rises from 10.61 units at the 80th percentile to 16.95 at the 90th and 22.68 at the 95th. The maximum values are 53.40, 69.30, and 88.21 units. Higher targets generally require more inventory protection, but the change depends on the forecast-error distribution of each series.

These seven-day values are preliminary planning levels. The final periodic-review policy requires a 14-day protection period because its 7-day review interval and 7-day lead time overlap.

## 9. Notebook 06: statistical inference and distribution analysis

Notebook 06 tests how stable the earlier point estimates are. It resamples complete product-store series rather than individual days, preserving the time-related observations within each series.

The 28-day mean has the lowest observed MAE, but the bootstrap interval for its difference from seasonal naive includes zero. The project therefore reports an observed ranking rather than claiming a statistically clear forecasting advantage.

The notebook also compares empirical weekly-sales quantiles with normal, Poisson, and negative-binomial calculations. Weekly variance exceeds weekly mean for every selected series, so the basic Poisson assumption is too restrictive. The normal approximation is useful for explaining sampling distributions and the Central Limit Theorem, but empirical forecast errors remain the primary uncertainty method.

Coverage uncertainty is also substantial. Central 80% coverage is 79.17%, with a 95% cluster-bootstrap interval from 68.75% to 89.58%. Central 90% coverage is 85.42%, with an interval from 72.92% to 95.83%.

Finally, notebook 06 recalibrates errors for the 14-day protection period required by weekly review and a 7-day lead time.

## 10. Notebook 07: inventory simulation and final evaluation

Notebook 07 fixes both policies before opening the final 28 days. The probabilistic policy uses a series-specific empirical buffer. The fixed policy uses the same buffer for every series, set equal to the probabilistic policy's average. This gives the policies comparable average protection while testing whether the allocation matters.

At the 90% target, the probabilistic policy raises fill rate from 96.55% to 97.89%, reduces unmet units from 119 to 73, and lowers average ending inventory from 70.40 to 68.99 units. At the 95% target, fill rate rises from 97.13% to 98.38%, unmet units fall from 99 to 56, and average ending inventory falls from 77.51 to 76.40 units.

The probabilistic policy nevertheless has slightly more stockout days. It reduces the size of shortages without reducing how often every small shortage occurs. All bootstrap intervals for policy differences include zero, so the evidence is suggestive rather than statistically clear.

Results also differ by profile. Stable series benefit most, intermittent series use much less inventory with unchanged fill rate, and variable series perform worse under the probabilistic policy in the final period.

## 11. Final conclusion

The project does not find a universal winner. Series-specific empirical uncertainty improves unit fill and slightly reduces average inventory at the higher targets, but the result is uncertain and depends on the demand profile and service metric. The evidence supports further use of probabilistic allocation, not a broad operational recommendation for Walmart.

## 12. How to read the project correctly

The project should be read as a transparent statistical decision study:

- EDA describes the data but does not establish causes.
- Validation compares methods on later observations.
- Quantiles express uncertainty rather than guarantee outcomes.
- Safety stock is calculated from historical forecast errors under stated assumptions.
- Inventory results are simulations, not observed Walmart operations.

That boundary keeps the project aligned with AAI 500 probability, statistics, statistical modeling, inference, and introductory machine learning.
