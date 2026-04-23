# Energy Price Forecast — ISO New England

A multi-model electricity price forecasting notebook for large industrial energy consumers in ISO-NE (New England). Combines Prophet trend decomposition, XGBoost with SHAP explainability, and ISO-NE Forward Capacity Market (FCM) analysis to produce actionable demand-response and shutdown-optimization decisions.

---

## What This Notebook Does

The notebook ingests live macroeconomic, weather, and energy-market data to:

1. **Forecast daily electricity prices** 12–18 months forward using XGBoost trained on 15+ features
2. **Explain** every forecast day with SHAP values — identifying the top drivers pushing prices above or below your profit threshold
3. **Model your true all-in electricity cost** including spot LMP, ISO-NE capacity charges (FCM), and transmission adders
4. **Identify capacity harvest days** — summer days where curtailment is profitable not because spot prices are high, but because the day could be the one annual system peak that sets your capacity tag for the entire coming year
5. **Optimize demand-response event scheduling** across three DR tiers (Emergency, Standard, Capacity Reserve)
6. **Schedule planned shutdowns** within maintenance constraints to maximise net energy savings
7. **Produce an executive summary** with break-even analysis, SHAP driver explanations, and a year-ahead curtailment calendar

---

## Key Concepts

### ISO-NE Capacity Market (FCM)
ISO-NE's Forward Capacity Market sets an annual capacity obligation for each load customer based on their consumption during the **single highest-demand hour of the year** (the "system peak," almost always a hot July or August weekday afternoon). The FCA #17 clearing price for the 2026/2027 commitment period is **$3.58/kW-month**.

For an 85 MW facility at 80% load factor this amounts to roughly **$3.65M/year** in capacity charges — dwarfing the impact of spot-price spikes. The notebook models this as:

- **`capacity_adder_mwh`** — amortised $/MWh overhead (~$6.13/MWh)
- **`all_in_effective_mwh`** — true total cost including spot + capacity + T&D
- **`peak_tag_prob`** — probability each forecast day contains the annual system peak (July weekdays dominate at ~2%/day)
- **`capacity_option_value_mwh`** — expected value of eliminating the annual capacity bill by curtailing on a given day (~$104/MWh on peak-risk July days)
- **`capacity_harvest_day`** — flag for days where spot price is below the profit threshold but total curtailment value exceeds it

### Profit Threshold
`PROFIT_THRESHOLD_MWH = 200` $/MWh is the energy-only price at which DR/shutdown operations become worth executing on spot price alone. After adding the $14.13/MWh capacity + T&D overhead, the **true equivalent spot break-even is $185.87/MWh** — meaning summer days at $190/MWh are genuinely unprofitable to run through.

---

## Data Sources

| Source | Series | Usage |
|--------|--------|-------|
| EIA API v2 | CT industrial electricity retail price | Target variable |
| NOAA GHCN | Boston Logan (USW00014739) daily TMAX/TMIN | Weather features |
| Prophet | Trend + seasonality decomposition | Baseline forecast |
| BLS API | Job openings, CPI | Macro features |
| Census API | MA population (FIPS 25) | Demand base |
| Synthetic | Oil, gas, coal price series | Commodity features |

---

## Configuration (`config` cell)

| Parameter | Default | Description |
|-----------|---------|-------------|
| `REGION` | ISO New England | Grid region |
| `FACILITY_MW` | 85 | Facility peak demand |
| `PROFIT_THRESHOLD_MWH` | 200 | $/MWh energy-only curtailment threshold |
| `CAPACITY_PRICE_KW_MONTH` | 3.58 | ISO-NE FCA #17 clearing price ($/kW-month) |
| `CAPACITY_TAG_FRACTION` | 1.0 | Fraction of FACILITY_MW on during system peak |
| `ANNUAL_CAPACITY_FACTOR` | 0.80 | Average load factor for MWh normalisation |
| `TRANSMISSION_ADDER_MWH` | 8.0 | T&D adder ($/MWh) |
| `SHUTDOWN_COST_PER_HOUR` | 12,500 | $/hr fixed shutdown cost |
| `DR_EVENTS_ENROLLED` | — | Number of enrolled DR events per year |
| `DR_HOURS_PER_EVENT` | — | Hours per DR event |
| `INCENTIVE_RATE` | 45 | $/MWh DR incentive payment |

---

## Notebook Structure

| Cell | Description |
|------|-------------|
| 1 | Imports & dependency check |
| 2 | **Config** — all tunable parameters |
| 3–5 | Live data ingestion (EIA, NOAA, BLS, Census) |
| 6–10 | Feature engineering & exploratory analysis |
| 11–14 | Prophet baseline + XGBoost model training |
| 15–17 | Model evaluation (MAE, RMSE, MAPE) + SHAP importance |
| 18–20 | 18-month forward forecast |
| 21–22 | Price spike & seasonality analysis |
| 23 | Profit threshold exceedance + SHAP driver explanation |
| **24** | **ISO-NE Capacity Pricing & Peak Tag Analysis** ← new |
| 25 | Demand-response opportunity modelling |
| 26 | Break-even & shutdown optimiser |
| 27 | Reserve scheduling |
| 28 | Tactical operations dashboard (matplotlib) |
| 29–30 | Strategic & tactical Plotly interactive charts |
| 31 | **Executive Summary** |

---

## Setup

### Requirements
```
python >= 3.10
xgboost >= 2.1
shap >= 0.49
prophet
pandas
numpy
matplotlib
plotly
scikit-learn
statsmodels
requests
python-dotenv
holidays
```

Install dependencies:
```bash
pip install xgboost shap prophet pandas numpy matplotlib plotly scikit-learn statsmodels requests python-dotenv holidays
```

### API Keys
Create a `.env` file in the project root (this file is `.gitignore`d and must never be committed):

```
EIA_API_KEY=your_eia_api_key_here
```

Free API keys:
- **EIA**: https://www.eia.gov/opendata/register.php
- **NOAA**: https://www.ncdc.noaa.gov/cdo-web/token (optional — notebook falls back to synthetic weather)
- **BLS**: https://www.bls.gov/developers/ (optional — public tier works without a key)
- **Census**: https://api.census.gov/data/key_signup.html (optional)

### Running
Open `energy_price_forecast.ipynb` in VS Code or Jupyter and run all cells in order. Total runtime is typically 2–4 minutes depending on API response times.

---

## Output Summary (example — April 2026 run)

- **59 days** forecast above $200/MWh spot (all January–February 2027)
- **364 days** above $200/MWh on all-in cost (capacity + T&D overhead = +$14.13/MWh)
- **106 capacity harvest days** in summer 2026 where curtailment is optimal despite sub-$200 spot prices
- **Max capacity option value**: $104/MWh on peak-risk July weekdays
- **Top SHAP driver** for winter price spikes: sustained 7-day price momentum (+$19.94/MWh)
- **Annual FCM capacity bill**: $3,651,600 for 85 MW at $3.58/kW-month

---

## Repository

```
C:\EnergyForecast\
├── energy_price_forecast.ipynb   # Main notebook
├── .env                          # API keys (gitignored)
├── .gitignore
└── README.md
```

Remote: https://github.com/iedmonds20/Energy-Price-Forecast
