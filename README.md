# From Forecasts to Stock Decisions

### Project Title: Probabilistic safety-stock design across Walmart stores using the M5 dataset.

This project is part of the AAI 500 Probability and Statistics for AI course in the Master of Science (M.S) in Applied Artificial Intelligence program at the University of San Diego.

**Project status:** Analysis complete. The project includes data audit and preparation, time-series EDA, statistical forecasting, empirical uncertainty, bootstrap inference, distribution diagnostics, periodic-review inventory simulation, and one final holdout evaluation.

## Project objective

Retailers must decide how much inventory to hold when future demand is uncertain. Too little inventory can lead to stockouts and missed sales, while too much inventory ties up cash and increases holding risk. A fixed buffer does not respond to the different demand patterns of individual products and stores.

This project investigates whether probabilistic demand forecasts can support better safety-stock decisions than a simple forecasting method combined with a fixed inventory buffer. It compares forecast accuracy, forecast uncertainty, simulated stockouts, service levels, and inventory held under clearly stated replenishment assumptions.

Our central research question is:

> Under stated replenishment assumptions, can probabilistic forecasts reduce simulated stockouts or inventory held compared with a simple forecasting and fixed-buffer approach?

## Installation

### 1. Clone the repository

```bash
git clone <repository-url>
cd <repository-folder>
```

### 2. Create the Python environment

