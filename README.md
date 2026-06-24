# Demand Forecasting for Dark Stores

A Prophet-based time-series forecasting system that predicts grocery order volumes 24 hours ahead across 8 delivery zones, enriched with weather and holiday signals. Forecast outputs are evaluated through inventory simulations, showing a meaningful reduction in stockout scenarios compared to naive baseline methods.

---

## Project Overview

Dark stores — micro-fulfillment centers that serve only online delivery orders — depend heavily on accurate short-term demand forecasts to stock the right quantities before each operational day. Over- or under-ordering directly translates to waste or missed orders.

This project builds a per-zone forecasting pipeline using Facebook Prophet, extends it with external signals (weather, public holidays), and evaluates the practical impact through an inventory simulation.

---

## Dataset

**Source:** UCI Machine Learning Repository — Online Retail Dataset  
**Records:** 541,909 transactions (Dec 2010 to Dec 2011)  
**After cleaning:** 397,924 records  
**Granularity:** Per-invoice line items with date, product, quantity, and country

The dataset is aggregated to daily order volumes per delivery zone before modeling.

---

## Delivery Zones

Eight zones are defined by mapping the top countries by order volume:

| Zone | Country |
|------|---------|
| Zone 1 | United Kingdom |
| Zone 2 | Germany |
| Zone 3 | France |
| Zone 4 | EIRE (Ireland) |
| Zone 5 | Spain |
| Zone 6 | Netherlands |
| Zone 7 | Belgium |
| Zone 8 | Switzerland |

---

## Repository Structure

```
dark-store-forecasting/
│
├── Online Retail.xlsx                        # Raw dataset
├── dark_store_demand_forecasting.ipynb       # Main notebook (end-to-end pipeline)
└── README.md
```

---

## Methodology

### 1. Data Cleaning

- Removed cancelled invoices (InvoiceNo starting with 'C')
- Filtered out negative quantities and zero-price entries
- Dropped rows with missing CustomerID
- Filled missing dates per zone with zero volume

### 2. External Signal Enrichment

**Holiday signals** via the `holidays` Python library:
- Binary flag for UK public holidays
- Pre-holiday flag (day before a bank holiday)
- Days-to-Christmas countdown (capped at 30)

**Weather signals** (UK climate profile):
- Daily average temperature in Celsius (monthly means with Gaussian noise)
- Daily rainfall in mm (exponential distribution)
- Binary cold-day flag (temp below 6C)
- Binary rainy-day flag (rainfall above 5mm)

### 3. Feature Engineering

35 features are constructed per zone before modeling:

- **Trend:** Linear time index from series start
- **Calendar:** Day of week, month, day of month, week of year, weekend/Monday flags
- **Fourier weekly seasonality:** 3 harmonic pairs (sin/cos at periods of 7 days)
- **Fourier annual seasonality:** 5 harmonic pairs (sin/cos at periods of 365.25 days)
- **Lag features:** lag-1, lag-7 (same day last week), lag-14
- **Rolling statistics:** 7-day and 14-day rolling mean and standard deviation
- **Holiday and event flags:** is_holiday, is_pre_holiday, days_to_xmas
- **Weather regressors:** avg_temp_c, rainfall_mm, cold_day, rainy_day

### 4. Forecasting Model

Each of the 8 zones has its own independently trained Prophet model. Key configuration:

- **Seasonality mode:** Multiplicative (order spikes scale with trend level)
- **Yearly seasonality:** Enabled
- **Weekly seasonality:** Enabled
- **Holiday effects:** UK public holidays injected via Prophet's holiday DataFrame
- **Extra regressors:** avg_temp_c, rainfall_mm (standardized)
- **Changepoint prior scale:** 0.05 (moderate trend flexibility)
- **Train/test split:** 80% train, 20% test (chronological, no shuffle)

### 5. Baseline

Naive lag-7 forecast: predict today's demand using the actual value from the same weekday last week. Missing lag values are replaced with the 7-day rolling mean.

### 6. Inventory Simulation

A single-period ordering policy is simulated over the test set:

```
ordered_quantity = forecast * (1 + 0.15)    # 15% safety stock buffer
stockout = 1  if  actual_demand > ordered_quantity  else  0
```

Stockout count, total unmet demand, and fill rate are computed per zone for both Prophet and the baseline.

---

## Results

### Forecast Accuracy

| Zone | MAE (Prophet) | MAE (Baseline) | MAE Improvement |
|------|--------------|----------------|-----------------|
| Zone 1 UK | 4,823 | 8,807 | +45.2% |
| Zone 2 Germany | 399 | 622 | +35.9% |
| Zone 3 France | 312 | 641 | +51.3% |
| Zone 4 EIRE | 541 | 975 | +44.5% |
| Zone 5 Spain | 185 | 368 | +49.6% |
| Zone 6 Netherlands | 1,102 | 3,460 | +68.1% |
| Zone 7 Belgium | 178 | 290 | +38.6% |
| Zone 8 Switzerland | 412 | 922 | +55.3% |

**Mean MAE improvement across all zones: +48.6%**

### Inventory Simulation (15% Safety Stock)

| Metric | Prophet | Naive Baseline |
|--------|---------|----------------|
| Total stockout days | 34 | 95 |
| Stockout reduction | 61 days | — |
| Avg fill rate | 97.41% | 91.13% |

---

## How to Run

### Requirements

```
python >= 3.9
prophet
holidays
pandas
numpy
scikit-learn
matplotlib
seaborn
openpyxl
```

Install dependencies:

```bash
pip install prophet holidays pandas numpy scikit-learn matplotlib seaborn openpyxl
```

### Steps

1. Place `Online Retail.xlsx` in the project root directory
2. Open `dark_store_demand_forecasting.ipynb` in Jupyter
3. Run all cells top to bottom

Each cell produces visible output. Training all 8 zones takes approximately 3 to 5 minutes depending on hardware.

---

## Key Findings

- Prophet captures weekly demand cycles reliably across all zones, with Monday and midweek peaks dominating the weekly pattern.
- Holiday flags reduce forecast error around Christmas and bank holidays, which are high-spike events that naive methods miss entirely.
- Temperature shows a moderate negative correlation with order volume in the UK zone, consistent with reduced outdoor activity driving more online grocery demand in cold periods.
- Zone 6 Netherlands shows the largest accuracy gain (+68.1% MAE reduction), suggesting the baseline performs poorly there due to high volatility that Prophet's trend decomposition handles better.
- The 24-hour inference function accepts live weather input and returns a point forecast, prediction interval, and recommended order quantity for any zone.

---

## Limitations

- Weather data is simulated from UK climate statistics rather than sourced from a real weather API. Integrating an actual weather feed (e.g. Open-Meteo) would improve signal quality.
- Smaller zones (Spain, Belgium, Switzerland) have sparse daily observations, which can make MAPE metrics unstable when actual volumes are near zero.
- The inventory simulation uses a simplified single-period model. A multi-period policy with lead times and holding costs would be more realistic for production use.

---

## Possible Extensions

- Integrate a live weather API for real temperature and rainfall data
- Add promotional event flags (sales campaigns, seasonal events)
- Experiment with XGBoost or LightGBM on the engineered features as an ensemble layer on top of Prophet
- Deploy the inference function as a REST API endpoint for integration with warehouse management systems
- Extend the forecast horizon to 48 or 72 hours using recursive multi-step prediction
