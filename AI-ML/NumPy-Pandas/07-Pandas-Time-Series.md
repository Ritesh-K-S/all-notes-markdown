# Chapter 07: Pandas Time Series

## Table of Contents
- [DateTime Basics and Parsing](#datetime-basics-and-parsing)
- [DatetimeIndex and Time-Based Indexing](#datetimeindex-and-time-based-indexing)
- [Date Ranges and Frequencies](#date-ranges-and-frequencies)
- [Resampling — Changing Time Frequency](#resampling--changing-time-frequency)
- [Rolling and Expanding Windows](#rolling-and-expanding-windows)
- [Shift, Lag, and Percent Change](#shift-lag-and-percent-change)
- [Time Zones](#time-zones)
- [Period and Timedelta](#period-and-timedelta)
- [Real-World Time Series Patterns](#real-world-time-series-patterns)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## DateTime Basics and Parsing

### What It Is
Time series data is data indexed or associated with timestamps — stock prices over time, website traffic per hour, temperature readings every minute. Pandas has a rich set of tools built specifically for this. Think of it as giving Pandas a calendar and a clock so it understands time natively.

### Why It Matters
- Time series is everywhere: finance, IoT, weather, healthcare, web analytics
- Pandas was originally built at a hedge fund (AQR Capital) — time series is in its DNA
- Proper datetime handling enables resampling, rolling calculations, and time-based filtering
- Wrong datetime parsing is the #1 source of bugs in time series projects

### How It Works

Pandas uses four main time-related concepts:

```
┌──────────────────────────────────────────────────────────────────────┐
│  Concept       │ Class              │ Example                      │
├──────────────────────────────────────────────────────────────────────┤
│  Timestamp     │ pd.Timestamp       │ '2024-01-15 10:30:00'        │
│  Time Span     │ pd.Period          │ 'January 2024'               │
│  Time Delta    │ pd.Timedelta       │ '5 days 3 hours'             │
│  Date Offset   │ pd.DateOffset      │ 'Business day', 'Month end'  │
└──────────────────────────────────────────────────────────────────────┘

        Timestamp (point in time)
            │
            ▼
────|────────●────────|─────── time axis
    Period (span of time)

Timedelta = distance between two Timestamps
```

### Code Examples

```python
import pandas as pd
import numpy as np

# =============================================
# CREATING TIMESTAMPS
# =============================================

# Single timestamp
ts = pd.Timestamp('2024-01-15')
ts = pd.Timestamp('2024-01-15 10:30:00')
ts = pd.Timestamp(year=2024, month=1, day=15, hour=10, minute=30)

# Accessing components
print(ts.year)         # 2024
print(ts.month)        # 1
print(ts.day)          # 15
print(ts.hour)         # 10
print(ts.dayofweek)    # 0 = Monday ... 6 = Sunday
print(ts.day_name())   # 'Monday'
print(ts.quarter)      # 1
print(ts.is_leap_year) # True
print(ts.dayofyear)    # 15

# =============================================
# PARSING DATES IN DATAFRAMES
# =============================================

# Method 1: Parse during read
# df = pd.read_csv('data.csv', parse_dates=['date_column'])

# Method 2: Convert existing column
df = pd.DataFrame({
    'date_str': ['2024-01-15', '2024-02-20', '2024-03-10', '2024-04-05'],
    'value': [100, 150, 130, 170]
})

df['date'] = pd.to_datetime(df['date_str'])

# Method 3: Handle messy date formats
messy_dates = pd.Series([
    '01/15/2024',       # US format
    '15-Jan-2024',      # abbreviated month
    'January 20, 2024', # full month name
    '2024.03.10'        # dot-separated
])

# format='mixed' handles multiple formats (slower but flexible)
parsed = pd.to_datetime(messy_dates, format='mixed')

# Specify exact format for speed (10-50x faster on large datasets)
dates_us = pd.to_datetime(pd.Series(['01/15/2024', '02/20/2024']), format='%m/%d/%Y')

# Handle ambiguous dates
# Is '01/02/2024' January 2nd or February 1st?
pd.to_datetime('01/02/2024', dayfirst=False)  # Jan 2 (US default)
pd.to_datetime('01/02/2024', dayfirst=True)   # Feb 1 (European)

# Handle Unix timestamps (seconds since 1970-01-01)
unix_ts = pd.Series([1704067200, 1704153600, 1704240000])
dates_from_unix = pd.to_datetime(unix_ts, unit='s')

# Millisecond timestamps (common in JavaScript/APIs)
ms_ts = pd.Series([1704067200000, 1704153600000])
dates_from_ms = pd.to_datetime(ms_ts, unit='ms')

# Handle errors gracefully
bad_dates = pd.Series(['2024-01-15', 'not_a_date', '2024-03-10'])
safe_parse = pd.to_datetime(bad_dates, errors='coerce')  # bad → NaT

# =============================================
# EXTRACTING DATE COMPONENTS
# =============================================

df['date'] = pd.to_datetime(df['date_str'])

df['year'] = df['date'].dt.year
df['month'] = df['date'].dt.month
df['day'] = df['date'].dt.day
df['weekday'] = df['date'].dt.day_name()    # 'Monday', 'Tuesday', ...
df['week_number'] = df['date'].dt.isocalendar().week  # ISO week number
df['quarter'] = df['date'].dt.quarter
df['is_weekend'] = df['date'].dt.dayofweek >= 5  # Saturday=5, Sunday=6
df['month_name'] = df['date'].dt.month_name()

# =============================================
# DATE FORMATTING (Timestamp → String)
# =============================================

df['formatted'] = df['date'].dt.strftime('%B %d, %Y')   # 'January 15, 2024'
df['iso_format'] = df['date'].dt.strftime('%Y-%m-%dT%H:%M:%S')

# Common format codes:
# %Y = 2024   %y = 24   %m = 01   %d = 15
# %H = 14     %M = 30   %S = 00   %B = January
# %b = Jan    %A = Monday   %a = Mon
```

> **Pro Tip**: Always specify `format=` in `pd.to_datetime()` when you know the date format. Without it, Pandas guesses the format for every row, which is 10-50x slower on large datasets.

---

## DatetimeIndex and Time-Based Indexing

### What It Is
A DatetimeIndex is when you use datetime values as the row index of your DataFrame. This unlocks powerful time-based operations: slicing by date range, resampling, time-zone conversion. Think of it like organizing your filing cabinet by date instead of by name.

### Why It Matters
- Enables intuitive slicing: `df['2024-01':'2024-06']` for first half of year
- Required for `resample()`, one of the most useful time series methods
- Makes time-based joins and alignment automatic
- Industry standard for financial and scientific time series

### Code Examples

```python
import pandas as pd
import numpy as np

# Create time series with DatetimeIndex
dates = pd.date_range('2024-01-01', periods=365, freq='D')
np.random.seed(42)
df = pd.DataFrame({
    'sales': np.random.randint(100, 1000, 365),
    'visitors': np.random.randint(500, 5000, 365)
}, index=dates)

# Or set existing column as index
# df = df.set_index('date')

# =============================================
# TIME-BASED SLICING (incredibly powerful!)
# =============================================

# Select a specific date
day_data = df.loc['2024-03-15']

# Select an entire month
march_data = df.loc['2024-03']

# Select a quarter
q1_data = df.loc['2024-01':'2024-03']

# Select a year
year_data = df.loc['2024']

# Between two specific dates
subset = df.loc['2024-06-01':'2024-06-30']

# Using .between_time() for intraday data
hourly_df = pd.DataFrame(
    {'price': np.random.randn(24)},
    index=pd.date_range('2024-01-01', periods=24, freq='h')
)
# Get only business hours (9 AM to 5 PM)
business_hours = hourly_df.between_time('09:00', '17:00')

# =============================================
# FILTERING BY DATE COMPONENTS
# =============================================

# Filter weekdays only
weekdays = df[df.index.dayofweek < 5]

# Filter specific months
summer = df[df.index.month.isin([6, 7, 8])]

# Filter Mondays
mondays = df[df.index.day_name() == 'Monday']

# Last day of each month
month_ends = df[df.index.is_month_end]

# First business day of each month
df_biz = df[df.index.dayofweek < 5]  # weekdays only
first_biz_days = df_biz.groupby(df_biz.index.to_period('M')).first()

# =============================================
# DATE ARITHMETIC
# =============================================

# Add/subtract time
df.index + pd.Timedelta(days=7)          # shift 7 days forward
df.index - pd.DateOffset(months=1)       # shift 1 month back

# Difference between dates
delta = pd.Timestamp('2024-12-31') - pd.Timestamp('2024-01-01')
print(delta.days)  # 365

# Days since a reference date (useful as feature)
df['days_since_start'] = (df.index - df.index[0]).days
```

---

## Date Ranges and Frequencies

### What It Is
`pd.date_range()` generates sequences of evenly-spaced dates. Frequency strings tell Pandas how far apart each date should be. Think of it like a metronome for time — you set the tempo (frequency) and it ticks at regular intervals.

### Why It Matters
- Creating time indices for synthetic data or simulation
- Filling gaps in irregular time series
- Defining bins for resampling
- Generating business calendars (skip weekends/holidays)

### Code Examples

```python
import pandas as pd

# =============================================
# BASIC DATE RANGES
# =============================================

# Daily dates
daily = pd.date_range(start='2024-01-01', end='2024-01-31', freq='D')

# Fixed number of periods
ten_days = pd.date_range(start='2024-01-01', periods=10, freq='D')

# Business days (skip weekends)
business = pd.date_range(start='2024-01-01', periods=20, freq='B')

# Hourly
hourly = pd.date_range(start='2024-01-01', periods=48, freq='h')

# Every 15 minutes
minutes = pd.date_range(start='2024-01-01 09:00', periods=32, freq='15min')

# Monthly (start of month)
months = pd.date_range(start='2024-01-01', periods=12, freq='MS')  # Month Start

# Monthly (end of month)
month_ends = pd.date_range(start='2024-01-01', periods=12, freq='ME')  # Month End

# Quarterly
quarters = pd.date_range(start='2024-01-01', periods=4, freq='QS')  # Quarter Start

# Yearly
years = pd.date_range(start='2020-01-01', periods=5, freq='YS')  # Year Start

# Custom: every 2 weeks
biweekly = pd.date_range(start='2024-01-01', periods=26, freq='2W')

# =============================================
# FREQUENCY ALIASES (most important ones)
# =============================================
# 
# 'D'    = Calendar day          'B'    = Business day
# 'W'    = Weekly (Sunday)       'W-MON'= Weekly (Monday)
# 'MS'   = Month start           'ME'   = Month end
# 'QS'   = Quarter start         'QE'   = Quarter end
# 'YS'   = Year start            'YE'   = Year end
# 'h'    = Hourly                'min'  = Minutely
# 's'    = Secondly              'BMS'  = Business month start
# 'BME'  = Business month end    'BQS'  = Business quarter start
#
# Combine with multiplier: '2h' = every 2 hours, '15min' = every 15 minutes

# =============================================
# REINDEXING TO FILL GAPS
# =============================================

# Irregular time series (gaps in data)
irregular = pd.Series(
    [100, 150, 200],
    index=pd.to_datetime(['2024-01-01', '2024-01-05', '2024-01-10'])
)

# Create complete daily index
full_index = pd.date_range('2024-01-01', '2024-01-10', freq='D')

# Reindex to fill gaps (NaN for missing dates)
regular = irregular.reindex(full_index)

# Fill the gaps
regular_filled = regular.interpolate(method='linear')  # interpolate
# Or: regular.ffill()  # forward fill
# Or: regular.fillna(0)  # fill with zero
```

---

## Resampling — Changing Time Frequency

### What It Is
Resampling changes the frequency of your time series data. **Downsampling** reduces frequency (daily → monthly), requiring aggregation. **Upsampling** increases frequency (monthly → daily), requiring interpolation or filling. Think of it like zooming in/out on a timeline.

### Why It Matters
- Converting raw tick data to OHLC candles (finance)
- Aggregating sensor readings from per-second to per-hour
- Aligning two datasets with different time frequencies
- Creating features at different time scales for ML models
- Equivalent to SQL's `GROUP BY date_trunc(...)` but more powerful

### How It Works

```
DOWNSAMPLING (Daily → Monthly):
Day 1:  100 ─┐
Day 2:  110  │
Day 3:  105  ├──→ January: mean = 108.3
Day 4:   95  │                sum = 650
Day 5:  120  │                max = 120
Day 6:  120 ─┘

UPSAMPLING (Monthly → Daily):
Jan: 100 ──→ Jan 1: 100
              Jan 2: ???  (fill with ffill, interpolate, or NaN)
              Jan 3: ???
              ...
Feb: 150 ──→ Feb 1: 150
```

### Code Examples

```python
import pandas as pd
import numpy as np

# Create daily data for 2 years
np.random.seed(42)
dates = pd.date_range('2023-01-01', '2024-12-31', freq='D')
df = pd.DataFrame({
    'sales': np.random.randint(50, 500, len(dates)),
    'visitors': np.random.randint(200, 2000, len(dates)),
    'revenue': np.random.uniform(1000, 10000, len(dates))
}, index=dates)

# =============================================
# DOWNSAMPLING (higher freq → lower freq)
# =============================================

# Daily → Monthly (sum)
monthly_sales = df['sales'].resample('ME').sum()

# Daily → Weekly (mean)
weekly_avg = df['sales'].resample('W').mean()

# Daily → Quarterly (multiple aggregations)
quarterly = df.resample('QE').agg({
    'sales': 'sum',
    'visitors': 'mean',
    'revenue': ['sum', 'mean', 'max']
})

# Daily → Monthly with OHLC (Open-High-Low-Close) — common in finance
ohlc = df['revenue'].resample('ME').ohlc()
# Columns: open (first), high (max), low (min), close (last)

# Custom aggregation
monthly_custom = df.resample('ME').agg(
    total_sales=('sales', 'sum'),
    peak_visitors=('visitors', 'max'),
    avg_revenue=('revenue', 'mean'),
    days_active=('sales', 'count')
)

# =============================================
# UPSAMPLING (lower freq → higher freq)
# =============================================

# Monthly data
monthly = pd.Series(
    [100, 150, 200, 180],
    index=pd.date_range('2024-01-01', periods=4, freq='MS')
)

# Upsample to daily
daily = monthly.resample('D').asfreq()  # NaN for new dates
# Output:
# 2024-01-01    100.0
# 2024-01-02      NaN
# 2024-01-03      NaN
# ...
# 2024-02-01    150.0
# ...

# Fill the gaps
daily_ffill = monthly.resample('D').ffill()          # forward fill
daily_bfill = monthly.resample('D').bfill()          # backward fill
daily_interp = monthly.resample('D').interpolate()    # linear interpolation

# =============================================
# RESAMPLING WITH GROUPER (for non-index datetime columns)
# =============================================

df_with_col = df.reset_index().rename(columns={'index': 'date'})

# Group by month using pd.Grouper (date is a column, not index)
monthly_grouped = df_with_col.groupby(
    pd.Grouper(key='date', freq='ME')
)['sales'].sum()

# Multiple groupers: month + another column
df_with_col['product'] = np.random.choice(['A', 'B', 'C'], len(df_with_col))

product_monthly = df_with_col.groupby([
    pd.Grouper(key='date', freq='ME'),
    'product'
])['sales'].sum().unstack(fill_value=0)

# =============================================
# PRACTICAL EXAMPLE: Year-over-Year Comparison
# =============================================

# Monthly totals
monthly_sales = df['sales'].resample('ME').sum()

# Create year-month for pivoting
yoy = pd.DataFrame({
    'year': monthly_sales.index.year,
    'month': monthly_sales.index.month,
    'sales': monthly_sales.values
})

yoy_pivot = yoy.pivot(index='month', columns='year', values='sales')
yoy_pivot['growth'] = (yoy_pivot[2024] - yoy_pivot[2023]) / yoy_pivot[2023] * 100
print(yoy_pivot)
```

> **Pro Tip**: `resample()` is just GroupBy with a time-based grouper. Everything you can do with GroupBy (`.agg()`, `.transform()`, `.apply()`) works with `resample()` too.

---

## Rolling and Expanding Windows

### What It Is
Rolling windows compute a function over a fixed-size sliding window of data. Expanding windows compute over all data from the start up to the current point. Think of rolling as looking through a fixed-width magnifying glass that slides along your data; expanding is like a growing snowball.

### Why It Matters
- **Moving averages** smooth noisy data and reveal trends
- **Rolling standard deviation** measures local volatility
- **Bollinger Bands** (rolling mean ± 2×rolling std) are used in trading
- Feature engineering for ML: rolling statistics capture temporal patterns
- Anomaly detection: flag values far from their rolling mean

### How It Works

```
Data:    [10, 20, 30, 40, 50, 60]

Rolling Mean (window=3):
  Step 1: [10, 20, 30] → mean = 20     → NaN, NaN, 20
  Step 2: [20, 30, 40] → mean = 30     → 30
  Step 3: [30, 40, 50] → mean = 40     → 40
  Step 4: [40, 50, 60] → mean = 50     → 50

Expanding Mean:
  Step 1: [10]                   → mean = 10
  Step 2: [10, 20]               → mean = 15
  Step 3: [10, 20, 30]           → mean = 20
  Step 4: [10, 20, 30, 40]      → mean = 25
  Step 5: [10, 20, 30, 40, 50]  → mean = 30
  Step 6: [10, 20, 30, 40, 50, 60] → mean = 35
```

### Code Examples

```python
import pandas as pd
import numpy as np

# Simulated stock price data
np.random.seed(42)
dates = pd.date_range('2024-01-01', periods=252, freq='B')  # ~1 year business days
price = 100 + np.random.randn(252).cumsum()  # random walk starting at 100

df = pd.DataFrame({'price': price}, index=dates)

# =============================================
# ROLLING WINDOWS
# =============================================

# Simple Moving Average (SMA)
df['SMA_20'] = df['price'].rolling(window=20).mean()   # 20-day MA
df['SMA_50'] = df['price'].rolling(window=50).mean()   # 50-day MA

# Rolling standard deviation (volatility)
df['volatility'] = df['price'].rolling(window=20).std()

# Bollinger Bands (price channel: mean ± 2*std)
df['BB_upper'] = df['SMA_20'] + 2 * df['volatility']
df['BB_lower'] = df['SMA_20'] - 2 * df['volatility']

# Rolling min and max (support and resistance levels)
df['rolling_high'] = df['price'].rolling(window=20).max()
df['rolling_low'] = df['price'].rolling(window=20).min()

# Rolling sum (e.g., 7-day total sales)
df['rolling_volume'] = df['price'].rolling(window=7).sum()

# --- min_periods: allow partial windows at the start ---
# By default, rolling(20) returns NaN for first 19 rows
# min_periods lets you compute with fewer observations
df['SMA_20_partial'] = df['price'].rolling(window=20, min_periods=1).mean()

# --- Centered window ---
# Look both forward and backward (useful for smoothing, NOT prediction)
df['SMA_centered'] = df['price'].rolling(window=11, center=True).mean()

# --- Time-based windows (instead of row count) ---
# Useful for irregular time series (not every day has data)
df['SMA_30D'] = df['price'].rolling(window='30D').mean()  # 30 calendar days

# =============================================
# EXPONENTIALLY WEIGHTED MOVING AVERAGE (EWM)
# =============================================
# Recent observations get exponentially more weight

# span = "effective window size" (higher = smoother)
df['EMA_20'] = df['price'].ewm(span=20).mean()

# EWM formula: weight of observation k days ago = (1 - α)^k
# where α = 2 / (span + 1)
# For span=20: α = 2/21 ≈ 0.095

# Comparison: SMA treats all points equally, EMA reacts faster to recent changes

# =============================================
# EXPANDING WINDOWS (cumulative)
# =============================================

# Cumulative mean (running average)
df['cum_mean'] = df['price'].expanding().mean()

# Cumulative max (all-time high)
df['all_time_high'] = df['price'].expanding().max()

# Drawdown: distance from all-time high (used in finance)
df['drawdown'] = (df['price'] - df['all_time_high']) / df['all_time_high']

# Cumulative sum
df['cum_return'] = df['price'].pct_change().expanding().sum()

# =============================================
# ROLLING WITH CUSTOM FUNCTIONS
# =============================================

# Rolling skewness
df['rolling_skew'] = df['price'].rolling(20).skew()

# Rolling correlation between two series
df['price2'] = 100 + np.random.randn(252).cumsum()
df['rolling_corr'] = df['price'].rolling(60).corr(df['price2'])

# Custom function: rolling max drawdown
def max_drawdown(window):
    """Maximum peak-to-trough decline in a window."""
    peak = window.expanding().max()
    drawdown = (window - peak) / peak
    return drawdown.min()

df['rolling_maxdd'] = df['price'].rolling(60).apply(max_drawdown, raw=False)

# =============================================
# MULTIPLE ROLLING STATISTICS AT ONCE
# =============================================

rolling = df['price'].rolling(20)
df['r_mean'] = rolling.mean()
df['r_std'] = rolling.std()
df['r_min'] = rolling.min()
df['r_max'] = rolling.max()
df['r_median'] = rolling.median()

# Or with .agg() (returns multiple columns)
stats = df['price'].rolling(20).agg(['mean', 'std', 'min', 'max'])
```

> **Pro Tip**: For large datasets, use `engine='numba'` in `.apply()` for 10-100x speedup: `df['col'].rolling(20).apply(func, engine='numba', raw=True)`. The function must work with raw NumPy arrays.

---

## Shift, Lag, and Percent Change

### What It Is
Shift moves your data forward or backward in time, creating lagged (past) or lead (future) versions of a column. Percent change computes the relative difference between consecutive values. Think of shift like scrolling a column up or down while the index stays in place.

### Why It Matters
- **Lag features** are the #1 most important feature type in time series ML
- **Returns calculation**: today's return = (today - yesterday) / yesterday
- **Detecting change**: compare current value to previous period
- **Avoiding data leakage**: shift ensures you only use past data for prediction

### How It Works

```
Original:    Shift(1):      Shift(-1):     pct_change():
┌──────┐    ┌──────┐       ┌──────┐       ┌──────────┐
│ 100  │    │ NaN  │       │ 110  │       │   NaN    │
│ 110  │    │ 100  │       │ 105  │       │  0.100   │  (+10%)
│ 105  │    │ 110  │       │ 120  │       │ -0.045   │  (-4.5%)
│ 120  │    │ 105  │       │ NaN  │       │  0.143   │  (+14.3%)
└──────┘    └──────┘       └──────┘       └──────────┘

Shift(1):  each value becomes "yesterday's value"
Shift(-1): each value becomes "tomorrow's value"
```

### Code Examples

```python
import pandas as pd
import numpy as np

np.random.seed(42)
dates = pd.date_range('2024-01-01', periods=30, freq='D')
df = pd.DataFrame({
    'price': 100 + np.random.randn(30).cumsum(),
    'volume': np.random.randint(1000, 5000, 30)
}, index=dates)

# =============================================
# SHIFT / LAG
# =============================================

# Lag: previous day's value
df['price_lag1'] = df['price'].shift(1)    # yesterday's price
df['price_lag7'] = df['price'].shift(7)    # price 7 days ago

# Lead: next day's value (careful — this is "future" data!)
df['price_lead1'] = df['price'].shift(-1)  # tomorrow's price

# Difference from previous value
df['price_diff'] = df['price'].diff()       # equivalent to price - price.shift(1)
df['price_diff7'] = df['price'].diff(7)     # difference from 7 days ago

# =============================================
# PERCENT CHANGE (RETURNS)
# =============================================

# Daily return
df['daily_return'] = df['price'].pct_change()
# Formula: (price[t] - price[t-1]) / price[t-1]

# Weekly return
df['weekly_return'] = df['price'].pct_change(periods=7)

# Log returns (preferred in finance — additive over time)
df['log_return'] = np.log(df['price'] / df['price'].shift(1))
# Equivalent: np.log1p(df['price'].pct_change())

# Cumulative return
df['cumulative_return'] = (1 + df['daily_return']).cumprod() - 1

# =============================================
# FEATURE ENGINEERING FOR ML
# =============================================

# Create multiple lag features
for lag in [1, 2, 3, 7, 14, 30]:
    df[f'price_lag_{lag}'] = df['price'].shift(lag)

# Rolling statistics as features
df['rolling_mean_7'] = df['price'].rolling(7).mean()
df['rolling_std_7'] = df['price'].rolling(7).std()

# Price relative to rolling mean (mean reversion indicator)
df['price_vs_ma'] = df['price'] / df['rolling_mean_7']

# Volume change
df['volume_change'] = df['volume'].pct_change()

# Momentum (rate of change over N periods)
df['momentum_7'] = df['price'] / df['price'].shift(7) - 1

# Volatility of returns
df['return_vol_7'] = df['daily_return'].rolling(7).std()

# =============================================
# AVOIDING DATA LEAKAGE WITH SHIFT
# =============================================

# Target: next day's price (what we want to predict)
df['target'] = df['price'].shift(-1)  # shift UP = future value

# Features: only use past data
df['feature_lag1'] = df['price'].shift(1)   # yesterday
df['feature_lag2'] = df['price'].shift(2)   # 2 days ago
df['feature_ma5'] = df['price'].shift(1).rolling(5).mean()  # MA of past 5 days (shifted!)

# WRONG: this MA includes today's price — data leakage!
# df['feature_ma5_wrong'] = df['price'].rolling(5).mean()

# Drop rows with NaN (from shifting/rolling)
df_clean = df.dropna()
```

> **Pro Tip**: When creating features for time series prediction, ALWAYS shift your rolling calculations by 1: `.shift(1).rolling(N).mean()`. Without the shift, the rolling window includes the current timestep, causing data leakage.

---

## Time Zones

### What It Is
Time zone handling converts timestamps between different time zones and makes your datetime data "timezone-aware." Think of it like a traveler's watch that can show any city's time.

### Why It Matters
- Global datasets have timestamps from different time zones
- APIs often return UTC; you need to convert to local time
- Daylight Saving Time (DST) creates unexpected gaps and overlaps
- Getting time zones wrong can shift your data by hours, ruining analysis

### Code Examples

```python
import pandas as pd

# =============================================
# MAKING TIMESTAMPS TIMEZONE-AWARE
# =============================================

# Localize: tell Pandas "this naive timestamp is in this timezone"
ts = pd.Timestamp('2024-06-15 14:00:00')
ts_utc = ts.tz_localize('UTC')
ts_nyc = ts.tz_localize('America/New_York')

# Convert between timezones
ts_london = ts_utc.tz_convert('Europe/London')
ts_tokyo = ts_utc.tz_convert('Asia/Tokyo')

print(f"UTC:    {ts_utc}")      # 2024-06-15 14:00:00+00:00
print(f"London: {ts_london}")   # 2024-06-15 15:00:00+01:00  (BST = UTC+1)
print(f"Tokyo:  {ts_tokyo}")    # 2024-06-15 23:00:00+09:00  (JST = UTC+9)

# =============================================
# TIMEZONE WITH SERIES / DATAFRAME
# =============================================

dates = pd.date_range('2024-01-01', periods=5, freq='D', tz='UTC')
df = pd.DataFrame({'value': range(5)}, index=dates)

# Convert index to another timezone
df_nyc = df.tz_convert('America/New_York')

# For a column (not index)
df2 = pd.DataFrame({
    'timestamp': pd.date_range('2024-01-01', periods=5, freq='D', tz='UTC'),
    'value': range(5)
})
df2['timestamp_est'] = df2['timestamp'].dt.tz_convert('US/Eastern')

# =============================================
# HANDLING DST (DAYLIGHT SAVING TIME)
# =============================================

# DST transition: clocks spring forward (1 hour gap)
# March 10, 2024 at 2:00 AM EST → 3:00 AM EDT
spring_forward = pd.date_range('2024-03-10', periods=24, freq='h', tz='US/Eastern')
# Notice: 2:00 AM doesn't exist on this day!

# DST transition: clocks fall back (1 hour repeated)
# November 3, 2024 at 2:00 AM EDT → 1:00 AM EST  
# 1:00 AM happens TWICE

# Handle ambiguous times during fall-back
ts = pd.Timestamp('2024-11-03 01:30:00')
# ts.tz_localize('US/Eastern')  # AmbiguousTimeError!
ts_dst = ts.tz_localize('US/Eastern', ambiguous=True)   # assume DST
ts_std = ts.tz_localize('US/Eastern', ambiguous=False)  # assume standard time

# =============================================
# BEST PRACTICE: STORE IN UTC, DISPLAY IN LOCAL
# =============================================

# When ingesting data from multiple timezones:
# 1. Convert everything to UTC immediately
# 2. Store/compute in UTC
# 3. Convert to local time only for display

events = pd.DataFrame({
    'event': ['Meeting NYC', 'Call London', 'Demo Tokyo'],
    'local_time': ['2024-06-15 09:00', '2024-06-15 14:00', '2024-06-16 10:00'],
    'timezone': ['America/New_York', 'Europe/London', 'Asia/Tokyo']
})

# Convert each to UTC
def to_utc(row):
    local = pd.Timestamp(row['local_time']).tz_localize(row['timezone'])
    return local.tz_convert('UTC')

events['utc_time'] = events.apply(to_utc, axis=1)
```

---

## Period and Timedelta

### What It Is
- **Period**: Represents a span of time (e.g., "January 2024", "Q1 2024", "the year 2024"). Unlike a Timestamp (a point), a Period has a start and end.
- **Timedelta**: Represents a duration — the difference between two timestamps. Like saying "5 days, 3 hours" without specifying when.

### Code Examples

```python
import pandas as pd

# =============================================
# PERIODS
# =============================================

# Create a Period
p = pd.Period('2024-01', freq='M')       # January 2024
print(p.start_time)                       # 2024-01-01
print(p.end_time)                         # 2024-01-31 23:59:59.999999999

# Period arithmetic
p + 1  # February 2024
p - 3  # October 2023

# PeriodIndex
periods = pd.period_range('2024-01', periods=12, freq='M')
df = pd.DataFrame({'sales': range(12)}, index=periods)

# Convert between Timestamp and Period
ts_index = pd.date_range('2024-01-01', periods=12, freq='MS')
period_index = ts_index.to_period('M')  # Timestamp → Period
back_to_ts = period_index.to_timestamp()  # Period → Timestamp

# =============================================
# TIMEDELTAS
# =============================================

# Create Timedeltas
td = pd.Timedelta(days=5, hours=3, minutes=30)
td = pd.Timedelta('5 days 3 hours 30 minutes')
td = pd.Timedelta('1W')  # 1 week

# Timedelta arithmetic
ts = pd.Timestamp('2024-06-15 10:00:00')
print(ts + pd.Timedelta(hours=8))   # 2024-06-15 18:00:00
print(ts - pd.Timedelta(days=30))   # 2024-05-16 10:00:00

# Difference between timestamps
delta = pd.Timestamp('2024-12-31') - pd.Timestamp('2024-01-01')
print(delta)            # 365 days
print(delta.days)       # 365
print(delta.seconds)    # 0
print(delta.total_seconds())  # 31536000.0

# Timedelta in DataFrames
df = pd.DataFrame({
    'order_date': pd.to_datetime(['2024-01-01', '2024-01-05', '2024-01-10']),
    'ship_date': pd.to_datetime(['2024-01-03', '2024-01-08', '2024-01-12'])
})

df['delivery_time'] = df['ship_date'] - df['order_date']
# Output: [2 days, 3 days, 2 days]

df['delivery_days'] = df['delivery_time'].dt.days
# Output: [2, 3, 2]

# Average delivery time
avg_delivery = df['delivery_time'].mean()
print(avg_delivery)  # 2 days 08:00:00
```

---

## Real-World Time Series Patterns

### Complete Example: Sales Analysis Pipeline

```python
import pandas as pd
import numpy as np

# Simulate 3 years of daily sales data
np.random.seed(42)
dates = pd.date_range('2022-01-01', '2024-12-31', freq='D')
n = len(dates)

# Create realistic seasonal pattern
base = 500
trend = np.linspace(0, 200, n)  # upward trend
seasonality = 100 * np.sin(2 * np.pi * np.arange(n) / 365)  # yearly cycle
weekly = 50 * np.sin(2 * np.pi * np.arange(n) / 7)  # weekly cycle
noise = np.random.normal(0, 50, n)

sales = pd.Series(
    (base + trend + seasonality + weekly + noise).clip(min=0),
    index=dates,
    name='sales'
)

# =============================================
# DECOMPOSE THE TIME SERIES
# =============================================

# Trend: 30-day moving average
trend_line = sales.rolling(30, center=True).mean()

# Seasonality: monthly averages
monthly_pattern = sales.groupby(sales.index.month).transform('mean')

# Year-over-Year growth
yoy_growth = sales.resample('ME').sum().pct_change(12)

# =============================================
# BUSINESS METRICS
# =============================================

# Month-to-Date (MTD) totals
df = sales.to_frame()
df['month'] = df.index.to_period('M')
df['mtd'] = df.groupby('month')['sales'].cumsum()

# Same day last year
df['sales_last_year'] = df['sales'].shift(365)
df['yoy_change'] = (df['sales'] - df['sales_last_year']) / df['sales_last_year']

# 7-day moving average (smoothed daily metric)
df['sales_7dma'] = df['sales'].rolling(7).mean()

# Day of week analysis
dow_analysis = df.groupby(df.index.day_name())['sales'].agg(['mean', 'std', 'count'])
dow_analysis = dow_analysis.reindex(['Monday', 'Tuesday', 'Wednesday', 
                                      'Thursday', 'Friday', 'Saturday', 'Sunday'])

# =============================================
# ANOMALY DETECTION WITH ROLLING STATS
# =============================================

rolling_mean = df['sales'].rolling(30).mean()
rolling_std = df['sales'].rolling(30).std()

# Z-score relative to rolling window
df['z_score'] = (df['sales'] - rolling_mean) / rolling_std

# Flag anomalies (|z| > 3)
df['is_anomaly'] = df['z_score'].abs() > 3
anomalies = df[df['is_anomaly']]
print(f"Anomalies detected: {len(anomalies)}")
```

---

## Common Mistakes

### 1. Not Specifying Date Format (Slow Parsing)
```python
# SLOW — Pandas guesses format for each row
pd.to_datetime(df['date_str'])  # tries many formats per element

# FAST — explicit format (10-50x faster)
pd.to_datetime(df['date_str'], format='%Y-%m-%d')
```

### 2. Mixing Timezone-Aware and Naive Datetimes
```python
# CRASHES — can't compare aware with naive
ts_utc = pd.Timestamp('2024-01-01', tz='UTC')
ts_naive = pd.Timestamp('2024-01-01')
# ts_utc > ts_naive  # TypeError!

# Fix: localize the naive one first
ts_naive_aware = ts_naive.tz_localize('UTC')
```

### 3. Data Leakage in Rolling Features
```python
# WRONG — rolling mean includes current value
df['feature'] = df['price'].rolling(7).mean()  # includes today!

# RIGHT — shift to use only past data
df['feature'] = df['price'].shift(1).rolling(7).mean()
```

### 4. Ignoring Business Days
```python
# WRONG — assuming 7-day shift = 1 week of trading data
df['price_last_week'] = df['price'].shift(7)

# RIGHT — use business day offset or count actual trading days
df['price_last_week'] = df['price'].shift(5)  # 5 business days = 1 week
```

### 5. Forgetting to Sort Before Time Operations
```python
# If data isn't sorted by time, rolling/shift give wrong results
df = df.sort_index()  # always sort by time first!
```

### 6. Resampling Without Setting DatetimeIndex
```python
# WRONG
# df.resample('ME')['sales'].sum()  # TypeError if index isn't DatetimeIndex

# RIGHT — use Grouper for non-index datetime columns
df.groupby(pd.Grouper(key='date', freq='ME'))['sales'].sum()
# Or set_index first: df.set_index('date').resample('ME')['sales'].sum()
```

---

## Interview Questions

### Q1: What's the difference between `resample()` and `groupby()` for time series?
**Answer**: `resample()` is a convenience method specifically for time-based grouping on a DatetimeIndex. It's equivalent to `groupby(pd.Grouper(freq=...))`. `resample()` is more concise and supports upsampling (with `.asfreq()`, `.ffill()`, `.interpolate()`), while raw `groupby` doesn't.

### Q2: Explain the difference between SMA and EMA. When would you use each?
**Answer**:
- **SMA (Simple Moving Average)**: Equal weight to all observations in the window. Better for identifying long-term trends and support/resistance levels.
- **EMA (Exponential Moving Average)**: Exponentially decaying weights — recent values matter more. Better for reactive signals, faster response to new information.
- Use SMA when you want a smoother, less reactive indicator. Use EMA when you need faster responsiveness to recent changes.
- Formula: $EMA_t = \alpha \cdot x_t + (1-\alpha) \cdot EMA_{t-1}$, where $\alpha = \frac{2}{span + 1}$

### Q3: How do you prevent data leakage in time series feature engineering?
**Answer**: Always use `.shift(1)` before computing rolling features so the window doesn't include the current timestep. Never use future data (negative shifts) in features. Use time-based train/test splits (not random splits). Ensure any aggregation (mean, std) only uses past data.

### Q4: How would you handle missing timestamps in an irregular time series?
**Answer**:
```python
# 1. Create a complete time index
full_idx = pd.date_range(df.index.min(), df.index.max(), freq='D')
# 2. Reindex to fill gaps with NaN
df = df.reindex(full_idx)
# 3. Choose a fill strategy
df = df.interpolate(method='time')  # or ffill, bfill
```

### Q5: What is the difference between `shift()` and `tshift()`/`DateOffset`?
**Answer**: `shift(n)` moves the data values up/down by n rows (index stays fixed). Using a `DateOffset` or `index + Timedelta` moves the index labels while data stays in place. `tshift()` was deprecated — use `df.index = df.index + pd.DateOffset(...)` instead.

### Q6: How do you handle time series data across different time zones?
**Answer**: Convert everything to UTC on ingestion using `tz_localize()` → `tz_convert('UTC')`. Store and compute in UTC. Convert to local time only for display or reporting. This avoids DST issues and ensures consistent comparisons.

---

## Quick Reference

| Operation | Code | Notes |
|-----------|------|-------|
| Parse dates | `pd.to_datetime(s, format='%Y-%m-%d')` | Specify format for speed |
| Date range | `pd.date_range(start, periods, freq)` | 'D', 'B', 'ME', 'h', etc. |
| Set datetime index | `df.set_index('date')` | Enables time slicing |
| Slice by date | `df.loc['2024-03']` | Year, month, or exact date |
| Extract year/month | `df['date'].dt.year` | `.dt` accessor |
| Day of week | `df['date'].dt.day_name()` | Monday, Tuesday, ... |
| Resample (down) | `df.resample('ME').sum()` | Daily → Monthly |
| Resample (up) | `df.resample('D').ffill()` | Monthly → Daily |
| Rolling mean | `df['col'].rolling(7).mean()` | 7-period moving avg |
| EWM | `df['col'].ewm(span=20).mean()` | Exponential weighting |
| Expanding | `df['col'].expanding().max()` | Cumulative max |
| Shift (lag) | `df['col'].shift(1)` | Previous row |
| Pct change | `df['col'].pct_change()` | Return from previous |
| Diff | `df['col'].diff()` | Absolute change |
| Localize TZ | `s.dt.tz_localize('UTC')` | Assign timezone |
| Convert TZ | `s.dt.tz_convert('US/Eastern')` | Change timezone |
| Period | `pd.Period('2024-01', freq='M')` | Time span |
| Timedelta | `pd.Timedelta(days=5)` | Duration |
| OHLC | `df.resample('ME').ohlc()` | Open/High/Low/Close |

---

*Previous: [06-Pandas-Data-Cleaning](06-Pandas-Data-Cleaning.md) | Next: [08-Pandas-Advanced-and-Performance](08-Pandas-Advanced-and-Performance.md)*
