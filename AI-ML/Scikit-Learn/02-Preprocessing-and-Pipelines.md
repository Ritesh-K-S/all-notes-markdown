# Preprocessing and Pipelines in Scikit-Learn

## Table of Contents
- [Why Preprocessing Matters](#why-preprocessing-matters)
- [Feature Scaling](#feature-scaling)
- [Encoding Categorical Variables](#encoding-categorical-variables)
- [Handling Missing Values (Imputation)](#handling-missing-values-imputation)
- [Feature Transformation](#feature-transformation)
- [Pipelines — Chaining Everything Together](#pipelines)
- [ColumnTransformer — Different Processing for Different Columns](#columntransformer)
- [FeatureUnion — Combining Feature Sets](#featureunion)
- [Advanced Pipeline Patterns](#advanced-pipeline-patterns)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Why Preprocessing Matters

### What It Is
Preprocessing is **preparing raw data** so that ML algorithms can work with it properly. Think of it like preparing ingredients before cooking — you wash vegetables, chop onions, measure spices. If you skip this, the dish (model) turns out terrible.

### Why It Matters
- Most ML algorithms assume **numeric, clean, scaled** data
- Raw data has missing values, mixed types, wildly different scales
- **Garbage in = garbage out** — no algorithm can fix bad input data
- Preprocessing often has MORE impact on results than choosing the "best" algorithm

### The Preprocessing Pipeline (Big Picture)

```
Raw Data
  │
  ▼
┌──────────────────┐
│ Handle Missing   │  → SimpleImputer, KNNImputer
│ Values           │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Encode           │  → OneHotEncoder, OrdinalEncoder, LabelEncoder
│ Categoricals     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Scale/Normalize  │  → StandardScaler, MinMaxScaler, RobustScaler
│ Features         │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Transform        │  → PolynomialFeatures, PowerTransformer, log
│ Features         │
└────────┬─────────┘
         │
         ▼
  Clean Data → Model
```

---

## Feature Scaling

### What It Is
Making all features live on a **similar scale**. Imagine comparing someone's age (0–100) with their salary (0–1,000,000). Without scaling, the salary feature would dominate simply because its numbers are bigger, not because it's more important.

### Why It Matters
- **Distance-based algorithms** (KNN, SVM, K-Means) are completely broken without scaling
- **Gradient descent** converges much faster with scaled features
- **Regularized models** (Ridge, Lasso) penalize larger-valued features more unfairly
- **Tree-based models** (Random Forest, XGBoost) do NOT need scaling — they split on thresholds

### When to Scale

| Algorithm | Needs Scaling? | Why |
|-----------|---------------|-----|
| Linear/Logistic Regression | ✅ Yes (with regularization) | Regularization penalizes by magnitude |
| SVM | ✅ Yes | Kernel distances sensitive to scale |
| KNN | ✅ Yes | Distance calculation dominated by large features |
| K-Means | ✅ Yes | Distance-based clustering |
| Neural Networks | ✅ Yes | Gradient descent convergence |
| Decision Tree | ❌ No | Splits on thresholds, scale-invariant |
| Random Forest | ❌ No | Ensemble of trees |
| XGBoost/LightGBM | ❌ No | Tree-based |
| Naive Bayes | ❌ No | Probability-based |

### The Four Scalers

#### 1. StandardScaler (Z-score Normalization)

$$z = \frac{x - \mu}{\sigma}$$

Centers data to mean=0, std=1.

```python
from sklearn.preprocessing import StandardScaler
import numpy as np

X = np.array([[10, 10000], [20, 20000], [30, 30000], [40, 40000]])

scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

print("Before scaling:")
print(f"  Mean: {X.mean(axis=0)}")         # [25.  25000.]
print(f"  Std:  {X.std(axis=0)}")          # [11.18  11180.34]

print("\nAfter scaling:")
print(f"  Mean: {X_scaled.mean(axis=0).round(2)}")  # [0. 0.]
print(f"  Std:  {X_scaled.std(axis=0).round(2)}")   # [1. 1.]

print("\nScaled data:")
print(X_scaled)
# [[-1.34  -1.34]
#  [-0.45  -0.45]
#  [ 0.45   0.45]
#  [ 1.34   1.34]]

# Access learned parameters
print(f"\nLearned mean: {scaler.mean_}")    # [25.  25000.]
print(f"Learned std:  {scaler.scale_}")     # [11.18  11180.34]
```

**Use when**: Data is approximately normally distributed. Default choice for most cases.

#### 2. MinMaxScaler (Min-Max Normalization)

$$x_{scaled} = \frac{x - x_{min}}{x_{max} - x_{min}}$$

Scales data to [0, 1] range (or custom range).

```python
from sklearn.preprocessing import MinMaxScaler

scaler = MinMaxScaler(feature_range=(0, 1))  # Default range
X_scaled = scaler.fit_transform(X)

print("MinMax scaled:")
print(X_scaled)
# [[0.   0.  ]
#  [0.33 0.33]
#  [0.67 0.67]
#  [1.   1.  ]]

# Custom range: scale to [-1, 1]
scaler_custom = MinMaxScaler(feature_range=(-1, 1))
X_custom = scaler_custom.fit_transform(X)
print("\nCustom range [-1, 1]:")
print(X_custom)
```

**Use when**: You need bounded values (e.g., neural networks with sigmoid), or data is NOT normally distributed.

#### 3. RobustScaler (Robust to Outliers)

$$x_{scaled} = \frac{x - \text{median}}{IQR}$$

Where $IQR = Q_{75} - Q_{25}$ (interquartile range).

```python
from sklearn.preprocessing import RobustScaler

# Data with outlier
X_outlier = np.array([[10], [20], [30], [40], [1000]])  # 1000 is an outlier

# StandardScaler is sensitive to outliers
standard = StandardScaler().fit_transform(X_outlier)
robust = RobustScaler().fit_transform(X_outlier)

print("With outlier present:")
print(f"StandardScaler: {standard.flatten().round(2)}")
# [-0.54, -0.51, -0.49, -0.46, 2.0]  ← Everything compressed

print(f"RobustScaler:   {robust.flatten().round(2)}")
# [-0.5, 0.0, 0.5, 1.0, 49.0]  ← Normal data well-spread, outlier clearly marked
```

**Use when**: Data has outliers that you don't want to remove.

#### 4. MaxAbsScaler

$$x_{scaled} = \frac{x}{|x_{max}|}$$

Scales to [-1, 1] without shifting center. Preserves sparsity.

```python
from sklearn.preprocessing import MaxAbsScaler

scaler = MaxAbsScaler()
X_scaled = scaler.fit_transform(X)
print(X_scaled)
# [[0.25  0.25]
#  [0.5   0.5 ]
#  [0.75  0.75]
#  [1.    1.  ]]
```

**Use when**: Sparse data (like text features) — doesn't destroy zero entries.

### Scaler Comparison Table

| Scaler | Formula | Range | Outlier Robust | Preserves Sparsity | Best For |
|--------|---------|-------|----------------|-------------------|----------|
| StandardScaler | $(x - \mu) / \sigma$ | unbounded | ❌ | ❌ | Normal data |
| MinMaxScaler | $(x - min) / (max - min)$ | [0, 1] | ❌ | ❌ | Bounded features |
| RobustScaler | $(x - median) / IQR$ | unbounded | ✅ | ❌ | Data with outliers |
| MaxAbsScaler | $x / |max|$ | [-1, 1] | ❌ | ✅ | Sparse data |
| Normalizer | $x / \|x\|$ | [-1, 1] | ❌ | ❌ | Row-wise (text/NLP) |

> **Pro Tip**: `Normalizer` is different — it normalizes each **row** (sample), not each column (feature). It's used in text processing where you want each document vector to have unit length.

---

## Encoding Categorical Variables

### What It Is
Converting text/category data (like "red", "blue", "green") into numbers that ML algorithms can understand. Algorithms can't read words — they need numbers.

### Why It Matters
- Most real-world data has categorical features (city, country, product type)
- Wrong encoding can introduce false ordinal relationships
- Right encoding can dramatically improve model performance

### Encoding Methods

#### 1. LabelEncoder (for target variable only)

```python
from sklearn.preprocessing import LabelEncoder

# Converts categories to integers
le = LabelEncoder()
y = ['cat', 'dog', 'fish', 'dog', 'cat', 'fish']
y_encoded = le.fit_transform(y)
print(y_encoded)         # [0, 1, 2, 1, 0, 2]
print(le.classes_)       # ['cat', 'dog', 'fish']

# Reverse transform
y_original = le.inverse_transform(y_encoded)
print(y_original)        # ['cat', 'dog', 'fish', 'dog', 'cat', 'fish']
```

> **Warning**: NEVER use LabelEncoder for features (X). It creates fake ordinal relationships (dog=1, fish=2 implies fish > dog). Use it ONLY for the target variable (y).

#### 2. OrdinalEncoder (for ordinal features)

```python
from sklearn.preprocessing import OrdinalEncoder

# For features that HAVE a natural order
X = np.array([['low'], ['medium'], ['high'], ['medium'], ['low']])

# Specify the order explicitly
enc = OrdinalEncoder(categories=[['low', 'medium', 'high']])
X_encoded = enc.fit_transform(X)
print(X_encoded)  # [[0.], [1.], [2.], [1.], [0.]]
# low=0, medium=1, high=2 — the order is meaningful!

# Handle unknown categories at prediction time
enc = OrdinalEncoder(
    categories=[['low', 'medium', 'high']],
    handle_unknown='use_encoded_value',
    unknown_value=-1  # Assign -1 to unknown categories
)
enc.fit(X)
print(enc.transform([['very_high']]))  # [[-1.]]
```

**Use when**: Feature has a natural order (e.g., education level: high school < bachelor < master < PhD).

#### 3. OneHotEncoder (for nominal features — THE DEFAULT CHOICE)

```python
from sklearn.preprocessing import OneHotEncoder

# For features WITHOUT natural order
X = np.array([['red'], ['blue'], ['green'], ['red'], ['blue']])

# Default: sparse output
enc = OneHotEncoder(sparse_output=False)  # dense output for readability
X_encoded = enc.fit_transform(X)
print(X_encoded)
# [[0. 0. 1.]   → red
#  [1. 0. 0.]   → blue
#  [0. 1. 0.]   → green
#  [0. 0. 1.]   → red
#  [1. 0. 0.]]  → blue

print(enc.get_feature_names_out())  # ['x0_blue', 'x0_green', 'x0_red']

# Drop first category to avoid multicollinearity (dummy encoding)
enc_drop = OneHotEncoder(sparse_output=False, drop='first')
X_drop = enc_drop.fit_transform(X)
print(X_drop)
# [[0. 1.]   → red (green=0, red=1)
#  [0. 0.]   → blue (green=0, red=0)
#  [1. 0.]   → green (green=1, red=0)
# ...

# Handle unknown categories
enc = OneHotEncoder(sparse_output=False, handle_unknown='ignore')
enc.fit(X)
print(enc.transform([['yellow']]))  # [[0. 0. 0.]]  ← All zeros
```

> **Pro Tip**: Use `drop='first'` for linear models to avoid the **dummy variable trap** (multicollinearity). Tree models don't need this.

#### 4. Pandas get_dummies (Quick Alternative)

```python
import pandas as pd

df = pd.DataFrame({'color': ['red', 'blue', 'green', 'red']})

# Quick one-hot encoding (but can't apply to test data separately!)
dummies = pd.get_dummies(df, columns=['color'], drop_first=True)
print(dummies)
#    color_green  color_red
# 0        False       True
# 1        False      False
# 2         True      False
# 3        False       True
```

> **Warning**: `pd.get_dummies` is convenient but dangerous in production — if test data has different categories than training data, columns won't match. Always use sklearn's OneHotEncoder for real projects.

### Encoding Comparison

| Encoder | Creates Order? | For Target? | For Features? | Handles Unknown? |
|---------|---------------|-------------|---------------|-----------------|
| LabelEncoder | Yes (arbitrary) | ✅ | ❌ Never | ❌ |
| OrdinalEncoder | Yes (meaningful) | ❌ | ✅ Ordinal only | ✅ |
| OneHotEncoder | No | ❌ | ✅ Nominal | ✅ |
| pd.get_dummies | No | ❌ | Quick & dirty | ❌ |

---

## Handling Missing Values (Imputation)

### What It Is
Filling in missing data with reasonable estimates. Like filling in blanks on an incomplete form — you use the best guess based on what you know.

### Why It Matters
- Most real-world datasets have missing values (NaN, null, blank)
- Most sklearn algorithms **crash** on missing values
- Dropping rows loses data; imputation preserves it
- The imputation strategy can significantly affect model quality

### Imputation Strategies

#### 1. SimpleImputer — Basic Strategies

```python
from sklearn.impute import SimpleImputer
import numpy as np
import pandas as pd

X = np.array([
    [1, 2, np.nan],
    [3, np.nan, 6],
    [7, 8, 9],
    [np.nan, 11, 12]
])

# Strategy: mean (default) — replace NaN with column mean
imp_mean = SimpleImputer(strategy='mean')
X_imputed = imp_mean.fit_transform(X)
print("Mean imputation:")
print(X_imputed)
# [[ 1.    2.    9.  ]   ← NaN → mean of [6,9,12] = 9.0
#  [ 3.    7.    6.  ]   ← NaN → mean of [2,8,11] = 7.0
#  [ 7.    8.    9.  ]
#  [ 3.67  11.   12. ]]  ← NaN → mean of [1,3,7] = 3.67

# Strategy: median — better for skewed data
imp_median = SimpleImputer(strategy='median')
X_median = imp_median.fit_transform(X)

# Strategy: most_frequent — for categorical data too
imp_freq = SimpleImputer(strategy='most_frequent')

# Strategy: constant — fill with a specific value
imp_const = SimpleImputer(strategy='constant', fill_value=0)
X_const = imp_const.fit_transform(X)
print("\nConstant (0) imputation:")
print(X_const)

# For categorical features
X_cat = np.array([['a', 'x'], ['b', None], [None, 'x'], ['a', 'y']])
imp_cat = SimpleImputer(strategy='most_frequent')
print("\nCategorical imputation:")
print(imp_cat.fit_transform(X_cat))
# [['a' 'x'], ['b' 'x'], ['a' 'x'], ['a' 'y']]

# Track which values were imputed (sklearn >= 1.0)
imp = SimpleImputer(strategy='mean', add_indicator=True)
X_with_indicator = imp.fit_transform(X)
print(f"\nWith indicator — shape: {X_with_indicator.shape}")
# Original 3 columns + 3 indicator columns (1 if was missing, 0 if not)
```

#### 2. KNNImputer — Smart Imputation Using Neighbors

```python
from sklearn.impute import KNNImputer

X = np.array([
    [1, 2, np.nan],
    [3, 4, 3],
    [np.nan, 6, 5],
    [8, 8, 7]
])

# Uses k-nearest neighbors to estimate missing values
# Finds similar rows and uses their values
imp_knn = KNNImputer(n_neighbors=2, weights='uniform')
X_imputed = imp_knn.fit_transform(X)
print("KNN imputation:")
print(X_imputed)
# Missing values filled based on closest neighbors
```

**Use when**: Features are correlated — neighboring samples provide better estimates than global mean.

#### 3. IterativeImputer — Multivariate Imputation (MICE)

```python
from sklearn.experimental import enable_iterative_imputer  # Must import first!
from sklearn.impute import IterativeImputer

# Models each feature with missing values as a function of other features
# Iteratively improves estimates
imp_iter = IterativeImputer(max_iter=10, random_state=42)
X_imputed = imp_iter.fit_transform(X)
print("Iterative imputation:")
print(X_imputed)
```

**Use when**: Missing data is not random (MAR/MNAR) and features are strongly correlated.

### Imputation Strategy Comparison

| Strategy | Best For | Handles Outliers | Computational Cost |
|----------|----------|-----------------|-------------------|
| Mean | Normally distributed numeric | ❌ | Very low |
| Median | Skewed numeric data | ✅ | Very low |
| Most Frequent | Categorical data | N/A | Very low |
| Constant | Domain-specific defaults | N/A | Very low |
| KNN | Correlated features | ❌ | Medium |
| Iterative (MICE) | Complex missing patterns | Depends | High |

> **Pro Tip**: Always create a **missing indicator** column alongside imputation. The fact that a value is missing is often informative! Use `add_indicator=True` in SimpleImputer.

---

## Feature Transformation

### What It Is
Mathematically transforming features to make them more suitable for ML algorithms — like converting currencies before comparing prices internationally.

### PowerTransformer — Making Data Normal

```python
from sklearn.preprocessing import PowerTransformer
import numpy as np

# Highly skewed data (log-normal)
np.random.seed(42)
X_skewed = np.random.exponential(size=(1000, 2))

# Yeo-Johnson: works with positive AND negative values
pt_yj = PowerTransformer(method='yeo-johnson')
X_normal = pt_yj.fit_transform(X_skewed)
print(f"Before: skew = {X_skewed.mean(axis=0).round(2)}")
print(f"After:  mean ≈ {X_normal.mean(axis=0).round(2)}, std ≈ {X_normal.std(axis=0).round(2)}")

# Box-Cox: only works with STRICTLY positive values
pt_bc = PowerTransformer(method='box-cox')
X_boxcox = pt_bc.fit_transform(X_skewed)  # X must be > 0
```

### PolynomialFeatures — Creating Interaction Terms

```python
from sklearn.preprocessing import PolynomialFeatures

X = np.array([[2, 3], [4, 5]])

# Degree 2: creates x1², x2², x1*x2
poly = PolynomialFeatures(degree=2, include_bias=False)
X_poly = poly.fit_transform(X)
print(f"Original features: {poly.n_features_in_}")
print(f"Output features: {X_poly.shape[1]}")
print(f"Feature names: {poly.get_feature_names_out()}")
# ['x0', 'x1', 'x0^2', 'x0 x1', 'x1^2']
print(X_poly)
# [[ 2.  3.  4.  6.  9.]
#  [ 4.  5. 16. 20. 25.]]

# Interaction only (no x², x³ etc.)
poly_interact = PolynomialFeatures(degree=2, interaction_only=True, include_bias=False)
X_interact = poly_interact.fit_transform(X)
print(f"\nInteraction only: {poly_interact.get_feature_names_out()}")
# ['x0', 'x1', 'x0 x1']
```

> **Warning**: Feature explosion! With 10 features and degree=3, you get 286 features. Use `interaction_only=True` to limit this.

### FunctionTransformer — Custom Transformations

```python
from sklearn.preprocessing import FunctionTransformer

# Apply any function as a transformer
log_transformer = FunctionTransformer(np.log1p, inverse_func=np.expm1)
X = np.array([[1, 10], [100, 1000]])

X_log = log_transformer.fit_transform(X)
print(X_log)
# [[0.69  2.40]
#  [4.62  6.91]]

# Works in pipelines!
```

### Binarizer — Threshold-Based Binary Features

```python
from sklearn.preprocessing import Binarizer

X = np.array([[1.5, -2.0, 3.3], [0.1, 4.5, -1.2]])

binarizer = Binarizer(threshold=1.0)  # Values > 1.0 → 1, else → 0
X_binary = binarizer.fit_transform(X)
print(X_binary)
# [[1. 0. 1.]
#  [0. 1. 0.]]
```

---

## Pipelines

### What It Is
A **Pipeline** chains multiple preprocessing steps and a model into a single object. Think of it as an assembly line in a factory — raw materials go in one end, finished product comes out the other. Every step is automated and in order.

### Why It Matters
- **Prevents data leakage**: fit/transform is done correctly automatically
- **Cleaner code**: one object instead of 5 separate ones
- **Reproducibility**: the entire workflow is a single saveable object
- **Works with GridSearchCV**: tune hyperparameters across ALL steps at once
- **Production-ready**: deploy ONE object, not a mess of separate transformers

### How Pipelines Work

```
          Pipeline.fit(X_train, y_train)
          ════════════════════════════════

X_train ──→ Step 1: Imputer.fit_transform(X) ──→ X_imputed
             │
X_imputed ─→ Step 2: Scaler.fit_transform(X) ──→ X_scaled
             │
X_scaled ──→ Step 3: Model.fit(X, y)          ──→ Fitted Model
             
          
          Pipeline.predict(X_test)
          ════════════════════════

X_test ───→ Step 1: Imputer.transform(X)      ──→ X_imputed
             │
X_imputed ─→ Step 2: Scaler.transform(X)      ──→ X_scaled
             │
X_scaled ──→ Step 3: Model.predict(X)         ──→ Predictions

Notice: fit_transform for training, ONLY transform for test — automatic!
```

### Code Example — Building Pipelines

```python
from sklearn.pipeline import Pipeline, make_pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.impute import SimpleImputer
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split

X, y = make_classification(n_samples=1000, n_features=20, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=42)

# ============================================
# Method 1: Pipeline with explicit names
# ============================================
pipe = Pipeline([
    ('imputer', SimpleImputer(strategy='mean')),      # Step 1
    ('scaler', StandardScaler()),                      # Step 2
    ('classifier', LogisticRegression(max_iter=200))   # Step 3 (final)
])

# Use it like a single estimator!
pipe.fit(X_train, y_train)
y_pred = pipe.predict(X_test)
score = pipe.score(X_test, y_test)
print(f"Pipeline accuracy: {score:.3f}")

# ============================================
# Method 2: make_pipeline (auto-generates names)
# ============================================
pipe_auto = make_pipeline(
    SimpleImputer(strategy='mean'),
    StandardScaler(),
    LogisticRegression(max_iter=200)
)
# Names are auto-generated: 'simpleimputer', 'standardscaler', 'logisticregression'

pipe_auto.fit(X_train, y_train)
print(f"make_pipeline accuracy: {pipe_auto.score(X_test, y_test):.3f}")

# ============================================
# Accessing pipeline steps
# ============================================

# Access by name
scaler_step = pipe.named_steps['scaler']
print(f"Scaler mean: {scaler_step.mean_[:3]}")

# Access by index
first_step = pipe[0]   # The imputer
last_step = pipe[-1]   # The classifier

# Slice the pipeline
preprocessing = pipe[:-1]  # Everything except the model
X_preprocessed = preprocessing.transform(X_test)

# ============================================
# Pipeline with GridSearchCV
# ============================================
from sklearn.model_selection import GridSearchCV

# Access step parameters with stepname__parameter syntax
param_grid = {
    'imputer__strategy': ['mean', 'median'],
    'classifier__C': [0.1, 1.0, 10.0],
    'classifier__penalty': ['l1', 'l2']
}

grid = GridSearchCV(pipe, param_grid, cv=5, scoring='accuracy')
grid.fit(X_train, y_train)
print(f"\nBest params: {grid.best_params_}")
print(f"Best score: {grid.best_score_:.3f}")
```

### Pipeline Rules

| Rule | Details |
|------|---------|
| All steps except last must be **transformers** | Must have `fit()` and `transform()` |
| Last step can be anything | Transformer, predictor, or `'passthrough'` |
| Methods cascade | `pipeline.predict()` calls transform on all steps, predict on last |
| Parameter access | Use `step_name__param_name` syntax (double underscore) |

---

## ColumnTransformer

### What It Is
A way to apply **different transformations to different columns**. In real data, you have numeric columns (scale them) and categorical columns (encode them) — ColumnTransformer handles this elegantly.

### Why It Matters
Real datasets are messy — they have numbers, categories, text, dates. You can't apply StandardScaler to a "city" column or OneHotEncoder to an "age" column. ColumnTransformer lets you route each column to the right transformer.

### How It Works

```
                    ColumnTransformer
                    ════════════════
                    
                         X (all columns)
                              │
              ┌───────────────┼───────────────┐
              │               │               │
         numeric cols    categorical cols   remaining
              │               │               │
              ▼               ▼               ▼
       ┌────────────┐  ┌────────────┐   ┌──────────┐
       │StandardScal│  │OneHotEncode│   │  drop /   │
       │    er      │  │    r       │   │passthrough│
       └─────┬──────┘  └─────┬──────┘   └────┬─────┘
              │               │               │
              └───────────────┼───────────────┘
                              │
                         X_transformed
                  (all results concatenated)
```

### Code Example — Full Real-World Pipeline

```python
import pandas as pd
import numpy as np
from sklearn.compose import ColumnTransformer, make_column_selector
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split

# ============================================
# Create realistic dataset
# ============================================
np.random.seed(42)
n = 500
data = pd.DataFrame({
    'age': np.random.randint(18, 70, n).astype(float),
    'income': np.random.normal(50000, 15000, n),
    'education': np.random.choice(['high_school', 'bachelor', 'master', 'phd'], n),
    'city': np.random.choice(['NYC', 'LA', 'Chicago', 'Houston'], n),
    'purchased': np.random.randint(0, 2, n)
})

# Add some missing values
data.loc[data.sample(50).index, 'age'] = np.nan
data.loc[data.sample(30).index, 'income'] = np.nan

X = data.drop('purchased', axis=1)
y = data['purchased']
X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=42)

print(X_train.dtypes)
# age          float64   ← numeric
# income       float64   ← numeric
# education    object    ← categorical
# city         object    ← categorical

# ============================================
# Define column groups
# ============================================
numeric_features = ['age', 'income']
categorical_features = ['education', 'city']

# ============================================
# Define transformers for each group
# ============================================

# Numeric: impute missing → scale
numeric_transformer = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', StandardScaler())
])

# Categorical: impute missing → one-hot encode
categorical_transformer = Pipeline([
    ('imputer', SimpleImputer(strategy='most_frequent')),
    ('onehot', OneHotEncoder(handle_unknown='ignore', sparse_output=False))
])

# ============================================
# Combine with ColumnTransformer
# ============================================
preprocessor = ColumnTransformer(
    transformers=[
        ('num', numeric_transformer, numeric_features),
        ('cat', categorical_transformer, categorical_features)
    ],
    remainder='drop'  # Drop columns not specified (or 'passthrough')
)

# ============================================
# Full pipeline: preprocessing + model
# ============================================
full_pipeline = Pipeline([
    ('preprocessor', preprocessor),
    ('classifier', LogisticRegression(max_iter=200))
])

# Fit and evaluate
full_pipeline.fit(X_train, y_train)
score = full_pipeline.score(X_test, y_test)
print(f"\nPipeline accuracy: {score:.3f}")

# ============================================
# Inspect what happened
# ============================================
# Get feature names after transformation
feature_names = full_pipeline.named_steps['preprocessor'].get_feature_names_out()
print(f"\nOutput features: {feature_names}")
# ['num__age', 'num__income', 'cat__education_bachelor', ...]

# Access nested parameters for GridSearchCV
print(full_pipeline.get_params().keys())
# 'preprocessor__num__imputer__strategy'
# 'preprocessor__cat__onehot__drop'
# 'classifier__C'
```

### Auto-Selecting Columns by Type

```python
from sklearn.compose import make_column_selector

# Automatically select columns by dtype — no hardcoding!
preprocessor_auto = ColumnTransformer(
    transformers=[
        ('num', numeric_transformer, make_column_selector(dtype_include=np.number)),
        ('cat', categorical_transformer, make_column_selector(dtype_include=object))
    ]
)

# This is more robust — automatically handles new columns of matching types
```

### remainder Parameter Options

```python
# remainder='drop'       → Discard columns not mentioned (default)
# remainder='passthrough' → Keep them as-is
# remainder=SomeTransformer() → Apply this transformer to remaining columns

preprocessor = ColumnTransformer(
    transformers=[
        ('num', StandardScaler(), numeric_features),
    ],
    remainder='passthrough'  # Keep categorical columns unchanged
)
```

---

## FeatureUnion

### What It Is
Combines **multiple feature extraction methods** side by side. While Pipeline chains steps sequentially (one after another), FeatureUnion runs steps **in parallel** and concatenates results horizontally.

### Why It Matters
Sometimes you want to extract different kinds of features from the same data and combine them — e.g., statistical features AND text features from a document.

### How It Works

```
Pipeline (sequential):        FeatureUnion (parallel):
                              
X ──→ A ──→ B ──→ C          X ──→ A ──┐
                                        ├──→ [A_out | B_out | C_out]
                              X ──→ B ──┤
                                        │
                              X ──→ C ──┘
```

```python
from sklearn.pipeline import FeatureUnion, make_union
from sklearn.decomposition import PCA
from sklearn.preprocessing import PolynomialFeatures, StandardScaler
from sklearn.datasets import make_classification

X, y = make_classification(n_samples=200, n_features=10, random_state=42)

# Combine PCA features + polynomial interaction features
combined_features = FeatureUnion([
    ('pca', PCA(n_components=3)),                    # 3 PCA features
    ('poly', PolynomialFeatures(degree=2, interaction_only=True,
                                 include_bias=False))  # Interaction features
])

X_combined = combined_features.fit_transform(X)
print(f"Original: {X.shape}")      # (200, 10)
print(f"Combined: {X_combined.shape}")  # (200, 3 + 55) = (200, 58)

# Use in a full pipeline
full_pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('features', combined_features),
    ('classifier', LogisticRegression(max_iter=200))
])
```

> **Note**: In modern sklearn, `ColumnTransformer` has largely replaced `FeatureUnion` for most use cases since it's more flexible and handles column routing.

---

## Advanced Pipeline Patterns

### Pattern 1: Caching Expensive Steps

```python
from sklearn.pipeline import Pipeline
from tempfile import mkdtemp

# Cache intermediate results to avoid recomputation during GridSearch
cachedir = mkdtemp()
pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('classifier', LogisticRegression())
], memory=cachedir)  # Cache fitted transformers

# Clean up
import shutil
# shutil.rmtree(cachedir)  # After you're done
```

### Pattern 2: Conditional Steps with passthrough

```python
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline

# Sometimes you want to optionally skip a step
pipe = Pipeline([
    ('scaler', StandardScaler()),       # Can be replaced with 'passthrough'
    ('classifier', LogisticRegression())
])

# In GridSearch, you can toggle steps on/off:
param_grid = {
    'scaler': [StandardScaler(), 'passthrough'],  # Try with and without scaling
    'classifier__C': [0.1, 1.0, 10.0]
}
```

### Pattern 3: Saving and Loading Pipelines

```python
import joblib

# Save the entire pipeline (preprocessing + model)
joblib.dump(full_pipeline, 'my_pipeline.pkl')

# Load it later — in production, just load and predict
loaded_pipeline = joblib.load('my_pipeline.pkl')
predictions = loaded_pipeline.predict(X_test)
# All preprocessing happens automatically!
```

### Pattern 4: Custom Transformer in a Pipeline

```python
from sklearn.base import BaseEstimator, TransformerMixin

class OutlierClipper(BaseEstimator, TransformerMixin):
    """Clips outliers to percentile boundaries."""
    
    def __init__(self, lower_percentile=5, upper_percentile=95):
        self.lower_percentile = lower_percentile
        self.upper_percentile = upper_percentile
    
    def fit(self, X, y=None):
        self.lower_ = np.percentile(X, self.lower_percentile, axis=0)
        self.upper_ = np.percentile(X, self.upper_percentile, axis=0)
        return self  # Always return self!
    
    def transform(self, X):
        X_clipped = np.clip(X, self.lower_, self.upper_)
        return X_clipped

# Use in a pipeline like any other transformer
pipe = Pipeline([
    ('clipper', OutlierClipper(lower_percentile=1, upper_percentile=99)),
    ('scaler', StandardScaler()),
    ('model', LogisticRegression())
])
```

> **Pro Tip**: Always inherit from both `BaseEstimator` (gives you `get_params`/`set_params`) and `TransformerMixin` (gives you `fit_transform` for free).

---

## Common Mistakes

### 1. Data Leakage Through Preprocessing Outside Pipeline

```python
# ❌ WRONG — Scaler sees test data
scaler = StandardScaler()
X_all_scaled = scaler.fit_transform(X)  # Fits on ALL data including test!
X_train, X_test = train_test_split(X_all_scaled, ...)

# ✅ CORRECT — Split first, then preprocess
X_train, X_test, y_train, y_test = train_test_split(X, y, ...)
pipe = Pipeline([('scaler', StandardScaler()), ('model', LogisticRegression())])
pipe.fit(X_train, y_train)
pipe.score(X_test, y_test)
```

### 2. OneHotEncoder Fails on Unseen Categories

```python
# ❌ WRONG — Crashes on new categories in test data
enc = OneHotEncoder()
enc.fit(X_train[['city']])
enc.transform(X_test[['city']])  # If test has 'Seattle', CRASH!

# ✅ CORRECT
enc = OneHotEncoder(handle_unknown='ignore')  # Unknown → all zeros
```

### 3. Applying Scaler to Entire DataFrame Including Target

```python
# ❌ WRONG
df_scaled = StandardScaler().fit_transform(df)  # Scales y too!

# ✅ CORRECT
X_scaled = StandardScaler().fit_transform(X)  # Only scale features
```

### 4. Using ColumnTransformer with Wrong Column Names After Split

```python
# ❌ WRONG — Positional indexing fails if columns reorder
preprocessor = ColumnTransformer(transformers=[('num', scaler, [0, 1])])

# ✅ CORRECT — Use column names (works with DataFrames)
preprocessor = ColumnTransformer(transformers=[('num', scaler, ['age', 'income'])])
```

### 5. Forgetting sparse_output in OneHotEncoder

```python
# If you combine with algorithms that don't support sparse matrices:
enc = OneHotEncoder(sparse_output=False)  # Returns dense array
# Or convert after:
# X_encoded = enc.transform(X).toarray()
```

---

## Interview Questions

### Conceptual

**Q1: Why do we need to scale features?**
> Some algorithms (KNN, SVM, neural networks) compute distances or gradients. Without scaling, features with larger magnitudes dominate. StandardScaler makes all features have equal influence by setting mean=0, std=1.

**Q2: What's the difference between StandardScaler and MinMaxScaler? When would you use each?**
> StandardScaler: centers to mean=0, std=1 — best for normally distributed data. No bounded range.
> MinMaxScaler: scales to [0,1] — best for data that's not normally distributed or when you need bounded values (neural network inputs). Sensitive to outliers.

**Q3: What is a Pipeline and why is it important?**
> A Pipeline chains preprocessing steps and a model into one object. It's critical because: (1) prevents data leakage by ensuring fit is only called on training data, (2) makes code cleaner, (3) enables GridSearchCV across all steps, (4) is deployable as a single object.

**Q4: Explain data leakage in the context of preprocessing.**
> If you fit a scaler on the FULL dataset (including test data) before splitting, the scaler's mean/std contains information from test samples. This makes your model appear better than it actually is. Prevention: always split first, or use Pipelines which enforce correct ordering.

**Q5: When would you use ColumnTransformer vs Pipeline?**
> Pipeline chains steps sequentially (all columns go through each step). ColumnTransformer applies different transformations to different column subsets in parallel. Use ColumnTransformer when your dataset has mixed types (numeric + categorical). They're often used together: ColumnTransformer inside a Pipeline.

**Q6: How would you handle a categorical feature with 1000 unique values?**
> OneHotEncoding creates 1000 columns — too many. Options: (1) Target encoding — replace category with mean of target, (2) Frequency encoding — replace with occurrence count, (3) Group rare categories into "Other", (4) Use embedding if using deep learning, (5) Hash encoding for controlled dimensionality.

### Coding

**Q7: Write a complete preprocessing pipeline for a dataset with numeric and categorical features, missing values, and outliers.**

```python
import numpy as np
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler, OneHotEncoder, RobustScaler
from sklearn.impute import SimpleImputer
from sklearn.ensemble import RandomForestClassifier

numeric_features = ['age', 'income', 'score']
categorical_features = ['gender', 'city', 'education']

numeric_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', RobustScaler())  # Robust to outliers
])

categorical_pipeline = Pipeline([
    ('imputer', SimpleImputer(strategy='constant', fill_value='missing')),
    ('encoder', OneHotEncoder(handle_unknown='ignore', sparse_output=False))
])

preprocessor = ColumnTransformer([
    ('num', numeric_pipeline, numeric_features),
    ('cat', categorical_pipeline, categorical_features)
])

full_pipeline = Pipeline([
    ('preprocess', preprocessor),
    ('model', RandomForestClassifier(n_estimators=100, random_state=42))
])

# One line to train, one line to predict
# full_pipeline.fit(X_train, y_train)
# full_pipeline.predict(X_test)
```

---

## Quick Reference

### Preprocessing Cheat Sheet

| Task | Tool | Key Parameter |
|------|------|--------------|
| Scale to mean=0, std=1 | `StandardScaler()` | `with_mean`, `with_std` |
| Scale to [0, 1] | `MinMaxScaler()` | `feature_range` |
| Scale ignoring outliers | `RobustScaler()` | `quantile_range` |
| Encode nominal features | `OneHotEncoder()` | `handle_unknown`, `drop` |
| Encode ordinal features | `OrdinalEncoder()` | `categories` (ordered) |
| Encode target labels | `LabelEncoder()` | — |
| Fill missing (simple) | `SimpleImputer()` | `strategy` |
| Fill missing (smart) | `KNNImputer()` | `n_neighbors` |
| Make data normal | `PowerTransformer()` | `method` |
| Create polynomial features | `PolynomialFeatures()` | `degree`, `interaction_only` |
| Custom function | `FunctionTransformer()` | `func`, `inverse_func` |

### Pipeline Syntax

```python
# Explicit names
Pipeline([('name1', Step1()), ('name2', Step2())])

# Auto names
make_pipeline(Step1(), Step2())

# Column-wise
ColumnTransformer([('name', transformer, columns)])

# Parallel features
FeatureUnion([('name1', Transformer1()), ('name2', Transformer2())])

# Access nested params (for GridSearchCV)
'pipeline_step__transformer_param'
# e.g., 'preprocessor__num__scaler__with_mean'
```

### Decision Flowchart: Which Scaler?

```
Data has outliers?
├── YES → RobustScaler
└── NO
    ├── Need bounded range?
    │   ├── YES → MinMaxScaler
    │   └── NO → StandardScaler (default)
    └── Sparse data?
        └── YES → MaxAbsScaler
```

---

*Previous: [01-Sklearn-Overview-and-API-Design](01-Sklearn-Overview-and-API-Design.md)*
*Next: [03-Supervised-Learning-Algorithms](03-Supervised-Learning-Algorithms.md)*
