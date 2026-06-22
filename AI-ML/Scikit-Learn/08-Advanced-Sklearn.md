# Chapter 08: Advanced Scikit-Learn — Custom Estimators, Transformers, and Model Persistence

## Table of Contents
- [Introduction](#introduction)
- [Custom Transformers](#custom-transformers)
- [Custom Estimators](#custom-estimators)
- [Model Persistence — Saving and Loading Models](#model-persistence--saving-and-loading-models)
- [Advanced Pipelines](#advanced-pipelines)
- [Parallelism and Performance](#parallelism-and-performance)
- [Callbacks and Logging](#callbacks-and-logging)
- [Sklearn Configuration and Global Settings](#sklearn-configuration-and-global-settings)
- [Production Patterns](#production-patterns)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Introduction

### What It Is
Advanced scikit-learn is about going beyond `fit/predict` — building your own reusable components, saving models for production, optimizing performance, and writing code that integrates cleanly into sklearn's ecosystem.

Think of it like learning to drive vs learning to build and maintain a car. Chapters 01–07 taught you to drive. This chapter teaches you to build custom engines and maintain the vehicle.

### Why It Matters
- **Custom Transformers**: Your real-world data needs domain-specific preprocessing that sklearn doesn't provide out of the box
- **Custom Estimators**: You may invent a new algorithm or need to wrap a non-sklearn model into sklearn's API
- **Model Persistence**: A model that only lives in a Jupyter notebook is useless — production means saving, loading, and versioning
- **Performance**: Training on millions of rows requires understanding parallelism, memory, and incremental learning

---

## Custom Transformers

### What They Are
Custom transformers are your own preprocessing steps that follow sklearn's API (`fit`, `transform`, `fit_transform`). They plug seamlessly into `Pipeline` and `GridSearchCV`.

**Analogy**: Sklearn gives you standard Lego bricks. Custom transformers let you 3D-print your own bricks that still snap into the same system.

### Why You Need Them
- Domain-specific feature engineering (e.g., extracting day-of-week from dates)
- Complex multi-column transformations
- Conditional logic that standard transformers can't handle
- Wrapping external libraries into sklearn-compatible components

### Method 1: FunctionTransformer (Quick & Simple)

For stateless transformations (no fitting needed):

```python
import numpy as np
import pandas as pd
from sklearn.preprocessing import FunctionTransformer

# Simple transformation: log(1 + x)
log_transformer = FunctionTransformer(
    func=np.log1p,           # Forward transformation
    inverse_func=np.expm1,    # Inverse (for inverse_transform)
    validate=True             # Check for NaN/Inf
)

X = np.array([[1, 10], [100, 1000], [10000, 100000]])
X_log = log_transformer.fit_transform(X)
print("Original:\n", X)
print("Log-transformed:\n", X_log)
print("Inverse:\n", log_transformer.inverse_transform(X_log))
```

```python
# FunctionTransformer with pandas DataFrames
def extract_date_features(df):
    """Extract useful features from datetime columns."""
    df = df.copy()
    df['hour'] = df['timestamp'].dt.hour
    df['day_of_week'] = df['timestamp'].dt.dayofweek
    df['is_weekend'] = df['day_of_week'].isin([5, 6]).astype(int)
    df['month'] = df['timestamp'].dt.month
    df = df.drop(columns=['timestamp'])
    return df

date_transformer = FunctionTransformer(extract_date_features)

# Example usage
df = pd.DataFrame({
    'timestamp': pd.date_range('2024-01-01', periods=5, freq='12h'),
    'value': [10, 20, 30, 40, 50]
})
print(date_transformer.fit_transform(df))
```

> **Limitation**: `FunctionTransformer` is stateless — it can't learn anything from training data (e.g., can't compute means for imputation). For stateful transforms, you need a class-based transformer.

### Method 2: Class-Based Transformer (Full Power)

Use `BaseEstimator` and `TransformerMixin` to build a proper transformer.

```python
from sklearn.base import BaseEstimator, TransformerMixin
import numpy as np

class OutlierClipper(BaseEstimator, TransformerMixin):
    """
    Clips outliers to [Q1 - factor*IQR, Q3 + factor*IQR].
    Learns clip boundaries from training data (stateful).
    """
    
    def __init__(self, factor=1.5):
        # __init__ should ONLY assign parameters as attributes
        # DO NOT do any computation here
        self.factor = factor
    
    def fit(self, X, y=None):
        # Learn clip boundaries from training data
        X = np.array(X)
        Q1 = np.percentile(X, 25, axis=0)
        Q3 = np.percentile(X, 75, axis=0)
        IQR = Q3 - Q1
        
        self.lower_bound_ = Q1 - self.factor * IQR
        self.upper_bound_ = Q3 + self.factor * IQR
        
        # MUST return self for chaining
        return self
    
    def transform(self, X):
        # Apply learned boundaries to new data
        X = np.array(X, dtype=float).copy()
        X = np.clip(X, self.lower_bound_, self.upper_bound_)
        return X

# Usage
X_train = np.array([[1, 10], [2, 20], [3, 30], [4, 40], [100, 500]])  # Last row = outlier
X_test = np.array([[2.5, 25], [200, 1000]])

clipper = OutlierClipper(factor=1.5)
clipper.fit(X_train)

print(f"Lower bounds: {clipper.lower_bound_}")
print(f"Upper bounds: {clipper.upper_bound_}")
print(f"Clipped train:\n{clipper.transform(X_train)}")
print(f"Clipped test:\n{clipper.transform(X_test)}")
```

### The Rules of Sklearn Estimators

These rules MUST be followed for compatibility with `Pipeline`, `GridSearchCV`, `clone()`, etc.:

```
RULE 1: __init__ must ONLY store parameters as attributes
        ✗ self.model = SomeModel()        # Don't create objects
        ✗ self.data = load_data()          # Don't load data
        ✓ self.alpha = alpha               # Just store params

RULE 2: Fitted attributes end with underscore _
        ✗ self.mean = X.mean()             # No trailing underscore
        ✓ self.mean_ = X.mean()            # Correct convention
        
RULE 3: fit() must return self
        return self                        # Always!

RULE 4: transform() must not modify input
        X = X.copy()                       # Always copy first

RULE 5: get_params() / set_params() must work
        (Free if you use BaseEstimator)
```

```python
# Demonstrating why these rules matter

from sklearn.base import clone

clipper = OutlierClipper(factor=2.0)
clipper.fit(X_train)

# clone() creates a fresh unfitted copy with same params
clipper_clone = clone(clipper)
print(f"Original factor: {clipper.factor}")       # 2.0
print(f"Clone factor: {clipper_clone.factor}")     # 2.0
print(f"Clone is fitted: {hasattr(clipper_clone, 'lower_bound_')}")  # False

# get_params() — used by GridSearchCV to read parameters
print(f"Params: {clipper.get_params()}")  # {'factor': 2.0}

# set_params() — used by GridSearchCV to set parameters
clipper.set_params(factor=3.0)
print(f"Updated factor: {clipper.factor}")  # 3.0
```

### Advanced Custom Transformer: DataFrame-Aware

```python
from sklearn.base import BaseEstimator, TransformerMixin
import pandas as pd
import numpy as np

class TargetEncoder(BaseEstimator, TransformerMixin):
    """
    Encodes categorical columns using target mean (with smoothing).
    
    smoothing formula:
        encoded = weight * category_mean + (1 - weight) * global_mean
        weight = count / (count + smoothing_factor)
    """
    
    def __init__(self, columns=None, smoothing=10):
        self.columns = columns
        self.smoothing = smoothing
    
    def fit(self, X, y):
        X = pd.DataFrame(X).copy()
        y = np.array(y)
        
        if self.columns is None:
            # Auto-detect object/category columns
            self.columns_ = X.select_dtypes(include=['object', 'category']).columns.tolist()
        else:
            self.columns_ = list(self.columns)
        
        self.global_mean_ = y.mean()
        self.encoding_maps_ = {}
        
        for col in self.columns_:
            # Group by category, compute mean and count
            stats = pd.DataFrame({'target': y, 'category': X[col]})
            agg = stats.groupby('category')['target'].agg(['mean', 'count'])
            
            # Apply smoothing
            weight = agg['count'] / (agg['count'] + self.smoothing)
            smoothed = weight * agg['mean'] + (1 - weight) * self.global_mean_
            
            self.encoding_maps_[col] = smoothed.to_dict()
        
        return self
    
    def transform(self, X):
        X = pd.DataFrame(X).copy()
        
        for col in self.columns_:
            # Map known categories, use global mean for unknown
            X[col] = X[col].map(self.encoding_maps_[col]).fillna(self.global_mean_)
        
        return X

# Usage
df_train = pd.DataFrame({
    'city': ['NYC', 'LA', 'NYC', 'SF', 'LA', 'NYC', 'SF', 'LA'],
    'size': [100, 200, 150, 300, 250, 120, 350, 180],
})
y_train = np.array([500, 400, 550, 800, 450, 520, 780, 420])

encoder = TargetEncoder(columns=['city'], smoothing=2)
encoder.fit(df_train, y_train)

print("Encoding map:", encoder.encoding_maps_)
print("\nTransformed:")
print(encoder.transform(df_train))

# Unknown categories get global mean
df_test = pd.DataFrame({'city': ['NYC', 'Chicago'], 'size': [130, 200]})
print("\nTest with unknown 'Chicago':")
print(encoder.transform(df_test))
```

### Custom Transformer with Feature Names (sklearn ≥1.0)

```python
from sklearn.base import BaseEstimator, TransformerMixin
from sklearn.utils.validation import check_is_fitted

class LogTransformer(BaseEstimator, TransformerMixin):
    """Log-transforms specified columns, compatible with set_output(transform='pandas')."""
    
    def __init__(self, offset=1.0):
        self.offset = offset
    
    def fit(self, X, y=None):
        # Store feature names if DataFrame
        if hasattr(X, 'columns'):
            self.feature_names_in_ = np.array(X.columns)
        self.n_features_in_ = X.shape[1] if hasattr(X, 'shape') else len(X[0])
        return self
    
    def transform(self, X):
        check_is_fitted(self)
        return np.log(np.array(X) + self.offset)
    
    def get_feature_names_out(self, input_features=None):
        """Required for set_output(transform='pandas') support."""
        check_is_fitted(self)
        if input_features is not None:
            return np.array([f"log_{f}" for f in input_features])
        if hasattr(self, 'feature_names_in_'):
            return np.array([f"log_{f}" for f in self.feature_names_in_])
        return np.array([f"log_x{i}" for i in range(self.n_features_in_)])

# Usage with pandas output
import sklearn
sklearn.set_config(transform_output="pandas")

log_t = LogTransformer(offset=1)
df = pd.DataFrame({'income': [50000, 60000, 70000], 'age': [25, 30, 35]})
result = log_t.fit_transform(df)
print(result)
#    log_income   log_age
# 0   10.819...  3.258...
# 1   11.002...  3.434...
# 2   11.156...  3.583...
```

---

## Custom Estimators

### What They Are
Custom estimators are your own models (classifiers/regressors) that follow sklearn's API. They can be used with `cross_val_score`, `GridSearchCV`, `Pipeline`, and all other sklearn utilities.

### Building a Custom Classifier

```python
from sklearn.base import BaseEstimator, ClassifierMixin
from sklearn.utils.validation import check_X_y, check_array, check_is_fitted
from sklearn.utils.multiclass import unique_labels
import numpy as np

class KNearestMeanClassifier(BaseEstimator, ClassifierMixin):
    """
    Classifies based on nearest class centroid (mean).
    Like KNN but compares to class centers, not individual points.
    
    Simple, fast, but assumes convex class boundaries.
    """
    
    def __init__(self, weighted=False):
        self.weighted = weighted  # Weight by inverse distance?
    
    def fit(self, X, y):
        # Validate inputs
        X, y = check_X_y(X, y)
        
        # Store classes seen during fit
        self.classes_ = unique_labels(y)
        
        # Compute centroid for each class
        self.centroids_ = {}
        for cls in self.classes_:
            self.centroids_[cls] = X[y == cls].mean(axis=0)
        
        # Store for validation
        self.X_ = X
        self.y_ = y
        self.n_features_in_ = X.shape[1]
        
        return self
    
    def predict(self, X):
        check_is_fitted(self)
        X = check_array(X)
        
        # Compute distance to each centroid
        predictions = []
        for x in X:
            distances = {
                cls: np.linalg.norm(x - centroid)
                for cls, centroid in self.centroids_.items()
            }
            predictions.append(min(distances, key=distances.get))
        
        return np.array(predictions)
    
    def predict_proba(self, X):
        """Probability estimates based on inverse distance to centroids."""
        check_is_fitted(self)
        X = check_array(X)
        
        probas = []
        for x in X:
            distances = np.array([
                np.linalg.norm(x - self.centroids_[cls])
                for cls in self.classes_
            ])
            # Convert distances to probabilities (inverse distance)
            inv_dist = 1.0 / (distances + 1e-10)
            probas.append(inv_dist / inv_dist.sum())
        
        return np.array(probas)

# Usage — works with all sklearn utilities!
from sklearn.model_selection import cross_val_score, GridSearchCV
from sklearn.datasets import load_iris
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler

X, y = load_iris(return_X_y=True)

# Cross-validation
model = KNearestMeanClassifier()
scores = cross_val_score(model, X, y, cv=5, scoring='accuracy')
print(f"KNearestMean CV accuracy: {scores.mean():.4f} ± {scores.std():.4f}")

# In a pipeline
pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('model', KNearestMeanClassifier())
])
scores = cross_val_score(pipe, X, y, cv=5)
print(f"Pipeline CV accuracy: {scores.mean():.4f} ± {scores.std():.4f}")

# GridSearchCV
param_grid = {'model__weighted': [True, False]}
grid = GridSearchCV(pipe, param_grid, cv=5)
grid.fit(X, y)
print(f"Best params: {grid.best_params_}")
```

### Building a Custom Regressor

```python
from sklearn.base import BaseEstimator, RegressorMixin
from sklearn.utils.validation import check_X_y, check_array, check_is_fitted

class WeightedAverageRegressor(BaseEstimator, RegressorMixin):
    """
    Predicts using weighted average of K nearest training samples.
    Weights: 1/distance^power
    """
    
    def __init__(self, k=5, power=2):
        self.k = k
        self.power = power
    
    def fit(self, X, y):
        X, y = check_X_y(X, y)
        self.X_train_ = X.copy()
        self.y_train_ = y.copy()
        self.n_features_in_ = X.shape[1]
        return self
    
    def predict(self, X):
        check_is_fitted(self)
        X = check_array(X)
        
        predictions = []
        for x in X:
            # Compute distances to all training points
            distances = np.linalg.norm(self.X_train_ - x, axis=1)
            
            # Get k nearest indices
            nearest_idx = np.argsort(distances)[:self.k]
            nearest_dist = distances[nearest_idx]
            nearest_y = self.y_train_[nearest_idx]
            
            # Weighted average (inverse distance weighting)
            weights = 1.0 / (nearest_dist ** self.power + 1e-10)
            prediction = np.average(nearest_y, weights=weights)
            predictions.append(prediction)
        
        return np.array(predictions)

# Test with sklearn utilities
from sklearn.datasets import make_regression
from sklearn.model_selection import cross_val_score

X_reg, y_reg = make_regression(n_samples=200, n_features=5, noise=10, random_state=42)

model = WeightedAverageRegressor(k=5, power=2)
scores = cross_val_score(model, X_reg, y_reg, cv=5, scoring='r2')
print(f"Custom Regressor R²: {scores.mean():.4f} ± {scores.std():.4f}")
```

### Sklearn Estimator Tags and Checks

```python
from sklearn.utils.estimator_checks import check_estimator

# Verify your estimator follows all sklearn conventions
# This runs ~30 tests covering edge cases
try:
    check_estimator(KNearestMeanClassifier())
    print("✓ Estimator passes all sklearn checks")
except Exception as e:
    print(f"✗ Failed: {e}")
```

> **Pro Tip**: Always run `check_estimator()` on your custom estimators. It catches subtle bugs like not handling single-sample inputs, not cloning properly, or mutating input data.

### Mixins Summary

| Mixin | What It Provides | Required Methods |
|-------|-----------------|-----------------|
| `BaseEstimator` | `get_params()`, `set_params()`, `__repr__()` | `__init__` with explicit params |
| `TransformerMixin` | `fit_transform()` | `fit()`, `transform()` |
| `ClassifierMixin` | `score()` (accuracy) | `fit()`, `predict()` |
| `RegressorMixin` | `score()` (R²) | `fit()`, `predict()` |
| `ClusterMixin` | `fit_predict()` | `fit()`, `predict()` |

---

## Model Persistence — Saving and Loading Models

### What It Is
Model persistence means saving a trained model to disk so you can load and use it later — in production, in another notebook, or months from now.

**Analogy**: Like saving your game progress. You don't want to retrain a model every time you restart your computer.

### Method 1: Joblib (Recommended for Sklearn)

```python
import joblib
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler

# Train a model
X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=42)

pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('model', RandomForestClassifier(n_estimators=100, random_state=42))
])
pipeline.fit(X_train, y_train)

print(f"Training accuracy: {pipeline.score(X_test, y_test):.4f}")

# ===== SAVE =====
joblib.dump(pipeline, 'iris_pipeline.joblib')
print("Model saved!")

# ===== LOAD =====
loaded_pipeline = joblib.load('iris_pipeline.joblib')
print(f"Loaded accuracy: {loaded_pipeline.score(X_test, y_test):.4f}")

# Verify predictions are identical
original_preds = pipeline.predict(X_test)
loaded_preds = loaded_pipeline.predict(X_test)
print(f"Predictions match: {np.array_equal(original_preds, loaded_preds)}")
```

### Method 2: Pickle (Standard Python)

```python
import pickle

# Save
with open('iris_pipeline.pkl', 'wb') as f:
    pickle.dump(pipeline, f)

# Load
with open('iris_pipeline.pkl', 'rb') as f:
    loaded = pickle.load(f)

print(f"Pickle loaded accuracy: {loaded.score(X_test, y_test):.4f}")
```

### Joblib vs Pickle

| Aspect | Joblib | Pickle |
|--------|--------|--------|
| **Speed (large arrays)** | Faster (optimized for NumPy) | Slower |
| **File size** | Smaller (compressed) | Larger |
| **Compression** | Built-in support | Manual |
| **Best for** | Sklearn models, NumPy arrays | General Python objects |
| **Import** | `import joblib` | `import pickle` (stdlib) |

```python
# Joblib with compression
joblib.dump(pipeline, 'model_compressed.joblib', compress=3)  # 0-9, higher=smaller

import os
size_normal = os.path.getsize('iris_pipeline.joblib')
size_compressed = os.path.getsize('model_compressed.joblib')
print(f"Normal: {size_normal:,} bytes")
print(f"Compressed: {size_compressed:,} bytes")
print(f"Reduction: {(1 - size_compressed/size_normal)*100:.1f}%")
```

### Method 3: Skops (Secure, Modern)

> **Security Warning**: `pickle` and `joblib` can execute arbitrary code when loading. Never load a model from an untrusted source!

```python
# skops provides secure serialization
# pip install skops

import skops.io as sio

# Save (secure format)
sio.dump(pipeline, 'iris_pipeline.skops')

# Load with type checking (won't execute arbitrary code)
unknown_types = sio.get_untrusted_types(file='iris_pipeline.skops')
print(f"Types in file: {unknown_types}")

# Only load if you trust these types
loaded = sio.load('iris_pipeline.skops', trusted=unknown_types)
```

### What to Save Beyond the Model

```python
import json
from datetime import datetime

# Save metadata alongside the model
metadata = {
    'model_version': '1.0.0',
    'trained_at': datetime.now().isoformat(),
    'sklearn_version': sklearn.__version__,
    'python_version': '3.10.12',
    'training_score': float(pipeline.score(X_test, y_test)),
    'feature_names': ['sepal_length', 'sepal_width', 'petal_length', 'petal_width'],
    'target_names': ['setosa', 'versicolor', 'virginica'],
    'training_samples': len(X_train),
    'hyperparameters': pipeline.get_params()
}

# Save model + metadata together
artifact = {
    'model': pipeline,
    'metadata': metadata
}
joblib.dump(artifact, 'iris_model_v1.0.0.joblib')

# Load and inspect
loaded_artifact = joblib.load('iris_model_v1.0.0.joblib')
print(json.dumps(loaded_artifact['metadata'], indent=2, default=str))
model = loaded_artifact['model']
```

### Version Compatibility Issues

```python
# PROBLEM: Model trained with sklearn 1.2 may not load with sklearn 1.4
# Solution 1: Pin sklearn version in requirements.txt
# Solution 2: Use ONNX for cross-framework portability

# Check version at load time
import sklearn
import warnings

def load_model_safe(filepath):
    """Load model with version check."""
    artifact = joblib.load(filepath)
    
    if 'metadata' in artifact:
        saved_version = artifact['metadata'].get('sklearn_version', 'unknown')
        current_version = sklearn.__version__
        
        if saved_version != current_version:
            warnings.warn(
                f"Model was trained with sklearn {saved_version}, "
                f"but current version is {current_version}. "
                f"Predictions may differ."
            )
    
    return artifact.get('model', artifact)
```

### ONNX Export (Cross-Platform)

```python
# Convert sklearn model to ONNX for deployment anywhere
# pip install skl2onnx onnxruntime

from skl2onnx import convert_sklearn
from skl2onnx.common.data_types import FloatTensorType

# Define input shape
initial_type = [('float_input', FloatTensorType([None, 4]))]  # 4 features

# Convert
onnx_model = convert_sklearn(pipeline, initial_types=initial_type)

# Save
with open('iris_model.onnx', 'wb') as f:
    f.write(onnx_model.SerializeToString())

# Load and predict with ONNX Runtime (no sklearn needed!)
import onnxruntime as rt

session = rt.InferenceSession('iris_model.onnx')
input_name = session.get_inputs()[0].name
onnx_preds = session.run(None, {input_name: X_test.astype(np.float32)})[0]

print(f"ONNX predictions match: {np.array_equal(original_preds, onnx_preds)}")
```

---

## Advanced Pipelines

### FeatureUnion — Parallel Feature Pipelines

`FeatureUnion` runs multiple transformers in parallel and concatenates their outputs.

```
Input X
  ├──→ Transformer A → Features A (5 cols)
  ├──→ Transformer B → Features B (3 cols)  →  Concatenate → [A | B | C] (11 cols)
  └──→ Transformer C → Features C (3 cols)
```

```python
from sklearn.pipeline import Pipeline, FeatureUnion
from sklearn.decomposition import PCA
from sklearn.preprocessing import StandardScaler, PolynomialFeatures
from sklearn.feature_selection import SelectKBest, f_classif

# Multiple feature extraction paths combined
feature_union = FeatureUnion([
    ('pca', Pipeline([
        ('scaler', StandardScaler()),
        ('pca', PCA(n_components=2))
    ])),
    ('select_best', Pipeline([
        ('scaler', StandardScaler()),
        ('select', SelectKBest(f_classif, k=2))
    ])),
    ('poly', Pipeline([
        ('scaler', StandardScaler()),
        ('poly', PolynomialFeatures(degree=2, interaction_only=True, include_bias=False)),
        ('select', SelectKBest(f_classif, k=3))
    ]))
])

# Full pipeline
full_pipeline = Pipeline([
    ('features', feature_union),
    ('model', RandomForestClassifier(n_estimators=100, random_state=42))
])

scores = cross_val_score(full_pipeline, X, y, cv=5, scoring='accuracy')
print(f"FeatureUnion Pipeline: {scores.mean():.4f} ± {scores.std():.4f}")
```

### ColumnTransformer — Different Transforms per Column

```python
from sklearn.compose import ColumnTransformer, make_column_selector
from sklearn.preprocessing import StandardScaler, OneHotEncoder, OrdinalEncoder
from sklearn.impute import SimpleImputer

# Real-world dataset with mixed types
df = pd.DataFrame({
    'age': [25, 30, None, 45, 50],
    'income': [50000, 60000, 70000, None, 90000],
    'education': ['BS', 'MS', 'PhD', 'BS', 'MS'],
    'city': ['NYC', 'LA', 'NYC', 'SF', 'LA'],
    'employed': [1, 1, 0, 1, 1]
})
y = np.array([0, 1, 0, 1, 1])

# Different preprocessing for different column types
preprocessor = ColumnTransformer(
    transformers=[
        ('num', Pipeline([
            ('imputer', SimpleImputer(strategy='median')),
            ('scaler', StandardScaler())
        ]), ['age', 'income']),
        
        ('ordinal', Pipeline([
            ('imputer', SimpleImputer(strategy='most_frequent')),
            ('encoder', OrdinalEncoder(categories=[['BS', 'MS', 'PhD']]))
        ]), ['education']),
        
        ('nominal', Pipeline([
            ('imputer', SimpleImputer(strategy='most_frequent')),
            ('encoder', OneHotEncoder(drop='first', sparse_output=False))
        ]), ['city']),
    ],
    remainder='passthrough'  # Keep 'employed' as-is
)

X_transformed = preprocessor.fit_transform(df)
print(f"Shape: {X_transformed.shape}")
print(f"Feature names: {preprocessor.get_feature_names_out()}")
```

### Auto Column Selection with Selectors

```python
from sklearn.compose import make_column_selector

# Auto-detect column types
preprocessor_auto = ColumnTransformer(
    transformers=[
        ('num', Pipeline([
            ('imputer', SimpleImputer(strategy='median')),
            ('scaler', StandardScaler())
        ]), make_column_selector(dtype_include=np.number)),
        
        ('cat', Pipeline([
            ('imputer', SimpleImputer(strategy='most_frequent')),
            ('encoder', OneHotEncoder(handle_unknown='ignore', sparse_output=False))
        ]), make_column_selector(dtype_include=object)),
    ]
)
```

### Pipeline Visualization

```python
from sklearn.utils import estimator_html_repr
from sklearn import set_config

# Pretty-print pipeline structure
set_config(display='diagram')  # Shows interactive diagram in Jupyter
print(full_pipeline)

# Or text representation
set_config(display='text')
print(full_pipeline)
```

### Accessing Pipeline Steps

```python
pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('selector', SelectKBest(f_classif, k=3)),
    ('model', RandomForestClassifier(random_state=42))
])
pipe.fit(X, y)

# Access individual steps
scaler = pipe.named_steps['scaler']
selector = pipe.named_steps['selector']
model = pipe.named_steps['model']

# Or by index
scaler = pipe[0]
model = pipe[-1]

# Get intermediate results
X_scaled = pipe[:1].transform(X)  # After scaler only
X_selected = pipe[:2].transform(X)  # After scaler + selector

print(f"After scaler: {X_scaled.shape}")
print(f"After selector: {X_selected.shape}")

# Feature importances from the model inside the pipeline
print(f"Importances: {model.feature_importances_}")
print(f"Selected features: {np.where(selector.get_support())[0]}")
```

---

## Parallelism and Performance

### n_jobs — Parallel Processing

Many sklearn estimators and utilities support `n_jobs` for parallel execution:

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_val_score, GridSearchCV
import time

X_large, y_large = make_classification(n_samples=50000, n_features=50, random_state=42)

# Sequential
start = time.time()
rf = RandomForestClassifier(n_estimators=200, n_jobs=1, random_state=42)
rf.fit(X_large, y_large)
print(f"Sequential fit: {time.time() - start:.2f}s")

# Parallel (all cores)
start = time.time()
rf = RandomForestClassifier(n_estimators=200, n_jobs=-1, random_state=42)
rf.fit(X_large, y_large)
print(f"Parallel fit: {time.time() - start:.2f}s")

# Parallel cross-validation
start = time.time()
scores = cross_val_score(rf, X_large, y_large, cv=5, n_jobs=-1)
print(f"Parallel CV: {time.time() - start:.2f}s, accuracy={scores.mean():.4f}")
```

### Incremental Learning (partial_fit)

For datasets too large to fit in memory:

```python
from sklearn.linear_model import SGDClassifier
from sklearn.preprocessing import StandardScaler

# Simulate streaming data in batches
X_stream, y_stream = make_classification(n_samples=100000, n_features=20, random_state=42)

# SGDClassifier supports partial_fit
model = SGDClassifier(loss='log_loss', random_state=42)
scaler = StandardScaler()

batch_size = 1000
classes = np.unique(y_stream)

# First batch: fit scaler, partial_fit model
X_batch = X_stream[:batch_size]
y_batch = y_stream[:batch_size]
scaler.partial_fit(X_batch)
X_scaled = scaler.transform(X_batch)
model.partial_fit(X_scaled, y_batch, classes=classes)

# Subsequent batches
for i in range(batch_size, len(X_stream), batch_size):
    X_batch = X_stream[i:i+batch_size]
    y_batch = y_stream[i:i+batch_size]
    
    scaler.partial_fit(X_batch)
    X_scaled = scaler.transform(X_batch)
    model.partial_fit(X_scaled, y_batch)

# Evaluate on last batch
accuracy = model.score(scaler.transform(X_stream[-1000:]), y_stream[-1000:])
print(f"Incremental learning accuracy: {accuracy:.4f}")
```

### Estimators Supporting `partial_fit`

| Estimator | Type | Notes |
|-----------|------|-------|
| `SGDClassifier` | Classification | Supports many loss functions |
| `SGDRegressor` | Regression | Online linear regression |
| `PassiveAggressiveClassifier` | Classification | Good for streaming |
| `MultinomialNB` | Classification | Text classification |
| `MiniBatchKMeans` | Clustering | Streaming KMeans |
| `MiniBatchDictionaryLearning` | Decomposition | Streaming matrix factorization |

### Memory Optimization

```python
# Use sparse matrices for high-dimensional sparse data (e.g., text)
from scipy.sparse import csr_matrix

# Dense: 10000 x 5000 × 8 bytes = 400 MB
# Sparse: Only non-zero elements stored

X_dense = np.random.choice([0, 0, 0, 0, 1], size=(10000, 5000))  # 80% zeros
X_sparse = csr_matrix(X_dense)

print(f"Dense memory:  {X_dense.nbytes / 1e6:.1f} MB")
print(f"Sparse memory: {(X_sparse.data.nbytes + X_sparse.indices.nbytes + X_sparse.indptr.nbytes) / 1e6:.1f} MB")

# Many sklearn algorithms work directly with sparse matrices
from sklearn.linear_model import LogisticRegression
model = LogisticRegression(max_iter=1000)
# model.fit(X_sparse, y)  # Works directly — no conversion needed
```

---

## Callbacks and Logging

### Verbose Output

```python
from sklearn.ensemble import GradientBoostingClassifier

# verbose > 0 shows training progress
gb = GradientBoostingClassifier(
    n_estimators=100,
    verbose=1,       # 1 = per-iteration, 2 = per-tree
    random_state=42
)
gb.fit(X_train, y_train)
```

### Warm Start — Resume Training

```python
from sklearn.ensemble import RandomForestClassifier

# Train with 50 trees
rf = RandomForestClassifier(n_estimators=50, warm_start=True, random_state=42)
rf.fit(X_train, y_train)
print(f"50 trees accuracy: {rf.score(X_test, y_test):.4f}")

# Add 50 more trees (total 100) without retraining the first 50
rf.n_estimators = 100
rf.fit(X_train, y_train)
print(f"100 trees accuracy: {rf.score(X_test, y_test):.4f}")

# Add 100 more trees (total 200)
rf.n_estimators = 200
rf.fit(X_train, y_train)
print(f"200 trees accuracy: {rf.score(X_test, y_test):.4f}")
```

> **Use Case**: Incrementally adding trees to find the right `n_estimators` without retraining from scratch.

---

## Sklearn Configuration and Global Settings

```python
import sklearn
from sklearn import set_config, get_config

# View current configuration
print(get_config())

# Set global options
set_config(
    display='diagram',              # Pipeline display mode
    transform_output='pandas',      # Return DataFrames from transform()
    enable_metadata_routing=False,  # Metadata routing (advanced)
    assume_finite=False             # Set True to skip NaN/Inf checks (faster)
)

# Context manager for temporary config
from sklearn import config_context

with config_context(assume_finite=True):
    # Faster but no input validation
    model.fit(X_train, y_train)
# Config reverts outside the block
```

### Transform Output as Pandas

```python
# Global setting: all transformers return DataFrames
set_config(transform_output='pandas')

scaler = StandardScaler()
result = scaler.fit_transform(pd.DataFrame(X, columns=['f1', 'f2', 'f3', 'f4']))
print(type(result))  # <class 'pandas.core.frame.DataFrame'>
print(result.head())

# Per-estimator setting
scaler_pandas = StandardScaler().set_output(transform='pandas')
scaler_numpy = StandardScaler().set_output(transform='default')

# Reset global
set_config(transform_output='default')
```

---

## Production Patterns

### Pattern 1: Full Training Pipeline

```python
import joblib
import json
import hashlib
from datetime import datetime
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_val_score

def train_and_save(X, y, feature_config, model_dir='models/'):
    """Complete training pipeline with metadata and versioning."""
    import os
    os.makedirs(model_dir, exist_ok=True)
    
    # Build pipeline
    preprocessor = ColumnTransformer([
        ('num', StandardScaler(), feature_config['numeric']),
        ('cat', OneHotEncoder(handle_unknown='ignore', sparse_output=False), 
         feature_config['categorical'])
    ])
    
    pipeline = Pipeline([
        ('preprocessor', preprocessor),
        ('model', RandomForestClassifier(n_estimators=200, random_state=42))
    ])
    
    # Evaluate
    cv_scores = cross_val_score(pipeline, X, y, cv=5, scoring='accuracy')
    
    # Train on full data
    pipeline.fit(X, y)
    
    # Create metadata
    metadata = {
        'version': datetime.now().strftime('%Y%m%d_%H%M%S'),
        'cv_accuracy_mean': float(cv_scores.mean()),
        'cv_accuracy_std': float(cv_scores.std()),
        'n_samples': len(X),
        'n_features': X.shape[1],
        'feature_config': feature_config,
        'sklearn_version': sklearn.__version__,
    }
    
    # Save
    version = metadata['version']
    joblib.dump(pipeline, f'{model_dir}/model_{version}.joblib', compress=3)
    with open(f'{model_dir}/metadata_{version}.json', 'w') as f:
        json.dump(metadata, f, indent=2, default=str)
    
    print(f"Model saved: model_{version}.joblib")
    print(f"CV Accuracy: {cv_scores.mean():.4f} ± {cv_scores.std():.4f}")
    
    return pipeline, metadata
```

### Pattern 2: Prediction Service

```python
import joblib
import json
import numpy as np
import pandas as pd

class PredictionService:
    """Load a saved model and serve predictions."""
    
    def __init__(self, model_path, metadata_path=None):
        self.pipeline = joblib.load(model_path)
        
        if metadata_path:
            with open(metadata_path) as f:
                self.metadata = json.load(f)
        else:
            self.metadata = {}
    
    def predict(self, data):
        """
        data: dict or DataFrame with feature values
        Returns: predictions and probabilities
        """
        if isinstance(data, dict):
            data = pd.DataFrame([data])
        
        predictions = self.pipeline.predict(data)
        
        result = {'predictions': predictions.tolist()}
        
        if hasattr(self.pipeline[-1], 'predict_proba'):
            probas = self.pipeline[-1].predict_proba(
                self.pipeline[:-1].transform(data)
            )
            result['probabilities'] = probas.tolist()
        
        return result
    
    def get_info(self):
        return {
            'model_type': type(self.pipeline[-1]).__name__,
            'metadata': self.metadata
        }

# Usage
# service = PredictionService('models/model_20240101.joblib', 'models/metadata_20240101.json')
# result = service.predict({'age': 30, 'income': 60000, 'city': 'NYC'})
```

### Pattern 3: A/B Testing Models

```python
class ModelABTest:
    """Run two models side-by-side and compare predictions."""
    
    def __init__(self, model_a_path, model_b_path, traffic_split=0.5):
        self.model_a = joblib.load(model_a_path)
        self.model_b = joblib.load(model_b_path)
        self.traffic_split = traffic_split
        self.results = {'a': [], 'b': []}
    
    def predict(self, X, request_id=None):
        """Route to model A or B based on traffic split."""
        # Deterministic routing based on request_id
        if request_id:
            use_a = int(hashlib.md5(str(request_id).encode()).hexdigest(), 16) % 100 < self.traffic_split * 100
        else:
            use_a = np.random.random() < self.traffic_split
        
        if use_a:
            pred = self.model_a.predict(X)
            self.results['a'].append(pred)
            return pred, 'model_a'
        else:
            pred = self.model_b.predict(X)
            self.results['b'].append(pred)
            return pred, 'model_b'
```

---

## Common Mistakes

### 1. Computation in `__init__`

```python
# ❌ WRONG: Doing work in __init__
class BadTransformer(BaseEstimator, TransformerMixin):
    def __init__(self, data_path='data.csv'):
        self.data = pd.read_csv(data_path)  # ✗ Loading data
        self.model = RandomForestClassifier()  # ✗ Creating objects
        self.n_features = self.data.shape[1]   # ✗ Computing values

# ✓ CORRECT: Only store parameters
class GoodTransformer(BaseEstimator, TransformerMixin):
    def __init__(self, data_path='data.csv'):
        self.data_path = data_path  # Just store the path
```

### 2. Forgetting `return self` in `fit()`

```python
# ❌ WRONG: Pipeline breaks silently
def fit(self, X, y=None):
    self.mean_ = X.mean(axis=0)
    # Missing: return self

# ✓ CORRECT:
def fit(self, X, y=None):
    self.mean_ = X.mean(axis=0)
    return self
```

### 3. Mutating Input Data

```python
# ❌ WRONG: Modifies the original array
def transform(self, X):
    X[X < 0] = 0  # This changes the caller's data!
    return X

# ✓ CORRECT: Work on a copy
def transform(self, X):
    X = np.array(X).copy()
    X[X < 0] = 0
    return X
```

### 4. Loading Untrusted Pickle/Joblib Files

```python
# ❌ DANGEROUS: Loading from untrusted source
model = joblib.load('model_from_internet.joblib')  # Could execute malicious code!

# ✓ SAFER: Use skops for untrusted files
import skops.io as sio
unknown_types = sio.get_untrusted_types(file='model.skops')
# Review types before loading
model = sio.load('model.skops', trusted=unknown_types)

# ✓ BEST: Only load models you trained yourself or from trusted sources
# ✓ Pin sklearn versions in requirements.txt
```

### 5. Not Saving the Full Pipeline

```python
# ❌ WRONG: Only saving the model, not the preprocessor
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_train)
model = RandomForestClassifier()
model.fit(X_scaled, y_train)
joblib.dump(model, 'model.joblib')  # How do you scale at prediction time?

# ✓ CORRECT: Save the entire pipeline
pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('model', RandomForestClassifier())
])
pipeline.fit(X_train, y_train)
joblib.dump(pipeline, 'pipeline.joblib')  # Everything included!
```

### 6. Ignoring sklearn Version Mismatch

```python
# ❌ Train with sklearn 1.2, deploy with sklearn 1.5 → might break

# ✓ Always record the version
import sklearn
print(f"sklearn version: {sklearn.__version__}")

# ✓ Pin in requirements.txt:
# scikit-learn==1.4.2

# ✓ Use ONNX for version-independent deployment
```

---

## Interview Questions

### Conceptual Questions

**Q1: Why would you build a custom transformer instead of using FunctionTransformer?**

`FunctionTransformer` is stateless — it applies the same function regardless of training data. A custom class-based transformer can learn from training data in `fit()` (e.g., computing means, learning encoding maps) and apply those learned values in `transform()`. Use `FunctionTransformer` for simple, stateless transforms; use a class for anything that needs to learn from data.

**Q2: What are the key rules a custom sklearn estimator must follow?**

1. `__init__` must only assign parameters as attributes (no computation)
2. Fitted attributes must end with underscore (e.g., `self.mean_`)
3. `fit()` must return `self`
4. `transform()` / `predict()` must not mutate input data
5. `get_params()` / `set_params()` must work correctly (free via `BaseEstimator`)
6. Must handle edge cases: single sample, all same values, etc.

**Q3: What's the security risk of pickle/joblib and how do you mitigate it?**

Pickle can execute arbitrary code during deserialization. A malicious `.pkl` file could run system commands, install malware, or exfiltrate data. Mitigations:
- Only load models from trusted sources
- Use `skops` for secure serialization
- Export to ONNX for deployment (no pickle needed)
- Verify file checksums/signatures before loading
- Run model loading in sandboxed environments

**Q4: Explain the difference between `Pipeline` and `FeatureUnion`.**

- `Pipeline`: Sequential — output of step N becomes input of step N+1. Used for chaining preprocessing → model.
- `FeatureUnion`: Parallel — runs multiple transformers on the same input and concatenates outputs horizontally. Used for combining different feature engineering approaches.

```
Pipeline:     X → A → B → C → output
FeatureUnion: X → A ─┐
              X → B ──├→ [A|B|C] → output  
              X → C ─┘
```

**Q5: How does `warm_start` work and when would you use it?**

`warm_start=True` tells the estimator to reuse the solution from the previous `fit()` call and add to it. For `RandomForest`, it keeps existing trees and adds new ones when you increase `n_estimators`. Use cases:
- Finding optimal `n_estimators` incrementally
- Online learning with ensemble methods
- Resuming interrupted training

**Q6: What's the difference between `partial_fit` and `warm_start`?**

| Aspect | `partial_fit` | `warm_start` |
|--------|--------------|-------------|
| Data | Sees new batch only | Sees full dataset again |
| Use case | Streaming/online learning | Incremental hyperparameter changes |
| Memory | Low (one batch at a time) | High (full dataset) |
| Models | SGD, NaiveBayes, MiniBatchKMeans | RandomForest, GradientBoosting |

**Q7: How would you deploy a sklearn model in production?**

Options ranked by complexity:
1. **Joblib + Flask/FastAPI**: Simple REST API wrapping `joblib.load()` + `predict()`
2. **ONNX Runtime**: Convert to ONNX, serve with ONNX Runtime (faster, no sklearn needed)
3. **MLflow**: Full lifecycle management with model registry and serving
4. **Cloud ML services**: AWS SageMaker, GCP Vertex AI, Azure ML
5. **Edge deployment**: ONNX → TensorRT / Core ML for mobile/embedded

**Q8: How do you handle unknown categories in production when using OneHotEncoder?**

```python
# Set handle_unknown='ignore' during training
encoder = OneHotEncoder(handle_unknown='ignore', sparse_output=False)
# Unknown categories get all-zero encoding at prediction time
```

### Coding Questions

**Q9: Write a custom transformer that adds rolling statistics.**

```python
class RollingFeatures(BaseEstimator, TransformerMixin):
    def __init__(self, windows=[3, 7]):
        self.windows = windows
    
    def fit(self, X, y=None):
        return self
    
    def transform(self, X):
        df = pd.DataFrame(X)
        result = df.copy()
        for w in self.windows:
            result = pd.concat([
                result,
                df.rolling(w, min_periods=1).mean().add_suffix(f'_roll_mean_{w}'),
                df.rolling(w, min_periods=1).std().fillna(0).add_suffix(f'_roll_std_{w}')
            ], axis=1)
        return result.values
```

**Q10: Create a complete pipeline with custom transformer, feature selection, and model, then save it.**

```python
from sklearn.pipeline import Pipeline
from sklearn.feature_selection import SelectKBest, f_classif

pipe = Pipeline([
    ('clipper', OutlierClipper(factor=1.5)),
    ('scaler', StandardScaler()),
    ('selector', SelectKBest(f_classif, k=10)),
    ('model', RandomForestClassifier(n_estimators=100, random_state=42))
])

pipe.fit(X_train, y_train)
print(f"Accuracy: {pipe.score(X_test, y_test):.4f}")

joblib.dump(pipe, 'full_pipeline.joblib', compress=3)
loaded = joblib.load('full_pipeline.joblib')
assert np.array_equal(pipe.predict(X_test), loaded.predict(X_test))
```

---

## Quick Reference

### Custom Estimator Skeleton

```python
from sklearn.base import BaseEstimator, TransformerMixin, ClassifierMixin

# Transformer skeleton
class MyTransformer(BaseEstimator, TransformerMixin):
    def __init__(self, param1=1.0):
        self.param1 = param1
    def fit(self, X, y=None):
        self.learned_value_ = ...
        return self
    def transform(self, X):
        check_is_fitted(self)
        X = np.array(X).copy()
        return ...

# Classifier skeleton
class MyClassifier(BaseEstimator, ClassifierMixin):
    def __init__(self, param1=1.0):
        self.param1 = param1
    def fit(self, X, y):
        X, y = check_X_y(X, y)
        self.classes_ = unique_labels(y)
        return self
    def predict(self, X):
        check_is_fitted(self)
        X = check_array(X)
        return ...
```

### Model Persistence Comparison

| Method | Security | Speed | Size | Cross-platform |
|--------|----------|-------|------|----------------|
| `joblib` | Low (arbitrary code) | Fast | Small (compressed) | No (Python only) |
| `pickle` | Low (arbitrary code) | Medium | Medium | No (Python only) |
| `skops` | High (type checking) | Medium | Medium | No (Python only) |
| `ONNX` | High (no code exec) | Fastest (runtime) | Smallest | Yes (any language) |

### Pipeline Components

| Component | Purpose | Import |
|-----------|---------|--------|
| `Pipeline` | Sequential steps | `sklearn.pipeline` |
| `FeatureUnion` | Parallel feature transforms | `sklearn.pipeline` |
| `ColumnTransformer` | Per-column transforms | `sklearn.compose` |
| `make_pipeline` | Pipeline without naming steps | `sklearn.pipeline` |
| `make_column_transformer` | ColumnTransformer shortcut | `sklearn.compose` |
| `make_column_selector` | Auto-detect column types | `sklearn.compose` |

### Key Imports Cheat Sheet

```python
# Custom estimators
from sklearn.base import BaseEstimator, TransformerMixin, ClassifierMixin, RegressorMixin
from sklearn.utils.validation import check_X_y, check_array, check_is_fitted
from sklearn.utils.multiclass import unique_labels

# Pipelines
from sklearn.pipeline import Pipeline, FeatureUnion, make_pipeline
from sklearn.compose import ColumnTransformer, make_column_transformer, make_column_selector

# Persistence
import joblib                  # joblib.dump(), joblib.load()
import pickle                  # pickle.dump(), pickle.load()
# import skops.io as sio       # sio.dump(), sio.load()

# Configuration
from sklearn import set_config, get_config, config_context
```

### Checklist: Is Your Custom Estimator Production-Ready?

| Check | Status |
|-------|--------|
| `__init__` only assigns parameters? | ☐ |
| Fitted attributes end with `_`? | ☐ |
| `fit()` returns `self`? | ☐ |
| `transform()`/`predict()` doesn't mutate input? | ☐ |
| `check_is_fitted()` in `transform()`/`predict()`? | ☐ |
| Works with `clone()`? | ☐ |
| Works in `Pipeline`? | ☐ |
| Works with `GridSearchCV`? | ☐ |
| Passes `check_estimator()`? | ☐ |
| Handles edge cases (empty input, single sample)? | ☐ |
| Has `get_feature_names_out()` (if transformer)? | ☐ |
| Saved model includes full pipeline? | ☐ |
| Sklearn version pinned in requirements? | ☐ |

---

*Previous Chapter: [07-Feature-Selection-and-Extraction.md](./07-Feature-Selection-and-Extraction.md) — SelectKBest, RFE, Feature Importance, Polynomial Features*
