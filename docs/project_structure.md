# Project structure and folder responsibilities

This document explains how the repository is organized. It should be updated when a new major folder or type of project artifact is introduced.

| Path | Purpose | Expected contents | Requirements and rules |
|---|---|---|---|
| `data/raw/` | Stores the original M5 files used as project inputs. | Original CSV files downloaded from the dataset source and a local README. | Do not edit, rename, or commit the CSV files. Analysis code should treat them as read-only inputs. |
| `data/processed/` | Stores reproducible datasets created from the raw files. | Long-form sales data, selected-series data, or other generated analysis tables. | Every file must be reproducible from tracked code. Generated data are not committed unless the team explicitly documents a small exception. |
| `notebooks/` | Contains the student-facing analysis in execution order. | Separate notebooks for data audit, data preparation and series selection, time-series EDA, forecasting, uncertainty and safety stock, and inventory simulation. | Each notebook must explain why a step is needed, display supporting output, interpret results, and state limitations. Use project-relative paths and run notebooks from top to bottom. |
| `scripts/` | Contains commands used to set up or maintain the local project. | The Kaggle download and verification helper. | Scripts must be safe to rerun, must not store credentials, and must validate external files before analysis. |
| `src/` | Contains reusable Python code shared by notebooks. | Data preparation, forecasting, simulation, and evaluation helpers. | Move code here only when reuse improves clarity. Keep the notebooks readable and do not hide essential statistical reasoning from the reader. |
| `reports/` | Stores written project outputs. | Draft or final reports and supporting material. | Report claims must agree with executed notebook evidence. Clearly distinguish observed M5 sales from simulated inventory outcomes. |
| `reports/figures/` | Stores figures exported for reports and presentations. | Clear charts and diagrams generated from notebooks. | Commit only figures that are used in a report. Each figure should have a clear filename, title, labels, units, and a traceable notebook source. |
| `docs/` | Stores technical project documentation. | Folder guides, data decisions, method specifications, assumptions, and evaluation definitions. | Update documentation as the implementation changes. Clearly separate completed work from planned work. |

## Root files

| File | Purpose |
|---|---|
| `README.md` | Provides the project overview, research questions, scope, limitations, and high-level workflow. |
| `setup.md` | Provides reproducible local environment and dataset setup instructions. |
| `requirements.txt` | Defines the Python packages used by the notebooks and source code. |
| `.gitignore` | Prevents raw data, generated data, local environments, and temporary files from being committed. |

## Notebook documentation standard

Each major notebook section should follow this sequence:

1. Explain why the section is needed.
2. Explain what the code or statistical check will do.
3. Display the calculation, table, or plot that provides evidence.
4. Interpret the observed result in plain English.
5. Explain what the result means for the next project decision.
6. State any relevant assumption or limitation.

This structure keeps the work understandable as an M.S.-level student project while preserving enough technical detail for another student to reproduce it.
