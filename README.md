# Prediction of Dry and Humid Temperature from Atmospheric Data

This repository contains a technical project completed for Weenat. The objective is to predict the humid temperature (T2M_U_med) at crop height (50 cm, unsheltered) using only atmospheric variables measured at 2 meters and derived from MeteoVision technology

## Project Objective

This technical test focuses on two main tasks:

1. Clean a raw hourly meteorological dataset (`dtf_test`) by applying strict physical and statistical filtering rules.
2. Build and compare three models to estimate the humid temperature from the following atmospheric parameters:

- T2M (air temperature at 2m)
- DEWT2M (dew point temperature)
- RH2M (relative humidity)
- WS2M / WS10M (wind speed at 2m and 10m)
- WD (wind direction)
- VPD (vapor pressure deficit)
- CLDC (cloud cover)
- SSI (solar radiation)
- PRECIP (precipitation)

## Dataset Description

Each observation includes:

- Timestamps: `date`, `hour`
- Sensor location: `station_id`, `lat`, `lon`
- Recorded variables: `T2M`, `DEWT2M`, `RH2M`, `WS2M`, `WS10M`, `WD`, `VPD`, `CLDC`, `SSI`, `PRECIP`
- Target variables:  
  - `T2M_D_med`: dry temperature (50 cm, unsheltered)
  - `T2M_U_med`: humid temperature (50 cm, unsheltered)

## Data Cleaning

Conducted in `Prétraitement.ipynb`. Steps include:

- Removal of error values: T = ±60°C or -40°C
- Filtering period: November to May only
- Remove daylight values: `SSI > 50`
- Physical consistency checks:
  - `T2M_U_med ≤ T2M_D_med + 0.5`
  - `T2M ≤ 10°C`
  - `|T2M - T2M_D_med - 1.5| ≤ 3°C`
- Drop rows with missing `T2M_U_med`
- Exclude days with fewer than 50 records per sensor
- Remove test/calibration stations near Nantes HQ

Time spent: approximately 1 hour

## Modeling Approaches

Implemented in `Modélisation.ipynb`. Three modeling strategies:

### 1. Stull Formula (simplified physical model)

No training needed. Applied the analytical approximation:

Tw = 0.2831 * RH2M^0.2735 * T2M + 0.0003018 * RH2M^2 + 0.01289 * RH2M - 4.0962

### 2. Polynomial Regression

- Degree 1 (validated by performance trade-off)
- Regularized Ridge regression
- Trained on data from 15/03/2020 to 14/03/2023
- Selected features: T2M, DEWT2M, RH2M, WS2M, CLDC

### 3. Random Forest (scikit-learn)

- Same train/test split as above
- Hyperparameter tuning with cross-validation
- Feature importance evaluated
- Initial overfitting addressed via parameter adjustment

## Results Summary

**Comparative table (before RF correction):**

| Model           | MSE Train | R^2 Train | MSE Test | R^2 Test |
|----------------|-----------|----------|----------|---------|
| Stull          | 1.894     | 0.858    | 1.835    | 0.760   |
| Polynomial     | 1.293     | 0.903    | 1.343    | 0.825   |
| Random Forest  | 0.191     | 0.986    | 1.508    | 0.803   |

**Train/Test Generalization Gap:**

| Model           | ΔR^2       | ΔMSE     |
|----------------|-----------|----------|
| Stull          | 0.098     | -0.059   |
| Polynomial     | 0.078     | 0.050    |
| Random Forest  | 0.183     | 1.316    |

**After correcting RF overfitting:**

- Train: MSE = 1.265, R^2 = 0.905
- Test: MSE = 1.354, R^2 = 0.823

## Residuals Analysis

Shapiro test p-values indicate that none of the residuals are normally distributed.

| Model         | Mean Residual | Std Dev | Shapiro p-value |
|---------------|---------------|---------|------------------|
| Stull         | -0.042        | 1.354   | 0.000            |
| Polynomial    | -0.005        | 1.159   | 0.000            |       
| Random Forest | -0.001        | 1.163   | 0.000            |

## Feature Importance (Random Forest)

| Variable   | Importance |
|------------|------------|
| T2M        | 0.9793     |
| DEWT2M     | 0.0129     |
| CLDC       | 0.0047     |
| WS10M      | 0.0011     |
| WS2M       | 0.0009     |
| PRECIP     | 0.0004     |
| SSI        | 0.0004     |
| WD         | 0.0002     |
| RH2M       | 0.0001     |
| VPD        | 0.0000     |

## Conclusion

Among the tested models, the polynomial regression (degree 1) proved to be the most reliable and well-balanced:

- It offers strong accuracy (R^2 = 0.825) with good generalization (low ΔR^2 and ΔMSE).
- It is interpretable, lightweight, and suitable for operational deployment.

The Stull formula, despite being purely analytical and non-trainable, delivered reasonable performance and can serve as a robust fallback model in edge-device or degraded operation scenarios.

The Random Forest initially suffered from overfitting, but after hyperparameter correction, it achieved solid results, comparable to the polynomial model. However, its complexity and marginal improvement may not justify its use in this context.

For frost-risk detection tasks (e.g., T2M_U_med < 0°C), the polynomial regression model offers the best trade-off between performance and reliability.