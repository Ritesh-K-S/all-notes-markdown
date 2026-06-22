# Chapter 12: Time Series Forecasting

## Table of Contents
- [12.1 Introduction to Time Series](#121-introduction-to-time-series)
- [12.2 Time Series Components](#122-time-series-components)
- [12.3 Stationarity and Transformations](#123-stationarity-and-transformations)
- [12.4 Autocorrelation and Partial Autocorrelation](#124-autocorrelation-and-partial-autocorrelation)
- [12.5 ARIMA Family Models](#125-arima-family-models)
- [12.6 SARIMA (Seasonal ARIMA)](#126-sarima-seasonal-arima)
- [12.7 Exponential Smoothing Methods](#127-exponential-smoothing-methods)
- [12.8 Facebook Prophet](#128-facebook-prophet)
- [12.9 ML Approaches to Time Series](#129-ml-approaches-to-time-series)
- [12.10 Model Evaluation for Time Series](#1210-model-evaluation-for-time-series)
- [12.11 Common Mistakes](#1211-common-mistakes)
- [12.12 Interview Questions](#1212-interview-questions)
- [12.13 Quick Reference](#1213-quick-reference)

---

## 12.1 Introduction to Time Series

### What It Is
A time series is a sequence of data points collected over time, where the order matters. Unlike regular ML data (where rows are independent), time series data has temporal dependencies — what happened yesterday affects today.

**Analogy**: A time series is like a diary. Each entry depends on previous entries. You can't randomly shuffle diary pages and still understand the story. The sequence IS the information.

### Why It Matters
- **Stock market prediction** — $6.7 trillion traded daily
- **Weather forecasting** — Saves lives, drives agriculture
- **Demand forecasting** — Amazon, Walmart optimize inventory
- **Energy consumption** — Power grids need next-hour predictions
- **Healthcare** — Heart rate monitoring, epidemic forecasting

### Types of Time Series Problems

| Type | Description | Example |
|------|-------------|---------|
| Forecasting | Predict future values | Next month's sales |
| Anomaly Detection | Find unusual patterns | Fraud detection in transactions |
| Classification | Classify entire sequences | Activity recognition from sensors |
| Segmentation | Find change points | Detecting regime changes in markets |

---

### Key Concepts

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

# Create a sample time series
dates = pd.date_range(start='2020-01-01', periods=365, freq='D')
np.random.seed(42)

# Simulating daily sales with trend + seasonality + noise
trend = np.linspace(100, 200, 365)                    # Upward trend
seasonality = 30 * np.sin(2 * np.pi * np.arange(365) / 365)  # Yearly cycle
weekly = 15 * np.sin(2 * np.pi * np.arange(365) / 7)         # Weekly cycle
noise = np.random.normal(0, 10, 365)                  # Random noise

sales = trend + seasonality + weekly + noise

ts = pd.Series(sales, index=dates, name='daily_sales')
print(ts.head())
print(f"\nShape: {ts.shape}")
print(f"Date range: {ts.index.min()} to {ts.index.max()}")
print(f"Frequency: {ts.index.freq}")

# Plot
ts.plot(figsize=(12, 4), title='Daily Sales')
plt.ylabel('Sales')
plt.show()
```

---

## 12.2 Time Series Components

### The Four Components

```
Original Time Series = Trend + Seasonality + Cyclical + Residual (Noise)

┌─────────────────────────────────────────────────────────────┐
│  Original:  ╱╲╱╲    ╱╲╱╲      ╱╲╱╲    ╱╲╱╲               │
│            ╱    ╲╱╲╱    ╲╱╲╱╲╱    ╲╱╲╱    ╲╱╲╱            │
│                                                              │
│  = Trend:     _______________                                │
│              ╱               ╲_______                        │
│             ╱                                                │
│                                                              │
│  + Seasonality:  ╱╲  ╱╲  ╱╲  ╱╲  ╱╲  ╱╲                   │
│                 ╱  ╲╱  ╲╱  ╲╱  ╲╱  ╲╱  ╲╱                  │
│                                                              │
│  + Residual:  ~~∿~∿~~∿~∿~~∿~∿~ (random noise)              │
└─────────────────────────────────────────────────────────────┘
```

| Component | Description | Example |
|-----------|-------------|---------|
| **Trend** | Long-term direction (up/down/flat) | Population growth |
| **Seasonality** | Fixed, known periodic patterns | Holiday shopping peaks |
| **Cyclical** | Variable-length fluctuations | Business cycles (3-7 years) |
| **Residual** | Random noise left after removing above | Unpredictable events |

### Additive vs. Multiplicative Decomposition

$$\text{Additive}: Y_t = T_t + S_t + R_t$$
$$\text{Multiplicative}: Y_t = T_t \times S_t \times R_t$$

- **Additive**: Seasonal fluctuations are CONSTANT in magnitude (±$100 every December)
- **Multiplicative**: Seasonal fluctuations GROW with the level (±10% every December)

```python
from statsmodels.tsa.seasonal import seasonal_decompose

# Decompose time series
decomposition = seasonal_decompose(ts, model='additive', period=7)  # Weekly seasonality

fig = decomposition.plot()
fig.set_size_inches(12, 10)
plt.tight_layout()
plt.show()

# Access individual components
trend = decomposition.trend
seasonal = decomposition.seasonal
residual = decomposition.resid

print(f"Trend range: {trend.dropna().min():.1f} to {trend.dropna().max():.1f}")
print(f"Seasonal amplitude: {seasonal.max():.1f}")
```

> **Pro Tip**: Use `model='multiplicative'` when the seasonal amplitude grows with the level of the series (common in financial data and growing businesses).

---

## 12.3 Stationarity and Transformations

### What Is Stationarity?

A time series is **stationary** if its statistical properties (mean, variance, autocorrelation) don't change over time.

```
Stationary:                    Non-Stationary:
Mean is constant               Mean changes over time
  ____    ____                    ____
 ╱    ╲  ╱    ╲                       ╱╲  ╱╲
╱      ╲╱      ╲╱╲╱           ╱╲  ╱╲╱  ╲╱  ╲
                              ╱  ╲╱
                             ╱
─────────────────────        ─────────────────────
```

### Why Stationarity Matters
- Most classical models (ARIMA) **assume** stationarity
- Statistical properties must be stable to make predictions
- Non-stationary data → spurious relationships → garbage forecasts

---

### 12.3.1 Testing for Stationarity

#### Augmented Dickey-Fuller (ADF) Test

**Hypothesis**:
- $H_0$: Series has a unit root (non-stationary)
- $H_1$: Series is stationary

```python
from statsmodels.tsa.stattools import adfuller, kpss

def adf_test(series, title=''):
    """Perform Augmented Dickey-Fuller test"""
    result = adfuller(series.dropna(), autolag='AIC')
    print(f'=== ADF Test: {title} ===')
    print(f'ADF Statistic: {result[0]:.4f}')
    print(f'p-value:       {result[1]:.4f}')
    print(f'Lags Used:     {result[2]}')
    print(f'Observations:  {result[3]}')
    for key, value in result[4].items():
        print(f'Critical Value ({key}): {value:.4f}')
    
    if result[1] <= 0.05:
        print("✅ STATIONARY (reject H0, p ≤ 0.05)")
    else:
        print("❌ NON-STATIONARY (fail to reject H0, p > 0.05)")
    print()

adf_test(ts, 'Original Series')
```

#### KPSS Test (Complementary)

**Hypothesis** (opposite of ADF!):
- $H_0$: Series is stationary
- $H_1$: Series is non-stationary

```python
def kpss_test(series, title=''):
    """Perform KPSS test for stationarity"""
    result = kpss(series.dropna(), regression='c', nlags='auto')
    print(f'=== KPSS Test: {title} ===')
    print(f'KPSS Statistic: {result[0]:.4f}')
    print(f'p-value:        {result[1]:.4f}')
    for key, value in result[3].items():
        print(f'Critical Value ({key}): {value:.4f}')
    
    if result[1] >= 0.05:
        print("✅ STATIONARY (fail to reject H0, p ≥ 0.05)")
    else:
        print("❌ NON-STATIONARY (reject H0, p < 0.05)")
    print()

kpss_test(ts, 'Original Series')
```

> **Best Practice**: Use BOTH tests together. If they disagree, differencing or transformation is needed.

---

### 12.3.2 Making Series Stationary

#### Method 1: Differencing

$$y'_t = y_t - y_{t-1}$$

```python
# First difference (removes linear trend)
ts_diff1 = ts.diff().dropna()
adf_test(ts_diff1, 'First Difference')

# Second difference (if first isn't enough)
ts_diff2 = ts.diff().diff().dropna()

# Seasonal differencing (removes seasonality)
ts_seasonal_diff = ts.diff(7).dropna()  # diff with period 7 (weekly)

# Plot comparison
fig, axes = plt.subplots(3, 1, figsize=(12, 8))
ts.plot(ax=axes[0], title='Original')
ts_diff1.plot(ax=axes[1], title='First Difference')
ts_seasonal_diff.plot(ax=axes[2], title='Seasonal Difference (period=7)')
plt.tight_layout()
plt.show()
```

#### Method 2: Log Transformation + Differencing

```python
# Log transform (stabilizes variance) then difference (removes trend)
ts_log = np.log(ts)
ts_log_diff = ts_log.diff().dropna()
adf_test(ts_log_diff, 'Log + First Difference')
```

---

## 12.4 Autocorrelation and Partial Autocorrelation

### What They Are

- **ACF (Autocorrelation Function)**: Correlation between the series and its lagged version. "How much does today correlate with 7 days ago?"
- **PACF (Partial Autocorrelation Function)**: Direct correlation with a lag, removing the effect of intermediate lags.

**Analogy**: ACF is like asking "Does your grandparent's height predict yours?" (includes the indirect effect through your parent). PACF is "Does your grandparent's height predict yours, AFTER removing the effect of your parent?"

### Why They Matter
- ACF and PACF plots are used to determine ARIMA parameters (p, d, q)
- They reveal the memory structure of your time series

```python
from statsmodels.graphics.tsaplots import plot_acf, plot_pacf

fig, axes = plt.subplots(1, 2, figsize=(14, 4))

# ACF plot
plot_acf(ts_diff1, lags=40, ax=axes[0])
axes[0].set_title('Autocorrelation Function (ACF)')

# PACF plot
plot_pacf(ts_diff1, lags=40, ax=axes[1], method='ywm')
axes[1].set_title('Partial Autocorrelation Function (PACF)')

plt.tight_layout()
plt.show()
```

### Reading ACF/PACF Plots for ARIMA Order Selection

```
ACF Pattern              → Suggests
──────────────────────────────────────
Cuts off after lag q     → MA(q) model
Decays gradually         → AR component needed

PACF Pattern             → Suggests
──────────────────────────────────────
Cuts off after lag p     → AR(p) model
Decays gradually         → MA component needed
```

| ACF Pattern | PACF Pattern | Model |
|-------------|-------------|-------|
| Cuts off at lag q | Tails off | MA(q) |
| Tails off | Cuts off at lag p | AR(p) |
| Tails off | Tails off | ARMA(p,q) |

---

## 12.5 ARIMA Family Models

### 12.5.1 AR (Autoregressive) Model

**Idea**: Today's value depends on previous values.

$$y_t = c + \phi_1 y_{t-1} + \phi_2 y_{t-2} + ... + \phi_p y_{t-p} + \epsilon_t$$

**Analogy**: Predicting tomorrow's temperature by looking at the past few days' temperatures.

---

### 12.5.2 MA (Moving Average) Model

**Idea**: Today's value depends on previous forecast errors.

$$y_t = c + \epsilon_t + \theta_1 \epsilon_{t-1} + \theta_2 \epsilon_{t-2} + ... + \theta_q \epsilon_{t-q}$$

**Analogy**: "If I was off by $5 yesterday, I'll adjust my prediction today."

---

### 12.5.3 ARIMA (p, d, q)

Combines AR + differencing + MA:

$$\text{ARIMA}(p, d, q)$$

- **p** = AR order (number of lagged terms)
- **d** = Differencing order (times to difference for stationarity)
- **q** = MA order (number of lagged forecast errors)

```python
from statsmodels.tsa.arima.model import ARIMA
import warnings
warnings.filterwarnings('ignore')

# Split data (time series: NO random splitting!)
train_size = int(len(ts) * 0.8)
train, test = ts[:train_size], ts[train_size:]

print(f"Train: {train.index[0]} to {train.index[-1]} ({len(train)} obs)")
print(f"Test:  {test.index[0]} to {test.index[-1]} ({len(test)} obs)")

# Fit ARIMA(2,1,2)
model = ARIMA(train, order=(2, 1, 2))
fitted = model.fit()

# Model summary
print(fitted.summary())

# Forecast
forecast = fitted.forecast(steps=len(test))

# Plot
plt.figure(figsize=(12, 5))
plt.plot(train.index, train, label='Train', color='blue')
plt.plot(test.index, test, label='Actual', color='green')
plt.plot(test.index, forecast, label='Forecast', color='red', linestyle='--')
plt.legend()
plt.title('ARIMA Forecast')
plt.show()

# Evaluate
from sklearn.metrics import mean_absolute_error, mean_squared_error
mae = mean_absolute_error(test, forecast)
rmse = np.sqrt(mean_squared_error(test, forecast))
print(f"MAE:  {mae:.2f}")
print(f"RMSE: {rmse:.2f}")
```

---

### 12.5.4 Auto ARIMA (Automatic Parameter Selection)

```python
# pip install pmdarima
import pmdarima as pm

# Automatically finds best (p,d,q)
auto_model = pm.auto_arima(
    train,
    start_p=0, max_p=5,       # AR range
    start_q=0, max_q=5,       # MA range
    d=None,                    # Auto-determine differencing
    seasonal=False,            # Set True for SARIMA
    stepwise=True,             # Faster search
    suppress_warnings=True,
    information_criterion='aic',  # Model selection criterion
    trace=True                 # Show search progress
)

print(f"\nBest model: ARIMA{auto_model.order}")
print(f"AIC: {auto_model.aic():.2f}")

# Forecast
forecast = auto_model.predict(n_periods=len(test))
```

### Model Selection Criteria

| Criterion | Formula | Use |
|-----------|---------|-----|
| AIC | $-2\ln(L) + 2k$ | Default choice, penalizes complexity |
| BIC | $-2\ln(L) + k\ln(n)$ | Stronger penalty for complexity (prefers simpler) |
| HQIC | $-2\ln(L) + 2k\ln(\ln(n))$ | Compromise between AIC and BIC |

Where $L$ = likelihood, $k$ = number of parameters, $n$ = observations.

---

### 12.5.5 Diagnostic Checks

After fitting ARIMA, check that residuals look like white noise:

```python
# Residual diagnostics
residuals = fitted.resid

fig, axes = plt.subplots(2, 2, figsize=(12, 8))

# 1. Residual plot (should look random)
axes[0,0].plot(residuals)
axes[0,0].set_title('Residuals Over Time')
axes[0,0].axhline(y=0, color='r', linestyle='--')

# 2. Histogram (should look normal)
axes[0,1].hist(residuals, bins=30, density=True)
axes[0,1].set_title('Residual Distribution')

# 3. ACF of residuals (should have no significant lags)
plot_acf(residuals, lags=30, ax=axes[1,0])
axes[1,0].set_title('ACF of Residuals')

# 4. Q-Q plot (should follow diagonal)
from scipy import stats
stats.probplot(residuals, plot=axes[1,1])
axes[1,1].set_title('Q-Q Plot')

plt.tight_layout()
plt.show()

# Ljung-Box test: Are residuals white noise?
from statsmodels.stats.diagnostic import acorr_ljungbox
lb_test = acorr_ljungbox(residuals, lags=[10, 20, 30])
print(lb_test)
# If p-values > 0.05 → residuals are white noise ✅ (good!)
```

---

## 12.6 SARIMA (Seasonal ARIMA)

### What It Is
SARIMA extends ARIMA to handle seasonality by adding seasonal AR, differencing, and MA components.

$$\text{SARIMA}(p, d, q) \times (P, D, Q, m)$$

- **(p, d, q)** = Non-seasonal ARIMA parameters
- **(P, D, Q)** = Seasonal ARIMA parameters
- **m** = Seasonal period (12 for monthly, 7 for daily with weekly pattern, 4 for quarterly)

```python
from statsmodels.tsa.statespace.sarimax import SARIMAX

# Generate monthly data with strong seasonality
dates_monthly = pd.date_range(start='2018-01-01', periods=60, freq='MS')
trend_m = np.linspace(100, 200, 60)
seasonal_m = 40 * np.sin(2 * np.pi * np.arange(60) / 12)  # 12-month cycle
noise_m = np.random.normal(0, 8, 60)
monthly_sales = pd.Series(trend_m + seasonal_m + noise_m, index=dates_monthly)

# Split
train_m = monthly_sales[:48]  # 4 years train
test_m = monthly_sales[48:]   # 1 year test

# Fit SARIMA(1,1,1)(1,1,1,12)
sarima_model = SARIMAX(
    train_m,
    order=(1, 1, 1),              # (p, d, q)
    seasonal_order=(1, 1, 1, 12), # (P, D, Q, m)
    enforce_stationarity=False,
    enforce_invertibility=False
)
sarima_fitted = sarima_model.fit(disp=False)
print(sarima_fitted.summary())

# Forecast with confidence intervals
forecast_result = sarima_fitted.get_forecast(steps=12)
forecast_mean = forecast_result.predicted_mean
confidence_int = forecast_result.conf_int()

# Plot
plt.figure(figsize=(12, 5))
plt.plot(train_m, label='Train')
plt.plot(test_m, label='Actual', color='green')
plt.plot(forecast_mean, label='Forecast', color='red', linestyle='--')
plt.fill_between(confidence_int.index, 
                 confidence_int.iloc[:, 0], 
                 confidence_int.iloc[:, 1], 
                 alpha=0.2, color='red', label='95% CI')
plt.legend()
plt.title('SARIMA Forecast with Confidence Intervals')
plt.show()
```

### Auto SARIMA

```python
auto_sarima = pm.auto_arima(
    train_m,
    start_p=0, max_p=3,
    start_q=0, max_q=3,
    d=None,
    seasonal=True,           # Enable seasonal
    m=12,                    # Monthly seasonality
    start_P=0, max_P=2,
    start_Q=0, max_Q=2,
    D=None,
    trace=True,
    error_action='ignore',
    suppress_warnings=True,
    stepwise=True
)

print(f"Best SARIMA: {auto_sarima.order} x {auto_sarima.seasonal_order}")
```

---

## 12.7 Exponential Smoothing Methods

### 12.7.1 Simple Exponential Smoothing (SES)

For data with **no trend, no seasonality**:

$$\hat{y}_{t+1} = \alpha y_t + (1-\alpha) \hat{y}_t$$

$\alpha$ (0 to 1) controls how fast old observations are "forgotten":
- $\alpha$ close to 1 → Reacts quickly to changes (short memory)
- $\alpha$ close to 0 → Smooth, slow to react (long memory)

```python
from statsmodels.tsa.holtwinters import SimpleExpSmoothing, ExponentialSmoothing

# Simple Exponential Smoothing
ses_model = SimpleExpSmoothing(train).fit(smoothing_level=0.3, optimized=False)
ses_forecast = ses_model.forecast(len(test))
```

---

### 12.7.2 Holt's Linear Method (Double Exponential)

For data with **trend, no seasonality**:

$$\hat{y}_{t+h} = l_t + h \cdot b_t$$

Where $l_t$ = level, $b_t$ = trend.

```python
from statsmodels.tsa.holtwinters import Holt

holt_model = Holt(train).fit(smoothing_level=0.3, smoothing_trend=0.1)
holt_forecast = holt_model.forecast(len(test))
```

---

### 12.7.3 Holt-Winters (Triple Exponential Smoothing)

For data with **trend AND seasonality**. The most complete exponential smoothing method.

```python
# Additive seasonality
hw_add = ExponentialSmoothing(
    train_m, 
    trend='add',           # Additive trend
    seasonal='add',        # Additive seasonality
    seasonal_periods=12    # Monthly data, yearly pattern
).fit()

# Multiplicative seasonality (when seasonal amplitude grows with level)
hw_mul = ExponentialSmoothing(
    train_m,
    trend='add',
    seasonal='mul',        # Multiplicative seasonality
    seasonal_periods=12
).fit()

# Forecast
hw_forecast_add = hw_add.forecast(12)
hw_forecast_mul = hw_mul.forecast(12)

# Plot comparison
plt.figure(figsize=(12, 5))
plt.plot(train_m, label='Train')
plt.plot(test_m, label='Actual', color='green')
plt.plot(hw_forecast_add, label='HW Additive', color='red', linestyle='--')
plt.plot(hw_forecast_mul, label='HW Multiplicative', color='purple', linestyle='--')
plt.legend()
plt.title('Holt-Winters Forecast')
plt.show()
```

### Exponential Smoothing Summary

| Method | Trend | Seasonal | Parameters |
|--------|:-----:|:--------:|-----------|
| SES | ❌ | ❌ | α (level) |
| Holt's | ✅ | ❌ | α, β (trend) |
| Holt-Winters | ✅ | ✅ | α, β, γ (seasonal) |

---

## 12.8 Facebook Prophet

### What It Is
Prophet is an additive regression model by Meta (Facebook) designed for business time series. It handles holidays, missing data, and changepoints automatically.

$$y(t) = g(t) + s(t) + h(t) + \epsilon_t$$

- $g(t)$ = trend (linear or logistic growth)
- $s(t)$ = seasonality (Fourier series)
- $h(t)$ = holidays/events
- $\epsilon_t$ = noise

### Why Use Prophet?
- Works great out-of-the-box for business data
- Handles missing data and outliers gracefully
- Built-in holiday effects
- Automatic changepoint detection
- Uncertainty intervals included
- Non-statisticians can use it effectively

```python
# pip install prophet
from prophet import Prophet

# Prophet requires specific column names: 'ds' (date) and 'y' (value)
df_prophet = pd.DataFrame({
    'ds': monthly_sales.index,
    'y': monthly_sales.values
})

train_prophet = df_prophet[:48]
test_prophet = df_prophet[48:]

# Initialize and fit
model = Prophet(
    yearly_seasonality=True,
    weekly_seasonality=False,    # Monthly data, no weekly pattern
    daily_seasonality=False,
    changepoint_prior_scale=0.05,  # Flexibility of trend changes (0.001-0.5)
    seasonality_prior_scale=10,    # Flexibility of seasonality
)

# Add custom seasonality if needed
# model.add_seasonality(name='quarterly', period=91.25, fourier_order=5)

# Add holidays (optional)
# holidays = pd.DataFrame({
#     'holiday': 'black_friday',
#     'ds': pd.to_datetime(['2020-11-27', '2021-11-26', '2022-11-25']),
#     'lower_window': -1,  # 1 day before
#     'upper_window': 1    # 1 day after
# })
# model = Prophet(holidays=holidays)

model.fit(train_prophet)

# Create future dataframe
future = model.make_future_dataframe(periods=12, freq='MS')

# Predict
forecast = model.predict(future)

# Plot
fig1 = model.plot(forecast)
plt.title('Prophet Forecast')
plt.show()

# Plot components (trend, seasonality)
fig2 = model.plot_components(forecast)
plt.show()

# Evaluate
prophet_pred = forecast.tail(12)['yhat'].values
mae = mean_absolute_error(test_prophet['y'], prophet_pred)
print(f"Prophet MAE: {mae:.2f}")
```

### Prophet Tuning Tips

```python
# For more flexible trend (captures sudden changes)
model = Prophet(changepoint_prior_scale=0.5)

# For smoother trend (less overfitting)
model = Prophet(changepoint_prior_scale=0.001)

# Logistic growth (bounded, e.g., market saturation)
model = Prophet(growth='logistic')
df_prophet['cap'] = 1000  # Upper bound
df_prophet['floor'] = 0   # Lower bound

# Cross-validation for Prophet
from prophet.diagnostics import cross_validation, performance_metrics

df_cv = cross_validation(model, initial='730 days', period='180 days', horizon='365 days')
df_p = performance_metrics(df_cv)
print(df_p[['horizon', 'mape', 'rmse', 'mae']].tail())
```

---

## 12.9 ML Approaches to Time Series

### 12.9.1 Feature Engineering for Time Series ML

Convert time series to supervised learning problem:

```python
def create_features(df, target_col, lags=7, window_sizes=[7, 14, 30]):
    """Create ML features from time series"""
    features = pd.DataFrame(index=df.index)
    
    # Lag features
    for lag in range(1, lags + 1):
        features[f'lag_{lag}'] = df[target_col].shift(lag)
    
    # Rolling statistics
    for window in window_sizes:
        features[f'rolling_mean_{window}'] = df[target_col].rolling(window).mean()
        features[f'rolling_std_{window}'] = df[target_col].rolling(window).std()
        features[f'rolling_min_{window}'] = df[target_col].rolling(window).min()
        features[f'rolling_max_{window}'] = df[target_col].rolling(window).max()
    
    # Date features
    features['day_of_week'] = df.index.dayofweek
    features['month'] = df.index.month
    features['day_of_month'] = df.index.day
    features['is_weekend'] = (df.index.dayofweek >= 5).astype(int)
    
    # Cyclical encoding
    features['month_sin'] = np.sin(2 * np.pi * features['month'] / 12)
    features['month_cos'] = np.cos(2 * np.pi * features['month'] / 12)
    features['dow_sin'] = np.sin(2 * np.pi * features['day_of_week'] / 7)
    features['dow_cos'] = np.cos(2 * np.pi * features['day_of_week'] / 7)
    
    # Target
    features['target'] = df[target_col]
    
    return features.dropna()

# Create features
df_ml = create_features(ts.to_frame('sales'), 'sales', lags=14)
print(f"Features created: {df_ml.shape[1] - 1}")
print(df_ml.columns.tolist())
```

---

### 12.9.2 XGBoost / LightGBM for Time Series

```python
from sklearn.ensemble import GradientBoostingRegressor
# Or: from xgboost import XGBRegressor
# Or: from lightgbm import LGBMRegressor

# Prepare features and target
feature_cols = [c for c in df_ml.columns if c != 'target']
X = df_ml[feature_cols]
y = df_ml['target']

# Time-based split (NOT random!)
split_idx = int(len(X) * 0.8)
X_train, X_test = X[:split_idx], X[split_idx:]
y_train, y_test = y[:split_idx], y[split_idx:]

# Train
model = GradientBoostingRegressor(
    n_estimators=200,
    max_depth=5,
    learning_rate=0.1,
    random_state=42
)
model.fit(X_train, y_train)

# Predict
y_pred = model.predict(X_test)

# Evaluate
mae = mean_absolute_error(y_test, y_pred)
rmse = np.sqrt(mean_squared_error(y_test, y_pred))
print(f"MAE:  {mae:.2f}")
print(f"RMSE: {rmse:.2f}")

# Feature importance
importances = pd.Series(model.feature_importances_, index=feature_cols)
importances.nlargest(10).plot(kind='barh', title='Top 10 Feature Importances')
plt.show()
```

---

### 12.9.3 Multi-Step Forecasting Strategies

```
Strategy 1: Recursive (One-Step)
─────────────────────────────────
Predict t+1, use prediction as input for t+2, etc.
[known, known, pred₁] → pred₂
[known, pred₁, pred₂] → pred₃
Problem: Errors compound!

Strategy 2: Direct (Multi-Output)
─────────────────────────────────
Train separate model for each horizon.
Model₁ predicts t+1
Model₂ predicts t+2
Model₃ predicts t+3
Problem: Doesn't capture dependencies between horizons.

Strategy 3: Multi-Output Model
─────────────────────────────────
One model predicts [t+1, t+2, ..., t+h] simultaneously.
Best for neural networks.
```

```python
# Recursive (iterative) forecasting
def recursive_forecast(model, last_features, n_steps, feature_cols):
    """Predict multiple steps ahead recursively"""
    predictions = []
    current_features = last_features.copy()
    
    for step in range(n_steps):
        # Predict next step
        pred = model.predict(current_features.values.reshape(1, -1))[0]
        predictions.append(pred)
        
        # Update lag features with the new prediction
        for i in range(len(feature_cols)-1, 0, -1):
            if f'lag_{i+1}' in feature_cols:
                current_features[f'lag_{i+1}'] = current_features[f'lag_{i}']
        current_features['lag_1'] = pred
    
    return predictions

# Direct multi-step: train separate model for each horizon
from sklearn.multioutput import MultiOutputRegressor

# Create multi-step targets
horizons = [1, 7, 14, 30]  # Forecast 1, 7, 14, 30 days ahead
for h in horizons:
    df_ml[f'target_h{h}'] = df_ml['target'].shift(-h)

df_ml_multi = df_ml.dropna()
targets = [f'target_h{h}' for h in horizons]
y_multi = df_ml_multi[targets]
X_multi = df_ml_multi[feature_cols]
```

---

### 12.9.4 Walk-Forward Validation

The correct way to validate time series models:

```python
from sklearn.model_selection import TimeSeriesSplit

def walk_forward_validation(model, X, y, n_splits=5):
    """Walk-forward (expanding window) validation"""
    tscv = TimeSeriesSplit(n_splits=n_splits)
    scores = []
    
    for fold, (train_idx, test_idx) in enumerate(tscv.split(X)):
        X_train_fold = X.iloc[train_idx]
        y_train_fold = y.iloc[train_idx]
        X_test_fold = X.iloc[test_idx]
        y_test_fold = y.iloc[test_idx]
        
        model.fit(X_train_fold, y_train_fold)
        y_pred = model.predict(X_test_fold)
        
        mae = mean_absolute_error(y_test_fold, y_pred)
        scores.append(mae)
        print(f"Fold {fold+1}: MAE = {mae:.2f} "
              f"(Train: {len(train_idx)}, Test: {len(test_idx)})")
    
    print(f"\nMean MAE: {np.mean(scores):.2f} ± {np.std(scores):.2f}")
    return scores

scores = walk_forward_validation(model, X, y, n_splits=5)
```

---

## 12.10 Model Evaluation for Time Series

### 12.10.1 Metrics

```python
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score

def evaluate_forecast(actual, predicted, model_name=""):
    """Calculate all time series metrics"""
    mae = mean_absolute_error(actual, predicted)
    rmse = np.sqrt(mean_squared_error(actual, predicted))
    mape = np.mean(np.abs((actual - predicted) / actual)) * 100
    r2 = r2_score(actual, predicted)
    
    # Symmetric MAPE (handles zeros better)
    smape = np.mean(2 * np.abs(actual - predicted) / (np.abs(actual) + np.abs(predicted))) * 100
    
    print(f"=== {model_name} ===")
    print(f"MAE:   {mae:.2f}")
    print(f"RMSE:  {rmse:.2f}")
    print(f"MAPE:  {mape:.2f}%")
    print(f"sMAPE: {smape:.2f}%")
    print(f"R²:    {r2:.4f}")
    
    return {'MAE': mae, 'RMSE': rmse, 'MAPE': mape, 'sMAPE': smape, 'R2': r2}

# Compare models
results_arima = evaluate_forecast(test.values, forecast.values, "ARIMA")
```

---

### 12.10.2 Baseline Models (Always Compare Against These!)

```python
# Naive forecast: tomorrow = today
naive_forecast = train.iloc[-1]  # Last known value repeated
naive_pred = pd.Series([naive_forecast] * len(test), index=test.index)

# Seasonal naive: same day last year/week
seasonal_naive = train.iloc[-7:]  # Last week's values repeated
# For weekly seasonality, repeat the last week

# Moving average baseline
ma_forecast = train.rolling(7).mean().iloc[-1]
ma_pred = pd.Series([ma_forecast] * len(test), index=test.index)

print("=== Baseline Comparison ===")
evaluate_forecast(test.values, naive_pred.values, "Naive")
evaluate_forecast(test.values, ma_pred.values, "Moving Average")
```

> **Key Rule**: If your complex model can't beat the naive baseline, it's not adding value!

---

## 12.11 Common Mistakes

### Mistake 1: Random Train/Test Split
```python
# ❌ WRONG - Future data leaks into training
from sklearn.model_selection import train_test_split
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# ✅ CORRECT - Always split by time
split_point = int(len(X) * 0.8)
X_train, X_test = X[:split_point], X[split_point:]
y_train, y_test = y[:split_point], y[split_point:]
```

### Mistake 2: Not Checking Stationarity Before ARIMA
```python
# ❌ WRONG - Applying ARIMA to non-stationary data
model = ARIMA(non_stationary_series, order=(2, 0, 2))  # d=0, no differencing!

# ✅ CORRECT - Test stationarity, then use appropriate d
adf_test(series)  # Check if stationary
model = ARIMA(series, order=(2, 1, 2))  # d=1 for differencing
# Or let auto_arima determine d automatically
```

### Mistake 3: Using Future Information in Features
```python
# ❌ WRONG - Rolling mean includes future values!
df['rolling_mean_7'] = df['sales'].rolling(7, center=True).mean()  # center=True uses future!

# ✅ CORRECT - Only use past values
df['rolling_mean_7'] = df['sales'].rolling(7).mean()  # Default: backward-looking only
df['rolling_mean_7'] = df['sales'].shift(1).rolling(7).mean()  # Even safer: shift first
```

### Mistake 4: Not Comparing Against Baselines
```python
# ❌ WRONG - "My LSTM has MAE of 5.3, ship it!"
# You don't know if that's good without a baseline

# ✅ CORRECT - Always compare
naive_mae = mean_absolute_error(test, [train.iloc[-1]] * len(test))
model_mae = mean_absolute_error(test, predictions)
improvement = (naive_mae - model_mae) / naive_mae * 100
print(f"Improvement over naive: {improvement:.1f}%")
# If < 10%, maybe your complex model isn't worth the complexity
```

### Mistake 5: Ignoring Prediction Intervals
```python
# ❌ WRONG - Only reporting point forecasts
forecast = model.predict(n_periods=30)

# ✅ CORRECT - Always include uncertainty
forecast, conf_int = model.predict(n_periods=30, return_conf_int=True)
# Report: "Sales will be 150 ± 20 (95% CI: 130-170)"
```

### Mistake 6: Overfitting with Too Many Lags
```python
# ❌ WRONG - Using 365 lag features for daily data
features = create_features(df, lags=365)  # Way too many!

# ✅ CORRECT - Use domain knowledge for relevant lags
# For daily data with weekly pattern: lags 1-7, 14, 21, 28
relevant_lags = [1, 2, 3, 4, 5, 6, 7, 14, 21, 28]
```

---

## 12.12 Interview Questions

### Q1: What is stationarity and why is it important for time series models?
**Answer**: A stationary series has constant mean, variance, and autocorrelation over time. It's important because ARIMA and many statistical models assume stationarity — they model the relationship between a value and its lags, which only makes sense if those relationships are stable. Non-stationary data can be made stationary through differencing, log transforms, or detrending.

### Q2: Explain the difference between ACF and PACF. How do you use them to determine ARIMA parameters?
**Answer**: ACF shows total correlation with lag k (including indirect effects through intermediate lags). PACF shows direct correlation with lag k (removing intermediate effects). For ARIMA order: if PACF cuts off at lag p → AR(p). If ACF cuts off at lag q → MA(q). If both tail off → need both AR and MA components (use AIC/BIC to select).

### Q3: When would you use SARIMA vs. Prophet vs. XGBoost for time series?
**Answer**: **SARIMA**: Strong seasonal patterns, smaller datasets, need statistical rigor and confidence intervals. **Prophet**: Business time series with holidays, missing data, multiple seasonalities, quick iteration. **XGBoost/LightGBM**: Many exogenous features available, non-linear relationships, very large datasets, when you need feature importance. In practice, try all three and compare.

### Q4: How do you handle multiple seasonalities (e.g., daily + weekly + yearly)?
**Answer**: SARIMA handles only one seasonal period. Options: (1) Prophet (handles multiple seasonalities natively via Fourier terms), (2) TBATS model (Box-Cox, ARMA errors, Trend, and Seasonal), (3) Feature engineering for ML models (encode each seasonality as separate features), (4) Hierarchical decomposition.

### Q5: What is walk-forward validation and why is it necessary?
**Answer**: Walk-forward validation is time series cross-validation where training data always precedes test data. At each fold, the training window expands (or slides) forward. It's necessary because random CV would leak future information into training. It simulates how the model would actually be used — trained on historical data to predict future data.

### Q6: How do you detect and handle structural breaks (change points) in a time series?
**Answer**: Detection: (1) Visual inspection, (2) CUSUM test, (3) Prophet's automatic changepoint detection, (4) Bai-Perron test. Handling: (1) Train only on data after the break, (2) Include a regime indicator feature, (3) Use models with adaptive components (like Prophet's changepoint flexibility), (4) Use piecewise linear models.

### Q7: Explain the difference between additive and multiplicative decomposition.
**Answer**: Additive: Y = T + S + R. Seasonal fluctuations are constant ($±100 every December). Multiplicative: Y = T × S × R. Seasonal fluctuations scale with the level (±10% every December, so $100 when sales are $1000 but $500 when sales are $5000). Use multiplicative when seasonal amplitude grows with the trend. Convert to additive by taking log(Y).

### Q8: Your time series model performs great on training but poorly on test. What do you check?
**Answer**: (1) **Data leakage** — features using future info? Scaling on full data? (2) **Overfitting** — too many parameters? Reduce model complexity (lower p, q, fewer lags). (3) **Non-stationarity** — did the underlying process change? Check for structural breaks. (4) **Seasonality shift** — holiday patterns changed? (5) **Outliers in test period** — COVID-like events? (6) **Verify temporal split** — ensure no shuffle contamination.

---

## 12.13 Quick Reference

### Model Selection Guide

| Data Characteristics | Recommended Models |
|---------------------|-------------------|
| No trend, no seasonality | SES, ARIMA(p,0,q) |
| Trend only | Holt's, ARIMA(p,1,q) |
| Trend + seasonality | SARIMA, Holt-Winters, Prophet |
| Multiple seasonalities | Prophet, TBATS, ML models |
| Many exogenous features | XGBoost, LightGBM, SARIMAX |
| Non-linear patterns | ML models (RF, GBM), Neural nets |
| Fast prototyping | Prophet, auto_arima |
| Few data points (< 50) | SES, Holt's, simple ARIMA |

### ARIMA Parameter Quick Guide

| Parameter | Meaning | How to Determine |
|-----------|---------|-----------------|
| p (AR) | Auto-regressive lags | PACF cutoff |
| d (I) | Differencing order | ADF test (usually 0, 1, or 2) |
| q (MA) | Moving average lags | ACF cutoff |
| P (Seasonal AR) | Seasonal AR lags | Seasonal PACF |
| D (Seasonal I) | Seasonal differencing | Usually 0 or 1 |
| Q (Seasonal MA) | Seasonal MA lags | Seasonal ACF |
| m | Seasonal period | Domain knowledge (12, 7, 4, 24) |

### Stationarity Tests Cheat Sheet

| Test | H₀ | Stationary If |
|------|-----|--------------|
| ADF | Non-stationary | p-value ≤ 0.05 (reject H₀) |
| KPSS | Stationary | p-value ≥ 0.05 (don't reject H₀) |
| PP (Phillips-Perron) | Non-stationary | p-value ≤ 0.05 |

### Time Series Metrics

| Metric | Formula | Notes |
|--------|---------|-------|
| MAE | mean(\|actual - pred\|) | Robust to outliers |
| RMSE | √mean((actual - pred)²) | Penalizes large errors |
| MAPE | mean(\|actual - pred\| / actual) × 100 | Percentage, undefined at 0 |
| sMAPE | mean(2\|a-p\| / (\|a\|+\|p\|)) × 100 | Symmetric, bounded |

### Common Seasonal Periods

| Data Frequency | Typical m | Pattern |
|---------------|:---------:|---------|
| Hourly | 24 | Daily cycle |
| Daily | 7 | Weekly cycle |
| Weekly | 52 | Yearly cycle |
| Monthly | 12 | Yearly cycle |
| Quarterly | 4 | Yearly cycle |

### Essential Python Imports

```python
# Statistical models
from statsmodels.tsa.arima.model import ARIMA
from statsmodels.tsa.statespace.sarimax import SARIMAX
from statsmodels.tsa.holtwinters import ExponentialSmoothing
from statsmodels.tsa.seasonal import seasonal_decompose
from statsmodels.tsa.stattools import adfuller, kpss
from statsmodels.graphics.tsaplots import plot_acf, plot_pacf

# Auto ARIMA
import pmdarima as pm

# Prophet
from prophet import Prophet

# ML approach
from sklearn.ensemble import GradientBoostingRegressor
from sklearn.model_selection import TimeSeriesSplit

# Evaluation
from sklearn.metrics import mean_absolute_error, mean_squared_error
```

---

*Previous: [11 - Feature Engineering for ML](11-Feature-Engineering-for-ML.md) | Next: [13 - Recommendation Systems](13-Recommendation-Systems.md)*
