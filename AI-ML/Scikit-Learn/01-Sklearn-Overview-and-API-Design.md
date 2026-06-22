# Scikit-Learn Overview and API Design

## Table of Contents
- [Introduction to Scikit-Learn](#introduction-to-scikit-learn)
- [Installation and Setup](#installation-and-setup)
- [The Estimator API — The Heart of Sklearn](#the-estimator-api)
- [Data Representation in Sklearn](#data-representation-in-sklearn)
- [Loading Datasets](#loading-datasets)
- [The fit/predict/transform Pattern](#the-fitpredicttransform-pattern)
- [Estimator Types Deep Dive](#estimator-types-deep-dive)
- [Sklearn Utilities and Helpers](#sklearn-utilities-and-helpers)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Introduction to Scikit-Learn

### What It Is
Scikit-Learn (sklearn) is a free, open-source Python library for machine learning. Think of it as a **toolbox** — you don't need to build a hammer from scratch every time you want to hit a nail. Sklearn gives you ready-made, well-tested tools (algorithms) to solve prediction and pattern-recognition problems.

### Why It Matters
- **Industry Standard**: Used in 80%+ of ML projects in production
- **Consistent API**: Learn one pattern, use it for 100+ algorithms
- **Battle-Tested**: Millions of users, thousands of contributors, extensively documented
- **Interoperable**: Works seamlessly with NumPy, Pandas, Matplotlib, and SciPy
- **No Deep Learning**: Focused on classical ML — when you don't need a neural network (which is most of the time), sklearn is your go-to

### How It Works — The Big Picture

```
┌─────────────────────────────────────────────────────────────┐
│                     SCIKIT-LEARN ECOSYSTEM                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Preprocessing │  │   Models     │  │   Evaluation     │  │
│  │              │  │              │  │                  │  │
│  │ - Scaling    │  │ - Classifiers│  │ - Metrics        │  │
│  │ - Encoding   │  │ - Regressors │  │ - Cross-val      │  │
│  │ - Imputation │  │ - Clusterers │  │ - GridSearch     │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│         │                  │                    │            │
│         └──────────────────┼────────────────────┘            │
│                            │                                 │
│                    ┌───────▼────────┐                        │
│                    │   PIPELINE     │                        │
│                    │  (Chains all   │                        │
│                    │   steps)       │                        │
│                    └────────────────┘                        │
│                                                              │
│  Foundation: NumPy arrays + SciPy sparse matrices            │
└─────────────────────────────────────────────────────────────┘
```

---

## Installation and Setup

### What It Is
Getting sklearn onto your system so you can start building ML models.

### Installation Methods

```python
# Method 1: pip (most common)
# pip install scikit-learn

# Method 2: conda (recommended for data science)
# conda install scikit-learn

# Method 3: With all optional dependencies
# pip install scikit-learn[all]
```

### Verify Installation

```python
import sklearn
print(sklearn.__version__)  # e.g., '1.4.0'

# Check all dependencies are met
sklearn.show_versions()  # Shows numpy, scipy, joblib versions
```

### Why It Matters
Version mismatches between sklearn, numpy, and scipy cause 90% of installation headaches. Always verify your installation.

> **Pro Tip**: Use virtual environments! Never install sklearn in your system Python.

```python
# Recommended setup
# python -m venv ml_env
# source ml_env/bin/activate  (Linux/Mac)
# ml_env\Scripts\activate     (Windows)
# pip install scikit-learn numpy pandas matplotlib
```

### Core Dependencies

| Package | Purpose | Minimum Version |
|---------|---------|-----------------|
| NumPy | Array operations | 1.19+ |
| SciPy | Scientific computing | 1.6+ |
| joblib | Parallel processing & model saving | 1.2+ |
| threadpoolctl | Thread management | 2.0+ |

---

## The Estimator API

### What It Is
The Estimator API is sklearn's **design philosophy** — a set of rules that every algorithm in the library follows. It's like how every car has a steering wheel, gas pedal, and brake in the same place, regardless of the brand. Once you learn the pattern, you can use ANY algorithm.

### Why It Matters
- You can swap algorithms with **one line of code**
- Learning curve: steep at first, then flat forever
- Makes code readable and maintainable
- Enables powerful composition (Pipelines)

### How It Works

Every object in sklearn follows this contract:

```
┌─────────────────────────────────────────────────┐
│              ESTIMATOR (Base Class)               │
├─────────────────────────────────────────────────┤
│                                                  │
│  Constructor:  __init__(hyperparameters)          │
│  ─────────────────────────────────────────       │
│  Sets configuration. Does NOT touch data.        │
│                                                  │
│  Methods:                                        │
│  ─────────────────────────────────────────       │
│  .fit(X, y)       → Learn from data             │
│  .predict(X)      → Make predictions            │
│  .transform(X)    → Transform data              │
│  .fit_transform(X)→ fit + transform in one call │
│  .score(X, y)     → Evaluate performance        │
│  .get_params()    → Get hyperparameters          │
│  .set_params()    → Set hyperparameters          │
│                                                  │
│  Attributes (after fit):                         │
│  ─────────────────────────────────────────       │
│  .coef_           → Learned coefficients         │
│  .feature_importances_ → Feature importance      │
│  .labels_         → Cluster assignments          │
│  (Always end with underscore _)                  │
│                                                  │
└─────────────────────────────────────────────────┘
```

> **Critical Rule**: Attributes ending with `_` (underscore) are **learned from data**. They only exist AFTER calling `.fit()`.

### The Three Types of Estimators

| Type | Purpose | Key Methods | Example |
|------|---------|-------------|---------|
| **Estimator** (Predictor) | Make predictions | `fit()`, `predict()` | LinearRegression, SVC |
| **Transformer** | Transform data | `fit()`, `transform()` | StandardScaler, PCA |
| **Meta-Estimator** | Combine estimators | `fit()`, `predict()` | Pipeline, GridSearchCV |

### Code Example — The Universal Pattern

```python
from sklearn.linear_model import LinearRegression
from sklearn.tree import DecisionTreeClassifier
from sklearn.preprocessing import StandardScaler
import numpy as np

# ============================================
# STEP 1: Create the estimator (set hyperparameters)
# ============================================
# No data is seen here — just configuration
model = LinearRegression(fit_intercept=True)
scaler = StandardScaler(with_mean=True, with_std=True)
tree = DecisionTreeClassifier(max_depth=5, random_state=42)

# ============================================
# STEP 2: Fit (learn from data)
# ============================================
X_train = np.array([[1, 2], [3, 4], [5, 6], [7, 8]])
y_train = np.array([1, 2, 3, 4])

model.fit(X_train, y_train)      # Learns coefficients
scaler.fit(X_train)               # Learns mean and std

# ============================================
# STEP 3: Use (predict or transform)
# ============================================
X_new = np.array([[9, 10]])

prediction = model.predict(X_new)          # Returns predicted y
X_scaled = scaler.transform(X_new)         # Returns scaled X

# ============================================
# STEP 4: Inspect learned parameters
# ============================================
print(f"Coefficients: {model.coef_}")        # [0.5, 0.5]
print(f"Intercept: {model.intercept_}")      # ~0.0
print(f"Mean: {scaler.mean_}")              # [4. 5.]
print(f"Std: {scaler.scale_}")             # [2.236 2.236]
```

### The Golden Rules

```python
# Rule 1: __init__ only stores parameters
class MyEstimator:
    def __init__(self, n_components=2):
        self.n_components = n_components  # ✅ Just store it
        # self.components_ = ...          # ❌ NEVER compute in __init__

# Rule 2: fit() always returns self (enables chaining)
scaler.fit(X_train)  # Returns the scaler itself
# So you can chain:
X_scaled = StandardScaler().fit(X_train).transform(X_test)

# Rule 3: fit_transform != fit then transform (sometimes)
# For efficiency, some transformers have optimized fit_transform
X_scaled = scaler.fit_transform(X_train)  # Faster than fit() + transform()

# Rule 4: NEVER fit on test data
scaler.fit(X_train)            # ✅ Learn from training data
X_test_scaled = scaler.transform(X_test)  # ✅ Apply to test data
# scaler.fit(X_test)           # ❌ DATA LEAKAGE!
```

---

## Data Representation in Sklearn

### What It Is
How sklearn expects you to organize your data — like how a library expects books to be on shelves, not scattered on the floor.

### Why It Matters
Wrong data shape = cryptic errors. Understanding data representation eliminates 50% of beginner bugs.

### The Standard Format

```
Features Matrix (X):              Target Vector (y):
┌─────────────────────────┐      ┌──────┐
│ feature1  feature2  ... │      │  y₁  │
│   x₁₁      x₁₂    ... │      │  y₂  │
│   x₂₁      x₂₂    ... │      │  y₃  │
│   ...       ...     ... │      │  ... │
│   xₙ₁      xₙ₂    ... │      │  yₙ  │
└─────────────────────────┘      └──────┘
  Shape: (n_samples, n_features)   Shape: (n_samples,)
  
  Rows = samples (observations)
  Columns = features (variables)
```

### Key Conventions

| Convention | Meaning | Example |
|-----------|---------|---------|
| `X` (uppercase) | Features matrix | 2D array/DataFrame |
| `y` (lowercase) | Target vector | 1D array/Series |
| `n_samples` | Number of rows | 1000 patients |
| `n_features` | Number of columns | age, weight, height |

### Code Example — Data Formats Sklearn Accepts

```python
import numpy as np
import pandas as pd
from sklearn.linear_model import LinearRegression

model = LinearRegression()

# ============================================
# Format 1: NumPy arrays (most common internally)
# ============================================
X_numpy = np.array([[1, 2], [3, 4], [5, 6]])
y_numpy = np.array([10, 20, 30])
model.fit(X_numpy, y_numpy)  # ✅ Works perfectly

# ============================================
# Format 2: Pandas DataFrame/Series (most common in practice)
# ============================================
X_pandas = pd.DataFrame({'age': [25, 30, 35], 'income': [50000, 60000, 70000]})
y_pandas = pd.Series([0, 1, 1], name='purchased')
model.fit(X_pandas, y_pandas)  # ✅ Sklearn handles conversion

# ============================================
# Format 3: Sparse matrices (for text/high-dimensional data)
# ============================================
from scipy.sparse import csr_matrix
X_sparse = csr_matrix([[1, 0, 0], [0, 1, 0], [0, 0, 1]])
y_sparse = np.array([1, 2, 3])
model.fit(X_sparse, y_sparse)  # ✅ Memory efficient

# ============================================
# Common shape errors and fixes
# ============================================
# Single feature must be 2D
X_single = np.array([1, 2, 3, 4, 5])
# model.fit(X_single, y)  # ❌ ValueError: Expected 2D array

X_single_fixed = X_single.reshape(-1, 1)  # Shape: (5, 1)
# model.fit(X_single_fixed, y)  # ✅ Now it's 2D

# Single sample must be 2D
X_one_sample = np.array([1, 2, 3])
# model.predict(X_one_sample)  # ❌ 
X_one_sample_fixed = X_one_sample.reshape(1, -1)  # Shape: (1, 3)
# model.predict(X_one_sample_fixed)  # ✅
```

> **Pro Tip**: When in doubt, use `.reshape(-1, 1)` for single features and `.reshape(1, -1)` for single samples.

### Data Type Requirements

```python
# Sklearn works with numeric data ONLY (internally)
# Categorical data must be encoded first

# ✅ Numeric types accepted:
# int32, int64, float32, float64

# ❌ Will fail or give wrong results:
# strings, objects, dates (must encode first)

# Check your dtypes:
print(X_pandas.dtypes)
# age       int64    ✅
# city      object   ❌ Must encode!
```

---

## Loading Datasets

### What It Is
Sklearn comes with built-in datasets for learning and testing — no downloads needed.

### Why It Matters
- Instant access to well-known datasets for experimentation
- Consistent format for tutorials and benchmarks
- Great for quick prototyping before using real data

### Dataset Types

| Type | Function | Size | Use Case |
|------|----------|------|----------|
| **Toy** | `load_*()` | Small (< 1000 samples) | Learning, quick tests |
| **Real-world** | `fetch_*()` | Large (downloads from internet) | Realistic benchmarks |
| **Generated** | `make_*()` | Customizable | Controlled experiments |

### Code Example — All Ways to Load Data

```python
from sklearn import datasets

# ============================================
# TOY DATASETS (bundled with sklearn)
# ============================================

# Classification
iris = datasets.load_iris()         # 150 samples, 4 features, 3 classes
digits = datasets.load_digits()     # 1797 samples, 64 features (8x8 images)
wine = datasets.load_wine()         # 178 samples, 13 features, 3 classes
breast_cancer = datasets.load_breast_cancer()  # 569 samples, 30 features

# Regression
boston = datasets.load_diabetes()    # 442 samples, 10 features
linnerud = datasets.load_linnerud() # 20 samples, 3 features

# ============================================
# Understanding the returned object (Bunch)
# ============================================
iris = datasets.load_iris()

print(type(iris))              # sklearn.utils.Bunch (dict-like)
print(iris.keys())             # ['data', 'target', 'feature_names', ...]

X = iris.data                  # Features: shape (150, 4)
y = iris.target                # Target: shape (150,) — 0, 1, 2
feature_names = iris.feature_names  # ['sepal length', 'sepal width', ...]
target_names = iris.target_names    # ['setosa', 'versicolor', 'virginica']

print(f"Samples: {X.shape[0]}, Features: {X.shape[1]}")
print(f"Classes: {target_names}")
print(f"First sample: {X[0]} → {target_names[y[0]]}")
# First sample: [5.1 3.5 1.4 0.2] → setosa

# ============================================
# Load as DataFrame directly (sklearn >= 1.0)
# ============================================
iris_df = datasets.load_iris(as_frame=True)
print(iris_df.frame.head())
#    sepal length  sepal width  petal length  petal width  target
# 0          5.1          3.5           1.4          0.2       0

# ============================================
# REAL-WORLD DATASETS (downloaded on first use)
# ============================================
# california = datasets.fetch_california_housing()  # Regression
# newsgroups = datasets.fetch_20newsgroups()        # Text classification
# olivetti = datasets.fetch_olivetti_faces()        # Face recognition

# ============================================
# GENERATED DATASETS (most useful for learning!)
# ============================================

# Classification with controlled difficulty
X, y = datasets.make_classification(
    n_samples=1000,      # Number of data points
    n_features=20,       # Total features
    n_informative=10,    # Features actually useful
    n_redundant=5,       # Linearly combined from informative
    n_classes=2,         # Binary classification
    random_state=42      # Reproducibility
)
print(f"Generated classification: X={X.shape}, y={y.shape}")

# Regression
X, y = datasets.make_regression(
    n_samples=500,
    n_features=5,
    n_informative=3,     # Only 3 features matter
    noise=10.0,          # Add Gaussian noise
    random_state=42
)

# Clustering (blobs)
X, y = datasets.make_blobs(
    n_samples=300,
    centers=4,           # 4 clusters
    cluster_std=1.0,     # Spread of clusters
    random_state=42
)

# Moons and circles (non-linear boundaries)
X_moons, y_moons = datasets.make_moons(n_samples=200, noise=0.1)
X_circles, y_circles = datasets.make_circles(n_samples=200, noise=0.05, factor=0.5)
```

> **Pro Tip**: `make_classification` with `n_informative < n_features` is perfect for testing feature selection methods. The "noise" features let you verify your feature selection is working.

---

## The fit/predict/transform Pattern

### What It Is
The three sacred verbs of sklearn. Every single operation follows this pattern:
- **fit**: "Learn something from this data"
- **predict**: "Use what you learned to make a prediction"
- **transform**: "Use what you learned to change this data"

### Why It Matters
This pattern enforces the separation of **learning** from **applying** — the #1 principle that prevents data leakage and ensures reproducible ML pipelines.

### The Mental Model

```
                    TRAINING PHASE              INFERENCE PHASE
                    ──────────────              ───────────────
                    
Training Data ──→ ┌───────────┐
                  │   fit()   │ → Learned State (stored internally)
                  └───────────┘           │
                                          ▼
                                  ┌──────────────┐
                  New Data ──────→│  predict()   │──→ Predictions
                                  │     OR       │
                                  │ transform()  │──→ Transformed Data
                                  └──────────────┘
```

### Detailed Breakdown

```python
import numpy as np
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split

# Generate example data
np.random.seed(42)
X = np.random.randn(100, 3) * [10, 1, 100]  # Different scales
y = (X[:, 0] + X[:, 1] > 5).astype(int)     # Binary target

# Split data
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# ============================================
# TRANSFORMER: StandardScaler
# ============================================

scaler = StandardScaler()

# fit() — Learns mean and std from TRAINING data only
scaler.fit(X_train)
# After fit: scaler.mean_ = [mean_col1, mean_col2, mean_col3]
#            scaler.scale_ = [std_col1, std_col2, std_col3]

# transform() — Applies the learned transformation
X_train_scaled = scaler.transform(X_train)  # Uses train's mean/std
X_test_scaled = scaler.transform(X_test)    # Uses SAME train's mean/std ✅

# fit_transform() — Shortcut for fit() + transform() on SAME data
X_train_scaled = scaler.fit_transform(X_train)  # Equivalent to above 2 lines

# ============================================
# PREDICTOR: LogisticRegression
# ============================================

model = LogisticRegression()

# fit() — Learns coefficients from training data
model.fit(X_train_scaled, y_train)
# After fit: model.coef_ = learned weights
#            model.intercept_ = learned bias

# predict() — Makes predictions on new data
y_pred = model.predict(X_test_scaled)
print(f"Predictions: {y_pred[:5]}")  # [0, 1, 0, 1, 1]

# predict_proba() — Returns probability estimates
y_proba = model.predict_proba(X_test_scaled)
print(f"Probabilities: {y_proba[:3]}")
# [[0.89, 0.11], [0.23, 0.77], [0.95, 0.05]]
#  P(class=0), P(class=1)

# score() — Returns accuracy (or R² for regressors)
accuracy = model.score(X_test_scaled, y_test)
print(f"Accuracy: {accuracy:.2f}")

# ============================================
# DECISION FUNCTION (for some models)
# ============================================
# Raw output before sigmoid/threshold
decision_scores = model.decision_function(X_test_scaled)
# Positive = class 1, Negative = class 0
```

### Why fit() and transform()/predict() are Separate

```python
# WRONG: Fitting on test data (DATA LEAKAGE!)
# ─────────────────────────────────────────────
scaler_bad = StandardScaler()
X_train_scaled = scaler_bad.fit_transform(X_train)
X_test_scaled = scaler_bad.fit_transform(X_test)  # ❌ Refits on test!
# Problem: Test data mean/std leak into the scaling
# This makes test results overly optimistic

# RIGHT: Fit on train, transform on test
# ─────────────────────────────────────────────
scaler_good = StandardScaler()
X_train_scaled = scaler_good.fit_transform(X_train)  # Learn + apply
X_test_scaled = scaler_good.transform(X_test)        # Only apply ✅
```

> **Important**: The separation exists because in production, you'll save the fitted scaler/model and apply it to NEW data that arrives one sample at a time. You can't "fit" on a single sample.

---

## Estimator Types Deep Dive

### Classifiers

```python
from sklearn.linear_model import LogisticRegression
from sklearn.tree import DecisionTreeClassifier
from sklearn.svm import SVC
from sklearn.neighbors import KNeighborsClassifier
from sklearn.naive_bayes import GaussianNB

# All classifiers follow the SAME interface:
classifiers = [
    LogisticRegression(),
    DecisionTreeClassifier(max_depth=5),
    SVC(kernel='rbf'),
    KNeighborsClassifier(n_neighbors=5),
    GaussianNB()
]

# You can loop through them — SAME API!
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split

X, y = make_classification(n_samples=200, n_features=10, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=42)

for clf in classifiers:
    clf.fit(X_train, y_train)          # Same method
    score = clf.score(X_test, y_test)  # Same method
    print(f"{clf.__class__.__name__:30s} Accuracy: {score:.3f}")

# Output:
# LogisticRegression                Accuracy: 0.880
# DecisionTreeClassifier            Accuracy: 0.860
# SVC                               Accuracy: 0.900
# KNeighborsClassifier              Accuracy: 0.860
# GaussianNB                        Accuracy: 0.860
```

### Regressors

```python
from sklearn.linear_model import LinearRegression, Ridge, Lasso
from sklearn.tree import DecisionTreeRegressor
from sklearn.svm import SVR

# Regressors: predict continuous values
# score() returns R² (not accuracy)

from sklearn.datasets import make_regression

X, y = make_regression(n_samples=200, n_features=5, noise=10, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=42)

regressors = [
    LinearRegression(),
    Ridge(alpha=1.0),
    Lasso(alpha=1.0),
    DecisionTreeRegressor(max_depth=5),
    SVR(kernel='rbf')
]

for reg in regressors:
    reg.fit(X_train, y_train)
    r2 = reg.score(X_test, y_test)  # R² score
    print(f"{reg.__class__.__name__:30s} R²: {r2:.3f}")
```

### Transformers

```python
from sklearn.preprocessing import (
    StandardScaler, MinMaxScaler, RobustScaler,
    LabelEncoder, OneHotEncoder
)
from sklearn.decomposition import PCA
from sklearn.feature_selection import SelectKBest

# Transformers: change the shape or values of X
# They have transform() instead of predict()

X = np.random.randn(100, 5)

# StandardScaler: zero mean, unit variance
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)
print(f"Mean after scaling: {X_scaled.mean(axis=0).round(2)}")  # [0, 0, 0, 0, 0]
print(f"Std after scaling: {X_scaled.std(axis=0).round(2)}")    # [1, 1, 1, 1, 1]

# PCA: reduce dimensions
pca = PCA(n_components=2)
X_reduced = pca.fit_transform(X)
print(f"Original shape: {X.shape}")   # (100, 5)
print(f"Reduced shape: {X_reduced.shape}")  # (100, 2)
print(f"Explained variance: {pca.explained_variance_ratio_}")
```

### get_params() and set_params()

```python
from sklearn.ensemble import RandomForestClassifier

# get_params() — inspect all hyperparameters
rf = RandomForestClassifier(n_estimators=100, max_depth=5, random_state=42)
params = rf.get_params()
print(params)
# {'n_estimators': 100, 'max_depth': 5, 'random_state': 42, ...}

# set_params() — change hyperparameters after creation
rf.set_params(n_estimators=200, max_depth=10)
print(rf.get_params()['n_estimators'])  # 200

# This is how GridSearchCV changes parameters internally!
```

### Cloning Estimators

```python
from sklearn.base import clone

# clone() creates a new estimator with same parameters but UN-fitted
original = RandomForestClassifier(n_estimators=100, random_state=42)
original.fit(X_train, y_train)  # Fitted

cloned = clone(original)  # Same params, but NOT fitted
# cloned.predict(X_test)  # ❌ NotFittedError!
cloned.fit(X_train, y_train)  # Must fit again
```

---

## Sklearn Utilities and Helpers

### Checking if an Estimator is Fitted

```python
from sklearn.utils.validation import check_is_fitted
from sklearn.linear_model import LinearRegression

model = LinearRegression()

try:
    check_is_fitted(model)
except Exception as e:
    print(f"Not fitted: {e}")
    # NotFittedError: This LinearRegression instance is not fitted yet.

model.fit([[1], [2], [3]], [1, 2, 3])
check_is_fitted(model)  # ✅ No error
```

### Type Checking Utilities

```python
from sklearn.base import is_classifier, is_regressor
from sklearn.linear_model import LogisticRegression, LinearRegression

print(is_classifier(LogisticRegression()))  # True
print(is_regressor(LinearRegression()))     # True
print(is_classifier(LinearRegression()))    # False
```

### Configuration and Global Settings

```python
import sklearn

# Set global configuration
sklearn.set_config(
    transform_output="pandas",  # Transformers return DataFrames (sklearn >= 1.2)
    display='diagram'           # Rich HTML display in Jupyter
)

# Temporary config change
with sklearn.config_context(transform_output="default"):
    # Inside this block, transformers return numpy arrays
    pass
```

> **Pro Tip (sklearn >= 1.2)**: Setting `transform_output="pandas"` makes ALL transformers return DataFrames with column names. This is incredible for debugging pipelines.

---

## Common Mistakes

### 1. Data Leakage Through fit_transform on Test Data

```python
# ❌ WRONG
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.fit_transform(X_test)  # Leaks test statistics!

# ✅ CORRECT
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)  # Only transform!
```

### 2. Passing 1D Array as Features

```python
# ❌ WRONG
X = [1, 2, 3, 4, 5]
# model.fit(X, y)  # ValueError: Expected 2D array

# ✅ CORRECT
X = np.array([1, 2, 3, 4, 5]).reshape(-1, 1)  # Shape: (5, 1)
```

### 3. Forgetting to fit Before predict

```python
# ❌ WRONG
model = LinearRegression()
# model.predict([[1, 2]])  # NotFittedError

# ✅ CORRECT
model = LinearRegression()
model.fit(X_train, y_train)
model.predict([[1, 2]])
```

### 4. Using the Wrong Metric

```python
# ❌ WRONG: Using accuracy for imbalanced datasets
# If 95% of data is class 0, a model predicting all 0s gets 95% accuracy!

# ✅ CORRECT: Use appropriate metrics
from sklearn.metrics import f1_score, roc_auc_score, classification_report
```

### 5. Not Setting random_state

```python
# ❌ WRONG: Results change every run
model = DecisionTreeClassifier()

# ✅ CORRECT: Reproducible results
model = DecisionTreeClassifier(random_state=42)
```

### 6. Modifying Data After Fitting

```python
# ❌ RISKY: Sklearn stores references, not copies (sometimes)
X_train_original = np.array([[1, 2], [3, 4]])
model.fit(X_train_original, y_train)
X_train_original[0, 0] = 999  # May corrupt model internals!

# ✅ SAFER: Use copies or don't modify training data after fitting
```

---

## Interview Questions

### Conceptual Questions

**Q1: What is the Estimator API in scikit-learn?**
> The Estimator API is a consistent interface that all sklearn objects implement. Every estimator has `fit()` to learn from data, and either `predict()` (for models) or `transform()` (for data transformers). This uniformity allows swapping algorithms with minimal code changes and enables composition via Pipelines.

**Q2: What's the difference between `fit_transform()` and `fit().transform()`?**
> Functionally identical for most transformers, but `fit_transform()` can be optimized internally (e.g., PCA computes both in one SVD pass). Always use `fit_transform()` on training data for potential speed gains. Never use `fit_transform()` on test data — only `transform()`.

**Q3: Why do learned attributes end with an underscore?**
> Convention to distinguish hyperparameters (set by user in `__init__`) from learned attributes (set by data in `fit()`). Example: `alpha` is a hyperparameter, `coef_` is learned. This makes it clear what's user-specified vs. data-derived.

**Q4: What is data leakage and how does sklearn's API prevent it?**
> Data leakage occurs when test/future data information bleeds into the training process. Sklearn prevents it by separating `fit()` (learning) from `transform()`/`predict()` (applying). You fit on training data only, then apply to test data. Pipelines enforce this automatically.

**Q5: How does sklearn handle feature names?**
> Since sklearn 1.0, estimators track feature names via `feature_names_in_` (set during fit). If you fit with a DataFrame, sklearn records column names and validates them during predict/transform. Use `get_feature_names_out()` to get output feature names after transformation.

### Coding Questions

**Q6: Write code to compare 5 different classifiers on the same dataset.**
```python
from sklearn.datasets import load_iris
from sklearn.model_selection import cross_val_score
from sklearn.linear_model import LogisticRegression
from sklearn.tree import DecisionTreeClassifier
from sklearn.svm import SVC
from sklearn.neighbors import KNeighborsClassifier
from sklearn.ensemble import RandomForestClassifier

X, y = load_iris(return_X_y=True)

models = {
    'Logistic Regression': LogisticRegression(max_iter=200),
    'Decision Tree': DecisionTreeClassifier(random_state=42),
    'SVM': SVC(random_state=42),
    'KNN': KNeighborsClassifier(),
    'Random Forest': RandomForestClassifier(random_state=42)
}

for name, model in models.items():
    scores = cross_val_score(model, X, y, cv=5, scoring='accuracy')
    print(f"{name:25s} Mean: {scores.mean():.3f} ± {scores.std():.3f}")
```

**Q7: What happens if you call predict() before fit()?**
> Raises `NotFittedError`. You can check programmatically with `sklearn.utils.validation.check_is_fitted(estimator)`.

---

## Quick Reference

### Sklearn Object Hierarchy

| Base Class | Has fit() | Has predict() | Has transform() | Example |
|-----------|-----------|---------------|-----------------|---------|
| Estimator | ✅ | ❌ | ❌ | Base class |
| Predictor | ✅ | ✅ | ❌ | LinearRegression |
| Transformer | ✅ | ❌ | ✅ | StandardScaler |
| Both | ✅ | ✅ | ✅ | Some clustering |

### Essential Import Patterns

```python
# Models
from sklearn.linear_model import LinearRegression, LogisticRegression, Ridge, Lasso
from sklearn.tree import DecisionTreeClassifier, DecisionTreeRegressor
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.svm import SVC, SVR
from sklearn.neighbors import KNeighborsClassifier

# Preprocessing
from sklearn.preprocessing import StandardScaler, MinMaxScaler, LabelEncoder, OneHotEncoder
from sklearn.impute import SimpleImputer

# Model Selection
from sklearn.model_selection import train_test_split, cross_val_score, GridSearchCV

# Metrics
from sklearn.metrics import accuracy_score, f1_score, mean_squared_error, r2_score

# Pipelines
from sklearn.pipeline import Pipeline, make_pipeline
from sklearn.compose import ColumnTransformer

# Datasets
from sklearn.datasets import load_iris, make_classification, make_regression
```

### Method Cheat Sheet

| Method | Purpose | Returns |
|--------|---------|---------|
| `fit(X, y)` | Learn from data | self |
| `predict(X)` | Predict target | array of predictions |
| `predict_proba(X)` | Predict probabilities | array of probabilities |
| `transform(X)` | Transform features | transformed array |
| `fit_transform(X)` | fit + transform | transformed array |
| `score(X, y)` | Evaluate model | float (accuracy/R²) |
| `get_params()` | Get hyperparameters | dict |
| `set_params(**p)` | Set hyperparameters | self |
| `decision_function(X)` | Raw prediction scores | array of scores |

### Key Attributes (After fit)

| Attribute | Found In | Meaning |
|-----------|----------|---------|
| `coef_` | Linear models | Feature weights |
| `intercept_` | Linear models | Bias term |
| `feature_importances_` | Tree-based models | Feature importance scores |
| `classes_` | Classifiers | Unique class labels |
| `n_features_in_` | All (sklearn≥1.0) | Number of features seen in fit |
| `feature_names_in_` | All (sklearn≥1.0) | Feature names from DataFrame |
| `mean_`, `scale_` | StandardScaler | Learned mean and std |
| `components_` | PCA | Principal components |

---

*Next Chapter: [02-Preprocessing-and-Pipelines](02-Preprocessing-and-Pipelines.md)*
