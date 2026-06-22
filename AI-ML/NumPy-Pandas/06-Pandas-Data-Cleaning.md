# Chapter 06: Pandas Data Cleaning

## Table of Contents
- [Missing Values — Detection and Handling](#missing-values--detection-and-handling)
- [Duplicates — Detection and Removal](#duplicates--detection-and-removal)
- [String Methods](#string-methods)
- [Type Conversion and Casting](#type-conversion-and-casting)
- [Outlier Detection and Treatment](#outlier-detection-and-treatment)
- [Data Validation and Consistency](#data-validation-and-consistency)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Missing Values — Detection and Handling

### What It Is
Missing values are gaps in your data — cells where information is absent. In Pandas, these are represented as `NaN` (Not a Number), `None`, or `pd.NaT` (for datetime). Think of it like a survey where someone skipped a question — the answer exists in reality, but you don't have it.

### Why It Matters
- **Most ML algorithms can't handle NaN** — your model will crash or silently produce garbage
- Real-world data is ALWAYS messy: sensor failures, optional fields, merge mismatches
- How you handle missing data can change your model's accuracy by 5-20%
- Interviews test this constantly: "How would you handle missing values in this dataset?"

### How It Works

Pandas uses **sentinel values** to mark missing data:

```
┌─────────────────────────────────────────────────────┐
│  Type           │  Missing Sentinel   │  Check With │
├─────────────────┼─────────────────────┼─────────────┤
│  float64        │  NaN (IEEE 754)     │  isna()     │
│  int64          │  Cannot store NaN!  │  —          │
│  Int64 (nullable)│ pd.NA              │  isna()     │
│  object (str)   │  None or NaN        │  isna()     │
│  datetime64     │  NaT                │  isna()     │
│  boolean        │  Cannot store NaN!  │  —          │
│  Boolean (nullable)│ pd.NA            │  isna()     │
└─────────────────┴─────────────────────┴─────────────┘
```

> **Key Insight**: Regular `int64` columns get promoted to `float64` when NaN is introduced. Use nullable `Int64` (capital I) dtype to avoid this.

### Code Examples

```python
import pandas as pd
import numpy as np

# --- Create data with missing values ---
df = pd.DataFrame({
    'name': ['Alice', 'Bob', None, 'Diana', 'Eve'],
    'age': [25, np.nan, 35, 28, np.nan],
    'salary': [70000, 85000, np.nan, np.nan, 95000],
    'city': ['NYC', 'LA', 'NYC', None, 'LA'],
    'hire_date': pd.to_datetime(['2020-01-01', '2019-06-15', pd.NaT, '2021-03-20', '2018-11-01'])
})

# =============================================
# STEP 1: DETECT MISSING VALUES
# =============================================

# Check for missing values (element-wise)
print(df.isna())       # DataFrame of True/False
print(df.isnull())     # Alias for isna() — identical behavior

# Count missing values per column
print(df.isna().sum())
# name        1
# age         2
# salary      2
# city        1
# hire_date   1

# Percentage of missing values per column
missing_pct = (df.isna().sum() / len(df)) * 100
print(missing_pct)

# Total missing values in entire DataFrame
total_missing = df.isna().sum().sum()

# Rows with ANY missing value
rows_with_missing = df[df.isna().any(axis=1)]

# Columns with missing values
cols_with_missing = df.columns[df.isna().any()].tolist()

# Heatmap-style summary (useful for large datasets)
def missing_summary(df):
    """Generate a comprehensive missing value report."""
    missing = df.isna().sum()
    missing_pct = (missing / len(df)) * 100
    dtypes = df.dtypes
    summary = pd.DataFrame({
        'missing_count': missing,
        'missing_pct': missing_pct.round(2),
        'dtype': dtypes
    })
    return summary[summary['missing_count'] > 0].sort_values('missing_pct', ascending=False)

print(missing_summary(df))

# =============================================
# STEP 2: HANDLE MISSING VALUES
# =============================================

# --- Option 1: DROP rows/columns with missing values ---

# Drop rows where ANY column is NaN
df_dropped = df.dropna()  # most aggressive — loses lots of data

# Drop rows where ALL columns are NaN
df.dropna(how='all')

# Drop rows only if specific columns have NaN
df.dropna(subset=['name', 'age'])

# Drop rows that have more than 2 NaN values
df.dropna(thresh=3)  # keep rows with at least 3 non-NaN values

# Drop columns (instead of rows)
df.dropna(axis=1)  # drops any column with NaN

# --- Option 2: FILL missing values ---

# Fill with a constant
df['city'].fillna('Unknown', inplace=False)
df['age'].fillna(0)  # caution: 0 may be misleading

# Fill with column statistics (most common approach)
df['age'].fillna(df['age'].mean())     # mean imputation
df['age'].fillna(df['age'].median())   # median (better for skewed data)
df['city'].fillna(df['city'].mode()[0])  # mode for categorical

# Forward fill: use the last valid value
df['salary'].ffill()   # propagate last valid observation forward
# [70000, 85000, NaN, NaN, 95000] → [70000, 85000, 85000, 85000, 95000]

# Backward fill: use the next valid value
df['salary'].bfill()   # propagate next valid observation backward
# [70000, 85000, NaN, NaN, 95000] → [70000, 85000, 95000, 95000, 95000]

# Limit the fill (don't propagate forever)
df['salary'].ffill(limit=1)  # fill at most 1 consecutive NaN

# Fill with interpolation (for numerical/time series data)
df['age'].interpolate(method='linear')   # linear interpolation
df['age'].interpolate(method='time')     # time-weighted (needs DatetimeIndex)

# --- Option 3: GROUP-WISE IMPUTATION (most sophisticated) ---
# Fill missing salary with the mean salary of their city
df['salary'] = df.groupby('city')['salary'].transform(
    lambda x: x.fillna(x.mean())
)

# --- Option 4: INDICATOR COLUMN (preserve the "missingness" signal) ---
df['salary_missing'] = df['salary'].isna().astype(int)
df['salary'] = df['salary'].fillna(df['salary'].median())
# Now the model knows which salaries were imputed

# =============================================
# NULLABLE DTYPES (Modern Pandas)
# =============================================

# Use nullable integer type to keep integers with NaN
s = pd.array([1, 2, None, 4], dtype=pd.Int64Dtype())
# Or shorthand:
s = pd.array([1, 2, None, 4], dtype='Int64')  # capital I!

# Nullable boolean
b = pd.array([True, False, None], dtype='boolean')

# Nullable string
st = pd.array(['hello', None, 'world'], dtype='string')

# Convert existing column to nullable
df['age'] = df['age'].astype('Int64')  # NaN becomes pd.NA, dtype stays integer
```

### When to Use Which Strategy

| Strategy | When to Use | When NOT to Use |
|----------|-------------|-----------------|
| Drop rows | < 5% missing, data is MCAR | Lots of missing data, systematic missingness |
| Mean/Median fill | Numerical, roughly symmetric | Skewed data (use median), categorical |
| Mode fill | Categorical columns | Too many categories |
| Forward/Back fill | Time series, sequential data | Random data, large gaps |
| Interpolation | Time series, smooth numeric trends | Categorical, non-sequential |
| Group-wise imputation | Missingness depends on a group variable | Independent missingness |
| ML-based imputation | Complex relationships, large datasets | Small datasets, simple patterns |
| Indicator column | Missingness itself is informative | Missingness is purely random |

> **Pro Tip**: Missing data has three types — **MCAR** (Missing Completely At Random), **MAR** (Missing At Random — depends on observed data), and **MNAR** (Missing Not At Random — depends on unobserved data). The type determines the correct handling strategy. Dropping rows only works reliably for MCAR.

---

## Duplicates — Detection and Removal

### What It Is
Duplicates are rows that appear more than once with identical (or nearly identical) values. Think of it like accidentally scanning the same item twice at checkout — your total will be wrong.

### Why It Matters
- Inflates statistics: mean, sum, count all become incorrect
- Causes data leakage in ML: same record in both train and test sets
- Database imports, web scraping, and file merges commonly create duplicates
- Can waste storage and slow down processing

### Code Examples

```python
import pandas as pd

df = pd.DataFrame({
    'id': [1, 2, 3, 2, 4, 3],
    'name': ['Alice', 'Bob', 'Charlie', 'Bob', 'Diana', 'Charlie'],
    'email': ['a@x.com', 'b@x.com', 'c@x.com', 'b@x.com', 'd@x.com', 'c@y.com'],
    'amount': [100, 200, 300, 200, 400, 350]
})

# --- Detect Duplicates ---

# Check for exact duplicate rows (all columns match)
print(df.duplicated())         # Boolean Series: True for duplicate rows
print(df.duplicated().sum())   # Count of duplicate rows

# Check duplicates on specific columns
print(df.duplicated(subset=['id']))         # rows 3 (id=2) and 5 (id=3)
print(df.duplicated(subset=['name']))       # same

# Keep last occurrence as "original" (first is marked as duplicate)
print(df.duplicated(subset=['id'], keep='last'))

# Mark ALL occurrences (including the first)
print(df.duplicated(subset=['id'], keep=False))

# View the actual duplicate rows
duplicates = df[df.duplicated(subset=['id'], keep=False)]
print(duplicates)

# --- Remove Duplicates ---

# Drop exact duplicate rows (keep first occurrence)
df_clean = df.drop_duplicates()

# Drop duplicates based on specific columns
df_clean = df.drop_duplicates(subset=['id'])          # keep='first' (default)
df_clean = df.drop_duplicates(subset=['id'], keep='last')  # keep last
df_clean = df.drop_duplicates(subset=['id'], keep=False)   # drop ALL duplicates

# --- Fuzzy/Near Duplicates ---
# Names with different casing or extra spaces
messy = pd.DataFrame({
    'name': ['Alice Smith', 'alice smith', 'ALICE SMITH', ' Alice Smith '],
    'score': [90, 92, 88, 91]
})

# Normalize before deduplication
messy['name_clean'] = messy['name'].str.strip().str.lower()
messy_clean = messy.drop_duplicates(subset=['name_clean'])

# --- Aggregating duplicates instead of dropping ---
# Instead of dropping, maybe you want to combine duplicate records
df_aggregated = df.groupby('id').agg({
    'name': 'first',
    'email': 'first',
    'amount': 'sum'   # sum the amounts for duplicate IDs
}).reset_index()
```

> **Pro Tip**: Before dropping duplicates, always ask: "Why do I have duplicates?" If they're from a merge, fix the merge. If they represent real repeated events (e.g., multiple purchases), dropping them loses real data.

---

## String Methods

### What It Is
Pandas provides vectorized string operations through the `.str` accessor on Series. These are like Python's built-in string methods but applied to every element in a column simultaneously. Think of it as broadcasting string operations across thousands of rows at once.

### Why It Matters
- Text data is inherently messy: inconsistent casing, extra spaces, mixed formats
- Feature extraction from text: extract numbers from strings, parse structured text
- Data standardization: cleaning names, addresses, product descriptions
- 80% of real-world data cleaning involves string manipulation

### Code Examples

```python
import pandas as pd
import numpy as np

# Sample messy text data
df = pd.DataFrame({
    'name': ['  Alice Smith  ', 'BOB jones', 'charlie BROWN', 'Diana Prince'],
    'email': ['alice@gmail.com', 'Bob@Yahoo.COM', 'charlie@outlook.com', None],
    'phone': ['(555) 123-4567', '555.234.5678', '555-345-6789', '5554567890'],
    'address': ['123 Main St, NYC, NY 10001', '456 Oak Ave, LA, CA 90001',
                '789 Pine Rd, Chicago, IL 60601', '321 Elm Blvd, Miami, FL 33101'],
    'product_code': ['PROD-001-A', 'PROD-023-B', 'PROD-105-C', 'PROD-042-A']
})

# =============================================
# CASE MANIPULATION
# =============================================
df['name_lower'] = df['name'].str.lower()        # all lowercase
df['name_upper'] = df['name'].str.upper()        # ALL UPPERCASE
df['name_title'] = df['name'].str.title()        # Title Case
df['name_capitalize'] = df['name'].str.capitalize()  # First letter only
df['name_swapcase'] = df['name'].str.swapcase()  # sWAP cASE

# =============================================
# WHITESPACE HANDLING
# =============================================
df['name_stripped'] = df['name'].str.strip()      # remove leading/trailing whitespace
df['name_lstrip'] = df['name'].str.lstrip()       # remove leading whitespace only
df['name_rstrip'] = df['name'].str.rstrip()       # remove trailing whitespace only

# =============================================
# SEARCHING AND MATCHING
# =============================================
# Check if string contains a substring
df['is_gmail'] = df['email'].str.contains('gmail', case=False, na=False)
# na=False: treat NaN as False instead of NaN

# Check if string starts/ends with
df['starts_with_a'] = df['name'].str.strip().str.startswith('A')
df['ends_com'] = df['email'].str.endswith('.com', na=False)

# Find position of substring (-1 if not found)
df['at_position'] = df['email'].str.find('@', na=-1)

# Count occurrences of a character
df['dash_count'] = df['phone'].str.count('-')

# =============================================
# EXTRACTING AND SPLITTING
# =============================================
# Split into multiple columns
df[['first_name', 'last_name']] = df['name'].str.strip().str.split(n=1, expand=True)

# Extract using regex (capture groups become columns)
df['area_code'] = df['phone'].str.extract(r'(\d{3})', expand=False)

# Extract multiple groups
extracted = df['address'].str.extract(r'(.+),\s*(.+),\s*(\w{2})\s*(\d{5})')
extracted.columns = ['street', 'city', 'state', 'zip']

# Extract product number from product code
df['prod_num'] = df['product_code'].str.extract(r'PROD-(\d+)', expand=False).astype(int)

# Get substring by position
df['email_domain'] = df['email'].str.split('@').str[1]  # after @

# =============================================
# REPLACING AND CLEANING
# =============================================
# Simple replace
df['phone_clean'] = df['phone'].str.replace(r'[^\d]', '', regex=True)
# Remove all non-digits: '(555) 123-4567' → '5551234567'

# Replace with regex
df['email_masked'] = df['email'].str.replace(
    r'(.{2}).+(@)', r'\1***\2', regex=True, na='N/A'
)

# Pad strings to fixed width
df['id_padded'] = pd.Series(['1', '23', '456']).str.zfill(5)  # '00001', '00023', '00456'
# Or: .str.pad(5, side='left', fillchar='0')

# =============================================
# STRING LENGTH AND SLICING
# =============================================
df['name_len'] = df['name'].str.len()        # length of each string
df['initials'] = df['name'].str.strip().str[0]  # first character
df['last_3'] = df['email'].str[-3:]           # last 3 characters

# =============================================
# CATEGORICAL / DUMMY ENCODING FROM STRINGS
# =============================================
# One-hot encode from delimited strings
tags = pd.Series(['python,ml', 'python,web', 'ml,nlp', 'web'])
dummies = tags.str.get_dummies(sep=',')
# Output:
#    ml  nlp  python  web
# 0   1    0       1    0
# 1   0    0       1    1
# 2   1    1       0    0
# 3   0    0       0    1

# =============================================
# CHAINING STRING OPERATIONS
# =============================================
# Clean a messy name column in one chain
df['name_clean'] = (
    df['name']
    .str.strip()           # remove whitespace
    .str.lower()           # lowercase
    .str.replace(r'\s+', ' ', regex=True)  # collapse multiple spaces
    .str.title()           # title case
)
```

> **Pro Tip**: Always pass `na=False` to `.str.contains()`, `.str.startswith()`, etc., when your column might have NaN. Without it, NaN rows return NaN (not False), which breaks Boolean filtering.

---

## Type Conversion and Casting

### What It Is
Type conversion (casting) changes a column's data type — like converting a "price" column stored as text `"$1,234.56"` into a proper float `1234.56`. Think of it like converting currencies: the value is the same, but the format changes.

### Why It Matters
- Data loaded from CSV is often all strings — you must convert to proper types
- Wrong types waste memory: `int64` for a column of 0s and 1s wastes 7 bytes per value
- Operations fail on wrong types: you can't compute the mean of a string column
- ML models require numeric inputs — categorical strings must be encoded

### Code Examples

```python
import pandas as pd
import numpy as np

# --- Basic type conversion with astype() ---
df = pd.DataFrame({
    'id': ['1', '2', '3', '4'],
    'price': ['10.5', '20.3', '15.7', '8.9'],
    'quantity': [10, 20, 30, 40],
    'is_active': [1, 0, 1, 1],
    'category': ['A', 'B', 'A', 'C'],
    'date_str': ['2024-01-15', '2024-02-20', '2024-03-10', '2024-04-05']
})

# String to integer
df['id'] = df['id'].astype(int)

# String to float
df['price'] = df['price'].astype(float)

# Integer to boolean
df['is_active'] = df['is_active'].astype(bool)

# String to category (saves memory for low-cardinality columns)
df['category'] = df['category'].astype('category')

# --- Handling conversion errors ---
messy = pd.Series(['1', '2', 'three', '4', 'N/A'])

# astype will CRASH on invalid values
# messy.astype(int)  # ValueError!

# pd.to_numeric with errors='coerce' converts failures to NaN
cleaned = pd.to_numeric(messy, errors='coerce')
# Output: [1.0, 2.0, NaN, 4.0, NaN]

# errors='ignore' returns the original series unchanged (usually not helpful)
# errors='raise' raises an exception (default)

# --- Date conversion ---
df['date'] = pd.to_datetime(df['date_str'])

# Handle messy date formats
messy_dates = pd.Series(['01/15/2024', '2024-02-20', 'Feb 10, 2024', 'invalid'])
parsed = pd.to_datetime(messy_dates, errors='coerce', format='mixed')
# 'invalid' becomes NaT

# Specify exact format (much faster for large datasets)
df['date'] = pd.to_datetime(df['date_str'], format='%Y-%m-%d')

# --- Money/Currency string to float ---
prices = pd.Series(['$1,234.56', '$789.00', '$12,345.67', 'N/A'])
prices_clean = (
    prices
    .str.replace('$', '', regex=False)    # remove dollar sign
    .str.replace(',', '', regex=False)    # remove commas
    .pipe(pd.to_numeric, errors='coerce') # convert to float
)
# Output: [1234.56, 789.0, 12345.67, NaN]

# --- Percentage string to float ---
pcts = pd.Series(['45.2%', '12.8%', '99.1%'])
pcts_float = pcts.str.rstrip('%').astype(float) / 100
# Output: [0.452, 0.128, 0.991]

# --- Memory-efficient downcasting ---
big_ints = pd.Series([1, 2, 3, 100, 200], dtype='int64')  # 8 bytes each
small_ints = pd.to_numeric(big_ints, downcast='integer')    # auto-picks int8 (1 byte)

big_floats = pd.Series([1.0, 2.5, 3.7], dtype='float64')
small_floats = pd.to_numeric(big_floats, downcast='float')  # auto-picks float32

# --- Check and convert dtypes of entire DataFrame ---
print(df.dtypes)

# Convert all object columns that look numeric
for col in df.select_dtypes(include='object').columns:
    try:
        df[col] = pd.to_numeric(df[col])
    except (ValueError, TypeError):
        pass  # keep as string

# --- Categorical encoding ---
df = pd.DataFrame({'color': ['red', 'blue', 'green', 'red', 'blue']})

# Label encoding (ordinal)
df['color_code'] = df['color'].astype('category').cat.codes
# red=2, green=1, blue=0 (alphabetical order)

# One-hot encoding
dummies = pd.get_dummies(df['color'], prefix='color', dtype=int)
# color_blue, color_green, color_red columns

# Ordinal with custom order
size_order = pd.CategoricalDtype(categories=['S', 'M', 'L', 'XL'], ordered=True)
df_sizes = pd.DataFrame({'size': ['M', 'L', 'S', 'XL', 'M']})
df_sizes['size'] = df_sizes['size'].astype(size_order)
# Now: df_sizes['size'].min() = 'S', df_sizes['size'].max() = 'XL'
# And: df_sizes[df_sizes['size'] > 'M']  works!
```

### Memory Comparison by dtype

| dtype | Bytes per value | Range | Use Case |
|-------|----------------|-------|----------|
| `int8` | 1 | -128 to 127 | Age, small counts |
| `int16` | 2 | -32,768 to 32,767 | Year, moderate counts |
| `int32` | 4 | ±2.1 billion | Most integers |
| `int64` | 8 | ±9.2 quintillion | Default (often overkill) |
| `float32` | 4 | ~7 decimal digits | Most ML, sufficient precision |
| `float64` | 8 | ~15 decimal digits | Default (finance, science) |
| `category` | 1-8 + overhead | N/A | Low-cardinality strings |
| `bool` | 1 | True/False | Flags |

---

## Outlier Detection and Treatment

### What It Is
Outliers are data points that are significantly different from other observations. They can be genuine extreme values (a billionaire in a salary dataset) or errors (age = 999). Like finding a penguin in a flock of flamingos — it stands out and might skew your analysis.

### Why It Matters
- Outliers skew mean, standard deviation, and regression coefficients
- Many ML models (linear regression, k-means) are sensitive to outliers
- Can indicate data quality issues, fraud, or genuinely interesting events
- Removing real outliers = losing valuable information; keeping errors = noise

### Code Examples

```python
import pandas as pd
import numpy as np

np.random.seed(42)
df = pd.DataFrame({
    'salary': np.concatenate([
        np.random.normal(50000, 15000, 97),  # normal salaries
        [200000, 500000, -5000]               # outliers
    ]),
    'age': np.concatenate([
        np.random.randint(22, 65, 97),
        [150, 5, -1]                          # impossible ages
    ])
})

# --- Method 1: IQR (Interquartile Range) Method ---
# Most commonly used, robust to the outliers themselves
Q1 = df['salary'].quantile(0.25)
Q3 = df['salary'].quantile(0.75)
IQR = Q3 - Q1

lower_bound = Q1 - 1.5 * IQR
upper_bound = Q3 + 1.5 * IQR

# Flag outliers
df['salary_outlier_iqr'] = ~df['salary'].between(lower_bound, upper_bound)

# Remove outliers
df_clean = df[df['salary'].between(lower_bound, upper_bound)]

# --- Method 2: Z-Score Method ---
# Works well for normally distributed data
from scipy import stats

z_scores = np.abs(stats.zscore(df['salary']))
df['salary_outlier_z'] = z_scores > 3  # common threshold

# Without scipy:
mean = df['salary'].mean()
std = df['salary'].std()
df['z_score'] = (df['salary'] - mean) / std
df['salary_outlier_z'] = df['z_score'].abs() > 3

# --- Method 3: Percentile Capping (Winsorization) ---
# Don't remove, just cap at bounds
lower = df['salary'].quantile(0.01)
upper = df['salary'].quantile(0.99)
df['salary_capped'] = df['salary'].clip(lower=lower, upper=upper)

# --- Method 4: Domain Knowledge Bounds ---
# Hard rules based on what's physically possible
df['age_valid'] = df['age'].between(0, 120)  # human age bounds
df.loc[~df['age_valid'], 'age'] = np.nan     # replace impossible values with NaN

# --- Reusable outlier detection function ---
def detect_outliers(df, columns, method='iqr', threshold=1.5):
    """
    Detect outliers using IQR or Z-score method.
    Returns a boolean DataFrame where True = outlier.
    """
    outlier_mask = pd.DataFrame(False, index=df.index, columns=columns)

    for col in columns:
        if method == 'iqr':
            Q1 = df[col].quantile(0.25)
            Q3 = df[col].quantile(0.75)
            IQR = Q3 - Q1
            outlier_mask[col] = ~df[col].between(
                Q1 - threshold * IQR,
                Q3 + threshold * IQR
            )
        elif method == 'zscore':
            z = (df[col] - df[col].mean()) / df[col].std()
            outlier_mask[col] = z.abs() > threshold

    return outlier_mask

outliers = detect_outliers(df, ['salary', 'age'], method='iqr')
print(f"Outliers found: {outliers.any(axis=1).sum()}")
```

### Outlier Treatment Decision Matrix

| Scenario | Action | Rationale |
|----------|--------|-----------|
| Data entry error (age = 999) | Remove or correct | Not real data |
| Genuine extreme (CEO salary) | Keep or cap | Real but influential |
| Sensor malfunction | Remove or interpolate | Not real measurement |
| Fraud detection | Keep and flag! | These ARE what you're looking for |
| Training ML model | Cap or transform (log) | Reduce influence without losing |

---

## Data Validation and Consistency

### What It Is
Data validation ensures your data meets expected rules and constraints — like spell-checking but for data quality. It catches impossible values, inconsistencies, and format violations.

### Why It Matters
- Garbage in = garbage out: no model can fix bad input data
- Production pipelines need automated validation to catch issues early
- Regulatory compliance often requires documented data quality checks

### Code Examples

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    'email': ['alice@gmail.com', 'bob@', 'charlie@outlook.com', 'not-an-email'],
    'age': [25, -5, 150, 30],
    'start_date': pd.to_datetime(['2020-01-01', '2019-06-15', '2025-01-01', '2021-03-20']),
    'end_date': pd.to_datetime(['2021-01-01', '2019-01-15', '2026-01-01', '2022-03-20']),
    'status': ['active', 'inactive', 'ACTIVE', 'unknown'],
    'score': [85, 105, -10, 72]
})

# --- Range validation ---
assert df['age'].between(0, 120).all(), "Invalid ages found!"
invalid_ages = df[~df['age'].between(0, 120)]

# Score should be 0-100
assert df['score'].between(0, 100).all(), "Invalid scores!"
df['score'] = df['score'].clip(0, 100)  # fix by capping

# --- Date consistency ---
# End date should be after start date
invalid_dates = df[df['end_date'] < df['start_date']]
print(f"Records where end < start: {len(invalid_dates)}")

# --- Categorical consistency ---
valid_statuses = ['active', 'inactive']
df['status_clean'] = df['status'].str.lower().str.strip()
invalid_status = df[~df['status_clean'].isin(valid_statuses)]

# --- Email validation (basic regex) ---
email_pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
df['valid_email'] = df['email'].str.match(email_pattern, na=False)

# --- Cross-column validation ---
# If status is 'inactive', end_date should not be null
mask = (df['status_clean'] == 'inactive') & (df['end_date'].isna())
assert not mask.any(), "Inactive records missing end_date!"

# --- Comprehensive validation report ---
def validate_dataframe(df, rules):
    """
    rules: dict of {column: {'type': dtype, 'range': (min, max), 'allowed': [values]}}
    """
    issues = []
    for col, checks in rules.items():
        if col not in df.columns:
            issues.append(f"Missing column: {col}")
            continue
        if 'range' in checks:
            lo, hi = checks['range']
            bad = (~df[col].between(lo, hi)).sum()
            if bad > 0:
                issues.append(f"{col}: {bad} values outside [{lo}, {hi}]")
        if 'allowed' in checks:
            bad = (~df[col].isin(checks['allowed'])).sum()
            if bad > 0:
                issues.append(f"{col}: {bad} values not in allowed set")
        if 'no_null' in checks and checks['no_null']:
            nulls = df[col].isna().sum()
            if nulls > 0:
                issues.append(f"{col}: {nulls} null values")
    return issues

rules = {
    'age': {'range': (0, 120), 'no_null': True},
    'score': {'range': (0, 100)},
    'status': {'allowed': ['active', 'inactive']}
}

issues = validate_dataframe(df, rules)
for issue in issues:
    print(f"⚠ {issue}")
```

---

## Common Mistakes

### 1. Filling NaN Before Understanding WHY It's Missing
```python
# WRONG: blindly fill with mean
df['salary'].fillna(df['salary'].mean())

# RIGHT: investigate first
# Are salaries missing because the person is an intern (no salary)?
# Then filling with mean is WRONG — fill with 0 or create a separate category
print(df[df['salary'].isna()][['name', 'position', 'department']])
```

### 2. String Operations Without `.str` Accessor
```python
# WRONG
df['name'].lower()  # AttributeError — .lower() is for Python strings, not Series

# CORRECT
df['name'].str.lower()  # vectorized via .str accessor
```

### 3. Not Handling NaN in String Columns
```python
# WRONG — crashes if column has NaN
df[df['email'].str.contains('gmail')]  # NaN rows produce NaN, not True/False

# CORRECT
df[df['email'].str.contains('gmail', na=False)]
```

### 4. Dropping Duplicates Without Specifying Subset
```python
# WRONG — drops only if ALL columns match (too strict)
df.drop_duplicates()  # might keep near-duplicates

# RIGHT — drop based on the meaningful key
df.drop_duplicates(subset=['customer_id'])
```

### 5. Converting Types Without Cleaning First
```python
# WRONG — crashes on dirty data
df['price'] = df['price'].astype(float)  # fails if '$' or ',' present

# RIGHT — clean, then convert
df['price'] = (
    df['price']
    .str.replace('[$,]', '', regex=True)
    .pipe(pd.to_numeric, errors='coerce')
)
```

### 6. Not Verifying Data After Cleaning
```python
# Always check after cleaning
print(f"Shape: {df.shape}")
print(f"Nulls remaining:\n{df.isna().sum()}")
print(f"Duplicates: {df.duplicated().sum()}")
print(f"Dtypes:\n{df.dtypes}")
print(f"Stats:\n{df.describe()}")
```

---

## Interview Questions

### Q1: How do you handle missing values in a dataset? Walk through your approach.
**Answer**:
1. **Investigate** — check percentage missing per column, understand WHY it's missing (MCAR/MAR/MNAR)
2. **Decide** — if < 5% and MCAR, consider dropping; otherwise impute
3. **Choose method** — mean/median for numeric (median if skewed), mode for categorical, group-wise imputation if missingness depends on another column
4. **Create indicators** — add `_missing` flag columns if missingness is informative
5. **Validate** — check that imputed values make sense statistically

### Q2: What's the difference between `NaN`, `None`, `pd.NA`, and `pd.NaT`?
**Answer**:
- `NaN` (NumPy): IEEE 754 floating-point "Not a Number". Used in float columns. `NaN != NaN` is True.
- `None`: Python's null object. Used in object-dtype columns.
- `pd.NA`: Pandas' experimental scalar NA for nullable dtypes (`Int64`, `boolean`, `string`). Propagates properly in arithmetic.
- `pd.NaT`: "Not a Time" — the missing value for datetime/timedelta columns.
All are detected by `pd.isna()` / `pd.isnull()`.

### Q3: You have a column with values like "$1,234.56". How do you convert it to numeric?
**Answer**:
```python
df['price'] = (
    df['price']
    .str.replace('[$,]', '', regex=True)
    .astype(float)
)
```

### Q4: What is the difference between `drop_duplicates()` and `groupby().first()`?
**Answer**: `drop_duplicates(subset=['key'])` keeps the first (or last) occurrence of each key, discarding other columns' values. `groupby('key').first()` also keeps the first non-null value per column for each key, which can pull values from different rows. They behave the same only when there are complete duplicate rows.

### Q5: How would you reduce the memory footprint of a 5GB DataFrame?
**Answer**:
1. Downcast numerics: `int64` → `int8/16/32`, `float64` → `float32`
2. Convert low-cardinality strings to `category` dtype
3. Parse dates as `datetime64` instead of strings
4. Use chunked reading with `pd.read_csv(chunksize=...)` if it doesn't fit in RAM
5. Drop unnecessary columns early

### Q6: How do you detect near-duplicate records (fuzzy matching)?
**Answer**: Normalize text first (lowercase, strip whitespace, remove punctuation), then use `drop_duplicates()`. For advanced cases, use `fuzzywuzzy`/`rapidfuzz` library to compute string similarity scores, or use `recordlinkage` library for probabilistic matching.

---

## Quick Reference

| Operation | Code | Notes |
|-----------|------|-------|
| Check nulls | `df.isna().sum()` | Per-column count |
| Null percentage | `df.isna().mean() * 100` | Percentage per column |
| Drop null rows | `df.dropna()` | `subset=`, `thresh=` options |
| Fill with value | `df['col'].fillna(value)` | Constant, mean, median |
| Forward fill | `df['col'].ffill()` | Last valid observation |
| Interpolate | `df['col'].interpolate()` | Linear by default |
| Check duplicates | `df.duplicated().sum()` | `subset=` for specific cols |
| Drop duplicates | `df.drop_duplicates()` | `keep='first'\|'last'\|False` |
| Lowercase | `df['col'].str.lower()` | All `.str` methods vectorized |
| Strip whitespace | `df['col'].str.strip()` | Also `lstrip()`, `rstrip()` |
| Regex extract | `df['col'].str.extract(r'(pattern)')` | Capture groups → columns |
| String contains | `df['col'].str.contains('x', na=False)` | Always use `na=False` |
| To numeric | `pd.to_numeric(s, errors='coerce')` | Failures → NaN |
| To datetime | `pd.to_datetime(s, errors='coerce')` | Failures → NaT |
| Downcast dtype | `df['col'].astype('int32')` | Or `pd.to_numeric(downcast=)` |
| Clip outliers | `df['col'].clip(lower, upper)` | Winsorization |
| Category dtype | `df['col'].astype('category')` | Saves memory for strings |

---

*Previous: [05-Pandas-Data-Manipulation](05-Pandas-Data-Manipulation.md) | Next: [07-Pandas-Time-Series](07-Pandas-Time-Series.md)*
