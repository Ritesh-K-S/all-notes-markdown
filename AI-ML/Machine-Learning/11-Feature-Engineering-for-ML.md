# Chapter 11: Feature Engineering for ML

## Table of Contents
- [11.1 Introduction to Feature Engineering](#111-introduction-to-feature-engineering)
- [11.2 Handling Categorical Variables](#112-handling-categorical-variables)
- [11.3 Feature Scaling and Normalization](#113-feature-scaling-and-normalization)
- [11.4 Handling Missing Values](#114-handling-missing-values)
- [11.5 Feature Transformation](#115-feature-transformation)
- [11.6 Feature Creation](#116-feature-creation)
- [11.7 Feature Selection](#117-feature-selection)
- [11.8 Feature Engineering for Specific Data Types](#118-feature-engineering-for-specific-data-types)
- [11.9 Pipelines for Production](#119-pipelines-for-production)
- [11.10 Common Mistakes](#1110-common-mistakes)
- [11.11 Interview Questions](#1111-interview-questions)
- [11.12 Quick Reference](#1112-quick-reference)

---

## 11.1 Introduction to Feature Engineering

### What It Is
Feature engineering is the art and science of transforming raw data into features that better represent the underlying problem to your ML model. Think of it as translating real-world information into a language that algorithms can understand.

**Analogy**: Imagine you're teaching a robot to predict house prices. The raw address "123 Oak Street" means nothing to it. But if you convert it to "distance from city center = 5km, crime rate = low, school rating = 8/10" — now the robot can learn!

### Why It Matters
- **"Applied ML is basically feature engineering"** — Andrew Ng
- A simple model with great features beats a complex model with bad features
- Features determine the ceiling of model performance; algorithms just get you closer to that ceiling
- Often the difference between a 70% model and a 95% model

### The Feature Engineering Workflow

```
Raw Data → Clean → Transform → Create → Select → Model
   │          │        │          │         │
   │     Handle NA  Encode    Combine   Remove
   │     Fix types  Scale     Domain    redundant
   │     Outliers   Normalize knowledge features
   │
   └── "Garbage in, garbage out"
```

> **Key Insight**: Feature engineering is where domain expertise meets data science. A healthcare expert knows that BMI matters more than raw weight. A finance expert knows that debt-to-income ratio matters more than raw income.

---

## 11.2 Handling Categorical Variables

### 11.2.1 Label Encoding

Assigns each category a unique integer. **Only for ordinal data** (has natural order).

```python
from sklearn.preprocessing import LabelEncoder, OrdinalEncoder
import pandas as pd
import numpy as np

# Example: Education level (ordinal - has natural order)
df = pd.DataFrame({
    'education': ['High School', 'Bachelor', 'Master', 'PhD', 'Bachelor', 'High School']
})

# Method 1: LabelEncoder (single column)
le = LabelEncoder()
df['education_encoded'] = le.fit_transform(df['education'])
print(df)
# Output: High School=1, Bachelor=0, Master=2, PhD=3 (alphabetical, NOT meaningful!)

# Method 2: OrdinalEncoder with specified order (CORRECT approach)
order = [['High School', 'Bachelor', 'Master', 'PhD']]  # Lowest to highest
oe = OrdinalEncoder(categories=order)
df['education_ordinal'] = oe.fit_transform(df[['education']])
print(df)
# Output: High School=0, Bachelor=1, Master=2, PhD=3 (meaningful order!)
```

> **Warning**: NEVER use label encoding for nominal data (no natural order like color, city). The model will assume Red=1 < Blue=2 < Green=3, which is meaningless.

---

### 11.2.2 One-Hot Encoding

Creates a binary column for each category. **For nominal (no order) data**.

```python
from sklearn.preprocessing import OneHotEncoder

df = pd.DataFrame({
    'color': ['Red', 'Blue', 'Green', 'Blue', 'Red'],
    'size': ['S', 'M', 'L', 'M', 'S']
})

# Method 1: pandas get_dummies (quick and easy)
df_encoded = pd.get_dummies(df, columns=['color'], prefix='color')
print(df_encoded)
# Output: color_Blue, color_Green, color_Red columns (0 or 1)

# Method 2: sklearn OneHotEncoder (for pipelines)
ohe = OneHotEncoder(sparse_output=False, drop='first')  # drop='first' avoids multicollinearity
encoded = ohe.fit_transform(df[['color']])
print(f"Feature names: {ohe.get_feature_names_out()}")
print(encoded)
```

**When to drop one column** (`drop='first'`):
- Linear models: YES (avoids multicollinearity / dummy variable trap)
- Tree models: NO (they handle it naturally)

**Problem with high cardinality**: If a column has 1000 unique values, one-hot creates 1000 columns!

---

### 11.2.3 Target Encoding (Mean Encoding)

Replaces each category with the mean of the target variable for that category. Great for high cardinality.

```python
from sklearn.preprocessing import TargetEncoder

# City with many unique values (high cardinality)
df = pd.DataFrame({
    'city': ['NYC', 'LA', 'NYC', 'Chicago', 'LA', 'NYC', 'Chicago', 'LA'],
    'price': [500000, 400000, 550000, 300000, 420000, 480000, 310000, 380000]
})

# Target encoding: replace city with mean price for that city
te = TargetEncoder(smooth='auto')
df['city_encoded'] = te.fit_transform(df[['city']], df['price'])
print(df[['city', 'city_encoded', 'price']])
# NYC → ~510000, LA → ~400000, Chicago → ~305000

# Manual implementation with smoothing (to prevent overfitting)
def target_encode_smooth(df, col, target, alpha=10):
    """Target encoding with global mean smoothing"""
    global_mean = df[target].mean()
    stats = df.groupby(col)[target].agg(['mean', 'count'])
    # Smoothing: blend category mean with global mean
    # More samples → trust category mean more
    smooth = (stats['count'] * stats['mean'] + alpha * global_mean) / (stats['count'] + alpha)
    return df[col].map(smooth)
```

> **Warning**: Target encoding MUST be fit on training data only! Fitting on full data causes severe data leakage.

---

### 11.2.4 Frequency Encoding

Replace categories with their frequency of occurrence:

```python
# Frequency encoding
freq = df['city'].value_counts(normalize=True)  # Proportions
df['city_freq'] = df['city'].map(freq)
print(df[['city', 'city_freq']])
# NYC: 0.375, LA: 0.375, Chicago: 0.25
```

---

### 11.2.5 Binary Encoding

Converts categories to binary, then splits into columns. Efficient for moderate cardinality (10-100 categories).

```python
# pip install category_encoders
import category_encoders as ce

# 8 cities → only 3 binary columns needed (2³=8)
encoder = ce.BinaryEncoder(cols=['city'])
df_binary = encoder.fit_transform(df[['city']])
print(df_binary)
```

### Encoding Selection Guide

| Method | Cardinality | Data Type | Models |
|--------|-------------|-----------|--------|
| One-Hot | Low (< 15) | Nominal | All |
| Ordinal | Any | Ordinal | All |
| Target | High (> 15) | Nominal | All (careful of leakage) |
| Frequency | Any | Nominal | Tree-based |
| Binary | Medium (10-100) | Nominal | All |
| Hash | Very High (> 1000) | Nominal | All (some info loss) |

---

## 11.3 Feature Scaling and Normalization

### Why Scaling Matters

```
Without scaling:              With scaling:
Feature A: [1, 2, 3]         Feature A: [0.0, 0.5, 1.0]
Feature B: [1000, 2000, 3000] Feature B: [0.0, 0.5, 1.0]

Distance calculation dominated    Equal contribution from
by Feature B!                     both features
```

**Models that NEED scaling**: KNN, SVM, Neural Networks, PCA, Logistic Regression, K-Means
**Models that DON'T need scaling**: Decision Trees, Random Forest, XGBoost, LightGBM

---

### 11.3.1 StandardScaler (Z-score Normalization)

$$z = \frac{x - \mu}{\sigma}$$

Transforms to mean=0, std=1. Best for normally distributed features.

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_train)  # Fit on TRAIN only!
X_test_scaled = scaler.transform(X_test)  # Transform test with TRAIN stats!

# After scaling:
print(f"Mean: {X_scaled.mean(axis=0)[:3]}")  # ≈ [0, 0, 0]
print(f"Std:  {X_scaled.std(axis=0)[:3]}")   # ≈ [1, 1, 1]
```

---

### 11.3.2 MinMaxScaler

$$x_{scaled} = \frac{x - x_{min}}{x_{max} - x_{min}}$$

Transforms to range [0, 1]. Best when you need bounded values.

```python
from sklearn.preprocessing import MinMaxScaler

scaler = MinMaxScaler(feature_range=(0, 1))  # Can change range
X_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)

# After scaling: all values between 0 and 1
print(f"Min: {X_scaled.min(axis=0)[:3]}")  # [0, 0, 0]
print(f"Max: {X_scaled.max(axis=0)[:3]}")  # [1, 1, 1]
```

> **Warning**: MinMaxScaler is sensitive to outliers! One extreme value compresses all other values.

---

### 11.3.3 RobustScaler

$$x_{scaled} = \frac{x - \text{median}}{IQR}$$

Uses median and interquartile range. **Best for data with outliers**.

```python
from sklearn.preprocessing import RobustScaler

# Handles outliers gracefully
scaler = RobustScaler()
X_scaled = scaler.fit_transform(X_train)
```

---

### 11.3.4 Normalizer (L2 Normalization)

Scales each **sample** (row) to unit norm. Used in text/NLP.

$$x_{normalized} = \frac{x}{||x||_2}$$

```python
from sklearn.preprocessing import Normalizer

# Each row has unit length (L2 norm = 1)
normalizer = Normalizer(norm='l2')
X_normalized = normalizer.fit_transform(X_train)
```

---

### Scaling Comparison

| Scaler | Handles Outliers | Range | Best For |
|--------|:---:|-------|----------|
| StandardScaler | ❌ | Unbounded | Gaussian-like features |
| MinMaxScaler | ❌ | [0, 1] | Neural networks, bounded features |
| RobustScaler | ✅ | Unbounded | Data with outliers |
| MaxAbsScaler | ❌ | [-1, 1] | Sparse data |
| Normalizer | - | Unit norm | Text/TF-IDF vectors |

---

## 11.4 Handling Missing Values

### 11.4.1 Understanding Missingness

| Type | Meaning | Example | Strategy |
|------|---------|---------|----------|
| MCAR | Missing Completely At Random | Sensor randomly fails | Any imputation |
| MAR | Missing At Random (depends on other features) | Rich people don't report income | Model-based imputation |
| MNAR | Missing Not At Random (depends on missing value itself) | Severely ill skip health surveys | Domain knowledge needed |

---

### 11.4.2 Simple Imputation

```python
from sklearn.impute import SimpleImputer
import pandas as pd
import numpy as np

df = pd.DataFrame({
    'age': [25, 30, np.nan, 45, np.nan, 35],
    'salary': [50000, np.nan, 70000, np.nan, 60000, 55000],
    'city': ['NYC', np.nan, 'LA', 'NYC', 'Chicago', np.nan]
})

# Numerical: mean, median, or constant
num_imputer = SimpleImputer(strategy='median')  # median is robust to outliers
df[['age', 'salary']] = num_imputer.fit_transform(df[['age', 'salary']])

# Categorical: most_frequent or constant
cat_imputer = SimpleImputer(strategy='most_frequent')
df[['city']] = cat_imputer.fit_transform(df[['city']])

# Or use constant fill
const_imputer = SimpleImputer(strategy='constant', fill_value='Unknown')
```

---

### 11.4.3 Advanced Imputation (KNN and Iterative)

```python
from sklearn.impute import KNNImputer, IterativeImputer

# KNN Imputer: fills missing values using K nearest neighbors
knn_imp = KNNImputer(n_neighbors=5, weights='distance')
X_imputed = knn_imp.fit_transform(X_train)

# Iterative Imputer (MICE - Multiple Imputation by Chained Equations)
# Models each feature with missing values as a function of other features
from sklearn.experimental import enable_iterative_imputer  # noqa
iter_imp = IterativeImputer(max_iter=10, random_state=42)
X_imputed = iter_imp.fit_transform(X_train)
```

---

### 11.4.4 Missing Value Indicators

Sometimes the FACT that a value is missing is informative:

```python
from sklearn.impute import SimpleImputer, MissingIndicator
from sklearn.pipeline import FeatureUnion

# Create both imputed values AND missing indicators
imputer = SimpleImputer(strategy='median')
indicator = MissingIndicator()

# Combine: original features (imputed) + binary "was_missing" features
X_imputed = imputer.fit_transform(X_train)
X_missing_flags = indicator.fit_transform(X_train)

# Or in a pipeline
X_combined = np.hstack([X_imputed, X_missing_flags])
```

> **Pro Tip**: For tree-based models (XGBoost, LightGBM), you can often leave NaN values as-is! They handle missing values natively and learn optimal split directions for them.

---

## 11.5 Feature Transformation

### 11.5.1 Log Transformation

Handles right-skewed distributions (income, prices, counts):

```python
import numpy as np

# Before: highly skewed (long right tail)
# After: more normal distribution
df['income_log'] = np.log1p(df['income'])  # log1p = log(1+x), handles zeros

# Inverse transform when needed
original = np.expm1(df['income_log'])  # expm1 = exp(x) - 1
```

```
Before (skewed):           After log transform:
   │                          │     ╭─╮
   │ ╭╮                      │    ╭╯  ╰╮
   │╭╯╰╮                     │   ╭╯    ╰╮
   │╯   ╰────────────        │  ╭╯      ╰╮
   └──────────────────        │ ╭╯        ╰──
                              └───────────────
   Most values clustered      More spread out,
   on left, long tail         symmetric distribution
```

---

### 11.5.2 Power Transforms (Box-Cox and Yeo-Johnson)

```python
from sklearn.preprocessing import PowerTransformer

# Yeo-Johnson: works with negative values too
pt = PowerTransformer(method='yeo-johnson')
X_transformed = pt.fit_transform(X_train)

# Box-Cox: only works with strictly positive values
pt_bc = PowerTransformer(method='box-cox')
X_transformed = pt_bc.fit_transform(X_train[X_train > 0])  # Must be positive!
```

---

### 11.5.3 Binning (Discretization)

Convert continuous features into categorical bins:

```python
from sklearn.preprocessing import KBinsDiscretizer

# Equal-width binning
binner = KBinsDiscretizer(n_bins=5, encode='ordinal', strategy='uniform')
df['age_binned'] = binner.fit_transform(df[['age']])

# Equal-frequency binning (same number of samples per bin)
binner_q = KBinsDiscretizer(n_bins=5, encode='ordinal', strategy='quantile')
df['income_binned'] = binner_q.fit_transform(df[['income']])

# Custom bins with pandas
df['age_group'] = pd.cut(df['age'], bins=[0, 18, 35, 50, 65, 100],
                         labels=['Teen', 'Young', 'Middle', 'Senior', 'Elderly'])
```

**When to bin:**
- Non-linear relationships (age vs. insurance cost)
- Reducing noise in continuous features
- Making features more interpretable
- When exact value matters less than range

---

### 11.5.4 Polynomial Features

Create interaction and polynomial terms:

$$[x_1, x_2] \rightarrow [1, x_1, x_2, x_1^2, x_1 x_2, x_2^2]$$

```python
from sklearn.preprocessing import PolynomialFeatures

# Creates x1², x2², x1*x2 (interactions + powers)
poly = PolynomialFeatures(degree=2, include_bias=False, interaction_only=False)
X_poly = poly.fit_transform(X_train)

print(f"Original features: {X_train.shape[1]}")
print(f"After polynomial:  {X_poly.shape[1]}")
print(f"Feature names: {poly.get_feature_names_out()[:10]}")

# interaction_only=True → only x1*x2, no x1², x2²
poly_int = PolynomialFeatures(degree=2, include_bias=False, interaction_only=True)
X_interactions = poly_int.fit_transform(X_train)
```

> **Warning**: Polynomial features grow combinatorially! 20 features with degree=3 → 1,771 features. Use with regularization (Lasso/Ridge).

---

## 11.6 Feature Creation

### 11.6.1 Domain-Specific Feature Engineering

```python
# E-commerce example
df['total_spend'] = df['quantity'] * df['unit_price']
df['avg_order_value'] = df['total_spend'] / df['num_orders']
df['days_since_last_purchase'] = (pd.Timestamp.now() - df['last_purchase']).dt.days
df['is_weekend_purchase'] = df['purchase_date'].dt.dayofweek >= 5

# Real estate example
df['price_per_sqft'] = df['price'] / df['sqft']
df['rooms_per_sqft'] = df['total_rooms'] / df['sqft']
df['age_of_house'] = 2026 - df['year_built']
df['is_renovated'] = (df['year_renovated'] > df['year_built']).astype(int)

# Finance example
df['debt_to_income'] = df['total_debt'] / df['annual_income']
df['credit_utilization'] = df['balance'] / df['credit_limit']
df['payment_to_income'] = df['monthly_payment'] / (df['annual_income'] / 12)
```

---

### 11.6.2 Aggregation Features (Group Statistics)

```python
# Customer transaction data → customer-level features
customer_features = df.groupby('customer_id').agg(
    total_transactions=('transaction_id', 'count'),
    total_amount=('amount', 'sum'),
    avg_amount=('amount', 'mean'),
    max_amount=('amount', 'max'),
    std_amount=('amount', 'std'),
    unique_products=('product_id', 'nunique'),
    first_purchase=('date', 'min'),
    last_purchase=('date', 'max'),
).reset_index()

# Merge back
df = df.merge(customer_features, on='customer_id', how='left')
```

---

### 11.6.3 Window/Lag Features (Time-based)

```python
# Rolling statistics
df['price_ma_7'] = df['price'].rolling(window=7).mean()   # 7-day moving average
df['price_ma_30'] = df['price'].rolling(window=30).mean() # 30-day moving average
df['price_std_7'] = df['price'].rolling(window=7).std()   # 7-day volatility

# Lag features (previous values)
df['price_lag_1'] = df['price'].shift(1)    # Yesterday's price
df['price_lag_7'] = df['price'].shift(7)    # Last week's price

# Difference features
df['price_diff'] = df['price'].diff()                    # Day-over-day change
df['price_pct_change'] = df['price'].pct_change()        # Percentage change

# Ratio to moving average
df['price_vs_ma7'] = df['price'] / df['price_ma_7']      # Above/below trend
```

---

### 11.6.4 Text Features

```python
from sklearn.feature_extraction.text import TfidfVectorizer, CountVectorizer

texts = ["machine learning is great", "deep learning is powerful", "ML and DL are AI"]

# TF-IDF (most common for traditional ML)
tfidf = TfidfVectorizer(max_features=100, ngram_range=(1, 2))  # Unigrams + bigrams
X_text = tfidf.fit_transform(texts)
print(f"Vocabulary size: {len(tfidf.vocabulary_)}")
print(f"Feature names: {tfidf.get_feature_names_out()[:10]}")

# Simple text statistics
df['text_length'] = df['text'].str.len()
df['word_count'] = df['text'].str.split().str.len()
df['avg_word_length'] = df['text'].apply(lambda x: np.mean([len(w) for w in x.split()]))
df['has_url'] = df['text'].str.contains(r'http', regex=True).astype(int)
df['num_uppercase'] = df['text'].str.count(r'[A-Z]')
```

---

### 11.6.5 Date/Time Features

```python
# Extract all useful components from datetime
df['date'] = pd.to_datetime(df['date'])

df['year'] = df['date'].dt.year
df['month'] = df['date'].dt.month
df['day'] = df['date'].dt.day
df['day_of_week'] = df['date'].dt.dayofweek      # Monday=0, Sunday=6
df['is_weekend'] = (df['date'].dt.dayofweek >= 5).astype(int)
df['quarter'] = df['date'].dt.quarter
df['hour'] = df['date'].dt.hour
df['is_month_start'] = df['date'].dt.is_month_start.astype(int)
df['is_month_end'] = df['date'].dt.is_month_end.astype(int)

# Cyclical encoding for periodic features (so Dec and Jan are "close")
df['month_sin'] = np.sin(2 * np.pi * df['month'] / 12)
df['month_cos'] = np.cos(2 * np.pi * df['month'] / 12)
df['hour_sin'] = np.sin(2 * np.pi * df['hour'] / 24)
df['hour_cos'] = np.cos(2 * np.pi * df['hour'] / 24)
```

> **Pro Tip**: Cyclical encoding prevents the model from thinking December (12) and January (1) are far apart. sin/cos encoding makes them adjacent on a circle.

---

## 11.7 Feature Selection

### Why Select Features?
- Reduces overfitting (fewer features = simpler model)
- Faster training and inference
- Improved interpretability
- Removes noise features that hurt performance

---

### 11.7.1 Filter Methods (Statistical Tests)

Fast, model-independent. Evaluate features individually.

```python
from sklearn.feature_selection import (SelectKBest, f_classif, mutual_info_classif,
                                        chi2, f_regression)

# For classification: ANOVA F-test
selector = SelectKBest(score_func=f_classif, k=10)  # Keep top 10 features
X_selected = selector.fit_transform(X_train, y_train)

# See which features were selected
selected_mask = selector.get_support()
selected_features = np.array(feature_names)[selected_mask]
print(f"Selected features: {selected_features}")

# Feature scores (higher = more important)
scores = pd.DataFrame({
    'feature': feature_names,
    'score': selector.scores_,
    'p_value': selector.pvalues_
}).sort_values('score', ascending=False)
print(scores.head(10))

# For classification: Mutual Information (captures non-linear relationships)
mi_selector = SelectKBest(score_func=mutual_info_classif, k=10)
X_mi = mi_selector.fit_transform(X_train, y_train)

# For regression: f_regression or mutual_info_regression
from sklearn.feature_selection import mutual_info_regression
selector_reg = SelectKBest(score_func=f_regression, k=10)
```

| Score Function | Data Type | Captures Non-linear? |
|---------------|-----------|:---:|
| f_classif (ANOVA) | Classification | ❌ |
| chi2 | Classification (non-negative) | ❌ |
| mutual_info_classif | Classification | ✅ |
| f_regression | Regression | ❌ |
| mutual_info_regression | Regression | ✅ |

---

### 11.7.2 Wrapper Methods (RFE - Recursive Feature Elimination)

Trains model, removes least important features, repeats:

```python
from sklearn.feature_selection import RFE, RFECV
from sklearn.ensemble import RandomForestClassifier

# RFE with fixed number of features
model = RandomForestClassifier(n_estimators=100, random_state=42)
rfe = RFE(estimator=model, n_features_to_select=10, step=1)
rfe.fit(X_train, y_train)

# Which features were selected?
selected = np.array(feature_names)[rfe.support_]
print(f"Selected features: {selected}")
print(f"Feature ranking: {rfe.ranking_}")  # 1 = selected

# RFECV: Automatically finds optimal number of features using CV
rfecv = RFECV(estimator=model, step=1, cv=5, scoring='f1', n_jobs=-1)
rfecv.fit(X_train, y_train)
print(f"Optimal number of features: {rfecv.n_features_}")
print(f"CV Score with optimal features: {rfecv.cv_results_['mean_test_score'].max():.4f}")
```

---

### 11.7.3 Embedded Methods (L1 Regularization & Feature Importance)

Features selected during model training:

```python
from sklearn.linear_model import Lasso, LogisticRegression
from sklearn.feature_selection import SelectFromModel

# Method 1: L1 (Lasso) Regularization — sets irrelevant feature weights to exactly 0
lasso = LogisticRegression(penalty='l1', solver='liblinear', C=0.1, random_state=42)
lasso.fit(X_train, y_train)

# Features with non-zero coefficients are selected
important_features = np.array(feature_names)[lasso.coef_[0] != 0]
print(f"Features selected by L1: {len(important_features)}")

# Method 2: Tree-based Feature Importance
from sklearn.ensemble import RandomForestClassifier

rf = RandomForestClassifier(n_estimators=100, random_state=42)
rf.fit(X_train, y_train)

# Plot feature importances
importances = pd.DataFrame({
    'feature': feature_names,
    'importance': rf.feature_importances_
}).sort_values('importance', ascending=False)
print(importances.head(15))

# Select features above threshold
selector = SelectFromModel(rf, threshold='median')  # Keep above-median importance
X_selected = selector.fit_transform(X_train, y_train)
print(f"Features kept: {selector.transform(X_train).shape[1]}")
```

---

### 11.7.4 Permutation Importance

More reliable than tree's built-in importance (which is biased toward high-cardinality features):

```python
from sklearn.inspection import permutation_importance

# Fit model first
rf.fit(X_train, y_train)

# Calculate permutation importance on VALIDATION set
perm_importance = permutation_importance(
    rf, X_test, y_test, 
    n_repeats=10, 
    random_state=42,
    scoring='f1'
)

# Sort and display
perm_df = pd.DataFrame({
    'feature': feature_names,
    'importance_mean': perm_importance.importances_mean,
    'importance_std': perm_importance.importances_std
}).sort_values('importance_mean', ascending=False)
print(perm_df.head(15))
```

---

### 11.7.5 Correlation-Based Removal

Remove highly correlated features (they provide redundant info):

```python
def remove_correlated_features(df, threshold=0.95):
    """Remove features that are >threshold correlated with each other"""
    corr_matrix = df.corr().abs()
    upper = corr_matrix.where(np.triu(np.ones(corr_matrix.shape), k=1).astype(bool))
    
    # Find features with correlation > threshold
    to_drop = [col for col in upper.columns if any(upper[col] > threshold)]
    print(f"Removing {len(to_drop)} highly correlated features: {to_drop}")
    return df.drop(columns=to_drop)

df_cleaned = remove_correlated_features(df_features, threshold=0.95)
```

---

### 11.7.6 Variance Threshold

Remove features with very low variance (near-constant):

```python
from sklearn.feature_selection import VarianceThreshold

# Remove features where >95% of values are the same
selector = VarianceThreshold(threshold=0.01)  # Adjust threshold
X_selected = selector.fit_transform(X_train)
print(f"Features removed: {X_train.shape[1] - X_selected.shape[1]}")
```

---

### Feature Selection Strategy

```
Start
  │
  ├── Remove constant/near-constant features (VarianceThreshold)
  │
  ├── Remove highly correlated features (correlation > 0.95)
  │
  ├── Quick filter (SelectKBest with mutual_info) → Remove obvious noise
  │
  ├── Train model with remaining features
  │
  ├── Check permutation importance → Remove unimportant features
  │
  └── Optional: RFECV for final fine-tuning
```

---

## 11.8 Feature Engineering for Specific Data Types

### 11.8.1 Geospatial Features

```python
from math import radians, sin, cos, sqrt, atan2

def haversine_distance(lat1, lon1, lat2, lon2):
    """Calculate distance between two GPS coordinates in km"""
    R = 6371  # Earth's radius in km
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    c = 2 * atan2(sqrt(a), sqrt(1-a))
    return R * c

# Distance to city center
df['dist_to_center'] = df.apply(
    lambda row: haversine_distance(row['lat'], row['lon'], city_center_lat, city_center_lon), 
    axis=1
)

# Cluster-based features
from sklearn.cluster import KMeans
coords = df[['lat', 'lon']].values
kmeans = KMeans(n_clusters=10, random_state=42).fit(coords)
df['geo_cluster'] = kmeans.labels_
```

---

### 11.8.2 IP Address / Network Features

```python
import ipaddress

def extract_ip_features(ip_str):
    """Extract features from IP address"""
    ip = ipaddress.ip_address(ip_str)
    return {
        'is_private': ip.is_private,
        'ip_first_octet': int(ip_str.split('.')[0]),
        'ip_class': 'A' if int(ip_str.split('.')[0]) < 128 else 'B'
    }
```

---

## 11.9 Pipelines for Production

### Why Pipelines?
- Prevent data leakage (fit only on train)
- Reproducible transformations
- Easy deployment (serialize entire pipeline)
- Clean, readable code

```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.ensemble import RandomForestClassifier

# Define column types
numeric_features = ['age', 'income', 'credit_score']
categorical_features = ['city', 'education', 'employment']

# Numeric pipeline: impute → scale
numeric_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', StandardScaler())
])

# Categorical pipeline: impute → encode
categorical_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='most_frequent')),
    ('encoder', OneHotEncoder(handle_unknown='ignore', sparse_output=False))
])

# Combine into ColumnTransformer
preprocessor = ColumnTransformer([
    ('num', numeric_pipeline, numeric_features),
    ('cat', categorical_pipeline, categorical_features)
])

# Full pipeline: preprocess → model
full_pipeline = Pipeline([
    ('preprocessor', preprocessor),
    ('classifier', RandomForestClassifier(n_estimators=100, random_state=42))
])

# Fit and predict (everything handled automatically!)
full_pipeline.fit(X_train, y_train)
y_pred = full_pipeline.predict(X_test)
score = full_pipeline.score(X_test, y_test)
print(f"Accuracy: {score:.4f}")

# Cross-validation works with pipeline (no leakage!)
from sklearn.model_selection import cross_val_score
scores = cross_val_score(full_pipeline, X, y, cv=5, scoring='f1')
print(f"CV F1: {scores.mean():.4f} ± {scores.std():.4f}")

# Save pipeline for deployment
import joblib
joblib.dump(full_pipeline, 'model_pipeline.pkl')

# Load and use in production
loaded_pipeline = joblib.load('model_pipeline.pkl')
prediction = loaded_pipeline.predict(new_data)
```

---

### Custom Transformers

```python
from sklearn.base import BaseEstimator, TransformerMixin

class DateFeatureExtractor(BaseEstimator, TransformerMixin):
    """Custom transformer to extract features from date columns"""
    
    def fit(self, X, y=None):
        return self  # Nothing to fit
    
    def transform(self, X):
        X = X.copy()
        X['year'] = X['date'].dt.year
        X['month'] = X['date'].dt.month
        X['day_of_week'] = X['date'].dt.dayofweek
        X['is_weekend'] = (X['date'].dt.dayofweek >= 5).astype(int)
        return X.drop(columns=['date'])

# Use in pipeline
pipeline = Pipeline([
    ('date_features', DateFeatureExtractor()),
    ('scaler', StandardScaler()),
    ('model', RandomForestClassifier())
])
```

---

## 11.10 Common Mistakes

### Mistake 1: Fitting Transformers on Full Data (Data Leakage!)
```python
# ❌ WRONG - Scaler sees test data statistics
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)  # Fits on ALL data
X_train, X_test = train_test_split(X_scaled, ...)

# ✅ CORRECT - Fit only on training data
X_train, X_test = train_test_split(X, ...)
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)   # Fit + transform
X_test_scaled = scaler.transform(X_test)          # Only transform!
```

### Mistake 2: One-Hot Encoding Before Train/Test Split
```python
# ❌ WRONG - Test set categories influence encoding
df_encoded = pd.get_dummies(df)
X_train, X_test = train_test_split(df_encoded, ...)

# ✅ CORRECT - Use Pipeline or encode after split
# Pipeline handles it automatically (see section 11.9)
```

### Mistake 3: Not Handling Unknown Categories in Production
```python
# ❌ WRONG - Crashes when new category appears in production
ohe = OneHotEncoder()
ohe.fit(X_train)
ohe.transform(X_production)  # Error if new category exists!

# ✅ CORRECT - handle_unknown='ignore' (sets all to 0)
ohe = OneHotEncoder(handle_unknown='ignore')
```

### Mistake 4: Creating Features from Target Variable
```python
# ❌ WRONG - Target leakage! 
df['avg_price_by_category'] = df.groupby('category')['price'].transform('mean')
# This uses the TARGET (price) to create a feature!

# ✅ CORRECT - Use only on training fold, with proper CV
# Use TargetEncoder with internal CV, or compute on train only
```

### Mistake 5: Too Many Polynomial/Interaction Features
```python
# ❌ WRONG - 100 features with degree=3 = 176,851 features!
poly = PolynomialFeatures(degree=3)
X_poly = poly.fit_transform(X_100_features)  # Memory explosion + overfitting

# ✅ CORRECT - Use sparingly or with regularization
poly = PolynomialFeatures(degree=2, interaction_only=True)  # Only interactions
# OR use with Lasso to auto-select
```

---

## 11.11 Interview Questions

### Q1: What is the difference between feature selection and feature extraction?
**Answer**: Feature *selection* picks a subset of original features (e.g., removing unimportant ones). Feature *extraction* creates new features from original ones (e.g., PCA creates principal components that are combinations of original features). Selection maintains interpretability; extraction often doesn't.

### Q2: How do you handle a categorical variable with 10,000 unique values?
**Answer**: One-hot encoding would create 10,000 columns (impractical). Options: (1) Target encoding with smoothing, (2) Frequency encoding, (3) Hash encoding, (4) Group rare categories into "Other", (5) Embedding layers (deep learning). Choice depends on model type and data size.

### Q3: Why do we scale features and which models require it?
**Answer**: Scaling ensures all features contribute equally to distance/gradient calculations. Models requiring it: KNN (distance-based), SVM (distance-based), Neural Networks (gradient-based), PCA (variance-based), Logistic Regression (gradient-based). Tree-based models (RF, XGBoost) don't need scaling because they split based on thresholds, not distances.

### Q4: Explain the difference between StandardScaler, MinMaxScaler, and RobustScaler.
**Answer**: StandardScaler: z-score (mean=0, std=1), assumes Gaussian, sensitive to outliers. MinMaxScaler: [0,1] range, preserves shape, very sensitive to outliers. RobustScaler: uses median/IQR, robust to outliers. Use RobustScaler when data has outliers, MinMaxScaler for neural networks, StandardScaler as default for normally distributed data.

### Q5: What is target encoding and what's the risk?
**Answer**: Target encoding replaces categories with the mean target value for that category. Risk: severe overfitting if done naively (model memorizes categories). Mitigation: (1) Smoothing (blend with global mean), (2) Use only training fold means during CV, (3) Add noise, (4) Use leave-one-out encoding.

### Q6: How do you handle cyclical features like day-of-week or month?
**Answer**: Use sin/cos encoding: `sin(2π * value/period)` and `cos(2π * value/period)`. This ensures December (12) and January (1) are adjacent, not 11 apart. For day-of-week: sin(2π * day/7) and cos(2π * day/7).

### Q7: What's the difference between filter, wrapper, and embedded feature selection methods?
**Answer**: **Filter**: Statistical tests independent of model (fast but ignores feature interactions). **Wrapper**: Uses model performance to select (slow but considers interactions, e.g., RFE). **Embedded**: Feature selection during training (Lasso, tree importance — good balance of speed and quality).

### Q8: How do you prevent data leakage in feature engineering?
**Answer**: (1) Always fit transformers on training data only, (2) Use sklearn Pipelines, (3) Never create features using target variable, (4) Be careful with time-based features (no future info), (5) In CV, all transformations must happen inside each fold.

---

## 11.12 Quick Reference

### Encoding Cheat Sheet

| Technique | When to Use | Cardinality | Model Type |
|-----------|-------------|:-----------:|------------|
| One-Hot | Nominal, low card | < 15 | All |
| Ordinal | Has natural order | Any | All |
| Target | Nominal, high card | > 15 | All (careful!) |
| Frequency | Quick baseline | Any | Tree-based |
| Binary | Medium card | 10-100 | All |

### Scaling Cheat Sheet

| Scaler | Formula | When to Use |
|--------|---------|-------------|
| StandardScaler | (x - μ) / σ | Default, Gaussian features |
| MinMaxScaler | (x - min) / (max - min) | Neural nets, bounded [0,1] |
| RobustScaler | (x - median) / IQR | Outliers present |
| Normalizer | x / ‖x‖ | Text vectors, row-wise |

### Feature Selection Decision Guide

| Situation | Method |
|-----------|--------|
| Quick baseline | VarianceThreshold + SelectKBest |
| Need interpretability | L1 regularization |
| Best performance | RFECV or Permutation Importance |
| Very high dimensions | Mutual Information filter → then wrapper |
| Production | Embedded (tree importance) within Pipeline |

### Must-Know sklearn Imports

```python
# Encoding
from sklearn.preprocessing import LabelEncoder, OrdinalEncoder, OneHotEncoder, TargetEncoder

# Scaling
from sklearn.preprocessing import StandardScaler, MinMaxScaler, RobustScaler

# Imputation
from sklearn.impute import SimpleImputer, KNNImputer

# Feature Selection
from sklearn.feature_selection import SelectKBest, RFE, RFECV, SelectFromModel
from sklearn.feature_selection import f_classif, mutual_info_classif, chi2

# Transformation
from sklearn.preprocessing import PolynomialFeatures, PowerTransformer, KBinsDiscretizer

# Pipeline
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
```

---

*Previous: [10 - Model Evaluation and Tuning](10-Model-Evaluation-and-Tuning.md) | Next: [12 - Time Series Forecasting](12-Time-Series-Forecasting.md)*
