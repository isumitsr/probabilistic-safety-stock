# Statistical inference and distribution method

## Purpose

Notebook 06 adds formal uncertainty to the descriptive and forecasting results. It uses no final-test sales.

## Distribution diagnostics

Training sales are aggregated into 269 complete seven-day periods for each selected series. The analysis compares empirical weekly quantiles with normal, Poisson, and negative-binomial quantiles.

Every selected series has variance greater than its mean. This overdispersion violates the basic Poisson equal-mean-and-variance assumption. Normal and negative-binomial results are treated as diagnostic comparisons. Empirical forecast-error distributions remain the main probability model because they preserve skewness and zero-heavy behavior.

## Cluster bootstrap

Daily and weekly outcomes within one product-store series are related. Resampling individual observations would incorrectly treat them as independent. The bootstrap therefore samples the 12 complete series with replacement and retains the four validation weeks within every sampled series.

The project uses 5,000 cluster-bootstrap repetitions for:

- validation MAE by forecast model;
- paired MAE differences relative to the 28-day mean; and
- 80% and 90% prediction-interval coverage.

The reported 2.5th and 97.5th percentiles form a 95% bootstrap interval. Because the series were purposively selected, these intervals describe stability across the project sample rather than all 30,490 M5 series.

## Central Limit Theorem connection

The bootstrap distribution of the average MAE difference is plotted with a normal curve having the same mean and standard deviation. This illustrates the normal approximation associated with the Central Limit Theorem. The project retains percentile-bootstrap intervals because there are only 12 clusters and the normal approximation need not be accurate.

## Safety-stock uncertainty

Each seven-day safety-stock quantile is recalculated from 2,000 bootstrap samples of the 52 training forecast errors for that series. These intervals show uncertainty in the buffer itself, not future-demand prediction intervals.

## Protection-period correction

A periodic-review system must cover demand over the review interval plus replenishment lead time:

```text
protection period = 7-day review interval + 7-day lead time = 14 days
```

Notebook 06 therefore calibrates empirical 14-day forecast-error quantiles from training data. These nonnegative upper-error quantiles are the probabilistic buffers used in notebook 07.
