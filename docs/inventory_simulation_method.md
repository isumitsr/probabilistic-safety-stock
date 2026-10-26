# Inventory simulation and final-evaluation method

## Policies

Notebook 07 compares two policies on the same 12 series and final 28-day demand path.

- The probabilistic policy assigns each series its empirical 14-day forecast-error quantile.
- The fixed policy assigns every series the same additive buffer.

At each target quantile, the fixed buffer equals the mean probabilistic buffer across the 12 series. Both policies therefore receive the same average buffer before inventory-unit rounding. This comparison tests the allocation of uncertainty rather than simply giving one policy more safety stock.

## Operating assumptions

- Review interval: 7 days.
- Replenishment lead time: 7 days.
- Protection period: 14 days.
- Unmet demand: lost sales, not backordered.
- Initial inventory: selected seven-day forecast plus the policy's initial buffer.
- Forecast model: the validation-selected baseline for each series.
- Inventory units and order-up-to levels: rounded up to whole units.

At each weekly review, scheduled orders arrive, the next 14 days are forecast, and an order raises inventory position to the policy's order-up-to level. That order arrives after seven days. Daily recorded sales then consume available inventory.

## Metrics

- Fill rate: fulfilled units divided by recorded demand.
- Stockout-day rate: share of evaluated days with unmet units.
- Unmet units: recorded sales that could not be filled in the simulation.
- Average ending inventory: mean units remaining after daily demand.
- Average starting inventory: mean units available before daily demand.

Fewer unmet units and fewer stockout days are related but not identical. A policy may experience more small shortages while avoiding a smaller number of large shortages.

## Final-test discipline

All policies, buffers, assumptions, and metrics are fixed before `d_1914` through `d_1941` are loaded. The final outcomes are not used to retune the models or buffers.

## Paired inference

The policies face identical series and demand paths. A 5,000-repetition cluster bootstrap samples complete series and calculates probabilistic-minus-fixed differences. Exact sign-flip tests are also reported for per-series unmet units, stockout-day rate, and ending inventory.

The null hypothesis is that the paired policy difference is centered at zero across the selected series. These tests do not support population-wide claims because the 12 series were purposively selected.
