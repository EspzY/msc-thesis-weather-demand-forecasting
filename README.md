# Weather-Augmented Demand Forecasting

This repository contains the code pipeline for a master thesis on
weather-augmented machine-learning demand forecasting for a Norwegian café
chain. The analysis asks whether operational weather information improves
store-category-day demand forecasts and whether the resulting error reductions
are operationally meaningful. The design is prediction-oriented and
decision-support oriented; it is not a causal identification design.

The raw transaction and weather source files are private and are not included in
the repository. The notebooks are written so that a thesis examiner can inspect
the construction of the modelling panel, feature sets, leakage controls and
benchmarking procedure.

## Notebook Pipeline

The table below follows the active clean pipeline. The requested core weather
and ML sequence is `00 -> 01a -> 01b -> 01c -> 01d -> 04 -> 05 -> 06 -> 07`.
Current project documentation also lists notebook `01e` as the ERA5 climatology
producer consumed by notebook `04`; run it before `04` if the ERA5 climatology
artifacts are not already present.

| Order | Notebook | Purpose | Main inputs | Main outputs |
|---:|---|---|---|---|
| 00 | `notebooks/00_setup_and_data_dictionary.ipynb` | Defines project-root detection, path conventions, data dictionary and lightweight availability checks. | Repository marker files and expected local data paths. | No saved files. |
| 01a | `notebooks/01a_parse_met_analysis.ipynb` | Parses MET Nordic Analysis files into a realised weather proxy by store and date. | MET Nordic Analysis NetCDF folders, store-grid mapping files. | `data/processed/realised_weather_daily.parquet`; `data/processed/realised_weather_daily_windows.parquet`. |
| 01b | `notebooks/01b_parse_meps_deterministic.ipynb` | Parses archived deterministic MEPS forecasts into local-time daily/window forecast tables. | MEPS deterministic NetCDF forecast trajectories. | `data/processed/forecast_meps_daily.parquet`; `data/processed/forecast_meps_daily_windows.parquet`; `data/processed/forecast_meps_horizon_hour_audit.parquet`. |
| 01c | `notebooks/01c_parse_meps_ensemble_features.ipynb` | Parses MEPS ensemble availability and short-horizon spread diagnostics for uncertainty calibration. | MEPS ensemble NetCDF folders for temperature, precipitation, wind components, humidity and cloud. | `data/processed/forecast_meps_ensemble_coverage_audit.parquet`; conditional `data/processed/forecast_meps_ensemble_daily_windows.parquet`. |
| 01d | `notebooks/01d_parse_historical_weather_calibration.ipynb` | Builds historical local-window calibration weather and pressure-regime support for the emulator. | Historical MET Nordic Analysis calibration files, including pressure/MSLP. | `data/processed/historical_weather_calibration_windows.parquet`; `data/processed/historical_weather_calibration_file_audit.parquet`; `data/processed/historical_weather_calibration_parameters.parquet`; audits under `outputs/historical_weather_calibration/`. |
| 01e | `notebooks/01e_parse_era5_longrun_climatology.ipynb` | Builds leakage-safe ERA5 long-run climatology used by the h=3 to h=10 emulator. | ERA5 single-level hourly time series and store-point metadata. | `data/processed/era5_longrun_weather_windows.parquet`; `data/processed/era5_longrun_climatology_parameters.parquet`; `data/processed/era5_longrun_same_calendar_day_climatology.parquet`; audits under `outputs/era5_longrun_climatology/`. |
| 04 | `notebooks/04_synthetic_weather_model.ipynb` | Constructs the operational weather forecast emulator: deterministic MEPS for h=0 to h=2 and synthetic climatology-drift weather for h=3 to h=10. | Processed realised weather, MEPS deterministic forecasts, ensemble/calibration outputs, historical calibration and ERA5 climatology artifacts. | `data/processed/weather_forecast_operational_windows.parquet`; `data/processed/weather_forecast_error_diagnostics_windows.parquet`; audits under `outputs/synthetic_weather_model/`. |
| 05 | `notebooks/05_weather_ensemble_features.ipynb` | Selects the main operational weather window and exports ML-ready weather features. | `data/processed/weather_forecast_operational_windows.parquet`. | `data/processed/weather_forecast_operational_ml_features.parquet`; feature metadata and audits under `outputs/weather_ensemble_features/`. |
| 06 | `notebooks/06_build_ml_panel.ipynb` | Assembles the forecast-origin-safe ML panel and feature registry for M1-M4. | Sales master panel, notebook 05 weather features and realised-weather diagnostics. | `data/processed/ml_forecast_panel_full.parquet`; `outputs/ml_panel/ml_panel_feature_registry.csv`; audits under `outputs/ml_panel/`. |
| 07 | `notebooks/07_ml_benchmark_models.ipynb` | Screens model families and feature sets under a chronological forecast-origin-aware split. | ML panel and feature registry from notebook 06. | `outputs/ml_models/benchmark_metrics_{run_mode}.csv`; `outputs/ml_models/benchmark_predictions_{run_mode}.parquet`; model, feature-set and gain summaries under `outputs/ml_models/`. |

