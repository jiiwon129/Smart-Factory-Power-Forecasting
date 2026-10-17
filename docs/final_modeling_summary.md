# Final Modeling Summary

This document summarizes the final team modeling notes for the BDA smart-factory power forecasting project.

## Problem

The project addressed heterogeneous power-consumption patterns across factory modules. Rather than using a single model for every module, the final design combined:

1. module-level pattern analysis,
2. K-means clustering of similar modules,
3. cluster-level forecasting,
4. full-process forecasting,
5. a weighted ensemble of cluster and full-process predictions.

## Preprocessing

- Dropped `operation` because it was constant at 1.
- Replaced duplicated time representation with a datetime-based time index.
- Aggregated raw 5-second measurements to 10-minute intervals to reduce short-term noise.
- Added lagged features for forecasting.

## Engineered Features

- `10min_pow`: difference in accumulated active energy
- min / max voltage and current statistics
- `std_activePow`
- time-weighted power factor
- time-weighted reactive power
- `pf_sign`
- `current_change_acc`
- `total_pf`
- `current_CV`
- `current_unbal_ratio`
- `current_Total`
- `voltage_CV`

## Clustering

K-means produced four module groups:

- base-load engine type
- work-synchronous type
- startup-load type
- environment-dependent type

## Validation Strategy

- Data range: 2024-12 through 2025-04
- Validation: final week of April
- Metrics: MAE, RMSE, SMAPE

## Model Comparison

### Baseline
- Linear Regression
- SMAPE: 1.7912%

### Full-process models
LightGBM, XGBoost, and CatBoost all achieved approximately:

- SMAPE: 0.18%
- RMSE: 1.09
- MAE: 0.88

LightGBM was selected for the full-process model based on feature-importance behavior.

### Cluster-level model
LightGBM was selected for cluster-level forecasting.

- Cluster 0 SMAPE: 0.184%
- Cluster 1 SMAPE: 0.1775%
- Cluster 2 SMAPE: 0.1814%
- Cluster 3 SMAPE: 0.179%

Overall:
- SMAPE: 0.18%
- MAE: 0.8946
- RMSE: 1.1209

### Ensemble
Final prediction combined:
- cluster model: 20%
- full-process model: 80%

## Operational Scenario

Using the May 2025 prediction:

- predicted power consumption: about 26.07 million kWh
- estimated electricity cost: about KRW 4.69 billion
- estimated emissions: about 11,038,955 kgCO₂
- suspected inefficient periods: 352 hours

Under the scenario of reducing electricity use by 5% only in the detected inefficient periods:

- estimated monthly savings: about KRW 122.9 million
- annualized savings: about KRW 1.47 billion
- estimated monthly CO₂ reduction: about 289 tons

These are scenario-based estimates, not verified operational savings. Actual deployment would require production schedule and MES/ERP data.

## LSTM Experiment

The final notes identify the LSTM experiment as Kim Jiwon's individual modeling task.

Test performance:
- MAE: 8.8554
- RMSE: 13.4537
- SMAPE: 1.8050%

The LSTM did not outperform the selected tree-based models and was retained as a comparison experiment rather than the final forecasting model.
