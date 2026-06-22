# Chapter 06: Feature Engineering

## Table of Contents
- [6.1 What is Feature Engineering](#61-what-is-feature-engineering)
- [6.2 Feature Creation — Building New Features](#62-feature-creation--building-new-features)
- [6.3 Feature Transformation](#63-feature-transformation)
- [6.4 Encoding Categorical Variables](#64-encoding-categorical-variables)
- [6.5 Feature Scaling and Normalization](#65-feature-scaling-and-normalization)
- [6.6 Feature Selection](#66-feature-selection)
- [6.7 Handling Date and Time Features](#67-handling-date-and-time-features)
- [6.8 Text Feature Engineering](#68-text-feature-engineering)
- [6.9 Advanced Techniques](#69-advanced-techniques)
- [6.10 Complete Feature Engineering Pipeline](#610-complete-feature-engineering-pipeline)

---

## 6.1 What is Feature Engineering

### What It Is
Feature engineering is the art and science of transforming raw data into features (input variables) that better represent the underlying problem to machine learning models, resulting in improved accuracy. Think of it as preparing ingredients for a recipe — the same raw vegetables can be served raw (poor model), or diced, seasoned, and sautéed (excellent model).

### Why It Matters
- **#1 factor** in Kaggle competition wins (more than model choice)
- A simple model with great features beats a complex model with bad features
- Encodes **domain knowledge** into the model
- Can turn an impossible problem into a solvable one
- "Applied machine learning is basically feature engineering" — Andrew Ng

### The Feature Engineering Mindset

```
RAW DATA                    ENGINEERED FEATURES              MODEL
━━━━━━━━━                   ━━━━━━━━━━━━━━━━━━━             ━━━━━━

timestamp ─────────► hour_of_day, is_weekend,        ┐
                     day_of_week, is_holiday          │
                                                      │
address ───────────► latitude, longitude,             │
                     distance_to_center,              ├──► Better
                     neighborhood_avg_price           │    Predictions!
                                                      │
purchase_history ──► total_spend, avg_order,          │
                     days_since_last, frequency       │
                                                      │
text_review ───────► sentiment_score, word_count,     │
                     has_negative_words               ┘
```

### Feature Engineering Taxonomy

| Category | What | Example |
|----------|------|---------|
| **Creation** | Build new features from existing | `age` from `birth_date` |
| **Transformation** | Change distribution/scale | `log(income)`, standardize |
| **Encoding** | Convert categories to numbers | One-hot, target encoding |
| **Scaling** | Bring features to same range | MinMax, StandardScaler |
| **Selection** | Choose the most useful features | Correlation, importance |
| **Extraction** | Pull features from complex data | TF-IDF from text, FFT from signals |
| **Interaction** | Combine features | `price_per_sqft = price / area` |

---

## 6.2 Feature Creation — Building New Features

### What It Is
Creating entirely new columns by combining, computing, or deriving information from existing data. This is where domain knowledge truly shines — knowing that `BMI = weight / height²` is more informative than weight and height separately.

### Why It Matters
- Raw features often don't capture the **actual signal**
- Ratios, differences, and aggregates often have stronger predictive power
- Helps models learn patterns they can't discover on their own (especially linear models)

### Code Examples

```python
import pandas as pd
import numpy as np

# ============================================
# Sample dataset: E-commerce customers
# ============================================
df = pd.DataFrame({
    'customer_id': range(1, 11),
    'registration_date': pd.to_datetime([
        '2021-01-15', '2021-03-22', '2020-07-10', '2022-01-01', '2020-12-05',
        '2021-06-18', '2021-09-30', '2022-05-15', '2020-03-01', '2021-11-11'
    ]),
    'last_purchase_date': pd.to_datetime([
        '2023-06-01', '2023-05-15', '2023-06-10', '2023-04-20', '2023-06-15',
        '2023-03-01', '2023-06-12', '2023-06-14', '2023-01-10', '2023-06-08'
    ]),
    'total_orders': [45, 12, 89, 5, 67, 23, 34, 8, 120, 56],
    'total_spend': [4500, 1800, 12000, 750, 8900, 3200, 5600, 1200, 18000, 7800],
    'num_returns': [3, 1, 12, 0, 5, 2, 4, 1, 8, 3],
    'age': [28, 35, 42, 22, 55, 31, 38, 25, 48, 33],
    'city': ['Mumbai', 'Delhi', 'Mumbai', 'Bangalore', 'Delhi', 
             'Mumbai', 'Chennai', 'Bangalore', 'Delhi', 'Mumbai'],
    'num_items_browsed': [500, 150, 1200, 80, 890, 300, 450, 100, 2000, 700],
    'num_items_carted': [100, 30, 200, 15, 150, 60, 80, 20, 350, 120]
})

print("Original columns:", list(df.columns))
reference_date = pd.Timestamp('2023-06-15')

# ============================================
# 1. RATIO FEATURES — Often the most powerful
# ============================================
# Average order value (how much they spend per order)
df['avg_order_value'] = df['total_spend'] / df['total_orders']

# Return rate (what fraction of orders get returned)
df['return_rate'] = df['num_returns'] / df['total_orders']

# Conversion rate (browsing → carting)
df['browse_to_cart_rate'] = df['num_items_carted'] / df['num_items_browsed']

# Cart-to-purchase rate
df['cart_to_purchase_rate'] = df['total_orders'] / df['num_items_carted']

print("\nRatio Features:")
print(df[['avg_order_value', 'return_rate', 'browse_to_cart_rate']].round(3))

# ============================================
# 2. TEMPORAL FEATURES — Time-based insights
# ============================================
# Customer tenure (how long they've been a customer)
df['tenure_days'] = (reference_date - df['registration_date']).dt.days

# Recency (days since last purchase — key for churn prediction!)
df['recency_days'] = (reference_date - df['last_purchase_date']).dt.days

# Purchase frequency (orders per month of tenure)
df['orders_per_month'] = df['total_orders'] / (df['tenure_days'] / 30)

# Is the customer "active" (purchased within last 30 days)?
df['is_active'] = (df['recency_days'] <= 30).astype(int)

print("\nTemporal Features:")
print(df[['tenure_days', 'recency_days', 'orders_per_month', 'is_active']])

# ============================================
# 3. BINNING / DISCRETIZATION — Create categories from numbers
# ============================================
# Age groups
df['age_group'] = pd.cut(df['age'], bins=[0, 25, 35, 45, 100], 
                          labels=['Young', 'Adult', 'Middle', 'Senior'])

# Spending tiers (quantile-based — equal-frequency bins)
df['spend_tier'] = pd.qcut(df['total_spend'], q=3, labels=['Low', 'Medium', 'High'])

# Custom bins based on domain knowledge
df['engagement_level'] = pd.cut(
    df['orders_per_month'],
    bins=[-np.inf, 1, 3, 5, np.inf],
    labels=['Dormant', 'Casual', 'Regular', 'Power']
)

print("\nBinned Features:")
print(df[['age', 'age_group', 'total_spend', 'spend_tier', 'engagement_level']])

# ============================================
# 4. AGGREGATION FEATURES — Group-level statistics
# ============================================
# City-level features (how does this customer compare to their city?)
city_stats = df.groupby('city').agg({
    'total_spend': ['mean', 'median', 'std'],
    'total_orders': 'mean',
    'age': 'mean'
}).reset_index()
city_stats.columns = ['city', 'city_avg_spend', 'city_med_spend', 
                       'city_std_spend', 'city_avg_orders', 'city_avg_age']

# Merge back
df = df.merge(city_stats, on='city', how='left')

# Relative features (how does this person compare to their group?)
df['spend_vs_city_avg'] = df['total_spend'] / df['city_avg_spend']
df['spend_zscore_city'] = (df['total_spend'] - df['city_avg_spend']) / df['city_std_spend']

print("\nAggregation Features:")
print(df[['customer_id', 'city', 'total_spend', 'city_avg_spend', 'spend_vs_city_avg']].round(2))

# ============================================
# 5. MATHEMATICAL TRANSFORMATIONS
# ============================================
# Polynomial features (captures non-linear relationships)
df['age_squared'] = df['age'] ** 2
df['tenure_squared'] = df['tenure_days'] ** 2

# Interaction features (combinations that might matter)
df['age_x_spend'] = df['age'] * df['total_spend']
df['tenure_x_orders'] = df['tenure_days'] * df['total_orders']

# Log transforms (for highly skewed features)
df['log_spend'] = np.log1p(df['total_spend'])      # log(1+x) to handle zeros
df['log_orders'] = np.log1p(df['total_orders'])

print("\nFinal feature count:", len(df.columns))
print("New features created:", len(df.columns) - 10)  # Started with 10
```

> **Pro Tip**: The best features often come from **domain expertise**. Talk to business stakeholders — they know patterns like "customers who browse but don't buy within 3 days rarely come back."

---

## 6.3 Feature Transformation

### What It Is
Changing the mathematical form of a feature to make it more suitable for modeling. Like adjusting the focus on a camera — the same scene looks clearer with the right settings.

### Why It Matters
- Many models assume **normally distributed** features
- Skewed data reduces model performance
- Transformations can **linearize** non-linear relationships
- Can reduce the impact of outliers

### When to Apply Which Transformation

| Transform | Formula | When to Use | Effect |
|-----------|---------|-------------|--------|
| Log | $\log(x+1)$ | Right-skewed, positive values | Compresses large values |
| Square Root | $\sqrt{x}$ | Moderate right skew | Gentler than log |
| Box-Cox | $\frac{x^\lambda - 1}{\lambda}$ | Find optimal transform | Learns best λ |
| Yeo-Johnson | Modified Box-Cox | Works with negatives | Flexible |
| Reciprocal | $\frac{1}{x}$ | Extreme right skew | Very aggressive |
| Power | $x^2, x^3$ | Left-skewed data | Amplifies large values |

### Code Examples

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from scipy import stats
from sklearn.preprocessing import PowerTransformer, QuantileTransformer

# ============================================
# Generate skewed data (simulating income distribution)
# ============================================
np.random.seed(42)
income = np.random.lognormal(mean=10, sigma=1, size=5000)
df = pd.DataFrame({'income': income})

print(f"Original: mean={income.mean():.0f}, median={np.median(income):.0f}, skew={stats.skew(income):.2f}")

# ============================================
# Compare Transformations
# ============================================
fig, axes = plt.subplots(2, 3, figsize=(15, 10))

# Original
axes[0, 0].hist(income, bins=50, color='steelblue', edgecolor='black', alpha=0.7)
axes[0, 0].set_title(f'Original\nSkew: {stats.skew(income):.2f}')

# Log Transform
log_income = np.log1p(income)  # log(1+x) handles zeros
axes[0, 1].hist(log_income, bins=50, color='forestgreen', edgecolor='black', alpha=0.7)
axes[0, 1].set_title(f'Log Transform\nSkew: {stats.skew(log_income):.2f}')

# Square Root Transform
sqrt_income = np.sqrt(income)
axes[0, 2].hist(sqrt_income, bins=50, color='coral', edgecolor='black', alpha=0.7)
axes[0, 2].set_title(f'Square Root\nSkew: {stats.skew(sqrt_income):.2f}')

# Box-Cox Transform (requires positive values)
boxcox_income, best_lambda = stats.boxcox(income)
axes[1, 0].hist(boxcox_income, bins=50, color='purple', edgecolor='black', alpha=0.7)
axes[1, 0].set_title(f'Box-Cox (λ={best_lambda:.2f})\nSkew: {stats.skew(boxcox_income):.2f}')

# Yeo-Johnson Transform (handles negatives too)
pt = PowerTransformer(method='yeo-johnson')
yj_income = pt.fit_transform(income.reshape(-1, 1)).flatten()
axes[1, 1].hist(yj_income, bins=50, color='orange', edgecolor='black', alpha=0.7)
axes[1, 1].set_title(f'Yeo-Johnson\nSkew: {stats.skew(yj_income):.2f}')

# Quantile Transform (forces any distribution to uniform/normal)
qt = QuantileTransformer(output_distribution='normal', random_state=42)
qt_income = qt.fit_transform(income.reshape(-1, 1)).flatten()
axes[1, 2].hist(qt_income, bins=50, color='teal', edgecolor='black', alpha=0.7)
axes[1, 2].set_title(f'Quantile (Normal)\nSkew: {stats.skew(qt_income):.2f}')

plt.tight_layout()
plt.savefig('transformations_compared.png', dpi=100, bbox_inches='tight')
plt.show()

# ============================================
# Practical Transformation Decision Guide
# ============================================
def suggest_transformation(series, name='feature'):
    """Analyze a numeric series and suggest appropriate transformation."""
    skew = series.skew()
    has_negatives = (series < 0).any()
    has_zeros = (series == 0).any()
    
    print(f"\n{'='*50}")
    print(f"Feature: {name}")
    print(f"  Skewness: {skew:.3f}")
    print(f"  Has negatives: {has_negatives}")
    print(f"  Has zeros: {has_zeros}")
    print(f"  Range: [{series.min():.2f}, {series.max():.2f}]")
    
    if abs(skew) < 0.5:
        print(f"  ✓ Approximately symmetric — no transform needed")
    elif skew > 0.5:  # Right-skewed
        if has_negatives:
            print(f"  → Use Yeo-Johnson transform (handles negatives)")
        elif has_zeros:
            print(f"  → Use log1p (log(1+x)) transform")
        else:
            print(f"  → Use log or Box-Cox transform")
    else:  # Left-skewed
        if has_negatives:
            print(f"  → Use Yeo-Johnson transform")
        else:
            print(f"  → Use square or exponential transform")
    
    return skew

# Test on multiple features
suggest_transformation(pd.Series(income), 'income')
suggest_transformation(pd.Series(np.random.normal(0, 1, 1000)), 'normal_feature')
suggest_transformation(pd.Series(np.random.exponential(5, 1000)), 'wait_time')
```

> **Important**: Always fit transformers on training data only, then apply to test data. Store the transformer object for inference.

---

## 6.4 Encoding Categorical Variables

### What It Is
Converting categorical (text) data into numbers that machine learning models can understand. Models only understand numbers — they can't process "Red", "Blue", "Green" directly.

### Why It Matters
- Most ML algorithms **require numeric input**
- Wrong encoding can introduce **false ordinal relationships** (e.g., Red=1, Blue=2 implies Blue > Red)
- Right encoding can **dramatically improve** model performance
- Encoding choice depends on the algorithm and cardinality

### Encoding Methods Comparison

| Method | When to Use | Pros | Cons |
|--------|-------------|------|------|
| **Label Encoding** | Ordinal data, tree models | Simple, no dimension increase | Implies false ordering for nominal |
| **One-Hot Encoding** | Nominal, low cardinality (<15) | No false ordering | High dimensions, sparse |
| **Target Encoding** | High cardinality | Powerful, single column | Risk of overfitting/leakage |
| **Frequency Encoding** | High cardinality, tree models | Simple, captures popularity | Collisions (same frequency) |
| **Binary Encoding** | Medium-high cardinality | Fewer dimensions than one-hot | Less interpretable |
| **Ordinal Encoding** | Naturally ordered categories | Preserves order, compact | Must know the ordering |

### Code Examples

```python
import pandas as pd
import numpy as np
from sklearn.preprocessing import LabelEncoder, OneHotEncoder, OrdinalEncoder
from category_encoders import TargetEncoder, BinaryEncoder

# Sample data
df = pd.DataFrame({
    'color': ['Red', 'Blue', 'Green', 'Red', 'Blue', 'Green', 'Red', 'Blue'],
    'size': ['Small', 'Medium', 'Large', 'XL', 'Small', 'Medium', 'Large', 'XL'],
    'city': ['Mumbai', 'Delhi', 'Mumbai', 'Bangalore', 'Delhi', 'Chennai', 'Mumbai', 'Delhi'],
    'brand': ['A', 'B', 'C', 'A', 'B', 'A', 'C', 'B'],
    'target': [1, 0, 1, 1, 0, 0, 1, 0]
})

print("Original Data:")
print(df)

# ============================================
# 1. LABEL ENCODING — Simple integer mapping
# ============================================
# Use for: ordinal features OR tree-based models (which can handle it)
# DON'T use for: linear models with nominal features (implies false order)

le = LabelEncoder()
df['color_label'] = le.fit_transform(df['color'])
print(f"\nLabel Encoding (color): {dict(zip(le.classes_, le.transform(le.classes_)))}")
# Output: {'Blue': 0, 'Green': 1, 'Red': 2}
# ⚠️ This implies Green > Blue, Red > Green — which is meaningless!

# ============================================
# 2. ORDINAL ENCODING — For naturally ordered categories
# ============================================
# Perfect for: size (S < M < L < XL), education level, rating
size_order = ['Small', 'Medium', 'Large', 'XL']
oe = OrdinalEncoder(categories=[size_order])
df['size_ordinal'] = oe.fit_transform(df[['size']])
print(f"\nOrdinal Encoding (size): Small=0, Medium=1, Large=2, XL=3")
print(df[['size', 'size_ordinal']])

# ============================================
# 3. ONE-HOT ENCODING — Gold standard for nominal data
# ============================================
# Creates binary column for each category
# Use for: nominal features with low cardinality (< 15 categories)

# Method 1: pandas get_dummies (simplest)
df_onehot = pd.get_dummies(df[['color']], prefix='color', drop_first=False)
print(f"\nOne-Hot Encoding (color):")
print(df_onehot)

# Method 2: With drop_first=True (avoid multicollinearity for linear models)
df_onehot_dropped = pd.get_dummies(df[['color']], prefix='color', drop_first=True)
print(f"\nOne-Hot with drop_first (color):")
print(df_onehot_dropped)
# Only Blue and Green columns — Red is implied when both are 0

# Method 3: sklearn OneHotEncoder (for pipelines)
ohe = OneHotEncoder(sparse_output=False, handle_unknown='ignore')
encoded = ohe.fit_transform(df[['color']])
encoded_df = pd.DataFrame(encoded, columns=ohe.get_feature_names_out(['color']))
print(f"\nsklearn One-Hot:")
print(encoded_df)

# ============================================
# 4. TARGET ENCODING (Mean Encoding) — For high cardinality
# ============================================
# Replaces category with the mean of the target variable for that category
# Use for: high-cardinality features (city with 1000 values)
# ⚠️ RISK: Overfitting and data leakage!

# Manual implementation (with smoothing to prevent overfitting)
def target_encode(df, col, target, smoothing=10):
    """
    Target encoding with smoothing to prevent overfitting.
    
    Formula: encoding = (count × category_mean + smoothing × global_mean) / (count + smoothing)
    
    Smoothing pulls rare categories toward the global mean.
    """
    global_mean = df[target].mean()
    stats_df = df.groupby(col)[target].agg(['mean', 'count'])
    
    # Apply smoothing
    smooth_encoding = (
        (stats_df['count'] * stats_df['mean'] + smoothing * global_mean) / 
        (stats_df['count'] + smoothing)
    )
    
    return df[col].map(smooth_encoding)

df['city_target_encoded'] = target_encode(df, 'city', 'target', smoothing=2)
print(f"\nTarget Encoding (city):")
print(df[['city', 'target', 'city_target_encoded']])

# ⚠️ CORRECT way: use cross-validation to prevent leakage
from sklearn.model_selection import KFold

def target_encode_cv(df, col, target, n_splits=5):
    """Target encoding with cross-validation to prevent leakage."""
    encoded = pd.Series(index=df.index, dtype=float)
    kf = KFold(n_splits=n_splits, shuffle=True, random_state=42)
    
    for train_idx, val_idx in kf.split(df):
        train_means = df.iloc[train_idx].groupby(col)[target].mean()
        encoded.iloc[val_idx] = df.iloc[val_idx][col].map(train_means)
    
    # Fill any NaN with global mean
    encoded.fillna(df[target].mean(), inplace=True)
    return encoded

# ============================================
# 5. FREQUENCY ENCODING — Simple and effective
# ============================================
# Replace category with its frequency (count or proportion)
# Use for: tree-based models, high cardinality

freq = df['city'].value_counts(normalize=True)  # Proportion
df['city_freq'] = df['city'].map(freq)
print(f"\nFrequency Encoding (city):")
print(df[['city', 'city_freq']])

# ============================================
# 6. BINARY ENCODING — Compromise between label and one-hot
# ============================================
# Converts label to binary representation
# 4 categories → only 2 columns (log2(4) = 2) instead of 4 one-hot columns
# Good for medium-cardinality features (15-100 categories)

# Example: 8 cities → 3 binary columns (log2(8) = 3)
# Mumbai=0→000, Delhi=1→001, Bangalore=2→010, Chennai=3→011
# Instead of 8 one-hot columns, you get 3!

# Using category_encoders library:
# be = BinaryEncoder(cols=['city'])
# df_binary = be.fit_transform(df[['city']])

# ============================================
# 7. HANDLING UNKNOWN CATEGORIES (at inference time)
# ============================================
# What happens when a new category appears that wasn't in training?

# Solution 1: Use 'handle_unknown' parameter
ohe = OneHotEncoder(handle_unknown='ignore', sparse_output=False)
ohe.fit(df[['color']])

# New data with unseen category
new_data = pd.DataFrame({'color': ['Red', 'Purple']})  # 'Purple' is new!
encoded_new = ohe.transform(new_data)
print(f"\nHandling unknown 'Purple': {encoded_new}")
# Purple → all zeros (no one-hot flag)

# Solution 2: Group rare categories into 'Other'
def group_rare_categories(series, threshold=0.05):
    """Replace categories appearing less than threshold% with 'Other'."""
    freq = series.value_counts(normalize=True)
    rare = freq[freq < threshold].index
    return series.replace(rare, 'Other')
```

### Encoding Decision Tree

```
           ┌─────────────────────────────────┐
           │   Is the feature ORDINAL?       │
           │   (has natural ordering)         │
           └──────────────┬──────────────────┘
                   ┌──────┴──────┐
                 Yes             No
                   │              │
           ┌───────▼──────┐  ┌───▼───────────────────┐
           │ ORDINAL      │  │ How many unique values?│
           │ ENCODING     │  └───┬───────────────────┘
           └──────────────┘      │
                          ┌──────┼──────────┐
                       < 15    15-100     > 100
                          │      │          │
                   ┌──────▼──┐ ┌─▼────┐ ┌──▼──────────┐
                   │ ONE-HOT │ │BINARY│ │ TARGET or   │
                   │ENCODING │ │ENCODE│ │ FREQUENCY   │
                   └─────────┘ └──────┘ │ ENCODING    │
                                        └─────────────┘
```

---

## 6.5 Feature Scaling and Normalization

### What It Is
Bringing all features to a similar range/scale so that no single feature dominates just because it has larger numbers. Imagine comparing temperatures (20-40°C) with salary (20000-200000) — without scaling, the model thinks salary is 1000x more important just because the numbers are bigger.

### Why It Matters

| Algorithm | Needs Scaling? | Why |
|-----------|---------------|-----|
| Linear Regression | **Yes** | Coefficients are scale-dependent |
| Logistic Regression | **Yes** | Gradient descent converges faster |
| SVM | **Yes** | Distance-based; features with larger range dominate |
| KNN | **Yes** | Distance-based |
| Neural Networks | **Yes** | Gradient descent optimization |
| Decision Trees / Random Forest | **No** | Split-based, not distance-based |
| XGBoost / LightGBM | **No** | Tree-based |
| Naive Bayes | **No** | Probability-based |

### Scaling Methods

| Method | Formula | Range | When to Use |
|--------|---------|-------|-------------|
| **StandardScaler** | $z = \frac{x - \mu}{\sigma}$ | ~[-3, 3] | Most common; normal-ish data |
| **MinMaxScaler** | $x' = \frac{x - x_{min}}{x_{max} - x_{min}}$ | [0, 1] | When you need bounded range |
| **RobustScaler** | $x' = \frac{x - median}{IQR}$ | Varies | Data with outliers |
| **MaxAbsScaler** | $x' = \frac{x}{|x_{max}|}$ | [-1, 1] | Sparse data |
| **Normalizer** | $x' = \frac{x}{\|x\|_2}$ | Unit norm | Per-sample (row) normalization |

### Code Examples

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from sklearn.preprocessing import (
    StandardScaler, MinMaxScaler, RobustScaler, 
    MaxAbsScaler, Normalizer
)

# ============================================
# Sample data with different scales
# ============================================
np.random.seed(42)
df = pd.DataFrame({
    'age': np.random.normal(35, 10, 1000),           # Range: ~15-55
    'income': np.random.lognormal(10, 1, 1000),       # Range: ~2k-200k
    'distance_km': np.random.exponential(5, 1000),    # Range: 0-40
    'score': np.random.uniform(0, 100, 1000)          # Range: 0-100
})

# Add some outliers
df.iloc[0, 1] = 1000000   # Extreme income outlier
df.iloc[1, 1] = 900000

print("Original Statistics:")
print(df.describe().round(2))

# ============================================
# 1. StandardScaler (Z-score normalization)
# ============================================
# Most common scaler. Centers at 0, std = 1
# Best when: roughly normal distribution, no extreme outliers
scaler = StandardScaler()
df_standard = pd.DataFrame(
    scaler.fit_transform(df), 
    columns=df.columns
)
print(f"\nStandardScaler — Mean: {df_standard.mean().round(4).values}")
print(f"StandardScaler — Std:  {df_standard.std().round(4).values}")

# ============================================
# 2. MinMaxScaler (Normalization to [0, 1])
# ============================================
# Maps everything to [0, 1] range
# Best when: need bounded values, uniform distribution
# ⚠️ Sensitive to outliers!
scaler_mm = MinMaxScaler()
df_minmax = pd.DataFrame(
    scaler_mm.fit_transform(df), 
    columns=df.columns
)
print(f"\nMinMaxScaler — Min: {df_minmax.min().round(4).values}")
print(f"MinMaxScaler — Max: {df_minmax.max().round(4).values}")

# ============================================
# 3. RobustScaler (Uses median and IQR — handles outliers!)
# ============================================
# Uses median and IQR instead of mean and std
# Best when: data has outliers (doesn't get affected by them)
scaler_robust = RobustScaler()
df_robust = pd.DataFrame(
    scaler_robust.fit_transform(df), 
    columns=df.columns
)

# ============================================
# Visual Comparison
# ============================================
fig, axes = plt.subplots(2, 2, figsize=(14, 10))

# Show effect on 'income' column (which has outliers)
col = 'income'
axes[0, 0].hist(df[col], bins=50, alpha=0.7, color='steelblue')
axes[0, 0].set_title(f'Original {col}\nRange: [{df[col].min():.0f}, {df[col].max():.0f}]')

axes[0, 1].hist(df_standard[col], bins=50, alpha=0.7, color='forestgreen')
axes[0, 1].set_title(f'StandardScaler\nRange: [{df_standard[col].min():.2f}, {df_standard[col].max():.2f}]')

axes[1, 0].hist(df_minmax[col], bins=50, alpha=0.7, color='coral')
axes[1, 0].set_title(f'MinMaxScaler\nRange: [0, 1] (most values near 0 due to outlier)')

axes[1, 1].hist(df_robust[col], bins=50, alpha=0.7, color='purple')
axes[1, 1].set_title(f'RobustScaler\nOutlier-resistant!')

plt.tight_layout()
plt.show()

# ============================================
# CRITICAL: Correct Usage in ML Pipeline
# ============================================
from sklearn.model_selection import train_test_split

X = df.drop('score', axis=1)
y = df['score']

# Split FIRST, then scale
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Fit on TRAIN only
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)  # fit + transform on train
X_test_scaled = scaler.transform(X_test)         # ONLY transform on test (no fit!)

# ⚠️ WRONG: scaler.fit_transform(X_test) — this causes DATA LEAKAGE!

print(f"\nCorrect scaling:")
print(f"Train mean: {X_train_scaled.mean(axis=0).round(4)}")  # Should be ~0
print(f"Test mean:  {X_test_scaled.mean(axis=0).round(4)}")    # NOT exactly 0 — that's correct!

# ============================================
# Feature Scaling in sklearn Pipeline (best practice)
# ============================================
from sklearn.pipeline import Pipeline
from sklearn.linear_model import LinearRegression

# This ensures scaling is always applied correctly
pipe = Pipeline([
    ('scaler', StandardScaler()),       # Step 1: Scale
    ('model', LinearRegression())       # Step 2: Model
])

# fit_transform on train, transform on test — handled automatically!
pipe.fit(X_train, y_train)
score = pipe.score(X_test, y_test)
print(f"\nPipeline R² score: {score:.4f}")
```

> **Pro Tip**: When using cross-validation, always put the scaler inside the pipeline (not outside). If you scale the full dataset first and then cross-validate, you've leaked information from the validation folds into training.

---

## 6.6 Feature Selection

### What It Is
Choosing the most relevant features from your dataset and discarding the rest. Like packing for a trip — bring what you need, leave the rest behind. More features ≠ better model.

### Why It Matters
- **Reduces overfitting** (fewer features = less noise to memorize)
- **Faster training** (fewer computations)
- **Better interpretability** (understand what drives predictions)
- **Curse of dimensionality** (with many features, you need exponentially more data)
- **Removes multicollinearity** (correlated features confuse linear models)

### Feature Selection Methods

```
Feature Selection Methods
━━━━━━━━━━━━━━━━━━━━━━━━

┌──────────────┐   ┌───────────────┐   ┌───────────────┐
│   FILTER     │   │   WRAPPER     │   │   EMBEDDED    │
│   Methods    │   │   Methods     │   │   Methods     │
├──────────────┤   ├───────────────┤   ├───────────────┤
│• Correlation │   │• Forward      │   │• L1 (Lasso)   │
│• Chi-squared │   │  Selection    │   │• Tree-based   │
│• Mutual Info │   │• Backward     │   │  importance   │
│• Variance    │   │  Elimination  │   │• Elastic Net  │
│  Threshold   │   │• RFE          │   │               │
├──────────────┤   ├───────────────┤   ├───────────────┤
│ Fast, simple │   │ Slow but      │   │ Built into    │
│ Model-free   │   │ thorough      │   │ model training│
└──────────────┘   └───────────────┘   └───────────────┘
```

### Code Examples

```python
import pandas as pd
import numpy as np
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.feature_selection import (
    VarianceThreshold, SelectKBest, f_classif, chi2,
    mutual_info_classif, RFE, SelectFromModel
)
from sklearn.linear_model import LassoCV

# ============================================
# Generate sample dataset with useful and useless features
# ============================================
X, y = make_classification(
    n_samples=1000, 
    n_features=20,           # Total features
    n_informative=8,         # Actually useful features
    n_redundant=4,           # Redundant (correlated with informative)
    n_repeated=2,            # Exact copies
    n_classes=2,
    random_state=42
)
feature_names = [f'feature_{i}' for i in range(20)]
df = pd.DataFrame(X, columns=feature_names)
df['target'] = y

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# ============================================
# METHOD 1: Variance Threshold (remove constant/near-constant features)
# ============================================
# Features with very low variance carry almost no information
selector_var = VarianceThreshold(threshold=0.01)  # Remove if variance < 0.01
X_var = selector_var.fit_transform(X_train)
removed = np.array(feature_names)[~selector_var.get_support()]
print(f"Variance Threshold: Removed {len(removed)} features")
print(f"  Removed: {list(removed)}")

# ============================================
# METHOD 2: Correlation-based (remove highly correlated features)
# ============================================
def remove_correlated_features(df, threshold=0.85):
    """Remove one of each pair of features with correlation > threshold."""
    corr_matrix = df.corr().abs()
    upper = corr_matrix.where(np.triu(np.ones_like(corr_matrix, dtype=bool), k=1))
    
    to_drop = [column for column in upper.columns if any(upper[column] > threshold)]
    
    print(f"\nCorrelation filter (threshold={threshold}):")
    print(f"  Features to drop: {to_drop}")
    return to_drop

to_drop = remove_correlated_features(df[feature_names])
# df_filtered = df.drop(columns=to_drop)

# ============================================
# METHOD 3: SelectKBest (univariate statistical tests)
# ============================================
# F-test (ANOVA) for classification
selector_f = SelectKBest(score_func=f_classif, k=10)
X_kbest = selector_f.fit_transform(X_train, y_train)

# Get scores and selected features
scores = pd.DataFrame({
    'feature': feature_names,
    'f_score': selector_f.scores_,
    'p_value': selector_f.pvalues_,
    'selected': selector_f.get_support()
}).sort_values('f_score', ascending=False)

print("\nSelectKBest (F-test) — Top 10:")
print(scores.head(10).to_string(index=False))

# Mutual Information (captures non-linear relationships)
mi_scores = mutual_info_classif(X_train, y_train, random_state=42)
mi_df = pd.DataFrame({
    'feature': feature_names,
    'mi_score': mi_scores
}).sort_values('mi_score', ascending=False)

print("\nMutual Information Scores — Top 10:")
print(mi_df.head(10).to_string(index=False))

# ============================================
# METHOD 4: RFE (Recursive Feature Elimination)
# ============================================
# Trains model, removes least important feature, repeats
# Like elimination rounds in a game show

rf = RandomForestClassifier(n_estimators=100, random_state=42)
rfe = RFE(estimator=rf, n_features_to_select=10, step=1)
rfe.fit(X_train, y_train)

rfe_results = pd.DataFrame({
    'feature': feature_names,
    'ranking': rfe.ranking_,
    'selected': rfe.support_
}).sort_values('ranking')

print("\nRFE Results (top 10 features selected):")
print(rfe_results[rfe_results['selected']].to_string(index=False))

# ============================================
# METHOD 5: Tree-based Feature Importance
# ============================================
rf.fit(X_train, y_train)
importances = pd.DataFrame({
    'feature': feature_names,
    'importance': rf.feature_importances_
}).sort_values('importance', ascending=False)

print("\nRandom Forest Feature Importance:")
print(importances.head(10).to_string(index=False))

# Visualize
import matplotlib.pyplot as plt
top_features = importances.head(15)
plt.figure(figsize=(10, 6))
plt.barh(top_features['feature'], top_features['importance'], color='steelblue')
plt.xlabel('Importance')
plt.title('Random Forest Feature Importance')
plt.gca().invert_yaxis()
plt.tight_layout()
plt.show()

# ============================================
# METHOD 6: L1 Regularization (Lasso) — Automatic feature selection
# ============================================
# Lasso automatically sets unimportant feature coefficients to ZERO
from sklearn.linear_model import LassoCV
from sklearn.preprocessing import StandardScaler

# Must scale features for Lasso
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)

lasso = LassoCV(cv=5, random_state=42)
lasso.fit(X_train_scaled, y_train)

lasso_results = pd.DataFrame({
    'feature': feature_names,
    'coefficient': np.abs(lasso.coef_),
    'selected': lasso.coef_ != 0
}).sort_values('coefficient', ascending=False)

print(f"\nLasso (alpha={lasso.alpha_:.4f}):")
print(f"  Features selected: {lasso_results['selected'].sum()} / {len(feature_names)}")
print(lasso_results[lasso_results['selected']].to_string(index=False))

# ============================================
# COMPARISON: Which features were selected by each method?
# ============================================
comparison = pd.DataFrame({
    'feature': feature_names,
    'SelectKBest': selector_f.get_support(),
    'RFE': rfe.support_,
    'RF_Important': importances['importance'].values > importances['importance'].median(),
    'Lasso': lasso.coef_ != 0
})
comparison['votes'] = comparison[['SelectKBest', 'RFE', 'RF_Important', 'Lasso']].sum(axis=1)
comparison = comparison.sort_values('votes', ascending=False)

print("\nFeature Selection Consensus (more votes = more likely important):")
print(comparison.to_string(index=False))

# Final selection: features selected by at least 3 methods
final_features = comparison[comparison['votes'] >= 3]['feature'].tolist()
print(f"\n✓ Final selected features ({len(final_features)}): {final_features}")
```

---

## 6.7 Handling Date and Time Features

### What It Is
Extracting meaningful features from datetime columns. A date like "2023-12-25 14:30:00" contains hidden information: it's a Monday, it's Christmas, it's afternoon, it's Q4 — all potentially predictive.

### Why It Matters
- Raw datetime values are useless to models (just a big number)
- Cyclical patterns (daily, weekly, monthly) drive many real-world phenomena
- Time-based features often among the **strongest predictors** in tabular data

### Code Examples

```python
import pandas as pd
import numpy as np

# ============================================
# Comprehensive DateTime Feature Engineering
# ============================================
# Sample: e-commerce orders
dates = pd.date_range('2022-01-01', periods=365*2, freq='D')
np.random.seed(42)
df = pd.DataFrame({
    'order_date': np.random.choice(dates, size=1000),
    'delivery_date': None  # Will calculate
})
df['delivery_date'] = df['order_date'] + pd.to_timedelta(
    np.random.randint(1, 14, size=1000), unit='D'
)

# ============================================
# 1. Basic Temporal Extractions
# ============================================
df['year'] = df['order_date'].dt.year
df['month'] = df['order_date'].dt.month
df['day'] = df['order_date'].dt.day
df['hour'] = df['order_date'].dt.hour
df['day_of_week'] = df['order_date'].dt.dayofweek     # 0=Monday, 6=Sunday
df['day_name'] = df['order_date'].dt.day_name()
df['week_of_year'] = df['order_date'].dt.isocalendar().week.astype(int)
df['quarter'] = df['order_date'].dt.quarter
df['day_of_year'] = df['order_date'].dt.dayofyear

# ============================================
# 2. Boolean / Binary Features
# ============================================
df['is_weekend'] = (df['day_of_week'] >= 5).astype(int)
df['is_month_start'] = df['order_date'].dt.is_month_start.astype(int)
df['is_month_end'] = df['order_date'].dt.is_month_end.astype(int)
df['is_quarter_start'] = df['order_date'].dt.is_quarter_start.astype(int)

# Custom: payday effect (assuming 1st and 15th)
df['is_payday_window'] = df['day'].isin([1, 2, 15, 16]).astype(int)

# Holiday flag (you'd use a holiday library in production)
# pip install holidays
# import holidays
# india_holidays = holidays.India(years=[2022, 2023])
# df['is_holiday'] = df['order_date'].isin(india_holidays).astype(int)

# ============================================
# 3. CYCLICAL ENCODING — Critical for time features!
# ============================================
# Problem: Month 12 and Month 1 are adjacent, but numerically far apart (12 vs 1)
# Solution: Encode as sine/cosine to preserve cyclical nature

# Month as cyclical (12-month cycle)
df['month_sin'] = np.sin(2 * np.pi * df['month'] / 12)
df['month_cos'] = np.cos(2 * np.pi * df['month'] / 12)

# Day of week as cyclical (7-day cycle)
df['dow_sin'] = np.sin(2 * np.pi * df['day_of_week'] / 7)
df['dow_cos'] = np.cos(2 * np.pi * df['day_of_week'] / 7)

# Hour as cyclical (24-hour cycle)
# df['hour_sin'] = np.sin(2 * np.pi * df['hour'] / 24)
# df['hour_cos'] = np.cos(2 * np.pi * df['hour'] / 24)

"""
Why Cyclical Encoding Works:
─────────────────────────────
Regular encoding:     Jan=1, Feb=2, ... Dec=12
Problem:              Distance(Dec, Jan) = 11, but they're 1 month apart!

Cyclical encoding:    Places months on a circle
                      Dec and Jan are now adjacent (as they should be!)

    Jan ●
   /       \
 Dec ●      ● Feb
  |           |
 Nov ●      ● Mar
  \       /
    ...
"""

# ============================================
# 4. Time-Difference Features (Duration / Recency)
# ============================================
# Delivery time (days between order and delivery)
df['delivery_days'] = (df['delivery_date'] - df['order_date']).dt.days

# Days since a reference date (recency)
reference = df['order_date'].max()
df['days_ago'] = (reference - df['order_date']).dt.days

# Time between events (for each customer — lag features)
# df['days_since_last_order'] = df.groupby('customer_id')['order_date'].diff().dt.days

# ============================================
# 5. Seasonal Features
# ============================================
# Season mapping (Northern Hemisphere)
def get_season(month):
    if month in [12, 1, 2]: return 'Winter'
    elif month in [3, 4, 5]: return 'Spring'
    elif month in [6, 7, 8]: return 'Summer'
    else: return 'Autumn'

df['season'] = df['month'].apply(get_season)

# Part of month
df['part_of_month'] = pd.cut(df['day'], bins=[0, 10, 20, 31], labels=['Beginning', 'Middle', 'End'])

print("DateTime features created:")
print(df.columns.tolist())
print(f"\nTotal features from 2 date columns: {len(df.columns) - 2}")
```

---

## 6.8 Text Feature Engineering

### What It Is
Converting text data into numerical features that capture meaning, sentiment, and structure.

### Code Examples

```python
import pandas as pd
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer, CountVectorizer

# Sample text data
texts = pd.Series([
    "This product is absolutely amazing! Best purchase ever.",
    "Terrible quality, broke after one day. Complete waste of money.",
    "It's okay, nothing special. Average product for the price.",
    "Exceeded all expectations! Will definitely buy again!",
    "Not worth the money. Very disappointed with the quality."
])

# ============================================
# 1. Basic Text Statistics (Simple but useful!)
# ============================================
df = pd.DataFrame({'text': texts})

df['char_count'] = df['text'].str.len()
df['word_count'] = df['text'].str.split().str.len()
df['avg_word_length'] = df['text'].apply(lambda x: np.mean([len(w) for w in x.split()]))
df['sentence_count'] = df['text'].str.count(r'[.!?]+')
df['exclamation_count'] = df['text'].str.count('!')
df['question_count'] = df['text'].str.count(r'\?')
df['capital_ratio'] = df['text'].apply(lambda x: sum(1 for c in x if c.isupper()) / len(x))
df['punctuation_ratio'] = df['text'].apply(
    lambda x: sum(1 for c in x if c in '!?.,;:') / len(x)
)

print("Text Statistics:")
print(df.drop('text', axis=1).round(3))

# ============================================
# 2. TF-IDF (Term Frequency - Inverse Document Frequency)
# ============================================
"""
TF-IDF scores words by:
  TF = How often the word appears in THIS document
  IDF = How rare the word is ACROSS ALL documents

High TF-IDF = word is common in this doc but rare overall = DISTINCTIVE word

Formula: TF-IDF(word, doc) = TF(word, doc) × log(N / DF(word))

Where:
  N = total number of documents
  DF(word) = number of documents containing the word
"""

tfidf = TfidfVectorizer(
    max_features=100,          # Keep top 100 words
    stop_words='english',      # Remove common words (the, is, and...)
    ngram_range=(1, 2),        # Include single words AND 2-word phrases
    min_df=1,                  # Word must appear in at least 1 doc
    max_df=0.95                # Ignore words in >95% of docs
)

tfidf_matrix = tfidf.fit_transform(texts)
tfidf_df = pd.DataFrame(
    tfidf_matrix.toarray(), 
    columns=tfidf.get_feature_names_out()
)

print(f"\nTF-IDF Features: {tfidf_df.shape[1]} dimensions")
print("Top words per document:")
for i in range(len(texts)):
    top_words = tfidf_df.iloc[i].nlargest(3)
    print(f"  Doc {i}: {list(top_words.index)}")

# ============================================
# 3. Sentiment-based Features (using simple lexicon)
# ============================================
# In production, use: from textblob import TextBlob
# or: from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer

positive_words = {'amazing', 'best', 'great', 'excellent', 'love', 'fantastic', 
                  'exceeded', 'definitely', 'wonderful', 'perfect'}
negative_words = {'terrible', 'worst', 'awful', 'hate', 'disappointed', 'broke',
                  'waste', 'poor', 'horrible', 'bad'}

def simple_sentiment_features(text):
    words = set(text.lower().split())
    pos_count = len(words & positive_words)
    neg_count = len(words & negative_words)
    return pos_count, neg_count, pos_count - neg_count

df[['positive_count', 'negative_count', 'sentiment_score']] = df['text'].apply(
    lambda x: pd.Series(simple_sentiment_features(x))
)

print("\nSentiment Features:")
print(df[['text', 'positive_count', 'negative_count', 'sentiment_score']].to_string())
```

---

## 6.9 Advanced Techniques

### Target Encoding with Regularization

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import KFold

def smoothed_target_encoding(train_df, test_df, col, target, smoothing=20, noise=0.01):
    """
    Production-grade target encoding with:
    1. Smoothing (pulls rare categories toward global mean)
    2. Cross-validation (prevents leakage on training data)
    3. Noise injection (prevents overfitting)
    """
    global_mean = train_df[target].mean()
    
    # For TEST data: use full training statistics
    agg = train_df.groupby(col)[target].agg(['mean', 'count'])
    smooth = (agg['count'] * agg['mean'] + smoothing * global_mean) / (agg['count'] + smoothing)
    test_encoded = test_df[col].map(smooth).fillna(global_mean)
    
    # For TRAIN data: use out-of-fold encoding (prevents leakage)
    train_encoded = pd.Series(index=train_df.index, dtype=float)
    kf = KFold(n_splits=5, shuffle=True, random_state=42)
    
    for train_idx, val_idx in kf.split(train_df):
        fold_agg = train_df.iloc[train_idx].groupby(col)[target].agg(['mean', 'count'])
        fold_smooth = (
            (fold_agg['count'] * fold_agg['mean'] + smoothing * global_mean) / 
            (fold_agg['count'] + smoothing)
        )
        train_encoded.iloc[val_idx] = train_df.iloc[val_idx][col].map(fold_smooth)
    
    train_encoded.fillna(global_mean, inplace=True)
    
    # Add small noise to prevent overfitting
    train_encoded += np.random.normal(0, noise, len(train_encoded))
    
    return train_encoded, test_encoded
```

### Polynomial and Interaction Features

```python
from sklearn.preprocessing import PolynomialFeatures
import pandas as pd
import numpy as np

# ============================================
# Polynomial Features (automated interaction creation)
# ============================================
# Creates all polynomial combinations up to degree d
# For features [a, b]: degree=2 creates [1, a, b, a², ab, b²]

X = pd.DataFrame({
    'height': [170, 165, 180, 175, 160],
    'weight': [70, 55, 85, 80, 50]
})

poly = PolynomialFeatures(degree=2, include_bias=False, interaction_only=False)
X_poly = poly.fit_transform(X)
poly_names = poly.get_feature_names_out(['height', 'weight'])

print(f"Original features: {X.shape[1]}")
print(f"Polynomial features: {X_poly.shape[1]}")
print(f"Feature names: {list(poly_names)}")
# ['height', 'weight', 'height^2', 'height weight', 'weight^2']

# ⚠️ WARNING: Polynomial features explode exponentially!
# 10 features, degree=2 → 65 features
# 10 features, degree=3 → 285 features
# Only use with feature selection afterwards!

# --- Interaction Only (no squares, just cross terms) ---
poly_inter = PolynomialFeatures(degree=2, include_bias=False, interaction_only=True)
X_inter = poly_inter.fit_transform(X)
inter_names = poly_inter.get_feature_names_out(['height', 'weight'])
print(f"\nInteraction only: {list(inter_names)}")
# ['height', 'weight', 'height weight']  ← Just the cross term

# ============================================
# Manual Domain-Specific Interactions
# ============================================
# Often better than automated polynomial features

# BMI (known medical formula)
X['bmi'] = X['weight'] / (X['height']/100) ** 2

# For housing data:
# df['price_per_sqft'] = df['price'] / df['area']
# df['room_density'] = df['rooms'] / df['area']
# df['bathroom_ratio'] = df['bathrooms'] / df['bedrooms']
```

### Feature Engineering for Time Series

```python
import pandas as pd
import numpy as np

# ============================================
# Lag Features and Rolling Statistics
# ============================================
# Essential for time series prediction

# Sample: daily sales data
np.random.seed(42)
dates = pd.date_range('2022-01-01', periods=365, freq='D')
sales = 100 + np.cumsum(np.random.randn(365)) + 20*np.sin(np.arange(365)*2*np.pi/7)
ts = pd.DataFrame({'date': dates, 'sales': sales})
ts.set_index('date', inplace=True)

# --- Lag features (yesterday's sales, last week's sales) ---
for lag in [1, 2, 3, 7, 14, 30]:
    ts[f'sales_lag_{lag}'] = ts['sales'].shift(lag)

# --- Rolling statistics (moving averages, volatility) ---
for window in [7, 14, 30]:
    ts[f'rolling_mean_{window}'] = ts['sales'].rolling(window).mean()
    ts[f'rolling_std_{window}'] = ts['sales'].rolling(window).std()
    ts[f'rolling_min_{window}'] = ts['sales'].rolling(window).min()
    ts[f'rolling_max_{window}'] = ts['sales'].rolling(window).max()

# --- Expanding features (cumulative) ---
ts['expanding_mean'] = ts['sales'].expanding().mean()

# --- Percent change ---
ts['pct_change_1d'] = ts['sales'].pct_change(1)
ts['pct_change_7d'] = ts['sales'].pct_change(7)

# --- Difference features ---
ts['diff_1d'] = ts['sales'].diff(1)
ts['diff_7d'] = ts['sales'].diff(7)

print(f"Time series features created: {len(ts.columns)} (from 1 column)")
print(ts.columns.tolist())

# ⚠️ IMPORTANT: Drop rows with NaN from lag/rolling (first few rows)
ts_clean = ts.dropna()
print(f"\nUsable rows: {len(ts_clean)} (lost {len(ts) - len(ts_clean)} to lag/rolling)")
```

---

## 6.10 Complete Feature Engineering Pipeline

### Production-Ready Pipeline

```python
import pandas as pd
import numpy as np
from sklearn.base import BaseEstimator, TransformerMixin
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder, OrdinalEncoder
from sklearn.impute import SimpleImputer

# ============================================
# Custom Feature Engineering Transformer
# ============================================
class FeatureEngineer(BaseEstimator, TransformerMixin):
    """
    Custom transformer that creates new features.
    Fits into sklearn Pipeline.
    """
    def __init__(self, create_interactions=True, create_ratios=True):
        self.create_interactions = create_interactions
        self.create_ratios = create_ratios
    
    def fit(self, X, y=None):
        # Store any statistics needed (e.g., for encoding)
        return self
    
    def transform(self, X):
        X = X.copy()
        
        if self.create_ratios:
            if 'total_spend' in X.columns and 'total_orders' in X.columns:
                X['avg_order_value'] = X['total_spend'] / X['total_orders'].replace(0, 1)
            
            if 'num_returns' in X.columns and 'total_orders' in X.columns:
                X['return_rate'] = X['num_returns'] / X['total_orders'].replace(0, 1)
        
        if self.create_interactions:
            numeric_cols = X.select_dtypes(include=[np.number]).columns[:5]  # Limit
            for i, col1 in enumerate(numeric_cols):
                for col2 in numeric_cols[i+1:]:
                    X[f'{col1}_x_{col2}'] = X[col1] * X[col2]
        
        return X

# ============================================
# Full Pipeline with ColumnTransformer
# ============================================
# Define column groups
numeric_features = ['age', 'income', 'experience', 'total_spend']
categorical_features = ['city', 'department', 'education']
ordinal_features = ['satisfaction_level']  # Low, Medium, High

# Numeric pipeline: impute → scale
numeric_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', StandardScaler())
])

# Categorical pipeline: impute → one-hot encode
categorical_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='most_frequent')),
    ('encoder', OneHotEncoder(handle_unknown='ignore', sparse_output=False))
])

# Ordinal pipeline: impute → ordinal encode
ordinal_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='most_frequent')),
    ('encoder', OrdinalEncoder(categories=[['Low', 'Medium', 'High']]))
])

