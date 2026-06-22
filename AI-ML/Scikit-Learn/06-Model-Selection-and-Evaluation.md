# Chapter 06: Model Selection and Evaluation in Scikit-Learn

## Table of Contents
- [Introduction to Model Selection](#introduction-to-model-selection)
- [Train-Test Split](#train-test-split)
- [Cross-Validation](#cross-validation)
- [Hyperparameter Tuning — GridSearchCV & RandomizedSearchCV](#hyperparameter-tuning--gridsearchcv--randomizedsearchcv)
- [Evaluation Metrics — Classification](#evaluation-metrics--classification)
- [Evaluation Metrics — Regression](#evaluation-metrics--regression)
- [Classification Report & Confusion Matrix](#classification-report--confusion-matrix)
- [ROC Curve and AUC](#roc-curve-and-auc)
- [Precision-Recall Curve](#precision-recall-curve)
- [Learning Curves & Validation Curves](#learning-curves--validation-curves)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Introduction to Model Selection

### What It Is
Model selection is the process of choosing the **best model** and the **best settings** (hyperparameters) for your specific problem. It answers two questions:
1. **Which algorithm** should I use? (Random Forest vs SVM vs Logistic Regression?)
2. **What settings** should I use? (How many trees? What learning rate?)

Think of it like buying a car: first you decide the type (sedan vs SUV vs truck), then you configure the options (engine, color, features).

### Why It Matters
- The difference between a mediocre and great model is often **tuning, not algorithm choice**
- Wrong evaluation → deploying a model that looks good in testing but fails in production
- Overfitting to the test set is a real, dangerous problem
- Proper model selection is what separates beginners from production-ready data scientists

### The Model Selection Workflow

```
┌────────────────────────────────────────────────────────────┐
│               MODEL SELECTION WORKFLOW                      │
├────────────────────────────────────────────────────────────┤
│                                                              │
│  1. SPLIT DATA                                              │
│     ┌──────────────────────────────────────┐                │
│     │  Full Dataset                        │                │
│     │  ┌─────────────────┐ ┌─────────────┐│                │
│     │  │  Training Set   │ │  Test Set    ││                │
│     │  │  (80%)          │ │  (20%)       ││                │
│     │  └─────────────────┘ └─────────────┘│                │
│     └──────────────────────────────────────┘                │
│                                                              │
│  2. CROSS-VALIDATE on training set                          │
│     → Try different algorithms & hyperparameters            │
│     → Pick the best combination                             │
│                                                              │
│  3. EVALUATE on test set (ONCE!)                            │
│     → Get unbiased performance estimate                     │
│     → NEVER go back and re-tune based on test results!      │
│                                                              │
│  4. DEPLOY the final model                                  │
│     → Retrain on ALL data (train + test) with best params  │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

---

## Train-Test Split

### What It Is
Split your data into two parts: one to **learn** from (training) and one to **evaluate** on (testing). The test set is data the model has **never seen** during training — it simulates real-world unseen data.

### Why You Can't Just Evaluate on Training Data
A model that memorizes training data will score 100% on it but fail on new data. That's like a student who memorizes test answers — they score perfectly on that test but can't solve new problems.

### Code Examples

#### Basic Split

```python
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.datasets import load_iris

X, y = load_iris(return_X_y=True)

# Basic 80/20 split
X_train, X_test, y_train, y_test = train_test_split(
    X, y,
    test_size=0.2,        # 20% for testing (common: 0.2-0.3)
    random_state=42,      # Reproducibility — ALWAYS set this!
    stratify=y            # Preserve class distribution in both sets
)

print(f"Training: {X_train.shape[0]} samples")
print(f"Testing:  {X_test.shape[0]} samples")

# Check stratification
from collections import Counter
print(f"Train class distribution: {Counter(y_train)}")
print(f"Test class distribution:  {Counter(y_test)}")

# Output:
# Training: 120 samples
# Testing:  30 samples
# Train class distribution: Counter({0: 40, 1: 40, 2: 40})
# Test class distribution:  Counter({0: 10, 1: 10, 2: 10})
```

#### Three-Way Split (Train / Validation / Test)

```python
# For hyperparameter tuning without cross-validation
# Split: 60% train, 20% validation, 20% test

# First split: separate test set
X_temp, X_test, y_temp, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

# Second split: separate validation from training
X_train, X_val, y_train, y_val = train_test_split(
    X_temp, y_temp, test_size=0.25, random_state=42, stratify=y_temp
    # 0.25 of 0.8 = 0.2 of total
)

print(f"Train: {len(X_train)}, Val: {len(X_val)}, Test: {len(X_test)}")
# Output: Train: 90, Val: 30, Test: 30
```

> **Pro Tip:** Use `stratify=y` for classification to ensure each class appears proportionally in both sets. This is critical for imbalanced datasets.

### When to Use What Split Ratio

| Dataset Size | Recommended Split | Reason |
|-------------|-------------------|--------|
| < 1,000 | Use cross-validation instead | Not enough data for reliable split |
| 1,000 - 100,000 | 80/20 or 70/30 | Standard split |
| > 100,000 | 90/10 or 95/5 | Even 5% test set is thousands of samples |
| > 1,000,000 | 98/1/1 (train/val/test) | 10K test samples is usually enough |

---

## Cross-Validation

### What It Is
Cross-validation is like taking 5 different practice exams, each covering different questions. Instead of one train/test split (which might be lucky or unlucky), you do **multiple splits** and average the results.

### Why It Matters
- More reliable performance estimate than a single split
- Uses ALL data for both training and testing
- Detects if your model is overfitting
- Essential when you don't have much data

### How K-Fold Cross-Validation Works

```
┌────────────────────────────────────────────────────────────┐
│              5-FOLD CROSS-VALIDATION                        │
├────────────────────────────────────────────────────────────┤
│                                                              │
│  Data: [████████████████████████████████████████████]         │
│                                                              │
│  Fold 1: [TEST ][Train][Train][Train][Train] → Score₁       │
│  Fold 2: [Train][TEST ][Train][Train][Train] → Score₂       │
│  Fold 3: [Train][Train][TEST ][Train][Train] → Score₃       │
│  Fold 4: [Train][Train][Train][TEST ][Train] → Score₄       │
│  Fold 5: [Train][Train][Train][Train][TEST ] → Score₅       │
│                                                              │
│  Final Score = mean(Score₁, Score₂, Score₃, Score₄, Score₅) │
│  Uncertainty  = std(Score₁, Score₂, Score₃, Score₄, Score₅) │
│                                                              │
│  Every sample is in the TEST set exactly ONCE!               │
│                                                              │
└────────────────────────────────────────────────────────────┘
```

### Code Examples

#### Basic cross_val_score

```python
from sklearn.model_selection import cross_val_score
from sklearn.ensemble import RandomForestClassifier

X, y = load_iris(return_X_y=True)

rf = RandomForestClassifier(n_estimators=100, random_state=42)

# 5-Fold Cross-Validation
scores = cross_val_score(
    rf, X, y,
    cv=5,                   # Number of folds (5 or 10 is standard)
    scoring='accuracy',     # Metric to evaluate
    n_jobs=-1               # Use all CPU cores
)

print(f"Fold scores: {scores}")
print(f"Mean accuracy: {scores.mean():.4f} ± {scores.std():.4f}")

# Output:
# Fold scores: [0.9667 0.9667 0.9333 0.9667 1.0000]
# Mean accuracy: 0.9667 ± 0.0211
```

#### cross_validate — Multiple Metrics at Once

```python
from sklearn.model_selection import cross_validate

# Get multiple metrics, plus train scores and fit/score times
cv_results = cross_validate(
    rf, X, y,
    cv=5,
    scoring=['accuracy', 'f1_weighted', 'precision_weighted'],
    return_train_score=True,  # Also report training scores (detect overfitting!)
    n_jobs=-1
)

# Results is a dict
for key, values in cv_results.items():
    if key.startswith('test_') or key.startswith('train_'):
        print(f"{key:>30s}: {values.mean():.4f} ± {values.std():.4f}")

# Output:
#              test_accuracy: 0.9667 ± 0.0211
#             train_accuracy: 1.0000 ± 0.0000  ← If much higher, overfitting!
#           test_f1_weighted: 0.9665 ± 0.0214
#   test_precision_weighted: 0.9694 ± 0.0193
```

#### Stratified K-Fold (For Imbalanced Classes)

```python
from sklearn.model_selection import StratifiedKFold

# StratifiedKFold preserves class proportions in each fold
# CRITICAL for imbalanced datasets!
skf = StratifiedKFold(
    n_splits=5,
    shuffle=True,       # Shuffle before splitting (recommended)
    random_state=42
)

# Use it with cross_val_score
scores = cross_val_score(rf, X, y, cv=skf, scoring='accuracy')
print(f"Stratified CV: {scores.mean():.4f} ± {scores.std():.4f}")

# Manual iteration over folds
for fold, (train_idx, test_idx) in enumerate(skf.split(X, y)):
    X_fold_train, X_fold_test = X[train_idx], X[test_idx]
    y_fold_train, y_fold_test = y[train_idx], y[test_idx]
    print(f"Fold {fold}: Train={len(train_idx)}, Test={len(test_idx)}, "
          f"Test class dist: {Counter(y_fold_test)}")
```

#### Other Cross-Validation Strategies

```python
from sklearn.model_selection import (
    LeaveOneOut,          # LOO: each sample is a test set once (expensive!)
    RepeatedStratifiedKFold,  # Repeat K-Fold multiple times
    TimeSeriesSplit,      # For time series data (no future leakage!)
    GroupKFold            # Ensure same group stays together (e.g., same patient)
)

# Repeated Stratified K-Fold — most robust estimate
rskf = RepeatedStratifiedKFold(n_splits=5, n_repeats=3, random_state=42)
scores = cross_val_score(rf, X, y, cv=rskf, scoring='accuracy')
print(f"Repeated CV (5×3=15 folds): {scores.mean():.4f} ± {scores.std():.4f}")

# Time Series Split — NEVER let future data leak into training!
tscv = TimeSeriesSplit(n_splits=5)
for train_idx, test_idx in tscv.split(X):
    print(f"Train: {train_idx[:3]}...{train_idx[-3:]}, Test: {test_idx}")
    
# GroupKFold — e.g., all data from one patient stays in same fold
from sklearn.model_selection import GroupKFold
groups = np.array([0,0,0,1,1,1,2,2,2,3,3,3])  # Patient IDs
gkf = GroupKFold(n_splits=4)
for train_idx, test_idx in gkf.split(X[:12], y[:12], groups):
    print(f"Train groups: {np.unique(groups[train_idx])}, "
          f"Test groups: {np.unique(groups[test_idx])}")
```

> **Pro Tip:** For time series data, ALWAYS use `TimeSeriesSplit`. Standard K-Fold causes data leakage by training on future data.

---

## Hyperparameter Tuning — GridSearchCV & RandomizedSearchCV

### What It Is
Most ML models have **hyperparameters** — settings you choose before training (like `n_estimators`, `max_depth`, `C`). Finding the best combination is called hyperparameter tuning.

- **GridSearchCV:** Try EVERY combination (exhaustive but slow)
- **RandomizedSearchCV:** Try RANDOM combinations (faster, often just as good)

### Why It Matters
- A well-tuned model can outperform a poorly tuned one by 5-15%
- Default parameters are rarely optimal for your specific data
- Manual tuning is tedious and doesn't scale
- Grid/random search + CV prevents overfitting to the test set

### How GridSearchCV Works

```
┌──────────────────────────────────────────────────────────────┐
│                    GRID SEARCH + CV                           │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Hyperparameter Grid:                                         │
│  max_depth = [3, 5, 10]                                      │
│  n_estimators = [50, 100, 200]                                │
│  → 3 × 3 = 9 combinations                                    │
│                                                                │
│  For EACH combination:                                        │
│  ┌─────────────────────────────────────────┐                  │
│  │ Run 5-Fold Cross-Validation             │                  │
│  │ → Get mean CV score for this combo      │                  │
│  └─────────────────────────────────────────┘                  │
│                                                                │
│  Total fits: 9 combinations × 5 folds = 45 model fits        │
│                                                                │
│  Pick the combination with BEST mean CV score                 │
│  Refit on full training data with best params                 │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Code Examples

#### GridSearchCV

```python
from sklearn.model_selection import GridSearchCV
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import make_classification

# Generate dataset
X, y = make_classification(n_samples=1000, n_features=20, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Define parameter grid
param_grid = {
    'n_estimators': [50, 100, 200],
    'max_depth': [3, 5, 10, None],
    'min_samples_split': [2, 5, 10],
    'max_features': ['sqrt', 'log2']
}

# Total combinations: 3 × 4 × 3 × 2 = 72
# Total fits with 5-fold CV: 72 × 5 = 360

# Create GridSearchCV
grid_search = GridSearchCV(
    estimator=RandomForestClassifier(random_state=42),
    param_grid=param_grid,
    cv=5,                    # 5-fold stratified CV (default for classifiers)
    scoring='accuracy',      # Metric to optimize
    refit=True,              # Refit best model on full training data
    n_jobs=-1,               # Parallel processing
    verbose=1,               # Print progress (0=silent, 1=summary, 2=all)
    return_train_score=True  # Also report training scores
)

# Run the search
grid_search.fit(X_train, y_train)

# Results
print(f"Best Parameters: {grid_search.best_params_}")
print(f"Best CV Score: {grid_search.best_score_:.4f}")
print(f"Test Score: {grid_search.score(X_test, y_test):.4f}")

# Output:
# Best Parameters: {'max_depth': 10, 'max_features': 'sqrt', 
#                    'min_samples_split': 2, 'n_estimators': 200}
# Best CV Score: 0.9138
# Test Score: 0.9200

# The best model is already fitted and ready to use!
best_model = grid_search.best_estimator_
predictions = best_model.predict(X_test)
```

#### Analyzing Grid Search Results

```python
import pandas as pd

# Convert results to DataFrame
results_df = pd.DataFrame(grid_search.cv_results_)

# Show top 5 parameter combinations
top5 = results_df.nsmallest(5, 'rank_test_score')[
    ['params', 'mean_test_score', 'std_test_score', 'mean_train_score', 'rank_test_score']
]
print(top5.to_string())

# Check for overfitting: large gap between train and test scores
print("\nOverfitting check:")
print(f"Best train score: {results_df.loc[grid_search.best_index_, 'mean_train_score']:.4f}")
print(f"Best test score:  {grid_search.best_score_:.4f}")
```

#### RandomizedSearchCV (Faster!)

```python
from sklearn.model_selection import RandomizedSearchCV
from scipy.stats import randint, uniform

# Instead of a grid, define DISTRIBUTIONS to sample from
param_distributions = {
    'n_estimators': randint(50, 500),          # Random integer between 50-500
    'max_depth': randint(3, 30),               # Random integer between 3-30
    'min_samples_split': randint(2, 20),       # Random integer between 2-20
    'min_samples_leaf': randint(1, 10),        # Random integer between 1-10
    'max_features': uniform(0.1, 0.9),         # Random float between 0.1-1.0
}

# RandomizedSearchCV — try 50 random combinations
random_search = RandomizedSearchCV(
    estimator=RandomForestClassifier(random_state=42),
    param_distributions=param_distributions,
    n_iter=50,               # Try 50 random combinations (vs 72+ for grid)
    cv=5,
    scoring='accuracy',
    refit=True,
    n_jobs=-1,
    random_state=42,
    verbose=1
)

random_search.fit(X_train, y_train)

print(f"Best Parameters: {random_search.best_params_}")
print(f"Best CV Score: {random_search.best_score_:.4f}")
print(f"Test Score: {random_search.score(X_test, y_test):.4f}")
```

#### Multi-Metric Grid Search

```python
# Optimize for multiple metrics simultaneously
grid_multi = GridSearchCV(
    estimator=RandomForestClassifier(random_state=42),
    param_grid={'n_estimators': [50, 100], 'max_depth': [5, 10]},
    cv=5,
    scoring={
        'accuracy': 'accuracy',
        'f1': 'f1_weighted',
        'precision': 'precision_weighted'
    },
    refit='f1',              # Use F1 score to pick the best model
    return_train_score=True,
    n_jobs=-1
)

grid_multi.fit(X_train, y_train)
print(f"Best by F1: {grid_multi.best_params_}")
print(f"Best F1 score: {grid_multi.best_score_:.4f}")
```

### GridSearchCV vs RandomizedSearchCV

| Aspect | GridSearchCV | RandomizedSearchCV |
|--------|-------------|-------------------|
| Strategy | Try ALL combinations | Try N random combinations |
| Speed | Slow (exponential in params) | Fast (linear in n_iter) |
| Guarantee | Finds global best in grid | Might miss global best |
| When to use | Few params, small grid | Many params, large ranges |
| Parameters | 3-4 params, 3-4 values each | Any number of params |
| Scaling | 3⁴ = 81 fits (4 params, 3 values) | You choose: 50, 100, etc. |

> **Pro Tip:** Start with `RandomizedSearchCV(n_iter=50)` to narrow the search space, then use `GridSearchCV` on a fine-grained grid around the best params found.

---

## Evaluation Metrics — Classification

### Why Accuracy Isn't Enough

```
Scenario: Email spam detection (99% ham, 1% spam)

Model A: Always predicts "ham" → 99% accuracy! (but catches 0% spam)
Model B: Catches 80% of spam  → 98.8% accuracy (much more useful!)

Accuracy is MISLEADING for imbalanced classes.
```

### The Confusion Matrix — Foundation of All Metrics

```
                    Predicted
                  Pos       Neg
Actual  Pos    TP (True    FN (False
               Positive)   Negative)
        Neg    FP (False   TN (True
               Positive)   Negative)

TP = Correctly predicted positive (spam correctly detected)
TN = Correctly predicted negative (ham correctly identified)
FP = Incorrectly predicted positive (ham marked as spam) — "False Alarm"
FN = Incorrectly predicted negative (spam not caught) — "Miss"
```

### Core Metrics

| Metric | Formula | Intuition | Use When |
|--------|---------|-----------|----------|
| **Accuracy** | $\frac{TP + TN}{TP + TN + FP + FN}$ | % correct overall | Balanced classes |
| **Precision** | $\frac{TP}{TP + FP}$ | "Of predicted positives, how many are right?" | Cost of FP is high (spam filter) |
| **Recall (Sensitivity)** | $\frac{TP}{TP + FN}$ | "Of actual positives, how many did we catch?" | Cost of FN is high (cancer detection) |
| **F1 Score** | $\frac{2 \cdot P \cdot R}{P + R}$ | Harmonic mean of precision & recall | Imbalanced classes |
| **Specificity** | $\frac{TN}{TN + FP}$ | "Of actual negatives, how many did we identify?" | When TN matters |

### The Precision-Recall Trade-Off

```
                  High Threshold (0.9)          Low Threshold (0.1)
                  ─────────────────             ─────────────────
                  Few predictions               Many predictions
                  HIGH Precision                 LOW Precision
                  LOW Recall                     HIGH Recall
                  
                  "Only flag when VERY sure"     "Flag anything suspicious"
                  Miss some positives            Catch almost everything
                  Few false alarms               Many false alarms

  Application Examples:
  ┌──────────────────────┐   ┌──────────────────────┐
  │ PRECISION matters:   │   │ RECALL matters:       │
  │ • Search engine      │   │ • Cancer screening    │
  │ • Spam filter        │   │ • Fraud detection     │
  │ • Recommendation     │   │ • Security threats    │
  │ (Don't annoy users!) │   │ (Don't miss cases!)   │
  └──────────────────────┘   └──────────────────────┘
```

### Code Examples

```python
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score
)
from sklearn.linear_model import LogisticRegression

# Create imbalanced dataset
X, y = make_classification(
    n_samples=1000, n_features=20, 
    weights=[0.9, 0.1],  # 90% class 0, 10% class 1 (imbalanced!)
    random_state=42
)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, 
                                                     stratify=y, random_state=42)

# Train model
lr = LogisticRegression(max_iter=1000, random_state=42)
lr.fit(X_train, y_train)
y_pred = lr.predict(X_test)

# All metrics
print(f"Accuracy:  {accuracy_score(y_test, y_pred):.4f}")
print(f"Precision: {precision_score(y_test, y_pred):.4f}")
print(f"Recall:    {recall_score(y_test, y_pred):.4f}")
print(f"F1 Score:  {f1_score(y_test, y_pred):.4f}")

# Output:
# Accuracy:  0.9450  ← Looks great!
# Precision: 0.7500  ← Of those flagged positive, 75% correct
# Recall:    0.6000  ← Only catches 60% of actual positives!
# F1 Score:  0.6667  ← Balanced view — not as rosy as accuracy
```

### Multi-Class Averaging Strategies

```python
# For multi-class problems, how to average metrics across classes?
from sklearn.metrics import precision_score

# 'macro'   — Compute for each class, then unweighted average (treats all classes equal)
# 'weighted' — Compute for each class, then weighted average by class support
# 'micro'   — Compute globally (total TP, total FP across all classes)

y_true_multi = [0, 0, 1, 1, 2, 2, 2]
y_pred_multi = [0, 1, 1, 1, 2, 0, 2]

print(f"Macro Precision:    {precision_score(y_true_multi, y_pred_multi, average='macro'):.4f}")
print(f"Weighted Precision: {precision_score(y_true_multi, y_pred_multi, average='weighted'):.4f}")
print(f"Micro Precision:    {precision_score(y_true_multi, y_pred_multi, average='micro'):.4f}")
```

---

## Evaluation Metrics — Regression

### Core Regression Metrics

| Metric | Formula | Interpretation | Range |
|--------|---------|---------------|-------|
| **MAE** | $\frac{1}{n}\sum\|y_i - \hat{y}_i\|$ | Average absolute error | [0, ∞) lower = better |
| **MSE** | $\frac{1}{n}\sum(y_i - \hat{y}_i)^2$ | Average squared error (penalizes large errors) | [0, ∞) lower = better |
| **RMSE** | $\sqrt{MSE}$ | Same units as target variable | [0, ∞) lower = better |
| **R² Score** | $1 - \frac{\sum(y_i - \hat{y}_i)^2}{\sum(y_i - \bar{y})^2}$ | % variance explained | (-∞, 1] higher = better |
| **MAPE** | $\frac{100}{n}\sum\|\frac{y_i - \hat{y}_i}{y_i}\|$ | % error (scale-independent) | [0, ∞) lower = better |

### When to Use Which

```
Use MAE when:  You want interpretable errors, outliers should be treated normally
Use MSE/RMSE when:  Large errors should be penalized more (squared penalty)
Use R² when:  You want to compare models on different datasets
Use MAPE when:  You need percentage errors (business reporting)
```

### Code Examples

```python
from sklearn.metrics import (
    mean_absolute_error, mean_squared_error, r2_score,
    mean_absolute_percentage_error
)
from sklearn.ensemble import RandomForestRegressor
from sklearn.datasets import fetch_california_housing

# Load data
housing = fetch_california_housing()
X_train, X_test, y_train, y_test = train_test_split(
    housing.data, housing.target, test_size=0.2, random_state=42
)

# Train model
rf_reg = RandomForestRegressor(n_estimators=100, random_state=42)
rf_reg.fit(X_train, y_train)
y_pred = rf_reg.predict(X_test)

# Compute all metrics
mae = mean_absolute_error(y_test, y_pred)
mse = mean_squared_error(y_test, y_pred)
rmse = np.sqrt(mse)
r2 = r2_score(y_test, y_pred)
mape = mean_absolute_percentage_error(y_test, y_pred)

print(f"MAE:  {mae:.4f}")     # Average error in target units
print(f"MSE:  {mse:.4f}")     # Squared error (penalizes outliers)
print(f"RMSE: {rmse:.4f}")    # Same units as target
print(f"R²:   {r2:.4f}")      # 1.0 = perfect, 0.0 = predicts mean
print(f"MAPE: {mape:.2%}")    # Percentage error

# Output:
# MAE:  0.3275
# MSE:  0.2558
# RMSE: 0.5058
# R²:   0.8054
# MAPE: 16.85%
```

### Understanding R² Score

```
R² = 1 means perfect predictions
R² = 0 means model is as good as predicting the mean (useless)
R² < 0 means model is WORSE than predicting the mean!

R² = 1 - (SS_res / SS_tot)
    SS_res = Σ(y - ŷ)²     (how far predictions are from truth)
    SS_tot = Σ(y - ȳ)²     (how far truth is from its mean)
```

> **Pro Tip:** R² can be negative! This means your model is terrible — worse than always predicting the average. Check your data and model if R² < 0.

---

## Classification Report & Confusion Matrix

### Classification Report

```python
from sklearn.metrics import classification_report, confusion_matrix
from sklearn.datasets import load_digits
from sklearn.svm import SVC

# Load multi-class dataset
X, y = load_digits(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, 
                                                     stratify=y, random_state=42)

# Train model
svc = SVC(kernel='rbf', random_state=42)
svc.fit(X_train, y_train)
y_pred = svc.predict(X_test)

# Classification report — ALL metrics in one beautiful table
print(classification_report(y_test, y_pred))

# Output:
#               precision    recall  f1-score   support
#            0       1.00      1.00      1.00        36
#            1       0.97      1.00      0.99        36
#            2       1.00      1.00      1.00        35
#            3       1.00      0.97      0.99        37
#            4       1.00      1.00      1.00        36
#            5       0.97      1.00      0.99        37
#            6       1.00      1.00      1.00        36
#            7       1.00      1.00      1.00        36
#            8       0.97      0.97      0.97        35
#            9       1.00      0.97      0.99        36
#     accuracy                           0.99       360
#    macro avg       0.99      0.99      0.99       360
# weighted avg       0.99      0.99      0.99       360

# "support" = number of true instances per class
```

### Confusion Matrix

```python
import matplotlib.pyplot as plt
from sklearn.metrics import ConfusionMatrixDisplay

# Method 1: Simple print
cm = confusion_matrix(y_test, y_pred)
print("Confusion Matrix:")
print(cm)

# Method 2: Beautiful visualization
fig, ax = plt.subplots(figsize=(10, 8))
ConfusionMatrixDisplay.from_estimator(
    svc, X_test, y_test,
    cmap='Blues',
    normalize=None,     # None=counts, 'true'=normalize by true labels, 'pred'=by predictions
    ax=ax
)
plt.title('Confusion Matrix')
plt.tight_layout()
plt.show()

# Normalized confusion matrix (shows percentages)
fig, ax = plt.subplots(figsize=(10, 8))
ConfusionMatrixDisplay.from_estimator(
    svc, X_test, y_test,
    cmap='Blues',
    normalize='true',   # Each row sums to 1.0 (recall per class)
    values_format='.2f',
    ax=ax
)
plt.title('Normalized Confusion Matrix (Recall per Class)')
plt.tight_layout()
plt.show()
```

---

## ROC Curve and AUC

### What It Is
The **ROC (Receiver Operating Characteristic) curve** plots True Positive Rate vs False Positive Rate at every possible classification threshold. **AUC (Area Under the Curve)** summarizes the curve as a single number.

Think of it like tuning a metal detector's sensitivity at an airport: too sensitive → catches everything but many false alarms; too insensitive → misses real threats. The ROC curve shows the trade-off at every sensitivity level.

### How It Works

```
  True Positive Rate (Recall)
  1.0 ┌──────────────────────┐
      │         ·····........│ Perfect model (AUC=1.0)
      │     ···              │
      │   ··                 │
  0.5 │  ·       ··········· │ Good model (AUC=0.85)
      │ ·     ···            │
      │·   ···               │
      │  ··                  │
  0.0 │·─────────────────────│ Random (AUC=0.5)
      └──────────────────────┘
      0.0        0.5       1.0
      False Positive Rate (1 - Specificity)

  AUC = 1.0  →  Perfect classifier
  AUC = 0.5  →  Random guessing (useless)
  AUC < 0.5  →  Worse than random (flip predictions!)
```

### Code Examples

```python
from sklearn.metrics import roc_curve, roc_auc_score, RocCurveDisplay
from sklearn.linear_model import LogisticRegression

# Binary classification
X, y = make_classification(n_samples=1000, n_features=20, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

lr = LogisticRegression(max_iter=1000, random_state=42)
lr.fit(X_train, y_train)

# Get probability scores (NOT hard predictions!)
y_proba = lr.predict_proba(X_test)[:, 1]  # Probability of class 1

# Compute ROC curve
fpr, tpr, thresholds = roc_curve(y_test, y_proba)
auc_score = roc_auc_score(y_test, y_proba)

print(f"AUC Score: {auc_score:.4f}")

# Plot ROC curve
plt.figure(figsize=(8, 6))
plt.plot(fpr, tpr, color='blue', linewidth=2, label=f'Model (AUC = {auc_score:.4f})')
plt.plot([0, 1], [0, 1], color='gray', linestyle='--', label='Random (AUC = 0.5)')
plt.fill_between(fpr, tpr, alpha=0.1, color='blue')
plt.xlabel('False Positive Rate')
plt.ylabel('True Positive Rate (Recall)')
plt.title('ROC Curve')
plt.legend(loc='lower right')
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

#### Comparing Multiple Models

```python
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.svm import SVC

models = {
    'Logistic Regression': LogisticRegression(max_iter=1000, random_state=42),
    'Random Forest': RandomForestClassifier(n_estimators=100, random_state=42),
    'Gradient Boosting': GradientBoostingClassifier(n_estimators=100, random_state=42),
    'SVM': SVC(probability=True, random_state=42)
}

plt.figure(figsize=(10, 7))

for name, model in models.items():
    model.fit(X_train, y_train)
    y_proba = model.predict_proba(X_test)[:, 1]
    fpr, tpr, _ = roc_curve(y_test, y_proba)
    auc = roc_auc_score(y_test, y_proba)
    plt.plot(fpr, tpr, linewidth=2, label=f'{name} (AUC={auc:.4f})')

plt.plot([0, 1], [0, 1], 'k--', linewidth=1, label='Random')
plt.xlabel('False Positive Rate')
plt.ylabel('True Positive Rate')
plt.title('ROC Curve — Model Comparison')
plt.legend(loc='lower right')
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

#### Multi-Class ROC (One-vs-Rest)

```python
from sklearn.preprocessing import label_binarize
from sklearn.metrics import roc_auc_score

# For multi-class, use one-vs-rest and average
X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, 
                                                     stratify=y, random_state=42)

lr_multi = LogisticRegression(max_iter=1000, random_state=42)
lr_multi.fit(X_train, y_train)
y_proba_multi = lr_multi.predict_proba(X_test)

# Multi-class AUC (one-vs-rest)
auc_ovr = roc_auc_score(y_test, y_proba_multi, multi_class='ovr')
print(f"Multi-class AUC (OVR): {auc_ovr:.4f}")
```

---

## Precision-Recall Curve

### What It Is
For **imbalanced datasets**, the Precision-Recall (PR) curve is more informative than ROC. It shows the trade-off between precision and recall at every threshold.

### When to Use PR Curve vs ROC Curve

| Situation | Use | Why |
|-----------|-----|-----|
| Balanced classes | ROC/AUC | Both work well |
| Imbalanced classes (rare positives) | PR Curve | ROC can look too optimistic |
| Care about positive class | PR Curve | Focuses on positive class performance |
| Overall discriminative ability | ROC/AUC | Standard, widely understood |

### Code Example

```python
from sklearn.metrics import precision_recall_curve, average_precision_score
from sklearn.metrics import PrecisionRecallDisplay

# Imbalanced dataset
X, y = make_classification(n_samples=1000, n_features=20, 
                           weights=[0.95, 0.05],  # 95% negative, 5% positive!
                           random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, 
                                                     stratify=y, random_state=42)

lr = LogisticRegression(max_iter=1000, random_state=42)
lr.fit(X_train, y_train)
y_proba = lr.predict_proba(X_test)[:, 1]

# Compute PR curve
precision, recall, thresholds = precision_recall_curve(y_test, y_proba)
ap = average_precision_score(y_test, y_proba)

# Plot
plt.figure(figsize=(8, 6))
plt.plot(recall, precision, color='blue', linewidth=2, 
         label=f'Model (AP = {ap:.4f})')
plt.axhline(y=y.mean(), color='gray', linestyle='--', 
            label=f'Baseline (prevalence = {y.mean():.2f})')
plt.xlabel('Recall')
plt.ylabel('Precision')
plt.title('Precision-Recall Curve (Imbalanced Data)')
plt.legend()
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()

# Average Precision (AP) is the area under the PR curve
print(f"Average Precision: {ap:.4f}")
```

#### Finding the Optimal Threshold

```python
# Default threshold is 0.5, but that's often not optimal
# Find threshold that maximizes F1 score

from sklearn.metrics import f1_score

thresholds_test = np.arange(0.1, 0.9, 0.01)
f1_scores = []

for thresh in thresholds_test:
    y_pred_thresh = (y_proba >= thresh).astype(int)
    f1 = f1_score(y_test, y_pred_thresh)
    f1_scores.append(f1)

optimal_threshold = thresholds_test[np.argmax(f1_scores)]
print(f"Optimal Threshold: {optimal_threshold:.2f}")
print(f"Best F1 Score: {max(f1_scores):.4f}")

# Use optimal threshold for predictions
y_pred_optimal = (y_proba >= optimal_threshold).astype(int)
print(f"\nWith default threshold (0.5):")
print(f"  Precision: {precision_score(y_test, lr.predict(X_test)):.4f}")
print(f"  Recall:    {recall_score(y_test, lr.predict(X_test)):.4f}")
print(f"\nWith optimal threshold ({optimal_threshold}):")
print(f"  Precision: {precision_score(y_test, y_pred_optimal):.4f}")
print(f"  Recall:    {recall_score(y_test, y_pred_optimal):.4f}")
```

---

## Learning Curves & Validation Curves

### What They Are
- **Learning Curve:** Shows how performance changes as you **add more training data**
- **Validation Curve:** Shows how performance changes as you **tune a hyperparameter**

These are diagnostic tools that tell you **why** your model is failing.

### Learning Curve — Do I Need More Data?

```
┌──────────────────────────────────────────────────────┐
│              LEARNING CURVE DIAGNOSIS                  │
├──────────────────────────────────────────────────────┤
│                                                        │
│  HIGH BIAS (Underfitting):     HIGH VARIANCE (Overfit):│
│                                                        │
│  Score                         Score                   │
│  1.0                           1.0 ___Train            │
│      ___Train                      \                   │
│  0.8 ──────────                0.8  \___               │
│      ___Test                   0.6       ___Test       │
│  0.6 ──────────                0.4                     │
│                                                        │
│  Train & Test converge LOW     Train high, Test low    │
│  → Model too simple            → Model too complex     │
│  → More data won't help!       → More data WILL help!  │
│  → Use more complex model      → Or simplify model     │
│                                                        │
└──────────────────────────────────────────────────────┘
```

```python
from sklearn.model_selection import learning_curve

X, y = load_digits(return_X_y=True)

# Compute learning curve
train_sizes, train_scores, test_scores = learning_curve(
    estimator=SVC(kernel='rbf', random_state=42),
    X=X, y=y,
    train_sizes=np.linspace(0.1, 1.0, 10),  # 10% to 100% of training data
    cv=5,
    scoring='accuracy',
    n_jobs=-1
)

# Plot
plt.figure(figsize=(10, 6))
plt.plot(train_sizes, train_scores.mean(axis=1), 'o-', color='blue', label='Train')
plt.fill_between(train_sizes, 
                 train_scores.mean(axis=1) - train_scores.std(axis=1),
                 train_scores.mean(axis=1) + train_scores.std(axis=1), 
                 alpha=0.1, color='blue')
plt.plot(train_sizes, test_scores.mean(axis=1), 'o-', color='red', label='Validation')
plt.fill_between(train_sizes, 
                 test_scores.mean(axis=1) - test_scores.std(axis=1),
                 test_scores.mean(axis=1) + test_scores.std(axis=1), 
                 alpha=0.1, color='red')
plt.xlabel('Training Set Size')
plt.ylabel('Accuracy')
plt.title('Learning Curve — SVM on Digits')
plt.legend(loc='lower right')
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

### Validation Curve — How Does a Hyperparameter Affect Performance?

```python
from sklearn.model_selection import validation_curve

# How does max_depth affect Random Forest performance?
param_range = [1, 2, 3, 5, 7, 10, 15, 20, 30, None]

train_scores, test_scores = validation_curve(
    estimator=RandomForestClassifier(n_estimators=100, random_state=42),
    X=X, y=y,
    param_name='max_depth',       # Hyperparameter to vary
    param_range=param_range,      # Values to try
    cv=5,
    scoring='accuracy',
    n_jobs=-1
)

# Convert None to a large number for plotting
param_labels = [str(p) if p is not None else '∞' for p in param_range]
x_positions = range(len(param_range))

plt.figure(figsize=(10, 6))
plt.plot(x_positions, train_scores.mean(axis=1), 'o-', color='blue', label='Train')
plt.plot(x_positions, test_scores.mean(axis=1), 'o-', color='red', label='Validation')
plt.fill_between(x_positions, 
                 test_scores.mean(axis=1) - test_scores.std(axis=1),
                 test_scores.mean(axis=1) + test_scores.std(axis=1), 
                 alpha=0.1, color='red')
plt.xticks(x_positions, param_labels)
plt.xlabel('max_depth')
plt.ylabel('Accuracy')
plt.title('Validation Curve — Effect of max_depth')
plt.legend()
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

---

## Common Mistakes

### 1. Data Leakage — The Silent Killer

```python
# BAD: Fitting scaler on ALL data, then splitting
from sklearn.preprocessing import StandardScaler
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)     # LEAKS test info into training!
X_train, X_test, y_train, y_test = train_test_split(X_scaled, y)

# GOOD: Fit scaler on training data ONLY
X_train, X_test, y_train, y_test = train_test_split(X, y)
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)  # fit on train
X_test_scaled = scaler.transform(X_test)         # transform test (no fit!)

# BEST: Use a Pipeline (handles this automatically)
from sklearn.pipeline import Pipeline
pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('model', LogisticRegression())
])
# Pipeline ensures scaler fits ONLY on each CV training fold
cross_val_score(pipe, X, y, cv=5)
```

### 2. Evaluating on Training Data

```python
# BAD: "My model has 99.8% accuracy!" (on training data)
model.fit(X_train, y_train)
print(model.score(X_train, y_train))  # Meaningless! Tests memorization, not learning

# GOOD: Evaluate on unseen test data
print(model.score(X_test, y_test))
```

### 3. Using Accuracy for Imbalanced Data

```python
# BAD: 99% accuracy on fraud detection (but catches 0% fraud)
print(f"Accuracy: {accuracy_score(y_test, y_pred):.4f}")

# GOOD: Use appropriate metrics
print(f"F1 Score: {f1_score(y_test, y_pred):.4f}")
print(f"Precision: {precision_score(y_test, y_pred):.4f}")
print(f"Recall: {recall_score(y_test, y_pred):.4f}")
print(f"AUC: {roc_auc_score(y_test, y_proba):.4f}")
```

### 4. Tuning on Test Set (Overfitting to Test)

```python
# BAD: Repeatedly testing and tuning based on test results
for params in many_parameter_combos:
    model = build_model(params)
    model.fit(X_train, y_train)
    score = model.score(X_test, y_test)  # Using test set to select params!
    # This overfits to the test set

# GOOD: Use cross-validation on training set for tuning
grid_search = GridSearchCV(model, param_grid, cv=5)
grid_search.fit(X_train, y_train)       # Tunes using CV on training data
final_score = grid_search.score(X_test, y_test)  # Test set used ONCE at the end
```

### 5. Not Using Stratified Splits for Classification

```python
# BAD: Random split on imbalanced data
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
# Test set might have 0 samples of the minority class!

# GOOD: Stratified split preserves class proportions
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, stratify=y)
```

### 6. Ignoring Standard Deviation in CV Scores

```python
# BAD: "Model A is better because it has 0.92 accuracy"
# Model A: 0.92 ± 0.08  (high variance — unstable!)
# Model B: 0.90 ± 0.01  (low variance — reliable!)

# GOOD: Always report mean ± std
scores = cross_val_score(model, X, y, cv=5)
print(f"Accuracy: {scores.mean():.4f} ± {scores.std():.4f}")
# Choose the model with good score AND low variance
```

---

## Interview Questions

### Conceptual

**Q1: What is data leakage and how do you prevent it?**
> Data leakage occurs when information from the test/validation set "leaks" into training, giving overly optimistic results. Common sources: (1) scaling before splitting, (2) feature engineering using future data, (3) duplicate records across train/test. Prevention: always split first, use Pipelines, use TimeSeriesSplit for temporal data.

**Q2: When would you use F1 score instead of accuracy?**
> When classes are imbalanced. If 99% of emails are not spam, a model that predicts "not spam" for everything gets 99% accuracy but 0% recall on spam. F1 score balances precision and recall, giving a more meaningful metric for the minority class.

**Q3: What's the difference between precision and recall? Give a real-world example where each matters more.**
> Precision = "of predicted positives, how many are correct?" Recall = "of actual positives, how many did we catch?" Precision matters in spam filters (don't put real emails in spam). Recall matters in cancer screening (don't miss any cancer cases).

**Q4: Why is cross-validation better than a single train-test split?**
> A single split depends on which samples end up in train vs test — it's a noisy estimate. Cross-validation uses every sample for both training and testing, giving a more reliable and stable performance estimate. It also reveals variance in performance across different data subsets.

**Q5: Explain the bias-variance trade-off using learning curves.**
> High bias (underfitting): both training and validation scores are low and converge — the model is too simple, more data won't help. High variance (overfitting): training score is high but validation score is much lower — the model memorizes training data, more data or regularization helps. The goal is the sweet spot where both are reasonably high.

**Q6: What's the difference between GridSearchCV and RandomizedSearchCV?**
> GridSearchCV exhaustively tries every parameter combination — guaranteed to find the best in the grid but exponentially slow. RandomizedSearchCV samples random combinations — much faster, surprisingly effective (often finds 95% of optimal with 20% of the compute). Use Random first to narrow the search, then Grid to fine-tune.

**Q7: What does AUC = 0.5 mean? AUC = 1.0?**
> AUC=0.5 means the model is no better than random guessing — its positive and negative predictions are equally likely to be correct. AUC=1.0 means perfect classification — there exists a threshold that separates all positives from all negatives. In practice, AUC > 0.8 is good, > 0.9 is excellent.

**Q8: How would you handle evaluation for a time series prediction model?**
> Never use standard K-Fold cross-validation — it leaks future data into training. Use `TimeSeriesSplit` which ensures training data always comes before test data chronologically. Also use walk-forward validation where you train on expanding windows and test on the next period.

### Coding

**Q9: Write a complete model selection pipeline with preprocessing, hyperparameter tuning, and evaluation.**
```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import GridSearchCV, train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import classification_report

# Split
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42)

# Pipeline with preprocessing + model
pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('clf', RandomForestClassifier(random_state=42))
])

# Hyperparameter grid (use __ to access pipeline step params)
param_grid = {
    'clf__n_estimators': [100, 200],
    'clf__max_depth': [5, 10, None]
}

# Grid search with CV
grid = GridSearchCV(pipe, param_grid, cv=5, scoring='f1_weighted', n_jobs=-1)
grid.fit(X_train, y_train)

# Final evaluation (ONCE!)
print(f"Best params: {grid.best_params_}")
print(f"Best CV score: {grid.best_score_:.4f}")
print(f"Test score: {grid.score(X_test, y_test):.4f}")
print(classification_report(y_test, grid.predict(X_test)))
```

---

## Quick Reference

### All Scoring Parameters for `scoring=`

| Task | Metric | `scoring=` Value |
|------|--------|-----------------|
| Classification | Accuracy | `'accuracy'` |
| | Precision | `'precision'` / `'precision_weighted'` |
| | Recall | `'recall'` / `'recall_weighted'` |
| | F1 | `'f1'` / `'f1_weighted'` / `'f1_macro'` |
| | AUC | `'roc_auc'` / `'roc_auc_ovr'` |
| | Log Loss | `'neg_log_loss'` |
| Regression | R² | `'r2'` |
| | MSE | `'neg_mean_squared_error'` |
| | MAE | `'neg_mean_absolute_error'` |
| | RMSE | `'neg_root_mean_squared_error'` |

> **Note:** Regression metrics are **negated** (prefixed with `neg_`) because sklearn maximizes scores. Higher is always better in sklearn's scoring.

### Cross-Validation Strategy Decision Table

| Situation | Use | Why |
|-----------|-----|-----|
| Standard classification | `StratifiedKFold(n_splits=5)` | Preserves class balance |
| Standard regression | `KFold(n_splits=5, shuffle=True)` | Simple, effective |
| Time series | `TimeSeriesSplit(n_splits=5)` | No future leakage |
| Grouped data (e.g., patients) | `GroupKFold` | No group leakage |
| Very small dataset | `LeaveOneOut()` or `RepeatedKFold` | Maximize data usage |
| Most robust estimate | `RepeatedStratifiedKFold(n_splits=5, n_repeats=3)` | Reduces variance |

### Import Cheat Sheet

```python
# Splitting
from sklearn.model_selection import train_test_split

# Cross-Validation
from sklearn.model_selection import (
    cross_val_score, cross_validate,
    KFold, StratifiedKFold, RepeatedStratifiedKFold,
    LeaveOneOut, TimeSeriesSplit, GroupKFold
)

# Hyperparameter Tuning
from sklearn.model_selection import GridSearchCV, RandomizedSearchCV

# Classification Metrics
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    classification_report, confusion_matrix, ConfusionMatrixDisplay,
    roc_curve, roc_auc_score, RocCurveDisplay,
    precision_recall_curve, average_precision_score, PrecisionRecallDisplay
)

# Regression Metrics
from sklearn.metrics import (
    mean_absolute_error, mean_squared_error, r2_score,
    mean_absolute_percentage_error
)

# Diagnostic Curves
from sklearn.model_selection import learning_curve, validation_curve
```

### Model Evaluation Workflow Cheat Sheet

```
1. Split data          →  train_test_split(stratify=y)
2. Build pipeline      →  Pipeline([('scaler', ...), ('model', ...)])
3. Tune params         →  GridSearchCV / RandomizedSearchCV (on train only!)
4. Check overfitting   →  Compare train vs CV scores, plot learning curves
5. Final evaluation    →  score(X_test, y_test) — ONCE!
6. Report metrics      →  classification_report / confusion_matrix
7. Deploy              →  Retrain on ALL data with best params
```

---

> **Key Takeaway:** Model evaluation is arguably **more important** than model building. A poorly evaluated good model can fail catastrophically in production. Always: (1) use cross-validation, not single splits, (2) match your metric to your business problem, (3) watch for data leakage, (4) use Pipelines to automate safe preprocessing.