Sales-panel construction is documented separately in
`notebooks/01_data_preparation_master_panel.ipynb`. It produces the master
sales/weather panels consumed before notebook `06`; include it in a full
end-to-end rerun when processed sales panels are absent.

## Model Specifications

- **M1: no-weather baseline.** Calendar, store/category identifiers, opening
  status, and origin-safe historical sales/campaign controls; no weather
  variables.
- **M2: operational point-weather model.** M1 plus forecast weather known at the
  forecast origin: temperature, precipitation, wind, humidity and cloud point
  forecast features.
- **M3: realised-weather benchmark.** M1 plus realised target-day weather. This
  is a perfect-information benchmark and is not operationally available.
- **M4: uncertainty-aware weather model.** M2 plus calibrated uncertainty,
  probability and interval features from the weather-emulator pipeline.

## Weather Data Sources

- **MET Nordic Analysis.** Used to construct realised weather proxies,
  historical calibration inputs and diagnostics; realised target-day values feed
  M3 and post-estimation diagnostics, not operational M1/M2/M4 features.
- **MEPS deterministic forecasts.** Archived deterministic forecasts provide the
  operational h=0, h=1 and h=2 weather inputs for M2 and M4.
- **MEPS ensemble forecasts.** Short-horizon ensemble availability and spread
  diagnostics support uncertainty calibration/audits; ensemble spread is not a
  direct demand feature in M1-M3.
- **ERA5 climatology.** Long-run climatology supports the synthetic h=3 to h=10
  climatology-drift emulator; ERA5 is calibration/climatology support and is not
  exposed directly as an ML demand feature.

## Leakage Controls

Operational model features are restricted to information available at the
forecast origin. Notebook `06` writes a feature registry that separates key
columns, targets, operational controls, forecast-weather features and
leakage-risk diagnostics.

Realised target-day weather is retained for two purposes only: the M3
perfect-information benchmark and diagnostic grouping/validation after
predictions are fixed. M1, M2 and M4 must not use realised target-day weather,
pressure/MSLP, ERA5 values or forecast-error diagnostics as direct demand
features.

Forecast evaluation is chronological and forecast-origin-aware. Notebook `07`
uses a chronological holdout; later LightGBM notebooks use expanding-window
rolling-origin evaluation. Random cross-validation is not used for the main
forecasting evidence because it would mix time periods and risk look-ahead
leakage.

## Reproduction

### Environment

Use the project conda environment:

```powershell
conda activate csvi_env
```

The documented environment uses Python with `pandas`, `numpy`, `pyarrow`,
`matplotlib`, `scikit-learn`, and tree-based ML libraries including
`lightgbm`, `xgboost` and `catboost`. A pinned environment file is not currently
documented in the inspected project files. TODO: add or verify
`environment.yml` before external replication.

### Execution Order

Run notebooks from the project root, in order:

1. `00_setup_and_data_dictionary.ipynb`
2. `01a_parse_met_analysis.ipynb`
3. `01b_parse_meps_deterministic.ipynb`
4. `01c_parse_meps_ensemble_features.ipynb`
5. `01d_parse_historical_weather_calibration.ipynb`
6. `01e_parse_era5_longrun_climatology.ipynb` if ERA5 climatology artifacts are
   absent or must be regenerated.
7. `04_synthetic_weather_model.ipynb`
8. `05_weather_ensemble_features.ipynb`
9. `06_build_ml_panel.ipynb`
10. `07_ml_benchmark_models.ipynb`

For a complete end-to-end run from raw sales transactions, also run
`01_data_preparation_master_panel.ipynb` before notebook `06`. TODO: verify the
exact placement if the processed sales master panels are not already present.

### Output Locations

- Processed data and modelling panels are written under `data/processed/`.
- Weather-emulator audits are written under `outputs/synthetic_weather_model/`.
- Weather-feature audits are written under `outputs/weather_ensemble_features/`.
- ML panel audits and the feature registry are written under `outputs/ml_panel/`.
- Model-screening metrics, predictions and summaries are written under
  `outputs/ml_models/`.

Private raw data, processed panels and generated model outputs are not included
in the repository unless explicitly provided by the thesis project owner.
