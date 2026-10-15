# Data preparation and series selection summary

## Data boundaries

| Period | Days | Dates | Rows for 12 series |
|---|---:|---|---:|
| Training | `d_1` to `d_1885` | 2011-01-29 to 2016-03-27 | 22,620 |
| Validation | `d_1886` to `d_1913` | 2016-03-28 to 2016-04-24 | 336 |
| Final test | `d_1914` to `d_1941` | 2016-04-25 to 2016-05-22 | Withheld |

Notebook 02 reads sales only through `d_1913`. Its processed development dataset contains no final-test sales.

## Selected series

The deterministic rules selected 12 series covering all 10 stores, all three states, all three product categories, and four demand profiles.

| Profile | Store | Category | Item | Mean daily sales | Zero-sales share | CV | Lag-7 correlation |
|---|---|---|---|---:|---:|---:|---:|
| Stable | CA_1 | FOODS | FOODS_3_586 | 45.950 | 0.004 | 0.358 | 0.591 |
| Stable | TX_1 | HOUSEHOLD | HOUSEHOLD_1_373 | 6.858 | 0.035 | 0.578 | 0.395 |
| Stable | WI_1 | HOBBIES | HOBBIES_1_345 | 3.608 | 0.136 | 0.739 | 0.246 |
| Weekly pattern | CA_2 | HOUSEHOLD | HOUSEHOLD_1_191 | 7.016 | 0.299 | 1.046 | 0.663 |
| Weekly pattern | TX_2 | HOBBIES | HOBBIES_1_323 | 4.334 | 0.077 | 0.665 | 0.438 |
| Weekly pattern | WI_2 | FOODS | FOODS_1_096 | 14.941 | 0.191 | 1.129 | 0.784 |
| Intermittent | CA_3 | HOBBIES | HOBBIES_2_110 | 0.150 | 0.900 | 3.474 | 0.028 |
| Intermittent | TX_3 | FOODS | FOODS_2_254 | 0.134 | 0.900 | 3.343 | 0.149 |
| Intermittent | WI_3 | HOUSEHOLD | HOUSEHOLD_1_161 | 0.118 | 0.900 | 3.214 | 0.155 |
| Variable | CA_4 | FOODS | FOODS_2_255 | 0.919 | 0.592 | 2.181 | 0.179 |
| Variable | TX_1 | HOBBIES | HOBBIES_1_209 | 2.173 | 0.405 | 2.665 | 0.015 |
| Variable | WI_1 | HOUSEHOLD | HOUSEHOLD_2_142 | 0.872 | 0.585 | 2.655 | 0.071 |

## Validation checks

- The long development dataset has 22,956 rows: 12 series multiplied by 1,913 days.
- Calendar and price joins preserve the row count.
- There are no duplicate series-date rows or missing dates.
- There are 2,009 rows without a selling price, and none has positive recorded sales.
- Stable and weekly-pattern series have complete training-period prices. Most missing prices occur in the intermittent group.

Missing prices remain missing because the available data do not establish a common reason for every gap.

## Limitations

- The 12 series are a purposive sample, not a statistically representative sample of all M5 series.
- Recorded sales may understate customer demand when an item is unavailable.
- The profile labels summarize training-period behavior and may change over time.
