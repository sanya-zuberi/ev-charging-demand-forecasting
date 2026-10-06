# EV Charging Demand Forecasting

Forecasting EV charging demand, both session volume and energy consumption, to support staffing, capacity planning, and energy procurement decisions.

## The Problem

EV charging stations need to plan staffing, physical capacity, and energy procurement ahead of time, but there was no forecast to guide those decisions, there were only historical session logs. This project builds a cleaned demand time series, compares three forecasting approaches, and produces 30/90/365-day forecasts with an interactive dashboard.

## Repo Structure

```
ev-charging-demand-forecasting/
├── processed data/      → cleaned hourly and daily demand series (hourly_demand.csv, daily_demand.csv)
├── notebooks/            → full pipeline, run in order (see "Reproducing" below)
├── output/               → forecast_all, insight_summary (PDF)
├── dashboard/            → Power BI .pbix file + page screenshots
├── slides/               → presentation slides (PPTX)
└── documentation/        → full project documentation (EV_Charging_Demand_Forecasting_Documentation.pdf)
```

## Data

The source dataset contains 294,024 individual EV charging sessions across 34 stations and 199 chargers (`sessions`, `stations`, `chargers` tables), spanning January 2022–December 2024.

The raw Excel file is not included in this repo. It is hosted as a Kaggle Dataset: **https://www.kaggle.com/datasets/sanyazuberi/ev-charging-raw-dataset**.

`processed data/` contains the cleaned, aggregated outputs of the cleaning notebook:
- `hourly_demand.csv`: session count and total energy (kWh), aggregated to hourly resolution
- `daily_demand.csv`: session count and total energy (kWh), aggregated to daily resolution

## Key Findings

- **Charging demand follows a clear daily rhythm**, near-zero overnight, peaking 10am–2pm, consistent with daytime/workplace charging rather than overnight home charging.
- **Weekday demand runs ~27% higher than weekends**, confirming a commuter-driven usage pattern.
- **Most of the historical growth in total demand reflects network expansion, not rising demand per station.** Station count grew from 8 (2022) to 33 (2024), and total session volume jumped sharply at almost exactly the same points.
- **A systematic data quality issue was identified in the source data:** 30% of all sessions occur before their charger's recorded installation date, affecting every charger in the dataset and correlating strongly (r ≈ 0.78) with how late each charger came online, indicating a data generation artifact rather than scattered bad records. Documented in full in `documentation/`, excluded from modelling since it isn't needed for this forecasting task.
- **The two models used for forecasting disagree on future growth**, and this is treated as a genuine, unresolved uncertainty rather than hidden behind one number, see "Models & Results" below.

## Models & Results

Three forecasting approaches were built and compared under identical, fair conditions, each forecasting a 90-day holdout period with no access to real values from within it.

| Target | Model | RMSE | MAPE |
|---|---|---|---|
| session_count | **XGBoost (calendar-only)** | **4.98** | 0.350 |
| session_count | Prophet | 5.31 | 0.413 |
| session_count | SARIMA | 5.46 | **0.342** |
| total_energy_kwh | **XGBoost (calendar-only)** | **233.9** | **0.370** |
| total_energy_kwh | Prophet | 262.1 | 0.449 |
| total_energy_kwh | SARIMA | 264.7 | 0.379 |

**XGBoost (calendar-only)**: using only hour, day of week, month, and year as features, had the lowest error on nearly every metric and was selected as the primary forecasting model. Prophet was kept alongside it in the final forecast to provide uncertainty intervals and because it makes a different, genuinely useful assumption about future growth (see below).

### Forecast: two models, two assumptions

Checked against the most recent full year of real data (2024 average: 19.04 sessions/hour, 882.03 kWh/hour):

| Model | 2025 Forecast Avg. | vs. 2024 |
|---|---|---|
| XGBoost | 19.03 sessions/hr, 881.79 kWh/hr | ≈ flat |
| Prophet | 20.28 sessions/hr, 930.48 kWh/hr | ≈ +6% |

XGBoost cannot extrapolate growth beyond the years it was trained on, so it holds flat at the 2024 level. Prophet extends the historical growth trend. Neither is proven correct, both are shown on the dashboard so planners can weigh either scenario rather than be given false certainty.

Full forecast output (30/90/365-day horizons, both models, both targets): `output/forecast_all`.

## Dashboard

Built in Power BI, two pages (Session Count, Energy), each with:
- Historical vs. forecast (both models overlaid, with a 30/90/365-day horizon slicer)
- Average demand by day of week
- Average demand by hour of day

See `dashboard/` for the `.pbix` file and page screenshots.

## Reproducing This Project

1. Download the raw dataset from the Kaggle link above.
2. Run the notebooks in `notebooks/` in this order:
   - Cleaning and EDA
   - Feature engineering
   - Prophet, SARIMA and XGBoost model notebooks, these three can run in any order relative to each other, but all require feature engineering to have run first
   - Model comparison: requires all three model notebooks to have completed
   - Forecast generation: requires feature engineering and the chosen model(s)' notebooks

## Tools Used

Python (pandas, Prophet, statsmodels, XGBoost, scikit-learn) · Kaggle Notebooks · Power BI · Microsoft Teams

## Limitations

- The installation-date inconsistency described above affects the `chargers` table broadly. Any future project relying on that specific field should investigate further before trusting it.
- No meaningful month-to-month seasonality was found in the data, which is unusual for real-world EV charging demand, likely a characteristic of this being a synthetic/simulated dataset.
- The two forecasting models make different, unverified assumptions about 2025 demand growth (see "Forecast" above), the dashboard presents both rather than a single resolved number.
- SARIMA was the slowest and least stable model to train on this dataset and did not outperform the alternatives. It was excluded from the final forecast stage for that reason.

