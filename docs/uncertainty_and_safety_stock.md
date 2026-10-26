# Uncertainty calibration and safety-stock planning

## Purpose

Notebook 05 measures uncertainty around each selected point forecast and converts it into planning buffers. It does not simulate an inventory system or claim observed service levels.

## Planning assumptions

- Lead time: 7 days.
- Review frequency: weekly.
- Forecast model: the baseline selected for each series in notebook 04.
- Final-test sales remain withheld.

The M5 files do not provide Walmart's supplier lead times or review policy. Seven days is therefore a transparent project assumption that matches the weekly patterns examined in the EDA.

## Empirical calibration

For each series, notebook 05 creates 52 weekly forecast origins from the most recent training year. At every origin it:

1. generates a seven-day forecast using only data available at that origin;
2. sums forecast and actual recorded sales over the next seven days; and
3. stores the error, defined as actual minus forecast.

The empirical 5th, 10th, 50th, 80th, 90th, and 95th percentiles of those errors describe the forecast-error distribution for that series. This approach preserves skewness and does not require normally distributed errors.

## Validation

Uncertainty is evaluated on four non-overlapping weekly periods in the 28-day validation window. Central 80% and 90% interval coverage and width are reported, along with pinball loss for the upper quantiles.

## Safety-stock calculation

For a chosen upper quantile `q`, the seven-day planning level is:

```text
planning level(q) = expected seven-day demand + max(error quantile(q), 0)
safety stock(q) = max(error quantile(q), 0)
```

The buffer is floored at zero because a negative upper-error quantile would otherwise recommend negative safety stock. The 80th, 90th, and 95th percentiles provide increasingly conservative planning choices.

These seven-day values are preliminary planning inputs. Notebook 06 recalibrates the buffers for the full 14-day protection period implied by a seven-day review interval plus a seven-day lead time, and notebook 07 evaluates the resulting policies in a simulation. The simulated service levels are conditional on the stated assumptions; M5 contains recorded sales rather than true demand, inventory positions, or observed stockout events.
