# Chapter 07: Ensemble Methods — Complete Guide

## Table of Contents
- [What Are Ensemble Methods?](#what-are-ensemble-methods)
- [Why Ensemble Methods Matter](#why-ensemble-methods-matter)
- [Bagging (Bootstrap Aggregating)](#bagging-bootstrap-aggregating)
- [Boosting](#boosting)
- [AdaBoost](#adaboost)
- [Gradient Boosting](#gradient-boosting)
- [XGBoost](#xgboost)
- [LightGBM](#lightgbm)
- [CatBoost](#catboost)
- [Stacking (Stacked Generalization)](#stacking)
- [Voting Ensembles](#voting-ensembles)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What Are Ensemble Methods?

### Simple Explanation (Like Explaining to a 15-Year-Old)

Imagine you're on a game show, and you can either ask **one genius** for the answer, or you can ask **100 average people** and go with the majority vote. Surprisingly, the 100 average people often beat the single genius! That's the core idea behind ensemble methods.

**Ensemble methods combine multiple "weak" models to create one "strong" model.** Instead of relying on a single decision tree (which might overfit or be biased), you build many trees and combine their predictions.

> **Key Insight:** A group of mediocre models working together almost always outperforms a single excellent model. This is called the "Wisdom of Crowds" in ML.

### Formal Definition

An ensemble method is a machine learning technique that combines multiple base learners (models) to produce a single predictive model that is more robust and accurate than any individual model.

$$\hat{y}_{ensemble} = f(h_1(x), h_2(x), ..., h_n(x))$$

Where $h_i(x)$ are individual base learners and $f$ is the combining function.

---

## Why Ensemble Methods Matter

### Real-World Relevance

| Application | Why Ensembles Win |
|---|---|
| Kaggle Competitions | ~90% of winning solutions use ensembles (especially XGBoost/LightGBM) |
| Fraud Detection | Reduces false positives while catching more fraud |
| Medical Diagnosis | Multiple model "opinions" give more reliable predictions |
| Credit Scoring | Banks use GBM variants for loan approval |
| Recommendation Systems | Netflix Prize was won with ensemble of 800+ models |
| Production ML Systems | Most enterprise ML uses gradient boosting for tabular data |

### When to Use Ensembles

- **Use when:** You need maximum predictive accuracy on tabular/structured data
- **Use when:** Single models are overfitting or underfitting
- **Use when:** You can afford slightly more computation for better results
- **Avoid when:** Interpretability is the #1 requirement (use single decision tree)
- **Avoid when:** Real-time inference with <1ms latency is critical
- **Avoid when:** Training data is very small (<100 samples)

---

## The Three Pillars of Ensembles

```
┌─────────────────────────────────────────────────────────────┐
│                    ENSEMBLE METHODS                          │
├────────────────┬──────────────────┬─────────────────────────┤
│    BAGGING     │     BOOSTING     │       STACKING          │
│                │                  │                         │
│ • Parallel     │ • Sequential     │ • Hierarchical          │
│ • Reduce       │ • Reduce Bias    │ • Combine diverse       │
│   Variance     │                  │   models                │
│ • Random       │ • AdaBoost       │ • Meta-learner on       │
│   Forest       │ • Gradient Boost │   top of base models    │
│                │ • XGBoost        │                         │
│                │ • LightGBM       │                         │
│                │ • CatBoost       │                         │
└────────────────┴──────────────────┴─────────────────────────┘
```

---

## Bagging (Bootstrap Aggregating)

### What It Is

Bagging = **B**ootstrap **Agg**regat**ing**. It creates multiple versions of the training data (via bootstrap sampling), trains a model on each version, and combines their predictions.

### How It Works — Step by Step

```
Original Dataset: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

Bootstrap Sample 1: [2, 5, 5, 8, 1, 3, 9, 2, 7, 4]  → Model 1
Bootstrap Sample 2: [1, 3, 3, 6, 7, 2, 10, 4, 8, 8] → Model 2
Bootstrap Sample 3: [4, 7, 1, 9, 2, 5, 6, 3, 3, 10] → Model 3
...
Bootstrap Sample N: [...]                              → Model N

Final Prediction:
  Classification → Majority Vote
  Regression     → Average of all model predictions
```

### The Math Behind Bagging

Each bootstrap sample has approximately 63.2% of unique samples:

$$P(\text{sample not picked in one draw}) = \left(1 - \frac{1}{n}\right)^n \approx e^{-1} \approx 0.368$$

So ~36.8% of data is left out (called **Out-of-Bag** or OOB samples) — free validation data!

### Why Bagging Reduces Variance

If individual models have variance $\sigma^2$ and correlation $\rho$:

$$Var_{ensemble} = \rho \cdot \sigma^2 + \frac{1 - \rho}{n} \cdot \sigma^2$$

When models are uncorrelated ($\rho \approx 0$), variance drops as $\frac{\sigma^2}{n}$.

### Code Example — Bagging Classifier

```python
import numpy as np
from sklearn.ensemble import BaggingClassifier
from sklearn.tree import DecisionTreeClassifier
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, classification_report

# Create synthetic dataset
X, y = make_classification(
    n_samples=1000,
    n_features=20,
    n_informative=15,      # 15 features are actually useful
    n_redundant=3,         # 3 features are linear combos of informative ones
    random_state=42
)

# Split data
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Single Decision Tree (for comparison)
single_tree = DecisionTreeClassifier(random_state=42)
single_tree.fit(X_train, y_train)
single_pred = single_tree.predict(X_test)
print(f"Single Tree Accuracy: {accuracy_score(y_test, single_pred):.4f}")
# Output: Single Tree Accuracy: 0.8750

# Bagging with 100 trees
bagging_clf = BaggingClassifier(
    estimator=DecisionTreeClassifier(),  # Base learner
    n_estimators=100,         # Number of models
    max_samples=0.8,          # 80% of data per bootstrap sample
    max_features=0.8,         # 80% of features per model (adds diversity)
    bootstrap=True,           # Use bootstrap sampling
    oob_score=True,           # Compute out-of-bag score
    random_state=42,
    n_jobs=-1                 # Use all CPU cores
)

bagging_clf.fit(X_train, y_train)
bagging_pred = bagging_clf.predict(X_test)
print(f"Bagging Accuracy: {accuracy_score(y_test, bagging_pred):.4f}")
# Output: Bagging Accuracy: 0.9300

print(f"OOB Score: {bagging_clf.oob_score_:.4f}")
# Output: OOB Score: 0.9225
```

### Random Forest — Bagging on Steroids

Random Forest = Bagging + Random Feature Selection at each split.

```python
from sklearn.ensemble import RandomForestClassifier
import matplotlib.pyplot as plt

# Random Forest with tuned hyperparameters
rf = RandomForestClassifier(
    n_estimators=200,         # Number of trees
    max_depth=15,             # Limit tree depth to prevent overfitting
    min_samples_split=5,      # Minimum samples to split a node
    min_samples_leaf=2,       # Minimum samples in a leaf
    max_features='sqrt',      # sqrt(n_features) considered at each split
    bootstrap=True,           # Bootstrap sampling
    oob_score=True,           # Out-of-bag evaluation
    class_weight='balanced',  # Handle imbalanced classes
    random_state=42,
    n_jobs=-1                 # Parallel processing
)

rf.fit(X_train, y_train)
rf_pred = rf.predict(X_test)
print(f"Random Forest Accuracy: {accuracy_score(y_test, rf_pred):.4f}")
# Output: Random Forest Accuracy: 0.9450

# Feature Importance — one of RF's best features!
importances = rf.feature_importances_
indices = np.argsort(importances)[::-1]

print("\nTop 10 Feature Importances:")
for i in range(10):
    print(f"  Feature {indices[i]}: {importances[indices[i]]:.4f}")

# Pro Tip: Use permutation importance for more reliable feature ranking
from sklearn.inspection import permutation_importance

perm_importance = permutation_importance(
    rf, X_test, y_test, n_repeats=10, random_state=42
)
print("\nPermutation Importance (Top 5):")
sorted_idx = perm_importance.importances_mean.argsort()[::-1]
for i in range(5):
    idx = sorted_idx[i]
    print(f"  Feature {idx}: {perm_importance.importances_mean[idx]:.4f} "
          f"± {perm_importance.importances_std[idx]:.4f}")
```

> **Pro Tip:** `max_features='sqrt'` for classification and `max_features=1.0` (or `n_features/3`) for regression is a strong default.

---

## Boosting

### What It Is

Boosting trains models **sequentially**, where each new model focuses on the mistakes of the previous ones. It's like a student who keeps practicing the problems they got wrong.

### Key Difference: Bagging vs. Boosting

```
BAGGING (Parallel):                    BOOSTING (Sequential):
                                       
 Data → Model 1 ─┐                    Data → Model 1
 Data → Model 2 ─┼→ Combine              ↓ (fix errors)
 Data → Model 3 ─┤                    Weighted Data → Model 2
 Data → Model N ─┘                       ↓ (fix errors)
                                       Weighted Data → Model 3
 All models are INDEPENDENT               ↓
 Goal: Reduce VARIANCE                 Final = Weighted Sum of all
                                       Models are DEPENDENT
                                       Goal: Reduce BIAS
```

| Aspect | Bagging | Boosting |
|--------|---------|----------|
| Training | Parallel | Sequential |
| Focus | Reduce variance | Reduce bias |
| Data sampling | Bootstrap (random) | Weighted (focus on errors) |
| Model weight | Equal | Based on performance |
| Overfitting risk | Low | Higher (need regularization) |
| Speed | Faster (parallelizable) | Slower (sequential) |
| Example | Random Forest | XGBoost, LightGBM |

---

## AdaBoost

### What It Is

**Ada**ptive **Boost**ing — the original boosting algorithm. It assigns higher weights to misclassified samples so the next model pays more attention to them.

### How It Works — Intuition

```
Round 1: Equal weights for all samples
         ┌─────────────────────────────────────┐
         │ ● ● ● ● ● ○ ○ ○ ○ ○               │ (all weight = 1/N)
         └─────────────────────────────────────┘
         Model 1 misclassifies: ●₃, ○₇, ○₈

Round 2: Increase weight of misclassified samples  
         ┌─────────────────────────────────────┐
         │ ● ● ●̲ ● ● ○ ○ ○̲ ○̲ ○               │ (●₃, ○₇, ○₈ get bigger)
         └─────────────────────────────────────┘
         Model 2 focuses on those hard cases

Round 3: Again increase weights of still-misclassified
         ...continue until T rounds

Final: Weighted vote of all T models
       (models with lower error get higher vote weight)
```

### The Math

1. Initialize weights: $w_i = \frac{1}{N}$ for all samples

2. For each round $t = 1, ..., T$:
   - Train weak learner $h_t$ on weighted data
   - Compute weighted error: $\epsilon_t = \sum_{i: h_t(x_i) \neq y_i} w_i$
   - Compute model weight: $\alpha_t = \frac{1}{2} \ln\left(\frac{1 - \epsilon_t}{\epsilon_t}\right)$
   - Update sample weights: $w_i \leftarrow w_i \cdot e^{-\alpha_t \cdot y_i \cdot h_t(x_i)}$
   - Normalize weights: $w_i \leftarrow \frac{w_i}{\sum_j w_j}$

3. Final prediction: $H(x) = \text{sign}\left(\sum_{t=1}^{T} \alpha_t \cdot h_t(x)\right)$

### Code Example — AdaBoost

```python
from sklearn.ensemble import AdaBoostClassifier
from sklearn.tree import DecisionTreeClassifier
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.metrics import accuracy_score
import numpy as np

# Generate data
X, y = make_classification(
    n_samples=2000, n_features=20, n_informative=10,
    n_clusters_per_class=2, random_state=42
)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# AdaBoost with decision stumps (depth=1 trees)
ada_clf = AdaBoostClassifier(
    estimator=DecisionTreeClassifier(max_depth=1),  # Weak learner = stump
    n_estimators=200,       # Number of boosting rounds
    learning_rate=0.1,      # Shrinkage — smaller = more rounds needed but better
    algorithm='SAMME',      # SAMME.R for real-valued predictions (usually better)
    random_state=42
)

ada_clf.fit(X_train, y_train)
ada_pred = ada_clf.predict(X_test)
print(f"AdaBoost Accuracy: {accuracy_score(y_test, ada_pred):.4f}")
# Output: AdaBoost Accuracy: 0.9125

# Learning rate vs n_estimators tradeoff
# Lower learning rate + more estimators = better but slower
for lr, n_est in [(1.0, 50), (0.1, 200), (0.01, 1000)]:
    ada = AdaBoostClassifier(
        estimator=DecisionTreeClassifier(max_depth=1),
        n_estimators=n_est, learning_rate=lr, random_state=42
    )
    scores = cross_val_score(ada, X_train, y_train, cv=5)
    print(f"  LR={lr:.2f}, N={n_est:4d} → CV Accuracy: {scores.mean():.4f} ± {scores.std():.4f}")

# Staged predictions — see how performance changes with more estimators
train_errors = []
test_errors = []
for y_pred_train, y_pred_test in zip(
    ada_clf.staged_predict(X_train),
    ada_clf.staged_predict(X_test)
):
    train_errors.append(1 - accuracy_score(y_train, y_pred_train))
    test_errors.append(1 - accuracy_score(y_test, y_pred_test))

print(f"\nError after 10 rounds:  Train={train_errors[9]:.4f}, Test={test_errors[9]:.4f}")
print(f"Error after 100 rounds: Train={train_errors[99]:.4f}, Test={test_errors[99]:.4f}")
print(f"Error after 200 rounds: Train={train_errors[199]:.4f}, Test={test_errors[199]:.4f}")
```

> **Important:** AdaBoost is sensitive to noise and outliers. If your data is noisy, prefer Gradient Boosting with regularization.

---

## Gradient Boosting

### What It Is

Instead of re-weighting samples (like AdaBoost), Gradient Boosting fits each new model to the **residual errors** (gradients) of the previous ensemble. It literally does gradient descent in function space.

### How It Works — The Intuition

Think of it like a painter:
1. First painter makes a rough sketch (captures the main pattern)
2. Second painter adds details the first missed
3. Third painter fixes even finer details
4. After 100 painters, you have a masterpiece

```
True Values:    [10, 20, 30, 40, 50]
Model 1 Pred:   [ 8, 22, 25, 38, 55]
Residuals 1:    [ 2, -2,  5,  2, -5]  ← Model 2 learns THESE

Model 2 Pred:   [1.5, -1, 4, 1.5, -4] (on residuals, with learning_rate=0.8)
Ensemble Pred:  [9.2, 21.2, 28.2, 39.2, 51.8]
Residuals 2:    [0.8, -1.2, 1.8, 0.8, -1.8]  ← Model 3 learns THESE

... continues until residuals are tiny
```

### The Math

For a loss function $L(y, F(x))$:

1. Initialize: $F_0(x) = \arg\min_\gamma \sum_{i=1}^{n} L(y_i, \gamma)$

2. For $m = 1, ..., M$:
   - Compute pseudo-residuals: $r_{im} = -\left[\frac{\partial L(y_i, F(x_i))}{\partial F(x_i)}\right]_{F=F_{m-1}}$
   - Fit a tree $h_m(x)$ to residuals $r_{im}$
   - Compute optimal step: $\gamma_m = \arg\min_\gamma \sum_{i=1}^{n} L(y_i, F_{m-1}(x_i) + \gamma h_m(x_i))$
   - Update: $F_m(x) = F_{m-1}(x) + \nu \cdot \gamma_m \cdot h_m(x)$

Where $\nu$ is the learning rate (shrinkage).

### Code Example — Gradient Boosting

```python
from sklearn.ensemble import GradientBoostingClassifier, GradientBoostingRegressor
from sklearn.datasets import make_classification, make_regression
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.metrics import accuracy_score, mean_squared_error
import numpy as np

# ============= CLASSIFICATION =============
X, y = make_classification(
    n_samples=2000, n_features=20, n_informative=12,
    random_state=42
)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

gb_clf = GradientBoostingClassifier(
    n_estimators=200,         # Number of boosting rounds
    learning_rate=0.1,        # Shrinkage — trades off with n_estimators
    max_depth=3,              # Depth of each tree (keep small! 3-8)
    min_samples_split=10,     # Regularization
    min_samples_leaf=5,       # Regularization
    subsample=0.8,            # Stochastic GB — use 80% of data per round
    max_features='sqrt',      # Random feature subset (like RF)
    validation_fraction=0.1,  # Hold out 10% for early stopping
    n_iter_no_change=10,      # Stop if no improvement for 10 rounds
    random_state=42
)

gb_clf.fit(X_train, y_train)
gb_pred = gb_clf.predict(X_test)
print(f"Gradient Boosting Accuracy: {accuracy_score(y_test, gb_pred):.4f}")
# Output: Gradient Boosting Accuracy: 0.9425
print(f"Number of estimators used: {gb_clf.n_estimators_}")  # May be < 200 due to early stopping

# ============= REGRESSION =============
X_reg, y_reg = make_regression(
    n_samples=1000, n_features=10, noise=10, random_state=42
)
X_train_r, X_test_r, y_train_r, y_test_r = train_test_split(
    X_reg, y_reg, test_size=0.2, random_state=42
)

gb_reg = GradientBoostingRegressor(
    n_estimators=300,
    learning_rate=0.05,
    max_depth=4,
    subsample=0.8,
    loss='squared_error',     # Options: 'squared_error', 'absolute_error', 'huber'
    random_state=42
)

gb_reg.fit(X_train_r, y_train_r)
y_pred_r = gb_reg.predict(X_test_r)
rmse = np.sqrt(mean_squared_error(y_test_r, y_pred_r))
print(f"\nGradient Boosting RMSE: {rmse:.4f}")

# Feature importance
print("\nTop 5 Features:")
for idx in np.argsort(gb_reg.feature_importances_)[::-1][:5]:
    print(f"  Feature {idx}: {gb_reg.feature_importances_[idx]:.4f}")
```

> **Pro Tip:** The golden rule of Gradient Boosting: **low learning rate + more trees = better generalization**. Start with `learning_rate=0.1` and `n_estimators=100-500`. Use early stopping to find the right number.

---

## XGBoost

### What It Is

**eXtreme Gradient Boosting** — an optimized, industrial-strength implementation of gradient boosting. Created by Tianqi Chen in 2014, it became the dominant algorithm for structured data competitions.

### Why XGBoost is Special

| Feature | Regular GB | XGBoost |
|---------|-----------|---------|
| Regularization | No built-in | L1 + L2 on leaf weights |
| Missing values | Manual handling | Built-in handling |
| Parallelization | Sequential only | Parallel feature sorting |
| Tree pruning | Post-pruning | Depth-first with max_depth |
| Cache optimization | No | Yes (block structure) |
| Out-of-core | No | Supports disk-based computation |
| Sparsity awareness | No | Yes (sparse-aware algorithm) |

### The XGBoost Objective

$$\text{Obj} = \sum_{i=1}^{n} L(y_i, \hat{y}_i) + \sum_{k=1}^{K} \Omega(f_k)$$

Where the regularization term is:

$$\Omega(f) = \gamma T + \frac{1}{2}\lambda \sum_{j=1}^{T} w_j^2$$

- $T$ = number of leaves in the tree
- $w_j$ = leaf weight (prediction value)
- $\gamma$ = penalty for number of leaves (pruning)
- $\lambda$ = L2 regularization on leaf weights

Using second-order Taylor expansion of the loss:

$$\text{Obj}^{(t)} \approx \sum_{i=1}^{n} \left[g_i f_t(x_i) + \frac{1}{2} h_i f_t^2(x_i)\right] + \Omega(f_t)$$

Where $g_i = \frac{\partial L}{\partial \hat{y}^{(t-1)}}$ (gradient) and $h_i = \frac{\partial^2 L}{\partial (\hat{y}^{(t-1)})^2}$ (hessian).

### Code Example — XGBoost

```python
import xgboost as xgb
import numpy as np
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.metrics import accuracy_score, classification_report
import time

# Generate data
X, y = make_classification(
    n_samples=5000, n_features=30, n_informative=20,
    n_classes=2, random_state=42
)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# ============= METHOD 1: Scikit-learn API =============
xgb_clf = xgb.XGBClassifier(
    n_estimators=300,          # Number of boosting rounds
    max_depth=6,               # Maximum tree depth
    learning_rate=0.1,         # Step size shrinkage (eta)
    subsample=0.8,             # Row sampling per tree
    colsample_bytree=0.8,     # Column sampling per tree
    colsample_bylevel=0.8,    # Column sampling per level
    min_child_weight=5,        # Min sum of instance weight in a child
    gamma=0.1,                 # Min loss reduction for split (pruning)
    reg_alpha=0.1,             # L1 regularization
    reg_lambda=1.0,            # L2 regularization
    scale_pos_weight=1,        # For imbalanced classes: sum(neg)/sum(pos)
    objective='binary:logistic',
    eval_metric='logloss',
    tree_method='hist',        # Fast histogram-based method
    random_state=42,
    n_jobs=-1,
    early_stopping_rounds=20   # Stop if no improvement
)

# Train with evaluation set for early stopping
start = time.time()
xgb_clf.fit(
    X_train, y_train,
    eval_set=[(X_test, y_test)],
    verbose=False
)
train_time = time.time() - start

xgb_pred = xgb_clf.predict(X_test)
print(f"XGBoost Accuracy: {accuracy_score(y_test, xgb_pred):.4f}")
print(f"Training Time: {train_time:.2f}s")
print(f"Best iteration: {xgb_clf.best_iteration}")
# Output: XGBoost Accuracy: 0.9560
# Output: Training Time: 0.85s

# ============= METHOD 2: Native API (more control) =============
dtrain = xgb.DMatrix(X_train, label=y_train)
dtest = xgb.DMatrix(X_test, label=y_test)

params = {
    'max_depth': 6,
    'eta': 0.1,               # learning_rate
    'objective': 'binary:logistic',
    'eval_metric': 'auc',
    'subsample': 0.8,
    'colsample_bytree': 0.8,
    'min_child_weight': 5,
    'gamma': 0.1,
    'lambda': 1.0,            # reg_lambda
    'alpha': 0.1,             # reg_alpha
    'tree_method': 'hist',
    'seed': 42
}

# Train with watchlist
watchlist = [(dtrain, 'train'), (dtest, 'eval')]
bst = xgb.train(
    params,
    dtrain,
    num_boost_round=500,
    evals=watchlist,
    early_stopping_rounds=20,
    verbose_eval=50  # Print every 50 rounds
)

# Predict (native API returns probabilities)
y_prob = bst.predict(dtest)
y_pred_native = (y_prob > 0.5).astype(int)
print(f"\nNative API Accuracy: {accuracy_score(y_test, y_pred_native):.4f}")

# Feature Importance (multiple types!)
importance_types = ['weight', 'gain', 'cover']
for imp_type in importance_types:
    importance = bst.get_score(importance_type=imp_type)
    top_features = sorted(importance.items(), key=lambda x: x[1], reverse=True)[:3]
    print(f"\n{imp_type.upper()} importance (top 3): {top_features}")
```

### XGBoost Hyperparameter Tuning Guide

```python
from sklearn.model_selection import RandomizedSearchCV
from scipy.stats import uniform, randint

# Define parameter distributions for random search
param_distributions = {
    'n_estimators': randint(100, 1000),
    'max_depth': randint(3, 10),
    'learning_rate': uniform(0.01, 0.3),
    'subsample': uniform(0.6, 0.4),
    'colsample_bytree': uniform(0.6, 0.4),
    'min_child_weight': randint(1, 10),
    'gamma': uniform(0, 0.5),
    'reg_alpha': uniform(0, 1),
    'reg_lambda': uniform(0.5, 2),
}

xgb_search = RandomizedSearchCV(
    xgb.XGBClassifier(
        objective='binary:logistic',
        tree_method='hist',
        random_state=42,
        n_jobs=-1,
        early_stopping_rounds=15
    ),
    param_distributions=param_distributions,
    n_iter=50,              # Try 50 random combinations
    scoring='roc_auc',
    cv=5,
    random_state=42,
    verbose=1,
    n_jobs=-1
)

# Note: For early stopping with RandomizedSearchCV, pass fit_params
# xgb_search.fit(X_train, y_train, eval_set=[(X_test, y_test)], verbose=False)

print(f"Best params: {xgb_search.best_params_}")
print(f"Best AUC: {xgb_search.best_score_:.4f}")
```

> **Pro Tip:** XGBoost tuning priority order: (1) `n_estimators` + `learning_rate`, (2) `max_depth` + `min_child_weight`, (3) `subsample` + `colsample_bytree`, (4) `gamma` + `reg_alpha` + `reg_lambda`.

---

## LightGBM

### What It Is

**Light Gradient Boosting Machine** by Microsoft. It's 10-20x faster than XGBoost on large datasets while achieving comparable or better accuracy. The secret? It grows trees **leaf-wise** instead of level-wise.

### Key Innovations

```
XGBoost (Level-wise growth):          LightGBM (Leaf-wise growth):

         [root]                                [root]
        /      \                              /      \
      [L1]    [L1]                          [L1]    [L1]
     /   \   /   \                         /   \      
   [L2] [L2][L2][L2]                    [L2] [L2]   ← Only splits the leaf
                                        /   \           with max loss reduction
                                      [L3] [L3]    

Level-wise: Balanced but wasteful      Leaf-wise: Faster convergence
(splits nodes that don't help much)    (but can overfit → use max_depth!)
```

| Innovation | How It Helps |
|---|---|
| **GOSS** (Gradient-based One-Side Sampling) | Keeps all samples with large gradients, randomly samples from small gradients |
| **EFB** (Exclusive Feature Bundling) | Bundles mutually exclusive features → reduces feature count |
| **Leaf-wise growth** | Deeper trees with same number of leaves → better accuracy |
| **Histogram-based splitting** | Bins continuous features → O(#bins) instead of O(#data) |
| **Categorical feature support** | Native handling without one-hot encoding |

### Code Example — LightGBM

```python
import lightgbm as lgb
import numpy as np
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, roc_auc_score
import time

# Generate larger dataset to show speed advantage
X, y = make_classification(
    n_samples=50000, n_features=50, n_informative=30,
    random_state=42
)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# ============= Scikit-learn API =============
lgb_clf = lgb.LGBMClassifier(
    n_estimators=500,
    max_depth=-1,              # No limit (leaf-wise needs num_leaves instead)
    num_leaves=31,             # Max leaves per tree (key param for LightGBM!)
    learning_rate=0.05,
    subsample=0.8,             # bagging_fraction
    subsample_freq=5,          # Perform bagging every 5 iterations
    colsample_bytree=0.8,     # feature_fraction
    min_child_samples=20,      # min_data_in_leaf
    min_child_weight=0.001,
    reg_alpha=0.1,             # L1 regularization
    reg_lambda=0.1,            # L2 regularization
    cat_features='auto',       # Auto-detect categorical columns
    objective='binary',
    metric='auc',
    boosting_type='gbdt',      # Options: 'gbdt', 'dart', 'goss'
    random_state=42,
    n_jobs=-1,
    verbose=-1                 # Suppress output
)

start = time.time()
lgb_clf.fit(
    X_train, y_train,
    eval_set=[(X_test, y_test)],
    callbacks=[
        lgb.early_stopping(stopping_rounds=20),
        lgb.log_evaluation(period=0)  # Suppress logs
    ]
)
lgb_time = time.time() - start

lgb_pred = lgb_clf.predict(X_test)
lgb_prob = lgb_clf.predict_proba(X_test)[:, 1]
print(f"LightGBM Accuracy: {accuracy_score(y_test, lgb_pred):.4f}")
print(f"LightGBM AUC: {roc_auc_score(y_test, lgb_prob):.4f}")
print(f"Training Time: {lgb_time:.2f}s")
print(f"Best iteration: {lgb_clf.best_iteration_}")
# LightGBM is typically 5-10x faster than XGBoost on this size data

# ============= Native API (more features) =============
train_data = lgb.Dataset(X_train, label=y_train)
test_data = lgb.Dataset(X_test, label=y_test, reference=train_data)

params = {
    'objective': 'binary',
    'metric': 'auc',
    'boosting_type': 'gbdt',
    'num_leaves': 31,
    'learning_rate': 0.05,
    'feature_fraction': 0.8,
    'bagging_fraction': 0.8,
    'bagging_freq': 5,
    'min_data_in_leaf': 20,
    'lambda_l1': 0.1,
    'lambda_l2': 0.1,
    'verbose': -1,
    'seed': 42
}

# Train with callbacks
callbacks = [lgb.early_stopping(20), lgb.log_evaluation(50)]
bst = lgb.train(
    params,
    train_data,
    num_boost_round=1000,
    valid_sets=[test_data],
    callbacks=callbacks
)

# Feature importance
importance_df = lgb.plot_importance  # For plotting
print(f"\nTop 10 features by gain:")
importance = bst.feature_importance(importance_type='gain')
indices = np.argsort(importance)[::-1][:10]
for i, idx in enumerate(indices):
    print(f"  {i+1}. Feature_{idx}: {importance[idx]:.1f}")
```

### LightGBM with Categorical Features (Native Support)

```python
import pandas as pd
import lightgbm as lgb
from sklearn.model_selection import train_test_split

# Example with categorical features
np.random.seed(42)
n = 10000
df = pd.DataFrame({
    'age': np.random.randint(18, 70, n),
    'income': np.random.normal(50000, 20000, n),
    'city': np.random.choice(['NYC', 'LA', 'Chicago', 'Houston', 'Phoenix'], n),
    'education': np.random.choice(['High School', 'Bachelor', 'Master', 'PhD'], n),
    'employed': np.random.choice([0, 1], n, p=[0.3, 0.7])
})
df['target'] = ((df['income'] > 50000) & (df['employed'] == 1)).astype(int)

# Convert to category dtype — LightGBM handles this natively!
cat_cols = ['city', 'education']
for col in cat_cols:
    df[col] = df[col].astype('category')

X = df.drop('target', axis=1)
y = df['target']
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# LightGBM will automatically detect category columns
lgb_cat = lgb.LGBMClassifier(
    n_estimators=200,
    num_leaves=31,
    learning_rate=0.1,
    random_state=42,
    verbose=-1
)

lgb_cat.fit(
    X_train, y_train,
    categorical_feature=cat_cols,  # Explicitly specify (or let it auto-detect)
    eval_set=[(X_test, y_test)],
    callbacks=[lgb.early_stopping(10), lgb.log_evaluation(0)]
)

print(f"Accuracy with native categoricals: {lgb_cat.score(X_test, y_test):.4f}")
# No one-hot encoding needed! LightGBM uses optimal split finding for categories
```

> **Pro Tip:** `num_leaves` is the most important parameter in LightGBM. A good starting point: `num_leaves = 2^max_depth - 1`. For `max_depth=7`, use `num_leaves=127` or less.

---

## CatBoost

### What It Is

**Cat**egorical **Boost**ing by Yandex. Specifically designed to handle categorical features elegantly and requires minimal hyperparameter tuning. It's the "just works" gradient boosting library.

### Key Innovations

| Feature | How It Works |
|---|---|
| **Ordered Target Encoding** | Encodes categoricals using target stats, but with a time-based ordering to prevent target leakage |
| **Ordered Boosting** | Uses different permutations of data to prevent prediction shift |
| **Symmetric Trees** | Uses oblivious decision trees (same split at each level) → faster inference |
| **Built-in GPU support** | Seamless GPU training |
| **Minimal tuning needed** | Strong defaults out of the box |

### Code Example — CatBoost

```python
from catboost import CatBoostClassifier, Pool
import numpy as np
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, roc_auc_score
import time

# Create dataset with categorical features
np.random.seed(42)
n = 20000
df = pd.DataFrame({
    'age': np.random.randint(18, 70, n),
    'income': np.random.normal(50000, 20000, n),
    'city': np.random.choice(['NYC', 'LA', 'Chicago', 'Houston', 'Phoenix', 
                               'Dallas', 'Seattle', 'Denver'], n),
    'education': np.random.choice(['High School', 'Bachelor', 'Master', 'PhD'], n),
    'job_type': np.random.choice(['Tech', 'Finance', 'Healthcare', 'Education',
                                   'Retail', 'Manufacturing'], n),
    'credit_score': np.random.randint(300, 850, n),
    'years_employed': np.random.randint(0, 40, n)
})
# Create target based on some logic
df['approved'] = (
    (df['income'] > 45000) & 
    (df['credit_score'] > 600) & 
    (df['years_employed'] > 2)
).astype(int)

X = df.drop('approved', axis=1)
y = df['approved']

# Identify categorical columns
cat_features = ['city', 'education', 'job_type']

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# CatBoost — note: NO need to encode categoricals!
cat_clf = CatBoostClassifier(
    iterations=500,            # n_estimators
    depth=6,                   # max_depth (uses symmetric/oblivious trees)
    learning_rate=0.1,
    l2_leaf_reg=3,             # L2 regularization
    random_strength=1,         # Randomness for scoring splits
    bagging_temperature=1,     # Bayesian bootstrap (0=no bagging, higher=more random)
    border_count=254,          # Number of bins for numerical features
    cat_features=cat_features, # Specify categorical columns!
    auto_class_weights='Balanced',  # Handle imbalanced classes
    eval_metric='AUC',
    random_seed=42,
    verbose=0                  # Suppress output
)

# Train with eval set
start = time.time()
cat_clf.fit(
    X_train, y_train,
    eval_set=(X_test, y_test),
    early_stopping_rounds=30,
    plot=False
)
cat_time = time.time() - start

cat_pred = cat_clf.predict(X_test)
cat_prob = cat_clf.predict_proba(X_test)[:, 1]
print(f"CatBoost Accuracy: {accuracy_score(y_test, cat_pred):.4f}")
print(f"CatBoost AUC: {roc_auc_score(y_test, cat_prob):.4f}")
print(f"Training Time: {cat_time:.2f}s")
print(f"Best iteration: {cat_clf.get_best_iteration()}")

# Feature importance
feature_importance = cat_clf.get_feature_importance()
feature_names = X.columns
sorted_idx = np.argsort(feature_importance)[::-1]
print("\nFeature Importance:")
for i, idx in enumerate(sorted_idx):
    print(f"  {feature_names[idx]}: {feature_importance[idx]:.2f}")

# SHAP values (built into CatBoost!)
shap_values = cat_clf.get_feature_importance(type='ShapValues', data=Pool(X_test, y_test, cat_features=cat_features))
print(f"\nSHAP values shape: {shap_values.shape}")  # (n_samples, n_features + 1)
```

### CatBoost vs XGBoost vs LightGBM — When to Use What

```python
# Quick comparison on same data
from catboost import CatBoostClassifier
import xgboost as xgb
import lightgbm as lgb
from sklearn.preprocessing import LabelEncoder

# For XGBoost/LightGBM, need to encode categoricals
X_train_encoded = X_train.copy()
X_test_encoded = X_test.copy()
le_dict = {}
for col in cat_features:
    le = LabelEncoder()
    X_train_encoded[col] = le.fit_transform(X_train[col])
    X_test_encoded[col] = le.transform(X_test[col])
    le_dict[col] = le

results = {}

# XGBoost
xgb_model = xgb.XGBClassifier(n_estimators=300, max_depth=6, learning_rate=0.1,
                                random_state=42, tree_method='hist', verbosity=0)
start = time.time()
xgb_model.fit(X_train_encoded, y_train)
results['XGBoost'] = {
    'time': time.time() - start,
    'accuracy': accuracy_score(y_test, xgb_model.predict(X_test_encoded))
}

# LightGBM
lgb_model = lgb.LGBMClassifier(n_estimators=300, num_leaves=31, learning_rate=0.1,
                                 random_state=42, verbose=-1)
start = time.time()
lgb_model.fit(X_train_encoded, y_train)
results['LightGBM'] = {
    'time': time.time() - start,
    'accuracy': accuracy_score(y_test, lgb_model.predict(X_test_encoded))
}

# CatBoost (no encoding needed!)
cat_model = CatBoostClassifier(iterations=300, depth=6, learning_rate=0.1,
                                 cat_features=cat_features, random_seed=42, verbose=0)
start = time.time()
cat_model.fit(X_train, y_train)
results['CatBoost'] = {
    'time': time.time() - start,
    'accuracy': accuracy_score(y_test, cat_model.predict(X_test))
}

print("\n" + "="*50)
print(f"{'Model':<12} {'Accuracy':<12} {'Time (s)':<10}")
print("="*50)
for model, res in results.items():
    print(f"{model:<12} {res['accuracy']:<12.4f} {res['time']:<10.2f}")
```

---

## Stacking (Stacked Generalization) {#stacking}

### What It Is

Stacking uses predictions from multiple diverse models as **input features** to a "meta-learner" (usually a simple model like logistic regression). It learns the optimal way to combine different models.

### How It Works

```
Level 0 (Base Models):
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
│ Random     │  │ XGBoost    │  │ LightGBM   │  │ Neural     │
│ Forest     │  │            │  │            │  │ Network    │
└─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
      │                │                │                │
      ▼                ▼                ▼                ▼
   pred_rf          pred_xgb        pred_lgb         pred_nn
      │                │                │                │
      └────────────────┴────────────────┴────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
Level 1             │  Meta-Learner   │  (e.g., Logistic Regression)
(Meta-Model):       │  Input: [pred_rf, pred_xgb, pred_lgb, pred_nn]
                    └────────┬────────┘
                             │
                             ▼
                      Final Prediction
```

> **Critical:** Base model predictions used for training the meta-learner MUST come from **cross-validation** (out-of-fold predictions), NOT from training on the same data. Otherwise, you get data leakage!

### Code Example — Stacking

```python
from sklearn.ensemble import (
    StackingClassifier, RandomForestClassifier, 
    GradientBoostingClassifier
)
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
from sklearn.neural_network import MLPClassifier
from sklearn.model_selection import cross_val_score
from sklearn.datasets import make_classification
from sklearn.metrics import accuracy_score
import numpy as np

# Generate data
X, y = make_classification(
    n_samples=3000, n_features=20, n_informative=15,
    random_state=42
)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Define diverse base models (diversity is key!)
estimators = [
    ('rf', RandomForestClassifier(n_estimators=100, random_state=42)),
    ('gb', GradientBoostingClassifier(n_estimators=100, random_state=42)),
    ('svm', SVC(probability=True, kernel='rbf', random_state=42)),
    ('mlp', MLPClassifier(hidden_layer_sizes=(100, 50), max_iter=500, random_state=42))
]

# Stacking Classifier
stack_clf = StackingClassifier(
    estimators=estimators,
    final_estimator=LogisticRegression(max_iter=1000),  # Meta-learner
    cv=5,                     # 5-fold CV for generating meta-features
    stack_method='predict_proba',  # Use probabilities (better than hard predictions)
    passthrough=False,        # Set True to also pass original features to meta-learner
    n_jobs=-1
)

stack_clf.fit(X_train, y_train)
stack_pred = stack_clf.predict(X_test)
print(f"Stacking Accuracy: {accuracy_score(y_test, stack_pred):.4f}")

# Compare with individual models
print("\nIndividual model comparison:")
for name, model in estimators:
    model.fit(X_train, y_train)
    pred = model.predict(X_test)
    print(f"  {name}: {accuracy_score(y_test, pred):.4f}")

# Custom stacking with more control
from sklearn.model_selection import KFold

def custom_stacking(X_train, y_train, X_test, base_models, meta_model, n_folds=5):
    """Manual stacking implementation for full control."""
    kf = KFold(n_splits=n_folds, shuffle=True, random_state=42)
    
    # Generate out-of-fold predictions
    meta_train = np.zeros((X_train.shape[0], len(base_models)))
    meta_test = np.zeros((X_test.shape[0], len(base_models)))
    
    for i, (name, model) in enumerate(base_models):
        test_preds = np.zeros((X_test.shape[0], n_folds))
        
        for fold, (train_idx, val_idx) in enumerate(kf.split(X_train)):
            X_fold_train = X_train[train_idx]
            y_fold_train = y_train[train_idx]
            X_fold_val = X_train[val_idx]
            
            model_clone = model.__class__(**model.get_params())
            model_clone.fit(X_fold_train, y_fold_train)
            
            # Out-of-fold predictions for meta-training
            meta_train[val_idx, i] = model_clone.predict_proba(X_fold_val)[:, 1]
            # Test predictions (averaged across folds)
            test_preds[:, fold] = model_clone.predict_proba(X_test)[:, 1]
        
        meta_test[:, i] = test_preds.mean(axis=1)
        print(f"  Base model '{name}' done")
    
    # Train meta-learner on out-of-fold predictions
    meta_model.fit(meta_train, y_train)
    final_pred = meta_model.predict(meta_test)
    return final_pred

# Use custom stacking
final_pred = custom_stacking(X_train, y_train, X_test, estimators, 
                              LogisticRegression(max_iter=1000))
print(f"\nCustom Stacking Accuracy: {accuracy_score(y_test, final_pred):.4f}")
```

---

## Voting Ensembles

### What It Is

The simplest ensemble: train different models and combine their predictions by voting (classification) or averaging (regression).

### Code Example — Voting

```python
from sklearn.ensemble import VotingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.svm import SVC
from sklearn.metrics import accuracy_score

# Hard Voting (majority vote)
hard_voting = VotingClassifier(
    estimators=[
        ('lr', LogisticRegression(max_iter=1000)),
        ('rf', RandomForestClassifier(n_estimators=100, random_state=42)),
        ('gb', GradientBoostingClassifier(n_estimators=100, random_state=42))
    ],
    voting='hard'
)

# Soft Voting (average probabilities — usually better!)
soft_voting = VotingClassifier(
    estimators=[
        ('lr', LogisticRegression(max_iter=1000)),
        ('rf', RandomForestClassifier(n_estimators=100, random_state=42)),
        ('gb', GradientBoostingClassifier(n_estimators=100, random_state=42)),
        ('svm', SVC(probability=True, random_state=42))  # Need probability=True for soft
    ],
    voting='soft',
    weights=[1, 2, 3, 1]  # Give more weight to better models
)

hard_voting.fit(X_train, y_train)
soft_voting.fit(X_train, y_train)

print(f"Hard Voting: {accuracy_score(y_test, hard_voting.predict(X_test)):.4f}")
print(f"Soft Voting: {accuracy_score(y_test, soft_voting.predict(X_test)):.4f}")
```

---

## Common Mistakes

### 1. Using Too Many Estimators Without Early Stopping
```python
# ❌ BAD: Fixed number, might overfit
xgb_bad = xgb.XGBClassifier(n_estimators=5000, learning_rate=0.01)
xgb_bad.fit(X_train, y_train)

# ✅ GOOD: Use early stopping
xgb_good = xgb.XGBClassifier(n_estimators=5000, learning_rate=0.01, 
                               early_stopping_rounds=50)
xgb_good.fit(X_train, y_train, eval_set=[(X_val, y_val)], verbose=False)
```

### 2. Not Tuning `num_leaves` in LightGBM
```python
# ❌ BAD: Default num_leaves=31 with max_depth=15 → overfitting
lgb_bad = lgb.LGBMClassifier(max_depth=15, num_leaves=31)

# ✅ GOOD: num_leaves should be < 2^max_depth
lgb_good = lgb.LGBMClassifier(max_depth=7, num_leaves=80)  # 80 < 128 (2^7)
```

### 3. Ignoring Feature Scale for Non-Tree Models in Stacking
```python
# ❌ BAD: SVM/LogReg as base model without scaling
# ✅ GOOD: Use Pipeline to scale before non-tree models
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler

estimators_good = [
    ('rf', RandomForestClassifier()),  # Trees don't need scaling
    ('svm', Pipeline([('scaler', StandardScaler()), ('svm', SVC(probability=True))]))
]
```

### 4. Data Leakage in Stacking
```python
# ❌ BAD: Training meta-learner on same predictions used for base training
base_model.fit(X_train, y_train)
meta_features = base_model.predict(X_train)  # LEAKAGE!

# ✅ GOOD: Use cross-validated out-of-fold predictions
from sklearn.model_selection import cross_val_predict
meta_features = cross_val_predict(base_model, X_train, y_train, cv=5, method='predict_proba')
```

### 5. Not Handling Categorical Features Properly
```python
# ❌ BAD: One-hot encoding high-cardinality categoricals for CatBoost
# (CatBoost handles them natively and better)

# ✅ GOOD: Let CatBoost handle categoricals
cat_model = CatBoostClassifier(cat_features=['city', 'country', 'product_id'])
```

### 6. Overfitting with Small Datasets
```python
# ❌ BAD: Complex ensemble on 200 samples
# ✅ GOOD: Use simpler models or heavy regularization
xgb_small_data = xgb.XGBClassifier(
    n_estimators=50, max_depth=3, reg_lambda=10, reg_alpha=5,
    subsample=0.5, colsample_bytree=0.5
)
```

---

## Interview Questions

### Conceptual Questions

**Q1: What's the difference between Bagging and Boosting?**
> Bagging trains models in parallel on random subsets to reduce variance. Boosting trains sequentially, with each model correcting the errors of the previous one, reducing bias.

**Q2: Why does Random Forest use `max_features='sqrt'`?**
> To decorrelate trees. If all trees see all features, they'll all split on the same strong features, making them correlated. Using a random subset forces trees to learn different patterns, reducing the correlation term in the variance formula.

**Q3: How does XGBoost handle missing values?**
> XGBoost learns the optimal default direction for missing values during training. At each split, it tries sending missing values both left and right, and picks the direction that minimizes the loss.

**Q4: Why is LightGBM faster than XGBoost?**
> Three reasons: (1) Leaf-wise growth instead of level-wise → fewer splits, (2) GOSS keeps only high-gradient samples, (3) EFB reduces feature dimensionality. Also, histogram-based splitting.

**Q5: When would you choose CatBoost over XGBoost?**
> When you have many categorical features (especially high-cardinality), when you want good results with minimal tuning, or when you need ordered boosting to prevent target leakage.

**Q6: Can ensembles overfit? How do you prevent it?**
> Yes! Boosting ensembles can overfit if you use too many estimators with high learning rate. Prevention: early stopping, regularization (L1/L2), subsampling, limiting tree depth, and using a validation set.

**Q7: Explain the bias-variance tradeoff in ensembles.**
> Bagging reduces variance (many models averaging out noise) without affecting bias much. Boosting reduces bias (sequentially fitting residuals) but can increase variance if not regularized. The ideal ensemble balances both.

**Q8: What is "Ordered Boosting" in CatBoost?**
> Standard boosting has prediction shift — the model is trained and predicts on the same data. Ordered boosting uses random permutations to ensure each sample's prediction is based only on models trained without seeing that sample.

### Coding/Practical Questions

**Q9: How do you tune XGBoost hyperparameters efficiently?**
> Start with learning_rate=0.1, find good n_estimators with early stopping. Then tune max_depth+min_child_weight. Then subsample+colsample_bytree. Finally, regularization (gamma, alpha, lambda). Use Optuna/Bayesian optimization for efficiency.

**Q10: How do you handle a 100M row dataset with gradient boosting?**
> Use LightGBM with GOSS, subsample the data, use histogram binning (max_bin=255), enable GPU training, use the `save_binary` feature for faster loading, or use distributed training.

---

## Quick Reference

### Algorithm Selection Cheat Sheet

| Scenario | Best Choice | Why |
|----------|-------------|-----|
| Tabular data, need best accuracy | LightGBM/XGBoost | State-of-the-art for structured data |
| Many categorical features | CatBoost | Native categorical handling |
| Need fast training (>1M rows) | LightGBM | GOSS + EFB + leaf-wise |
| Need interpretability | Random Forest | Feature importance + simple to explain |
| Imbalanced classification | XGBoost (`scale_pos_weight`) | Good built-in handling |
| Minimal tuning needed | CatBoost | Strong defaults |
| Competition/Maximum accuracy | Stacking (LGB + XGB + Cat) | Combines diverse models |
| Small dataset (<1000 rows) | Random Forest with low `max_depth` | Resistant to overfitting |

### Hyperparameter Cheat Sheet

| Parameter | XGBoost | LightGBM | CatBoost |
|-----------|---------|----------|----------|
| # Trees | `n_estimators` | `n_estimators` | `iterations` |
| Learning rate | `learning_rate`/`eta` | `learning_rate` | `learning_rate` |
| Tree depth | `max_depth` | `max_depth` | `depth` |
| Leaf count | N/A | `num_leaves` (KEY!) | N/A |
| L1 reg | `reg_alpha` | `reg_alpha`/`lambda_l1` | N/A |
| L2 reg | `reg_lambda` | `reg_lambda`/`lambda_l2` | `l2_leaf_reg` |
| Row sampling | `subsample` | `bagging_fraction` | `subsample` |
| Col sampling | `colsample_bytree` | `feature_fraction` | `rsm` |
| Min samples leaf | `min_child_weight` | `min_data_in_leaf` | `min_data_in_leaf` |

### Default Starting Points

```python
# XGBoost defaults
xgb_defaults = {
    'n_estimators': 300, 'learning_rate': 0.1, 'max_depth': 6,
    'subsample': 0.8, 'colsample_bytree': 0.8, 'min_child_weight': 5,
    'gamma': 0, 'reg_lambda': 1, 'reg_alpha': 0
}

# LightGBM defaults  
lgb_defaults = {
    'n_estimators': 300, 'learning_rate': 0.1, 'num_leaves': 31,
    'subsample': 0.8, 'colsample_bytree': 0.8, 'min_child_samples': 20,
    'reg_alpha': 0.1, 'reg_lambda': 0.1
}

# CatBoost defaults
cat_defaults = {
    'iterations': 500, 'learning_rate': 0.1, 'depth': 6,
    'l2_leaf_reg': 3, 'bagging_temperature': 1
}
```

### Key Takeaways

- **Bagging** → Reduces variance → Random Forest
- **Boosting** → Reduces bias → XGBoost, LightGBM, CatBoost
- **Stacking** → Combines diverse models → Meta-learner on top
- Always use **early stopping** with boosting
- **LightGBM** for speed, **CatBoost** for categoricals, **XGBoost** for control
- **Diversity** is more important than individual model accuracy in ensembles
- For production: LightGBM > XGBoost > CatBoost (inference speed)

---

*Next Chapter: [08-Clustering-Algorithms.md](08-Clustering-Algorithms.md)*