```bash
python3.11 -m venv .venv
source .venv/bin/activate       # macOS/Linux
# .venv\Scripts\Activate.ps1   # Windows PowerShell

python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

### 3. Authenticate with Kaggle

The M5 files are subject to Kaggle's competition rules. Before the first download:

1. Sign in to [Kaggle](https://www.kaggle.com/).
2. Open the [M5 Forecasting - Accuracy data page](https://www.kaggle.com/competitions/m5-forecasting-accuracy/data) and accept the competition rules.
3. Generate an API token from [Kaggle API settings](https://www.kaggle.com/settings/api).
4. Authenticate on your computer:

```bash
python -c "import kagglehub; kagglehub.login()"
```

Never commit a Kaggle token or credential file to this repository.

### 4. Download and verify the data

Run the project download helper from the repository root:

```bash
python scripts/download_data.py
```

The script downloads the five official competition files into `data/raw/`, verifies their checksums, and skips files that are already valid. The raw CSV files are excluded from Git because of their size and Kaggle's competition terms.

Complete this step before running the analysis notebooks.

### 5. Start the project notebooks

Start JupyterLab from the repository root:

```bash
jupyter lab
```

Open the notebooks in numeric order, beginning with:

```text
notebooks/01_data_audit_and_eda.ipynb
```

The first notebook also checks the local data and calls the download helper automatically if a required file is missing or invalid. Once the data audit passes, run the remaining numbered notebooks in order.

See [`setup.md`](setup.md) for the complete environment, authentication, data-download, and verification instructions.

For a detailed explanation of the work completed so far, read the [implementation walkthrough](docs/implementation_walkthrough.md). For formulas, algorithms, data schemas, and validation rules, use the [technical implementation reference](docs/technical_implementation_reference.md).

## Team members and contributors

- Sumit Chaurasia
- Lala Ram
- AAI 500 Group 1

## Methods used

- Exploratory data analysis of demand, seasonality, intermittency, and variability
- Fixed-origin time-series forecast validation
- Empirical forecast-error distributions and quantile forecasts
- Cluster bootstrap confidence intervals and paired inference
- Normal, Poisson, negative-binomial, and empirical distribution diagnostics
- Safety-stock planning under explicit lead-time assumptions
- Periodic-review inventory-policy simulation and final holdout evaluation

## Technologies

- Python
- pandas and NumPy for data preparation
- Matplotlib/ Seaborn for visualization
- Jupyter Notebook/ VSCode Notebooks
- Git and GitHub for version control and collaboration
- Trello for Project Management

## Project description

### Dataset

The project uses the public M5 Walmart sales dataset. The competition data contain daily unit sales for 3,049 products sold across 10 stores in California, Texas, and Wisconsin. This produces 30,490 product-store time series. The data also include calendar events and weekly selling prices.

| File | Purpose | Main contents |
|---|---|---|
| `sales_train_validation.csv` | Historical demand | Product and store identifiers plus daily sales from `d_1` through `d_1913` |
| `sales_train_evaluation.csv` | Extended evaluation data | Historical sales through `d_1941`, including the competition evaluation period |
| `calendar.csv` | Date and event information | Dates, weekdays, months, years, events, and SNAP indicators |
| `sell_prices.csv` | Price information | Store, item, selling week, and selling price |

The raw sales file is in wide format. Notebook 02 reshapes the selected daily columns into a long time-series table and merges calendar and price information using the appropriate date and store-item keys.

### Series selection

The full M5 dataset is large, so the analysis uses a documented sample of 12 product-store series. The sample covers different stores, categories, and demand profiles, including stable, weekly-pattern, intermittent, and variable series. The reproducible selection rule and final series list are recorded in notebook 02 and the supporting documentation.

### Analysis workflow

1. Audit the raw files, data types, missing values, duplicate keys, and date coverage. Completed in notebook 01.
2. Protect the final 28-day test period and create chronological training and validation periods. Completed in notebook 02.
3. Select a reproducible group of product-store series covering different demand patterns. Completed in notebook 02.
4. Convert the selected sales to long format and join calendar and price data. Completed in notebook 02.
5. Study sparsity, variability, recent history, weekday patterns, autocorrelation, and seven-day demand. Completed in notebook 03.
6. Build and compare point-forecast baselines on the validation period. Completed in notebook 04.
7. Estimate predictive uncertainty and translate it into safety-stock planning levels. Completed in notebook 05.
8. Add distribution diagnostics, cluster-bootstrap inference, and a 14-day protection-period calibration. Completed in notebook 06.
9. Simulate the fixed-buffer and probabilistic policies and evaluate them once on the protected final period. Completed in notebook 07.

### Inventory-policy comparison

The probabilistic policy uses an upper forecast quantile for demand over the 14-day protection period. In simplified form:

```text
Order-up-to level = protection-period forecast + uncertainty buffer
Safety stock = order-up-to level - expected protection-period demand
```

The comparison policy uses the same point forecast and a fixed additive buffer matched to the probabilistic policy's average buffer. Both policies use a 7-day review interval, 7-day lead time, lost sales, and identical demand paths.

### Research questions and hypotheses

**Research question 1:** Do probabilistic forecasts produce better simulated service outcomes than a fixed-buffer policy?

**Hypothesis 1:** At comparable average inventory levels, a probabilistic policy will produce a lower simulated stockout rate or higher simulated service level than the fixed-buffer policy.

**Research question 2:** How does the target service level affect inventory held and stockouts?

**Hypothesis 2:** Increasing the target service level will reduce simulated stockouts but require more inventory.

**Research question 3:** Does better average forecast accuracy always produce better inventory decisions?

**Hypothesis 3:** The model with the best point-forecast metric will not necessarily produce the best balance between service and inventory because the quality of the uncertainty estimate also matters.

### Evaluation metrics

Forecast performance is evaluated with MAE, RMSE, WAPE, and quantile or pinball loss where appropriate. Predictive uncertainty is assessed using interval coverage and interval width. Inventory policies are compared using simulated stockout-day rate, fill rate, unmet units, average ending inventory, and the tradeoff between service and inventory.

### Main conclusion

At the 90% and 95% targets, the probabilistic policy produced higher final-period fill rate, fewer unmet units, and slightly lower average ending inventory than the matched fixed-buffer policy. However, it produced slightly more stockout days, and every 95% cluster-bootstrap interval for the policy differences included zero. The evidence is suggestive rather than conclusive and differs by demand profile. See the [final project summary](reports/final_project_summary.md).

### Scope and limitations

The M5 data record observed sales, not complete inventory positions or unmet customer demand. The dataset also does not provide the supplier lead times, replenishment rules, or inventory costs required to reproduce Walmart's actual decisions. Therefore, lead time, starting inventory, and ordering rules are explicit modeling assumptions. Stockouts, service levels, and inventory outcomes reported in this project are simulation results, not measured Walmart outcomes.

The results apply only to the selected product-store series and the assumptions used in the simulation. They should not be interpreted as a recommendation for Walmart's actual inventory system without additional operational data.

## Repository structure

```text
.
├── README.md
├── setup.md
├── requirements.txt
├── docs/
│   ├── README.md
│   ├── project_structure.md
│   ├── implementation_walkthrough.md
│   ├── technical_implementation_reference.md
│   ├── series_selection_and_splits.md
│   ├── forecasting_method.md
│   ├── uncertainty_and_safety_stock.md
│   ├── statistical_inference_method.md
│   └── inventory_simulation_method.md
├── data/
│   ├── raw/              # Downloaded M5 files, not committed to GitHub
│   └── processed/        # Generated analysis-ready data, if needed
├── scripts/
│   └── download_data.py  # Kaggle download and checksum verification
├── notebooks/
│   ├── 01_data_audit_and_eda.ipynb
│   ├── 02_data_preparation_and_series_selection.ipynb
│   ├── 03_time_series_eda.ipynb
│   ├── 04_forecasting.ipynb
│   ├── 05_uncertainty_and_safety_stock.ipynb
│   ├── 06_statistical_inference_and_distribution_analysis.ipynb
│   └── 07_inventory_policy_simulation_and_final_evaluation.ipynb
├── src/
│   └── __init__.py         # Reserved for genuinely reusable project code
├── reports/
│   ├── initial_data_audit_summary.md
│   ├── data_preparation_and_selection_summary.md
│   ├── time_series_eda_summary.md
│   ├── forecasting_summary.md
│   ├── uncertainty_and_safety_stock_summary.md
│   ├── statistical_inference_summary.md
│   ├── final_project_summary.md
│   └── figures/
```

## Data use

The M5 files remain subject to the Kaggle competition rules and are not redistributed through this repository. Users should download the data directly from Kaggle and follow its terms of use.

## Acknowledgments

We thank the University of San Diego faculty and Prof. Haisav Chokshi for guidance in the AAI 500 Probability and Statistics for AI course. We also acknowledge the M5 competition organizers and the authors of the M5 competition paper for making the retail forecasting data and documentation available.

### Reference

Makridakis, S., Spiliotis, E., & Assimakopoulos, V. (2022). The M5 competition: Background, organization, and implementation. *International Journal of Forecasting, 38*(4), 1325–1336. https://doi.org/10.1016/j.ijforecast.2021.07.007
