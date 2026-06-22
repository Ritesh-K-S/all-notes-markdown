# Chapter 07: Feature Selection and Extraction in Scikit-Learn

## Table of Contents
- [Introduction](#introduction)
- [Why Feature Selection Matters](#why-feature-selection-matters)
- [Feature Selection vs Feature Extraction](#feature-selection-vs-feature-extraction)
- [Filter Methods](#filter-methods)
- [Wrapper Methods](#wrapper-methods)
- [Embedded Methods](#embedded-methods)
- [Feature Extraction](#feature-extraction)
- [Polynomial Features](#polynomial-features)
- [Putting It All Together — Pipelines](#putting-it-all-together--pipelines)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Introduction

### What It Is
Feature selection is the process of choosing the most relevant input variables (features) for your model. Think of it like packing for a trip — you don't bring your entire wardrobe; you pick only what's useful for where you're going.

Feature extraction, on the other hand, creates *new* features by combining or transforming existing ones — like mixing paint colors to create new ones.

### Why It Matters

| Problem | Impact |
|---------|--------|
| Too many features | Model is slow, overfits, hard to interpret |
| Irrelevant features | Model gets confused by noise |
| Redundant features | Multicollinearity, unstable coefficients |
| Too few features | Model underfits, misses patterns |

> **Rule of Thumb**: More features ≠ better model. The "curse of dimensionality" means that as features grow, the data becomes sparse, and models need exponentially more data to generalize well.

### Real-World Impact
- **Faster training**: 100 features vs 10,000 features = orders of magnitude difference
- **Better accuracy**: Removing noise helps the model focus on real signal
- **Interpretability**: Stakeholders can understand a model with 10 features, not 10,000
- **Lower cost**: Fewer features = less data to collect in production

---

## Why Feature Selection Matters

### The Curse of Dimensionality

Imagine you have 1 feature and 100 data points — the space is well-covered. Now imagine 100 features with the same 100 data points — the space is almost empty. Your model has no nearby neighbors to learn from.

```
1D: |●●●●●●●●●●| → Dense, well-covered

2D: |● ●  ●   |
    |  ●    ● |  → Starting to get sparse
    |●   ●  ● |

100D: Virtually all points are "far" from each other
```

### Mathematical Intuition

For $d$ dimensions and $n$ samples, the average distance between points grows as:

$$\text{Average distance} \propto n^{-1/d}$$

This means you need exponentially more data as dimensions increase:
- 10 features → ~100 samples might suffice
- 100 features → ~10,000+ samples needed
- 1000 features → ~1,000,000+ samples needed

---

## Feature Selection vs Feature Extraction

| Aspect | Feature Selection | Feature Extraction |
|--------|------------------|-------------------|
| What it does | Picks a subset of original features | Creates new features from original ones |
| Original features preserved? | Yes | No (transformed) |
| Interpretability | High (same features) | Low (new abstract features) |
| Methods | Filter, Wrapper, Embedded | PCA, LDA, Polynomial Features |
| When to use | When you need interpretability | When you need dimensionality reduction |
| Example | Keep "age" and "income", drop "zipcode" | Create PC1 = 0.7*age + 0.3*income |

---

## Filter Methods

### What They Are
Filter methods score each feature independently using a statistical test and select the top-scoring ones. They're called "filters" because they filter out low-quality features *before* model training.

**Analogy**: Like a bouncer at a club checking IDs before letting people in — quick, independent check for each person.

### How They Work

```
All Features → Statistical Test → Rank Features → Select Top-K
     [f1, f2, f3, ..., fn]              [f3 > f1 > f7 > ...]    [f3, f1, f7]
```

### SelectKBest

Selects the top `k` features based on a scoring function.

```python
import numpy as np
from sklearn.datasets import load_iris, make_classification
from sklearn.feature_selection import SelectKBest, f_classif, chi2, mutual_info_classif
from sklearn.model_selection import train_test_split

# Create a dataset with some informative and some noise features
X, y = make_classification(
    n_samples=1000,
    n_features=20,        # Total features
    n_informative=5,      # Only 5 are actually useful
    n_redundant=5,        # 5 are linear combinations of informative ones
    n_repeated=0,         # No duplicates
    n_classes=2,
    random_state=42
)

print(f"Original shape: {X.shape}")  # (1000, 20)

# Select top 10 features using ANOVA F-test
selector = SelectKBest(score_func=f_classif, k=10)
X_selected = selector.fit_transform(X, y)

print(f"After selection: {X_selected.shape}")  # (1000, 10)

# See scores for each feature
scores = selector.scores_
print(f"\nFeature scores:")
for i, score in enumerate(scores):
    selected = "✓" if selector.get_support()[i] else "✗"
    print(f"  Feature {i:2d}: score={score:8.2f} {selected}")
```

### Scoring Functions Explained

| Function | Use Case | What It Measures | Assumptions |
|----------|----------|-----------------|-------------|
| `f_classif` | Classification | ANOVA F-statistic between feature and target | Linear relationship, normal distribution |
| `chi2` | Classification | Chi-squared statistic | Non-negative features only |
| `mutual_info_classif` | Classification | Mutual information (information gain) | Captures non-linear relationships |
| `f_regression` | Regression | F-statistic for regression | Linear relationship |
| `mutual_info_regression` | Regression | Mutual information | Captures non-linear relationships |

#### ANOVA F-test (`f_classif`)

Measures whether the means of a feature differ significantly across classes.

$$F = \frac{\text{Between-class variance}}{\text{Within-class variance}} = \frac{\sum_{k} n_k(\bar{x}_k - \bar{x})^2 / (K-1)}{\sum_{k}\sum_{i \in k}(x_i - \bar{x}_k)^2 / (N-K)}$$

Where:
- $K$ = number of classes
- $n_k$ = samples in class $k$
- $\bar{x}_k$ = mean of feature in class $k$
- $\bar{x}$ = overall mean

**Intuition**: If a feature has very different average values for class 0 vs class 1, it's probably useful for classification.

```python
from sklearn.feature_selection import f_classif, f_regression

# For classification tasks
X_iris, y_iris = load_iris(return_X_y=True)
f_scores, p_values = f_classif(X_iris, y_iris)

print("Iris dataset - ANOVA F-test:")
feature_names = ['sepal_length', 'sepal_width', 'petal_length', 'petal_width']
for name, f, p in zip(feature_names, f_scores, p_values):
    significance = "***" if p < 0.001 else "**" if p < 0.01 else "*" if p < 0.05 else "ns"
    print(f"  {name:15s}: F={f:8.2f}, p={p:.2e} {significance}")

# Output:
# sepal_length   : F=  119.26, p=1.67e-31 ***
# sepal_width    : F=   49.16, p=4.49e-17 ***
# petal_length   : F= 1180.16, p=2.86e-91 ***  ← Most discriminative
# petal_width    : F=  960.01, p=4.17e-85 ***
```

#### Chi-Squared Test (`chi2`)

Measures the dependence between a feature and the target. Works only with non-negative features (e.g., word counts, TF-IDF scores).

$$\chi^2 = \sum \frac{(O - E)^2}{E}$$

Where $O$ = observed frequency, $E$ = expected frequency under independence.

```python
from sklearn.feature_selection import chi2
from sklearn.preprocessing import MinMaxScaler

# chi2 requires non-negative values, so scale to [0, 1]
X_scaled = MinMaxScaler().fit_transform(X_iris)

chi2_scores, chi2_pvalues = chi2(X_scaled, y_iris)

print("\nIris dataset - Chi-squared test:")
for name, score, p in zip(feature_names, chi2_scores, chi2_pvalues):
    print(f"  {name:15s}: χ²={score:6.2f}, p={p:.2e}")
```

#### Mutual Information (`mutual_info_classif`)

Measures how much knowing a feature reduces uncertainty about the target. Unlike F-test, it captures **non-linear** relationships.

$$MI(X; Y) = \sum_{x}\sum_{y} p(x,y) \log\frac{p(x,y)}{p(x)p(y)}$$

- MI = 0 → Features are independent
- MI > 0 → Features share information

```python
from sklearn.feature_selection import mutual_info_classif

# Mutual information - captures non-linear relationships
mi_scores = mutual_info_classif(X_iris, y_iris, random_state=42)

print("\nIris dataset - Mutual Information:")
for name, mi in zip(feature_names, mi_scores):
    print(f"  {name:15s}: MI={mi:.4f}")

# Compare with f_classif for non-linear data
from sklearn.datasets import make_moons
X_moons, y_moons = make_moons(n_samples=500, noise=0.2, random_state=42)

f_scores_moons, _ = f_classif(X_moons, y_moons)
mi_scores_moons = mutual_info_classif(X_moons, y_moons, random_state=42)

print("\nMoons dataset (non-linear):")
print(f"  F-test scores: {f_scores_moons}")      # Low - can't see non-linear pattern
print(f"  MI scores:     {mi_scores_moons}")      # Higher - captures non-linear dependency
```

> **Pro Tip**: Use `mutual_info_classif` when you suspect non-linear relationships. Use `f_classif` when you want speed and your data is roughly linear.

### SelectPercentile

Instead of picking top-K, pick the top percentage of features:

```python
from sklearn.feature_selection import SelectPercentile

# Select top 25% of features
selector = SelectPercentile(score_func=f_classif, percentile=25)
X_top25 = selector.fit_transform(X, y)

print(f"Original: {X.shape[1]} features")
print(f"Top 25%:  {X_top25.shape[1]} features")
```

### VarianceThreshold

Removes features with low variance (near-constant features). This is purely unsupervised — doesn't use target `y`.

```python
from sklearn.feature_selection import VarianceThreshold

# Create data with a constant feature
X_with_constant = np.column_stack([
    X_iris,
    np.ones(150),             # Constant feature (zero variance)
    np.random.choice([0, 1], size=150, p=[0.99, 0.01])  # Nearly constant
])

print(f"Before: {X_with_constant.shape[1]} features")

# Remove features with variance below threshold
# For binary features: Var[X] = p(1-p), threshold=0.01 removes features
# where 99% of values are the same
selector = VarianceThreshold(threshold=0.01)
X_filtered = selector.fit_transform(X_with_constant)

print(f"After:  {X_filtered.shape[1]} features")
print(f"Removed features at indices: {np.where(~selector.get_support())[0]}")
```

> **When to use**: As a first pass to eliminate obviously useless features. Especially useful after one-hot encoding where some categories might be extremely rare.

---

## Wrapper Methods

### What They Are
Wrapper methods evaluate feature subsets by actually training a model. They "wrap" a model around the feature selection process.

**Analogy**: Like trying on different outfit combinations and checking the mirror each time — more thorough but much slower.

### How They Work

```
Feature Subset → Train Model → Evaluate Performance → Adjust Subset → Repeat
     {f1, f3}       SVM           accuracy=0.85        Add f7
     {f1, f3, f7}   SVM           accuracy=0.91        Add f2
     {f1, f2, f3, f7} SVM         accuracy=0.90        Remove f2 (overfit)
     → Final: {f1, f3, f7}
```

### Recursive Feature Elimination (RFE)

RFE works by repeatedly training a model, ranking features by importance, and removing the least important one(s).

```
Step 1: Train with ALL features → Remove weakest
Step 2: Train with remaining → Remove weakest
Step 3: Train with remaining → Remove weakest
...
Step n: Left with desired number of features
```

```python
from sklearn.feature_selection import RFE, RFECV
from sklearn.ensemble import RandomForestClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import make_classification

# Create dataset
X, y = make_classification(
    n_samples=500, n_features=15, n_informative=5,
    n_redundant=3, random_state=42
)

# RFE with Random Forest
estimator = RandomForestClassifier(n_estimators=100, random_state=42)
rfe = RFE(
    estimator=estimator,
    n_features_to_select=5,  # Keep 5 features
    step=1                    # Remove 1 feature per iteration
)

rfe.fit(X, y)

# Results
print("Feature Rankings (1 = selected):")
for i, (rank, selected) in enumerate(zip(rfe.ranking_, rfe.support_)):
    status = "✓ SELECTED" if selected else f"  rank={rank}"
    print(f"  Feature {i:2d}: {status}")

print(f"\nSelected features: {np.where(rfe.support_)[0]}")
```

### RFECV — RFE with Cross-Validation

The problem with plain RFE: how do you know the optimal number of features? RFECV solves this by trying different numbers and using cross-validation to find the best.

```python
from sklearn.feature_selection import RFECV
from sklearn.model_selection import StratifiedKFold

# RFECV automatically finds the optimal number of features
rfecv = RFECV(
    estimator=RandomForestClassifier(n_estimators=100, random_state=42),
    step=1,                           # Remove 1 feature at a time
    cv=StratifiedKFold(5),           # 5-fold stratified CV
    scoring='accuracy',               # Metric to optimize
    min_features_to_select=1,         # Minimum features to consider
    n_jobs=-1                         # Parallel processing
)

rfecv.fit(X, y)

print(f"Optimal number of features: {rfecv.n_features_}")
print(f"Selected features: {np.where(rfecv.support_)[0]}")
print(f"Best CV score: {rfecv.cv_results_['mean_test_score'].max():.4f}")

# Plot the performance vs number of features
import matplotlib.pyplot as plt

n_features_range = range(1, len(rfecv.cv_results_['mean_test_score']) + 1)
plt.figure(figsize=(10, 5))
plt.plot(n_features_range, rfecv.cv_results_['mean_test_score'], 'b-o')
plt.fill_between(
    n_features_range,
    np.array(rfecv.cv_results_['mean_test_score']) - np.array(rfecv.cv_results_['std_test_score']),
    np.array(rfecv.cv_results_['mean_test_score']) + np.array(rfecv.cv_results_['std_test_score']),
    alpha=0.2
)
plt.xlabel('Number of Features')
plt.ylabel('CV Accuracy')
plt.title('RFECV - Optimal Number of Features')
plt.axvline(x=rfecv.n_features_, color='r', linestyle='--', label=f'Optimal: {rfecv.n_features_}')
plt.legend()
plt.grid(True)
plt.show()
```

### SequentialFeatureSelector (Forward/Backward)

Added in sklearn 0.24. Uses greedy forward or backward search.

```python
from sklearn.feature_selection import SequentialFeatureSelector

# Forward selection: start with 0 features, add one at a time
sfs_forward = SequentialFeatureSelector(
    estimator=RandomForestClassifier(n_estimators=50, random_state=42),
    n_features_to_select=5,
    direction='forward',     # 'forward' or 'backward'
    scoring='accuracy',
    cv=5,
    n_jobs=-1
)

sfs_forward.fit(X, y)
print(f"Forward selection features: {np.where(sfs_forward.get_support())[0]}")

# Backward selection: start with all features, remove one at a time
sfs_backward = SequentialFeatureSelector(
    estimator=RandomForestClassifier(n_estimators=50, random_state=42),
    n_features_to_select=5,
    direction='backward',
    scoring='accuracy',
    cv=5,
    n_jobs=-1
)

sfs_backward.fit(X, y)
print(f"Backward selection features: {np.where(sfs_backward.get_support())[0]}")
```

**Forward vs Backward**:
| Aspect | Forward | Backward |
|--------|---------|----------|
| Starts with | 0 features | All features |
| Each step | Adds best feature | Removes worst feature |
| Faster when | Target # features is small | Target # features is large |
| Misses | Interactions (added one at a time) | — |
| Captures | Can miss feature pairs | Better at preserving interactions |

> **Pro Tip**: Backward selection is generally better at capturing feature interactions but is slower when you have many features.

---

## Embedded Methods

### What They Are
Embedded methods perform feature selection as part of the model training process itself. The model has a built-in mechanism to assess feature importance.

**Analogy**: Like a chef who tastes while cooking and adjusts ingredients — the selection happens during the process, not before or after.

### How They Work

```
Training Process
├── Model learns weights/importances for each feature
├── Features with zero/low weight are effectively "ignored"
└── Final model implicitly selects features
```

### SelectFromModel

A meta-transformer that uses any estimator's `feature_importances_` or `coef_` attribute to select features.

```python
from sklearn.feature_selection import SelectFromModel
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.linear_model import LogisticRegression, Lasso

# ----- Method 1: Tree-based Feature Importance -----
rf = RandomForestClassifier(n_estimators=200, random_state=42)
rf.fit(X, y)

# Select features with importance > mean importance
selector_rf = SelectFromModel(rf, threshold='mean')
X_rf = selector_rf.transform(X)

print(f"Random Forest selection: {X.shape[1]} → {X_rf.shape[1]} features")
print(f"Selected: {np.where(selector_rf.get_support())[0]}")

# Visualize importances
importances = rf.feature_importances_
indices = np.argsort(importances)[::-1]

print("\nFeature Importances (sorted):")
for i, idx in enumerate(indices):
    bar = "█" * int(importances[idx] * 50)
    selected = "✓" if selector_rf.get_support()[idx] else " "
    print(f"  [{selected}] Feature {idx:2d}: {importances[idx]:.4f} {bar}")
```

```python
# ----- Method 2: L1 Regularization (Lasso) -----
# L1 penalty drives coefficients to exactly zero → automatic feature selection

from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline

# Lasso for regression / LogisticRegression with L1 for classification
lasso_selector = SelectFromModel(
    LogisticRegression(penalty='l1', C=0.1, solver='saga', max_iter=5000, random_state=42),
    threshold=1e-5  # Features with |coef| > threshold are selected
)

# Must scale features before L1 regularization!
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

lasso_selector.fit(X_scaled, y)
X_l1 = lasso_selector.transform(X_scaled)

print(f"\nL1 (Lasso) selection: {X.shape[1]} → {X_l1.shape[1]} features")
print(f"Selected: {np.where(lasso_selector.get_support())[0]}")

# Show non-zero coefficients
coefs = lasso_selector.estimator_.coef_[0]
print("\nCoefficients:")
for i, coef in enumerate(coefs):
    if abs(coef) > 1e-5:
        print(f"  Feature {i:2d}: coef={coef:+.4f} ✓")
    else:
        print(f"  Feature {i:2d}: coef={coef:+.4f}   (eliminated)")
```

### Why L1 Drives Coefficients to Zero

The L1 penalty adds $\lambda \sum |w_i|$ to the loss function. Geometrically:

```
L2 (Ridge):                    L1 (Lasso):
  Constraint region:             Constraint region:
       ╭───╮                          /\
      │     │                        /  \
      │  ●  │  ← Circular          ●    ● ← Diamond
      │     │                        \  /
       ╰───╯                          \/
  
  Loss contours hit circle         Loss contours hit corners
  → small but non-zero weights     → corners = zero on some axis
                                   → SPARSE solutions!
```

The diamond shape of L1 constraint has corners on the axes, making it likely that the optimal point has some weights exactly at zero.

### Feature Importance from Trees

How tree-based models compute feature importance:

$$\text{Importance}(f) = \sum_{\text{nodes using } f} \frac{n_{\text{node}}}{n_{\text{total}}} \cdot \Delta \text{impurity}$$

Where $\Delta\text{impurity}$ is the reduction in Gini impurity or entropy at that split.

```python
# Compare different tree-based methods
from sklearn.ensemble import GradientBoostingClassifier, ExtraTreesClassifier

models = {
    'Random Forest': RandomForestClassifier(n_estimators=200, random_state=42),
    'Gradient Boosting': GradientBoostingClassifier(n_estimators=200, random_state=42),
    'Extra Trees': ExtraTreesClassifier(n_estimators=200, random_state=42)
}

print("Feature importance comparison:")
print(f"{'Feature':<10}", end="")
for name in models:
    print(f"{name:<20}", end="")
print()

for name, model in models.items():
    model.fit(X, y)

for i in range(X.shape[1]):
    print(f"F{i:<9}", end="")
    for name, model in models.items():
        imp = model.feature_importances_[i]
        print(f"{imp:<20.4f}", end="")
    print()
```

> **Warning**: Tree-based feature importances are biased toward high-cardinality features (features with many unique values). Use permutation importance for unbiased estimates.

### Permutation Importance (Unbiased)

```python
from sklearn.inspection import permutation_importance

# Train model
rf = RandomForestClassifier(n_estimators=100, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3, random_state=42)
rf.fit(X_train, y_train)

# Permutation importance on TEST set (unbiased)
perm_importance = permutation_importance(
    rf, X_test, y_test,
    n_repeats=30,        # Repeat permutation 30 times for stability
    random_state=42,
    scoring='accuracy',
    n_jobs=-1
)

# Sort by importance
sorted_idx = perm_importance.importances_mean.argsort()[::-1]

print("Permutation Importance (test set):")
for idx in sorted_idx:
    mean = perm_importance.importances_mean[idx]
    std = perm_importance.importances_std[idx]
    if mean > 0.001:
        print(f"  Feature {idx:2d}: {mean:.4f} ± {std:.4f}")
```

> **Key Insight**: Always compute permutation importance on the **test set**, not training set. Training set importance can be inflated for overfit models.

### Comparison of All Methods

```python
from sklearn.feature_selection import (
    SelectKBest, f_classif, mutual_info_classif,
    RFE, SelectFromModel
)
from sklearn.ensemble import RandomForestClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.preprocessing import StandardScaler

# Standardize
X_std = StandardScaler().fit_transform(X)

# Method 1: Filter (F-test)
filter_selector = SelectKBest(f_classif, k=5).fit(X_std, y)

# Method 2: Filter (Mutual Info)
mi_selector = SelectKBest(mutual_info_classif, k=5).fit(X_std, y)

# Method 3: Wrapper (RFE)
rfe_selector = RFE(LogisticRegression(max_iter=5000), n_features_to_select=5).fit(X_std, y)

# Method 4: Embedded (L1)
l1_model = LogisticRegression(penalty='l1', C=0.5, solver='saga', max_iter=5000)
l1_selector = SelectFromModel(l1_model, max_features=5).fit(X_std, y)

# Method 5: Embedded (Tree)
rf_selector = SelectFromModel(
    RandomForestClassifier(n_estimators=200, random_state=42), max_features=5
).fit(X_std, y)

# Compare
methods = {
    'F-test': filter_selector.get_support(),
    'Mutual Info': mi_selector.get_support(),
    'RFE': rfe_selector.get_support(),
    'L1': l1_selector.get_support(),
    'RF Importance': rf_selector.get_support()
}

print(f"{'Method':<15} {'Selected Features'}")
print("-" * 50)
for name, support in methods.items():
    features = np.where(support)[0]
    print(f"{name:<15} {features}")

# Features selected by multiple methods are most likely truly important
consensus = np.zeros(X.shape[1])
for support in methods.values():
    consensus += support.astype(int)

print(f"\nConsensus (selected by N methods):")
for i in np.argsort(consensus)[::-1]:
    if consensus[i] > 0:
        print(f"  Feature {i:2d}: selected by {int(consensus[i])}/5 methods")
```

> **Pro Tip**: Use "consensus" feature selection — features selected by multiple methods are more likely to be truly important.

---

## Feature Extraction

### What It Is
Feature extraction transforms existing features into new, more compact representations. Unlike selection (which picks from existing), extraction creates new features.

### Polynomial Features

Creates interaction terms and polynomial terms from existing features.

For features $[x_1, x_2]$ with degree=2:
$$[1, x_1, x_2, x_1^2, x_1 x_2, x_2^2]$$

```python
from sklearn.preprocessing import PolynomialFeatures
import pandas as pd

# Simple example
X_simple = np.array([[2, 3], [4, 5], [6, 7]])

# Degree 2 polynomial features
poly = PolynomialFeatures(degree=2, include_bias=False)
X_poly = poly.fit_transform(X_simple)

# Show what was created
feature_names = poly.get_feature_names_out(['x1', 'x2'])
df = pd.DataFrame(X_poly, columns=feature_names)
print("Polynomial Features (degree=2):")
print(df.to_string(index=False))

# Output:
#  x1  x2  x1^2  x1 x2  x2^2
#   2   3     4      6      9
#   4   5    16     20     25
#   6   7    36     42     49
```

```python
# Interaction-only features (no powers like x1², x2²)
poly_interact = PolynomialFeatures(degree=2, interaction_only=True, include_bias=False)
X_interact = poly_interact.fit_transform(X_simple)
names = poly_interact.get_feature_names_out(['x1', 'x2'])
print(f"\nInteraction only: {names}")
# ['x1', 'x2', 'x1 x2']

# Real-world example: predicting house price
# Features: [sqft, bedrooms, age]
# Interactions capture: sqft*bedrooms (big house with many rooms = luxury)
#                       sqft*age (old big house = might need renovation)
```

#### Feature Count Explosion

The number of polynomial features grows combinatorially:

$$\text{Number of features} = \binom{n + d}{d} - 1 \text{ (without bias)}$$

Where $n$ = original features, $d$ = degree.

| Original Features | Degree 2 | Degree 3 | Degree 4 |
|:-:|:-:|:-:|:-:|
| 5 | 20 | 55 | 125 |
| 10 | 65 | 285 | 1,000 |
| 50 | 1,325 | 23,425 | 316,250 |
| 100 | 5,150 | 176,850 | 4,598,125 |

> **Warning**: With 100 features and degree 3, you'll get 176,850 features! Always combine with feature selection or regularization.

```python
# Practical pattern: Polynomial + Regularization
from sklearn.pipeline import Pipeline
from sklearn.linear_model import Ridge

pipe = Pipeline([
    ('poly', PolynomialFeatures(degree=2, include_bias=False)),
    ('scaler', StandardScaler()),
    ('model', Ridge(alpha=1.0))  # Regularization prevents overfitting
])

# Or: Polynomial + Feature Selection
pipe_select = Pipeline([
    ('poly', PolynomialFeatures(degree=2, include_bias=False)),
    ('select', SelectKBest(f_regression, k=20)),  # Keep only top-20
    ('model', Ridge(alpha=1.0))
])
```

### Custom Feature Extraction with FunctionTransformer

```python
from sklearn.preprocessing import FunctionTransformer

# Create custom transformations
log_transformer = FunctionTransformer(
    func=np.log1p,        # log(1 + x) — safe for zeros
    inverse_func=np.expm1  # Inverse: exp(x) - 1
)

# Use in pipeline
from sklearn.pipeline import Pipeline
pipe = Pipeline([
    ('log_transform', log_transformer),
    ('model', LogisticRegression())
])
```

---

## Polynomial Features

### Deep Dive: When and Why

**When to use Polynomial Features:**
1. Linear model + non-linear data → polynomial features make it linear in higher-D
2. Known domain interactions (e.g., BMI = weight/height²)
3. Small number of features (< 20)

**When NOT to use:**
1. Already using non-linear models (trees, neural nets)
2. Many features (explosion problem)
3. No domain reason to expect interactions

```python
# Demonstration: Linear model fails on non-linear data
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_squared_error

# Non-linear relationship: y = x² + noise
np.random.seed(42)
X_demo = np.random.uniform(-3, 3, (200, 1))
y_demo = X_demo[:, 0]**2 + np.random.normal(0, 0.5, 200)

X_train, X_test, y_train, y_test = train_test_split(X_demo, y_demo, test_size=0.3, random_state=42)

# Plain linear regression — fails
lr = LinearRegression().fit(X_train, y_train)
print(f"Linear R²: {lr.score(X_test, y_test):.4f}")  # ~0.0 (terrible)

# Polynomial degree 2 — perfect for y = x²
poly = PolynomialFeatures(degree=2, include_bias=False)
X_train_poly = poly.fit_transform(X_train)
X_test_poly = poly.transform(X_test)

lr_poly = LinearRegression().fit(X_train_poly, y_train)
print(f"Poly(2) R²: {lr_poly.score(X_test_poly, y_test):.4f}")  # ~0.99 (excellent)

# Higher degree — overfits
poly_high = PolynomialFeatures(degree=15, include_bias=False)
X_train_high = poly_high.fit_transform(X_train)
X_test_high = poly_high.transform(X_test)

lr_high = LinearRegression().fit(X_train_high, y_train)
print(f"Poly(15) R² train: {lr_high.score(X_train_high, y_train):.4f}")  # ~1.0
print(f"Poly(15) R² test:  {lr_high.score(X_test_high, y_test):.4f}")   # Might be negative!
```

---

## Putting It All Together — Pipelines

### Complete Feature Selection Pipeline

```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.feature_selection import SelectKBest, f_classif, SelectFromModel
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_val_score
import pandas as pd

# Simulate a real dataset
np.random.seed(42)
n = 1000

data = pd.DataFrame({
    'age': np.random.randint(18, 70, n),
    'income': np.random.normal(50000, 15000, n),
    'credit_score': np.random.randint(300, 850, n),
    'num_products': np.random.randint(1, 10, n),
    'tenure_months': np.random.randint(1, 120, n),
    'noise_1': np.random.randn(n),        # Irrelevant
    'noise_2': np.random.randn(n),        # Irrelevant
    'noise_3': np.random.randn(n),        # Irrelevant
    'category': np.random.choice(['A', 'B', 'C'], n),
    'region': np.random.choice(['North', 'South', 'East', 'West'], n)
})

# Target (depends on age, income, credit_score)
y = ((data['age'] > 40) & (data['income'] > 50000) & (data['credit_score'] > 600)).astype(int)

# Define column types
numeric_features = ['age', 'income', 'credit_score', 'num_products', 'tenure_months', 'noise_1', 'noise_2', 'noise_3']
categorical_features = ['category', 'region']

# Build pipeline with feature selection
pipeline = Pipeline([
    # Step 1: Preprocessing
    ('preprocessor', ColumnTransformer([
        ('num', StandardScaler(), numeric_features),
        ('cat', OneHotEncoder(drop='first', sparse_output=False), categorical_features)
    ])),
    
    # Step 2: Feature Selection
    ('feature_selection', SelectKBest(f_classif, k=5)),
    
    # Step 3: Model
    ('classifier', RandomForestClassifier(n_estimators=100, random_state=42))
])

# Evaluate
scores = cross_val_score(pipeline, data, y, cv=5, scoring='accuracy')
print(f"Pipeline with feature selection:")
print(f"  Accuracy: {scores.mean():.4f} ± {scores.std():.4f}")

# Compare without feature selection
pipeline_no_select = Pipeline([
    ('preprocessor', ColumnTransformer([
        ('num', StandardScaler(), numeric_features),
        ('cat', OneHotEncoder(drop='first', sparse_output=False), categorical_features)
    ])),
    ('classifier', RandomForestClassifier(n_estimators=100, random_state=42))
])

scores_no_select = cross_val_score(pipeline_no_select, data, y, cv=5, scoring='accuracy')
print(f"\nPipeline WITHOUT feature selection:")
print(f"  Accuracy: {scores_no_select.mean():.4f} ± {scores_no_select.std():.4f}")
```

### Tuning Feature Selection in GridSearch

```python
from sklearn.model_selection import GridSearchCV

# Pipeline where feature selection is a tunable step
pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('selector', SelectKBest(f_classif)),
    ('classifier', RandomForestClassifier(random_state=42))
])

# Tune both the number of features AND model hyperparameters
param_grid = {
    'selector__k': [3, 5, 7, 10, 'all'],
    'classifier__n_estimators': [50, 100, 200],
    'classifier__max_depth': [3, 5, None]
}

grid_search = GridSearchCV(
    pipeline, param_grid,
    cv=5, scoring='accuracy', n_jobs=-1
)
grid_search.fit(X, y)

print(f"Best params: {grid_search.best_params_}")
print(f"Best score: {grid_search.best_score_:.4f}")
```

---

## Common Mistakes

### 1. Data Leakage in Feature Selection

**WRONG** — Selecting features on the entire dataset before splitting:
```python
# ❌ WRONG: Feature selection before train/test split
selector = SelectKBest(f_classif, k=5)
X_selected = selector.fit_transform(X, y)  # Uses ALL data including test!

X_train, X_test, y_train, y_test = train_test_split(X_selected, y)
# Test performance is INFLATED — you've already "peeked" at test data
```

**RIGHT** — Feature selection only on training data:
```python
# ✓ CORRECT: Feature selection inside pipeline or after split
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3)

selector = SelectKBest(f_classif, k=5)
X_train_selected = selector.fit_transform(X_train, y_train)  # Fit on TRAIN only
X_test_selected = selector.transform(X_test)                   # Transform test

# BEST: Use a Pipeline (handles this automatically)
pipeline = Pipeline([
    ('selector', SelectKBest(f_classif, k=5)),
    ('model', RandomForestClassifier())
])
pipeline.fit(X_train, y_train)  # selector.fit only sees X_train
```

### 2. Using Chi-Squared on Negative Values

```python
# ❌ WRONG: chi2 requires non-negative features
from sklearn.feature_selection import chi2
# chi2(X_with_negatives, y)  # Will raise ValueError or give wrong results

# ✓ CORRECT: Scale to non-negative first
from sklearn.preprocessing import MinMaxScaler
X_positive = MinMaxScaler().fit_transform(X)
chi2_scores, p_values = chi2(X_positive, y)
```

### 3. Trusting Tree-Based Feature Importance Blindly

```python
# Tree importance is biased toward high-cardinality features
# Example: random ID column might appear "important"

# ✓ ALWAYS validate with permutation importance
from sklearn.inspection import permutation_importance

rf.fit(X_train, y_train)
perm_imp = permutation_importance(rf, X_test, y_test, n_repeats=10, random_state=42)
# Use perm_imp.importances_mean for unbiased importance
```

### 4. Too Aggressive Feature Selection

```python
# ❌ Selecting too few features can discard important information
# Especially if features interact (x1 alone not useful, but x1*x2 is)

# ✓ Use RFECV to find optimal number data-driven
# ✓ Check learning curves after selection
# ✓ Start conservative (keep more features), then reduce
```

### 5. Not Scaling Before L1 Regularization

```python
# ❌ L1 penalizes based on magnitude — unscaled features are unfairly treated
# Feature in range [0, 1000] will have tiny coefficient → selected
# Feature in range [0, 1] will have large coefficient → penalized

# ✓ ALWAYS StandardScaler before L1-based selection
pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('selector', SelectFromModel(LogisticRegression(penalty='l1', solver='saga')))
])
```

### 6. Ignoring Feature Correlation

```python
# Two highly correlated features: model picks one, drops the other randomly
# This makes importance scores UNSTABLE

# ✓ Check correlation matrix first
import pandas as pd
correlation_matrix = pd.DataFrame(X).corr().abs()

# Find highly correlated pairs
upper_triangle = correlation_matrix.where(
    np.triu(np.ones(correlation_matrix.shape), k=1).astype(bool)
)
high_corr_pairs = [
    (i, col) for col in upper_triangle.columns 
    for i in upper_triangle.index 
    if upper_triangle.loc[i, col] > 0.9
]
print(f"Highly correlated pairs: {high_corr_pairs}")
```

---

## Interview Questions

### Conceptual Questions

**Q1: What's the difference between filter, wrapper, and embedded methods?**

| Aspect | Filter | Wrapper | Embedded |
|--------|--------|---------|----------|
| Uses model? | No (statistical test) | Yes (trains model) | Yes (built into training) |
| Speed | Fastest | Slowest | Medium |
| Accuracy | Good | Best (usually) | Good-to-best |
| Overfitting risk | Low | High (if not CV) | Low (regularized) |
| Example | SelectKBest | RFE | Lasso, Tree importance |

**Q2: Why does L1 regularization produce sparse solutions but L2 doesn't?**

The L1 constraint region is a diamond (polytope with corners on axes), while L2 is a sphere. The loss function's level curves are more likely to first touch the diamond at a corner, where one or more weights are exactly zero. L2's smooth sphere surface means the intersection point typically has all weights non-zero.

**Q3: What is the curse of dimensionality and how does feature selection help?**

As dimensions increase, data becomes increasingly sparse. Distance metrics become less meaningful (all points are "equally far"). Feature selection reduces dimensions to mitigate this, concentrating data in a lower-dimensional space where models can find meaningful patterns.

**Q4: How would you handle feature selection with 100,000 features (e.g., genomics)?**

1. Start with VarianceThreshold (remove constant/near-constant)
2. Apply univariate filter (SelectKBest with mutual_info) — fast even for many features
3. Use L1 regularization (Lasso/ElasticNet) for further reduction
4. Optionally apply RFECV on the remaining subset
5. Use domain knowledge to validate selections

**Q5: When would you prefer mutual information over F-test for feature selection?**

Mutual information captures **any** statistical dependency (linear or non-linear), while F-test only captures linear relationships. Use MI when:
- You suspect non-linear feature-target relationships
- Features interact in complex ways
- You don't want to assume normality

Use F-test when:
- Speed matters (faster than MI)
- Relationships are approximately linear
- You want p-values for statistical significance

**Q6: What's the problem with using training set feature importance?**

Training set importance can be inflated for features the model has memorized (overfit). A random noise feature might appear important on training data because the model uses it to memorize noise. Always evaluate importance on held-out test data using permutation importance.

**Q7: How do you handle feature selection when features are highly correlated?**

Options:
1. Remove one of each highly correlated pair before selection
2. Use PCA/dimensionality reduction first
3. Use ElasticNet (L1+L2) — handles groups of correlated features better than pure L1
4. Use group-based selection methods
5. Average importance across correlated groups

**Q8: Explain the difference between `feature_importances_` and permutation importance.**

- `feature_importances_` (MDI): Based on impurity decrease at splits. Computed during training. Biased toward high-cardinality and correlated features.
- Permutation importance: Measures accuracy drop when feature is shuffled. Computed post-training on any dataset. Unbiased but slower.

### Coding Questions

**Q9: Implement a pipeline that does feature selection and model training without data leakage.**

```python
from sklearn.pipeline import Pipeline
from sklearn.model_selection import cross_val_score

pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('selector', SelectKBest(f_classif, k=10)),
    ('model', RandomForestClassifier(random_state=42))
])

# cross_val_score handles train/test properly per fold
scores = cross_val_score(pipeline, X, y, cv=5, scoring='accuracy')
```

**Q10: How would you find the optimal number of features?**

```python
# Method 1: RFECV
rfecv = RFECV(estimator=RandomForestClassifier(), cv=5, scoring='accuracy')
rfecv.fit(X, y)
optimal_k = rfecv.n_features_

# Method 2: GridSearch over k
param_grid = {'selector__k': range(1, X.shape[1] + 1)}
grid = GridSearchCV(pipeline, param_grid, cv=5)
grid.fit(X, y)
optimal_k = grid.best_params_['selector__k']
```

---

## Quick Reference

### Method Selection Guide

```
START
  │
  ├─ Need speed? → Filter Methods (SelectKBest, VarianceThreshold)
  │
  ├─ Need best accuracy? → Wrapper Methods (RFECV)
  │
  ├─ Need interpretability? → Embedded (L1, Tree Importance)
  │
  ├─ Non-linear relationships? → mutual_info_classif
  │
  ├─ Very high dimensions (>10K)? → VarianceThreshold → Filter → L1
  │
  └─ Don't know? → Start with SelectFromModel(RandomForest)
```

### Scikit-Learn Feature Selection API

| Class | Type | Key Parameters | Returns |
|-------|------|---------------|---------|
| `VarianceThreshold` | Filter | `threshold` | Low-variance removed |
| `SelectKBest` | Filter | `score_func`, `k` | Top-k features |
| `SelectPercentile` | Filter | `score_func`, `percentile` | Top % features |
| `RFE` | Wrapper | `estimator`, `n_features_to_select` | Recursively selected |
| `RFECV` | Wrapper | `estimator`, `cv`, `scoring` | Optimal # features |
| `SequentialFeatureSelector` | Wrapper | `estimator`, `direction` | Forward/backward selected |
| `SelectFromModel` | Embedded | `estimator`, `threshold` | Above-threshold features |

### Scoring Functions

| Function | Task | Captures Non-linear? | Requires |
|----------|------|---------------------|----------|
| `f_classif` | Classification | No | Any features |
| `chi2` | Classification | No | Non-negative features |
| `mutual_info_classif` | Classification | Yes | Any features |
| `f_regression` | Regression | No | Any features |
| `mutual_info_regression` | Regression | Yes | Any features |

### Key Attributes After Fitting

| Attribute | Available On | Meaning |
|-----------|-------------|---------|
| `.get_support()` | All selectors | Boolean mask of selected features |
| `.scores_` | SelectKBest, SelectPercentile | Score per feature |
| `.pvalues_` | SelectKBest (some functions) | P-value per feature |
| `.ranking_` | RFE, RFECV | Rank per feature (1=selected) |
| `.n_features_` | RFECV | Optimal number of features |
| `.feature_importances_` | Tree models | MDI importance |
| `.coef_` | Linear models | Feature coefficients |

### Decision Cheat Sheet

| Scenario | Recommended Method |
|----------|-------------------|
| Quick baseline | `SelectKBest(f_classif, k=10)` |
| Non-linear data | `SelectKBest(mutual_info_classif, k=10)` |
| Remove useless features fast | `VarianceThreshold(threshold=0.01)` |
| Find optimal feature count | `RFECV(estimator, cv=5)` |
| Automatic with trees | `SelectFromModel(RandomForestClassifier())` |
| Sparse model needed | `SelectFromModel(Lasso(alpha=0.01))` |
| Text/NLP features | `chi2` (works with TF-IDF) |
| Production pipeline | Pipeline + GridSearchCV over `k` |

---

*Next Chapter: [08-Advanced-Sklearn.md](./08-Advanced-Sklearn.md) — Custom Estimators, Transformers, and Model Persistence*
