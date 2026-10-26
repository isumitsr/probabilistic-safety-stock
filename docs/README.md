# Technical documentation

This directory contains the technical documentation for the project. Documentation will be added incrementally as each project section is designed and completed.

## Current documentation

- [`project_structure.md`](project_structure.md): explains the purpose, expected contents, inputs, outputs, and rules for each repository folder.
- [`implementation_walkthrough.md`](implementation_walkthrough.md): explains the current project from objective through interpretation, including why each stage is needed and what its results mean.
- [`technical_implementation_reference.md`](technical_implementation_reference.md): documents the implemented data flow, formulas, algorithms, schemas, checks, and execution contract.
- [`series_selection_and_splits.md`](series_selection_and_splits.md): defines the protected periods, deterministic selection rules, and processed outputs used by later notebooks.
- [`forecasting_method.md`](forecasting_method.md): records the point-forecast baselines, validation design, metrics, and selection rule.
- [`uncertainty_and_safety_stock.md`](uncertainty_and_safety_stock.md): documents empirical uncertainty calibration and the safety-stock planning assumptions.
- [`statistical_inference_method.md`](statistical_inference_method.md): explains the distribution checks, cluster bootstrap, confidence intervals, and 14-day protection-period correction.
- [`inventory_simulation_method.md`](inventory_simulation_method.md): records the periodic-review policies, matched-buffer comparison, final-test simulation, and paired inference.

All implemented stages now have focused documentation. Any later sensitivity analysis should be documented separately rather than changing the final-test design after seeing its result.