# Combine all transformers
preprocessor = ColumnTransformer(
    transformers=[
        ('num', numeric_pipeline, numeric_features),
        ('cat', categorical_pipeline, categorical_features),
        ('ord', ordinal_pipeline, ordinal_features)
    ],
    remainder='drop'  # Drop columns not specified
)

# Full pipeline: feature engineering → preprocessing → model
from sklearn.ensemble import GradientBoostingClassifier

full_pipeline = Pipeline([
    ('feature_eng', FeatureEngineer(create_interactions=False)),
    ('preprocessor', preprocessor),
    ('model', GradientBoostingClassifier(n_estimators=100, random_state=42))
])

# Usage:
# full_pipeline.fit(X_train, y_train)
# predictions = full_pipeline.predict(X_test)
# score = full_pipeline.score(X_test, y_test)

print("Pipeline steps:")
for name, step in full_pipeline.steps:
    print(f"  {name}: {step.__class__.__name__}")
```

---

## Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Feature engineering after train/test split | Leaking test info into features | Split first, engineer on train, apply to test |
| Creating too many features without selection | Overfitting, slow training | Always follow up with feature selection |
| Using target info in features | Data leakage → unrealistic accuracy | Only use features available at prediction time |
| Not handling unknown categories | Model crashes on new data | Use `handle_unknown='ignore'` or 'Other' category |
| One-hot encoding high-cardinality features | Memory explosion (1000+ columns) | Use target/frequency encoding instead |
| Scaling before splitting | Test data statistics contaminate train | Fit scaler on train only |
| Ignoring cyclical nature of time | Dec and Jan appear far apart | Use sin/cos encoding |
| Not documenting feature definitions | Can't reproduce, can't debug | Maintain a feature dictionary |

---

## Interview Questions

1. **What is feature engineering and why is it important?**
   - Transforming raw data into informative features for ML models
   - Often more impactful than model selection; captures domain knowledge

2. **How would you handle a categorical feature with 10,000 unique values?**
   - Target encoding (with smoothing + CV to avoid leakage)
   - Frequency encoding
   - Group rare categories into "Other"
   - Embedding (if using neural networks)
   - Hash encoding (fixed-dimension)

3. **When should you use StandardScaler vs MinMaxScaler?**
   - StandardScaler: data is approximately normal, you want mean=0/std=1
   - MinMaxScaler: need bounded [0,1] range, uniform distribution, image data
   - RobustScaler: data has outliers (uses median/IQR)

4. **What's the difference between feature selection and feature extraction?**
   - Selection: choose a subset of existing features (keeps interpretability)
   - Extraction: create new features from existing ones (PCA, autoencoders — may lose interpretability)

5. **How do you prevent data leakage in feature engineering?**
   - Split before any engineering; fit on train, transform test
   - Don't use future data for predictions (time series: no future lags)
   - Target encoding must use cross-validation on training data
   - Statistics (mean for imputation) only from training set

6. **Explain cyclical encoding for time features.**
   - Time features are cyclical: hour 23 and hour 0 are 1 hour apart, not 23
   - Encode as: $sin(\frac{2\pi \cdot x}{period})$ and $cos(\frac{2\pi \cdot x}{period})$
   - Need BOTH sin and cos to uniquely identify position on the circle

7. **How would you approach feature engineering for a new dataset?**
   - EDA first → understand distributions, relationships
   - Domain knowledge → create meaningful ratios, differences, aggregations
   - Temporal features → if dates exist, extract everything
   - Encoding → appropriate method based on cardinality
   - Scaling → based on model choice
   - Selection → remove irrelevant/redundant features
   - Validate → check for leakage, test model performance

---

## Quick Reference

| Task | Method | Code |
|------|--------|------|
| Create ratio | Division | `df['ratio'] = df['a'] / df['b']` |
| Bin numeric | Cut/Qcut | `pd.cut(df['col'], bins=5)` |
| Log transform | Log1p | `np.log1p(df['col'])` |
| One-hot encode | Dummies | `pd.get_dummies(df['col'], drop_first=True)` |
| Target encode | Group mean | `df.groupby('cat')['target'].transform('mean')` |
| Standard scale | Z-score | `StandardScaler().fit_transform(X)` |
| Cyclical encode | Sin/Cos | `np.sin(2*np.pi*df['hour']/24)` |
| Lag features | Shift | `df['lag_1'] = df['value'].shift(1)` |
| Rolling stats | Rolling | `df['col'].rolling(7).mean()` |
| Remove correlated | Threshold | Drop one if `abs(corr) > 0.85` |
| Feature importance | Tree model | `rf.feature_importances_` |
| Auto polynomial | PolynomialFeatures | `PolynomialFeatures(degree=2)` |
| Handle unknown cats | Encoder param | `OneHotEncoder(handle_unknown='ignore')` |
| Full pipeline | ColumnTransformer | `ColumnTransformer([('num', pipe, cols)])` |

---

*Previous: [Chapter 05 - Exploratory Data Analysis](05-Exploratory-Data-Analysis.md)*  
*Next: [Chapter 07 - Data Pipelines and ETL](07-Data-Pipelines-and-ETL.md)*
