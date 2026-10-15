# Point-forecast method

## Purpose

Notebook 04 compares simple, interpretable forecasting rules before uncertainty and inventory decisions are introduced. The comparison uses a fixed 28-day validation period and leaves the final 28 days untouched.

## Data design

- Training history: `d_1` through `d_1885`.
- Validation horizon: `d_1886` through `d_1913`.
- Final test: `d_1914` through `d_1941`, withheld.
- Forecasts are made from one origin. Actual validation values are not fed back into later predictions.

## Baselines

| Model | Forecast rule | Reason for inclusion |
|---|---|---|
| Last value | Repeats the final training observation | Minimal persistence benchmark |
| Seasonal naive, 7 days | Repeats the final observed week four times | Directly represents weekly repetition found in the EDA |
| Mean, 28 days | Repeats the mean of the final 28 training days | Smooth local-level benchmark |

## Evaluation and selection

MAE is the primary per-series selection metric because it remains interpretable in units sold. RMSE gives extra weight to large misses. WAPE provides a scale-adjusted summary when total actual demand is positive.

The model with the lowest validation MAE is selected separately for each series. Ties are resolved deterministically by model name. This rule is fixed before uncertainty calibration.

WAPE is undefined when all actual values in a validation series are zero. Those cases retain MAE and RMSE and are not assigned an artificial WAPE value.
