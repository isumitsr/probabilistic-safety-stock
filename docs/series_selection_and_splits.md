# Series selection and chronological splits

## Purpose

This document records the data boundaries and series-selection rules used after the raw-data audit. Later forecasting notebooks should use these decisions without selecting a different sample after seeing validation or final-test results.

## Chronological periods

| Period | M5 days | Dates | Use |
|---|---|---|---|
| Training | `d_1` to `d_1885` | 2011-01-29 to 2016-03-27 | Series selection, EDA, and model fitting |
| Validation | `d_1886` to `d_1913` | 2016-03-28 to 2016-04-24 | Forecast comparison and model choice |
| Final test | `d_1914` to `d_1941` | 2016-04-25 to 2016-05-22 | One final evaluation after methods are fixed |

Notebook 02 reads sales only through `d_1913`. Its processed development dataset therefore contains no final-test sales.

## Selection coverage

The project uses 12 product-store series. The fixed selection plan covers all 10 stores, all three states, and all three product categories. It includes three series in each of four profiles:

- stable;
- weekly pattern;
- intermittent; and
- variable.

The store-category slots are fixed before ranking products. This prevents the sample from drifting toward one large store or one high-volume category.

## Training-only measures

The ranking measures use `d_1` through `d_1885`:

- mean daily recorded sales;
- share of days with zero recorded sales;
- coefficient of variation;
- number of nonzero days in the latest 365 training days; and
- lag-7 autocorrelation.

Every ranking uses `item_id` as the final sort field. This makes ties deterministic across environments.

## Profile rules

| Profile | Eligibility | Ranking within the fixed store-category slot |
|---|---|---|
| Stable | Mean at least 1 unit, zero share at most 25%, at least 180 recent nonzero days | Lowest coefficient of variation |
| Weekly pattern | Mean at least 0.5 unit, zero share at most 40%, at least 100 recent nonzero days | Highest lag-7 autocorrelation |
| Intermittent | Zero share from 65% to 90%, 20 to 150 recent nonzero days | Highest zero-sales share |
| Variable | Mean at least 0.5 unit, zero share at most 60%, at least 60 recent nonzero days | Highest coefficient of variation |

These are project sampling labels. They do not claim that a product always follows one demand process. Lag-7 correlation is evidence of repeated weekly timing, not proof of its cause.

## Generated outputs

Notebook 02 creates two reproducible local files:

- `data/processed/selected_series_metadata.csv` contains the 12 selected series and their training-only measures.
- `data/processed/selected_series_development.csv` contains joined training and validation observations through `d_1913`.

Generated datasets remain excluded from Git. Running notebook 02 rebuilds them from the checksum-verified raw files.
