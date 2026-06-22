# Chapter 04: Ensemble Methods in Scikit-Learn

## Table of Contents
- [Introduction to Ensemble Methods](#introduction-to-ensemble-methods)
- [Bagging — Random Forest](#bagging--random-forest)
- [Boosting — AdaBoost & Gradient Boosting](#boosting--adaboost--gradient-boosting)
- [Voting Classifiers](#voting-classifiers)
- [Stacking (Stacked Generalization)](#stacking-stacked-generalization)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Introduction to Ensemble Methods

### What It Is
Imagine you're making a big decision — like buying a car. Would you trust one person's opinion or ask 10 different experts? Ensemble methods do exactly this: they combine multiple "weak" models to create one powerful "strong" model.

Instead of relying on a single decision tree (which might overfit or underfit), ensembles combine many models so their individual errors cancel out.

### Why It Matters
- **Better accuracy** — Ensembles almost always beat single models
- **Reduced overfitting** — Averaging multiple models smooths out noise
- **Robustness** — Less sensitive to outliers and noise in data
- **Winners of ML competitions** — Random Forest & Gradient Boosting dominate Kaggle
- **Production-ready** — Used at Netflix, Uber, Airbnb for recommendation & fraud detection

### The Three Big Ensemble Strategies

| Strategy | How It Works | Example |
|----------|-------------|---------|
| **Bagging** | Train models on random subsets, average results | Random Forest |
| **Boosting** | Train models sequentially, each fixing previous errors | AdaBoost, GradientBoosting |
| **Stacking** | Train models, then train a meta-model on their outputs | StackingClassifier |

### How Ensembles Reduce Error

```
Total Error = Bias² + Variance + Irreducible Noise

Bagging  → Reduces VARIANCE (overfitting)
Boosting → Reduces BIAS (underfitting)
Stacking → Reduces BOTH (when done right)
```

---

## Bagging — Random Forest

### What It Is
**Bagging** (Bootstrap AGGregatING) trains multiple models on random subsets of data, then takes a vote (classification) or average (regression).

**Random Forest** = Bagging + Decision Trees + Random Feature Selection

Think of it like asking 100 people to guess the number of jellybeans in a jar. Each person might be way off, but the average of all guesses is usually very close to the truth.

### Why It Matters
- Works out-of-the-box with minimal tuning
- Handles both classification and regression
- Resistant to overfitting (unlike single decision trees)
- Provides feature importance scores for free
- Handles missing values and mixed data types well

### How It Works

```
┌─────────────────────────────────────────────────────┐
│              RANDOM FOREST ALGORITHM                  │
├─────────────────────────────────────────────────────┤
│                                                       │
│   Original Dataset (N samples, M features)           │
│          │                                           │
│          ▼                                           │
│   ┌──────────────────────────────────┐               │
│   │  Bootstrap Sampling (with replacement)│           │
│   └──────────────────────────────────┘               │
│          │          │          │                     │
│          ▼          ▼          ▼                     │
│      Subset 1   Subset 2   Subset 3  ... Subset B   │
│      (~63% of   (~63% of   (~63% of                 │
│       data)      data)      data)                   │
│          │          │          │                     │
│          ▼          ▼          ▼                     │
│      Tree 1     Tree 2     Tree 3    ... Tree B     │
│   (random √M  (random √M  (random √M              │
│    features)   features)   features)               │
│          │          │          │                     │
│          ▼          ▼          ▼                     │
│      Pred 1     Pred 2     Pred 3    ... Pred B     │
│          │          │          │                     │
│          └──────────┼──────────┘                     │
│                     ▼                                │
│            Majority Vote (Classification)            │
│            or Average (Regression)                   │
│                     │                                │
│                     ▼                                │
│              Final Prediction                        │
└─────────────────────────────────────────────────────┘
```

**Key Insight:** Each tree only sees ~63.2% of the data (due to bootstrap sampling) and only √M features at each split. This **decorrelates** the trees, making the ensemble much better than any single tree.

### Mathematical Foundation

For a Random Forest with $B$ trees:

**Classification:**
$$\hat{y} = \text{mode}(h_1(x), h_2(x), \ldots, h_B(x))$$

**Regression:**
$$\hat{y} = \frac{1}{B} \sum_{b=1}^{B} h_b(x)$$

**Variance Reduction:**
If each tree has variance $\sigma^2$ and pairwise correlation $\rho$:
$$\text{Var}(\bar{h}) = \rho \sigma^2 + \frac{1 - \rho}{B} \sigma^2$$

As $B \to \infty$, the second term vanishes. Random feature selection reduces $\rho$, which reduces the first term.

### Code Examples

#### Basic Random Forest Classifier

```python
import numpy as np
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, classification_report

# Load data
X, y = load_iris(return_X_y=True)

# Split into train/test
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

# Create Random Forest with 100 trees
rf = RandomForestClassifier(
    n_estimators=100,       # Number of trees (more = better but slower)
    max_depth=None,         # No limit on tree depth (let trees grow fully)
    min_samples_split=2,    # Minimum samples to split a node
    min_samples_leaf=1,     # Minimum samples in a leaf
    max_features='sqrt',    # Use √(n_features) at each split (decorrelates trees)
    bootstrap=True,         # Use bootstrap sampling
    oob_score=True,         # Calculate out-of-bag score (free validation!)
    random_state=42,        # Reproducibility
    n_jobs=-1               # Use all CPU cores
)

# Train
rf.fit(X_train, y_train)

# Predict
y_pred = rf.predict(X_test)

# Evaluate
print(f"Accuracy: {accuracy_score(y_test, y_pred):.4f}")
print(f"OOB Score: {rf.oob_score_:.4f}")  # Free validation without a test set!
print("\nClassification Report:")
print(classification_report(y_test, y_pred))

# Output:
# Accuracy: 1.0000
# OOB Score: 0.9583
```

#### Feature Importance

```python
import pandas as pd
import matplotlib.pyplot as plt

# Get feature importances (based on impurity decrease)
importances = rf.feature_importances_
feature_names = load_iris().feature_names

# Create a sorted DataFrame
feat_imp_df = pd.DataFrame({
    'Feature': feature_names,
    'Importance': importances
}).sort_values('Importance', ascending=False)

print(feat_imp_df)
# Output:
#          Feature  Importance
# petal length (cm)    0.4427
# petal width (cm)     0.4225
# sepal length (cm)    0.1002
# sepal width (cm)     0.0346

# Plot feature importances
plt.figure(figsize=(8, 5))
plt.barh(feat_imp_df['Feature'], feat_imp_df['Importance'])
plt.xlabel('Importance (Mean Decrease in Impurity)')
plt.title('Random Forest Feature Importance')
plt.tight_layout()
plt.show()
```

#### Random Forest Regressor

```python
from sklearn.datasets import make_regression
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_squared_error, r2_score

# Generate synthetic regression data
X, y = make_regression(n_samples=1000, n_features=10, noise=10, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Create regressor
rf_reg = RandomForestRegressor(
    n_estimators=200,       # More trees for regression (smoother predictions)
    max_depth=20,           # Limit depth to prevent overfitting
    min_samples_leaf=5,     # Require at least 5 samples per leaf
    max_features=0.33,      # Use 1/3 of features (common for regression)
    random_state=42,
    n_jobs=-1
)

rf_reg.fit(X_train, y_train)
y_pred = rf_reg.predict(X_test)

print(f"RMSE: {np.sqrt(mean_squared_error(y_test, y_pred)):.4f}")
print(f"R² Score: {r2_score(y_test, y_pred):.4f}")

# Output:
# RMSE: 15.2341
# R² Score: 0.9847
```

#### Pro Tip: Permutation Importance (More Reliable Than Default)

```python
from sklearn.inspection import permutation_importance

# Permutation importance — shuffles each feature and measures accuracy drop
# More reliable than impurity-based importance for high-cardinality features
perm_importance = permutation_importance(
    rf, X_test, y_test, 
    n_repeats=30,           # Repeat shuffling 30 times for stability
    random_state=42,
    n_jobs=-1
)

# Sort by importance
sorted_idx = perm_importance.importances_mean.argsort()[::-1]
for idx in sorted_idx:
    print(f"{feature_names[idx]:>20s}: "
          f"{perm_importance.importances_mean[idx]:.4f} "
          f"± {perm_importance.importances_std[idx]:.4f}")
```

### Key Hyperparameters

| Parameter | Default | Effect | Tuning Guide |
|-----------|---------|--------|--------------|
| `n_estimators` | 100 | More trees = better but diminishing returns | Start 100, go to 500+ for complex data |
| `max_depth` | None | Controls tree complexity | None (let trees grow) or limit to 10-30 |
| `max_features` | 'sqrt' | Features per split | 'sqrt' for classification, 0.33 for regression |
| `min_samples_split` | 2 | Min samples to split | 2-10; higher = less overfitting |
| `min_samples_leaf` | 1 | Min samples in leaf | 1-10; higher = smoother predictions |
| `bootstrap` | True | Use sampling with replacement | Almost always True |

> **Pro Tip:** Random Forests rarely overfit with more trees. The main cost is computation time. If in doubt, add more trees.

---

## Boosting — AdaBoost & Gradient Boosting

### What It Is
If Bagging is "wisdom of crowds" (many independent opinions), Boosting is "learning from mistakes" (each model focuses on what the previous one got wrong).

Think of it like studying for an exam:
- **Attempt 1:** You take a practice test and get some questions wrong
- **Attempt 2:** You study ONLY the topics you got wrong, take another test
- **Attempt 3:** You focus on what you STILL get wrong
- **Result:** After many rounds, you've mastered everything

### Why It Matters
- Often achieves **highest accuracy** among all methods
- GradientBoosting → XGBoost → LightGBM → CatBoost (evolution of boosting)
- Dominates structured/tabular data competitions
- Used in production at Google (search ranking), Facebook (ad click prediction)

### How Boosting Works

```
┌─────────────────────────────────────────────────────────┐
│                   BOOSTING PROCESS                        │
├─────────────────────────────────────────────────────────┤
│                                                           │
│   Round 1: All samples have EQUAL weight                 │
│   ┌──────────────┐                                       │
│   │   Model 1    │ → Predictions → Identify ERRORS       │
│   └──────────────┘                                       │
│          │                                               │
│          ▼  Increase weight on MISCLASSIFIED samples     │
│                                                           │
│   Round 2: Wrong samples now have HIGHER weight          │
│   ┌──────────────┐                                       │
│   │   Model 2    │ → Focuses on hard cases               │
│   └──────────────┘                                       │
│          │                                               │
│          ▼  Again increase weight on remaining errors    │
│                                                           │
│   Round 3: Model focuses on HARDEST cases                │
│   ┌──────────────┐                                       │
│   │   Model 3    │ → Handles edge cases                  │
│   └──────────────┘                                       │
│          │                                               │
│          ▼                                               │
│                                                           │
│   Final: Weighted combination of all models              │
│   F(x) = α₁·h₁(x) + α₂·h₂(x) + α₃·h₃(x) + ...      │
│   (better models get higher α weights)                   │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### AdaBoost (Adaptive Boosting)

#### Mathematical Foundation

For iteration $t$:

1. **Calculate error** of model $h_t$:
$$\epsilon_t = \frac{\sum_{i=1}^{N} w_i \cdot \mathbb{1}[y_i \neq h_t(x_i)]}{\sum_{i=1}^{N} w_i}$$

2. **Calculate model weight** (better models get more say):
$$\alpha_t = \frac{1}{2} \ln\left(\frac{1 - \epsilon_t}{\epsilon_t}\right)$$

3. **Update sample weights** (misclassified get more weight):
$$w_i^{(t+1)} = w_i^{(t)} \cdot e^{-\alpha_t \cdot y_i \cdot h_t(x_i)}$$

4. **Final prediction:**
$$H(x) = \text{sign}\left(\sum_{t=1}^{T} \alpha_t \cdot h_t(x)\right)$$

#### AdaBoost Code Example

```python
from sklearn.ensemble import AdaBoostClassifier
from sklearn.tree import DecisionTreeClassifier
from sklearn.datasets import make_classification

# Generate a harder dataset
X, y = make_classification(
    n_samples=1000, n_features=20, n_informative=10,
    n_redundant=5, random_state=42
)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# AdaBoost with Decision Stumps (depth=1 trees)
ada = AdaBoostClassifier(
    estimator=DecisionTreeClassifier(max_depth=1),  # Weak learner = decision stump
    n_estimators=200,       # Number of boosting rounds
    learning_rate=0.1,      # Shrinks contribution of each model (lower = more robust)
    algorithm='SAMME.R',    # Use probability estimates (better than SAMME)
    random_state=42
)

ada.fit(X_train, y_train)
print(f"AdaBoost Accuracy: {ada.score(X_test, y_test):.4f}")

# Output:
# AdaBoost Accuracy: 0.8900

# Track how error decreases with more estimators
from sklearn.metrics import accuracy_score

staged_scores = [accuracy_score(y_test, pred) 
                 for pred in ada.staged_predict(X_test)]
print(f"Best accuracy at {np.argmax(staged_scores)+1} estimators: {max(staged_scores):.4f}")
```

### Gradient Boosting

#### How It Differs from AdaBoost
- AdaBoost: Re-weights **samples** based on errors
- Gradient Boosting: Fits new model to **residual errors** (gradient of loss)

#### Mathematical Foundation

At each step $m$:

1. **Compute residuals** (negative gradient of loss function):
$$r_{im} = -\frac{\partial L(y_i, F_{m-1}(x_i))}{\partial F_{m-1}(x_i)}$$

2. **Fit a tree** $h_m(x)$ to residuals $r_{im}$

3. **Update model:**
$$F_m(x) = F_{m-1}(x) + \eta \cdot h_m(x)$$

Where $\eta$ is the learning rate (shrinkage).

For MSE loss: residuals = $y_i - F_{m-1}(x_i)$ (literally the errors!)

```
Intuition:
                                                
  y = truth    F₀ = mean    r₁ = y - F₀     h₁ fits r₁
  [10]         [7]          [3]              [2.8]
  [5]          [7]          [-2]             [-1.9]
  [8]          [7]          [1]              [0.9]
  
  F₁ = F₀ + η·h₁ = 7 + 0.1·[2.8, -1.9, 0.9] = [7.28, 6.81, 7.09]
  
  New residuals are SMALLER. Repeat until residuals ≈ 0.
```

#### Gradient Boosting Code Example

```python
from sklearn.ensemble import GradientBoostingClassifier, GradientBoostingRegressor

# Classification
gb_clf = GradientBoostingClassifier(
    n_estimators=200,        # Number of boosting stages
    learning_rate=0.1,       # Shrinks each tree's contribution (trade-off with n_estimators)
    max_depth=3,             # Shallow trees! (typically 3-8 for boosting)
    min_samples_split=10,    # Prevents overfitting
    min_samples_leaf=5,      # Minimum samples per leaf
    subsample=0.8,           # Stochastic GB: use 80% of data per tree (reduces variance)
    max_features='sqrt',     # Random feature subsampling
    random_state=42
)

gb_clf.fit(X_train, y_train)
print(f"Gradient Boosting Accuracy: {gb_clf.score(X_test, y_test):.4f}")

# Output:
# Gradient Boosting Accuracy: 0.9200
```

#### Gradient Boosting Regressor

```python
from sklearn.datasets import fetch_california_housing

# Load real-world dataset
housing = fetch_california_housing()
X_train, X_test, y_train, y_test = train_test_split(
    housing.data, housing.target, test_size=0.2, random_state=42
)

gb_reg = GradientBoostingRegressor(
    n_estimators=300,
    learning_rate=0.05,      # Lower learning rate = more trees needed but better
    max_depth=5,
    subsample=0.8,
    min_samples_leaf=10,
    loss='squared_error',    # Also: 'absolute_error', 'huber', 'quantile'
    random_state=42
)

gb_reg.fit(X_train, y_train)
y_pred = gb_reg.predict(X_test)

print(f"RMSE: {np.sqrt(mean_squared_error(y_test, y_pred)):.4f}")
print(f"R² Score: {r2_score(y_test, y_pred):.4f}")

# Output:
# RMSE: 0.5321
# R² Score: 0.7892
```

#### HistGradientBoosting (Faster, Better)

```python
from sklearn.ensemble import HistGradientBoostingClassifier

# HistGradientBoosting — inspired by LightGBM
# 10-100x faster than GradientBoosting for large datasets
# Handles missing values natively!
hgb = HistGradientBoostingClassifier(
    max_iter=200,            # Number of boosting iterations
    learning_rate=0.1,
    max_depth=None,          # Uses max_leaf_nodes instead
    max_leaf_nodes=31,       # Controls tree complexity
    min_samples_leaf=20,
    l2_regularization=0.1,   # L2 penalty (prevents overfitting)
    early_stopping=True,     # Stop if validation score stops improving
    validation_fraction=0.1, # Use 10% of training data for early stopping
    n_iter_no_change=10,     # Stop after 10 rounds of no improvement
    random_state=42
)

hgb.fit(X_train, y_train)
print(f"HistGB Accuracy: {hgb.score(X_test, y_test):.4f}")

# Output:
# HistGB Accuracy: 0.9250
```

> **Pro Tip:** For datasets with >10,000 samples, ALWAYS use `HistGradientBoostingClassifier` over `GradientBoostingClassifier`. It's dramatically faster due to histogram-based splits.

### Comparison: AdaBoost vs Gradient Boosting

| Aspect | AdaBoost | Gradient Boosting |
|--------|----------|-------------------|
| Strategy | Re-weight samples | Fit to residuals |
| Base learner | Typically stumps | Shallow trees (depth 3-8) |
| Loss function | Exponential | Any differentiable loss |
| Speed | Moderate | Slower (but HistGB is fast) |
| Robustness to noise | Sensitive (upweights noisy samples) | More robust |
| When to use | Simple datasets, few features | Complex datasets, most use cases |

---

## Voting Classifiers

### What It Is
A Voting Classifier combines predictions from multiple **different** models. Instead of 100 trees (all the same algorithm), you might combine a Random Forest + SVM + Logistic Regression.

Think of it like a panel of judges with different expertise — a music expert, a dance expert, and an acting expert all voting on a talent show.

### Why It Matters
- Different models capture different patterns in data
- Reduces risk of one model's weakness dominating
- Easy to implement and explain
- Often boosts accuracy by 1-3% over best single model

### Types of Voting

| Type | How It Works | When to Use |
|------|-------------|-------------|
| **Hard Voting** | Majority vote on predicted class | When models don't output probabilities |
| **Soft Voting** | Average predicted probabilities, pick highest | Generally better (uses more information) |

```
Example with 3 models predicting class for one sample:

Hard Voting:
  Model A → Class 1    ┐
  Model B → Class 0    ├→ Majority = Class 1 (2 vs 1)
  Model C → Class 1    ┘

Soft Voting:
  Model A → [0.3, 0.7]   (70% confidence for Class 1)
  Model B → [0.6, 0.4]   (60% confidence for Class 0)
  Model C → [0.2, 0.8]   (80% confidence for Class 1)
  Average → [0.37, 0.63] → Class 1 (63% avg confidence)
```

### Code Example

```python
from sklearn.ensemble import VotingClassifier, RandomForestClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
from sklearn.neighbors import KNeighborsClassifier
from sklearn.datasets import make_classification

# Create dataset
X, y = make_classification(n_samples=1000, n_features=20, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Define diverse base models
models = [
    ('rf', RandomForestClassifier(n_estimators=100, random_state=42)),
    ('lr', LogisticRegression(max_iter=1000, random_state=42)),
    ('svm', SVC(probability=True, random_state=42)),  # probability=True needed for soft voting
    ('knn', KNeighborsClassifier(n_neighbors=7))
]

# Hard Voting
hard_voter = VotingClassifier(estimators=models, voting='hard')
hard_voter.fit(X_train, y_train)
print(f"Hard Voting Accuracy: {hard_voter.score(X_test, y_test):.4f}")

# Soft Voting (generally better)
soft_voter = VotingClassifier(estimators=models, voting='soft')
soft_voter.fit(X_train, y_train)
print(f"Soft Voting Accuracy: {soft_voter.score(X_test, y_test):.4f}")

# Compare with individual models
for name, model in models:
    model.fit(X_train, y_train)
    print(f"  {name:>5s} Accuracy: {model.score(X_test, y_test):.4f}")

# Output:
# Hard Voting Accuracy: 0.9150
# Soft Voting Accuracy: 0.9250
#    rf Accuracy: 0.9050
#    lr Accuracy: 0.8800
#   svm Accuracy: 0.9100
#   knn Accuracy: 0.8750
```

#### Weighted Voting

```python
# Give more weight to better models
weighted_voter = VotingClassifier(
    estimators=models,
    voting='soft',
    weights=[3, 1, 2, 1]  # RF gets 3x weight, SVM gets 2x
)
weighted_voter.fit(X_train, y_train)
print(f"Weighted Voting Accuracy: {weighted_voter.score(X_test, y_test):.4f}")
```

> **Pro Tip:** For voting to work well, models should be **diverse** (different algorithms) and **uncorrelated** in their errors. Combining 5 random forests won't help much!

---

## Stacking (Stacked Generalization)

### What It Is
Stacking takes voting to the next level: instead of a simple vote/average, it trains a **meta-model** that learns the best way to combine base model predictions.

Analogy: Instead of judges just voting, you have a "head judge" who has learned which judges are good at what. The head judge knows Judge A is great at classical music but bad at rock, so it weights accordingly.

### Why It Matters
- Most powerful ensemble technique
- Can learn complex non-linear combinations of models
- Winners of many Kaggle competitions use stacking
- Sklearn makes it easy with `StackingClassifier`

### How It Works

```
┌──────────────────────────────────────────────────────────┐
│                    STACKING ARCHITECTURE                   │
├──────────────────────────────────────────────────────────┤
│                                                            │
│   LAYER 0 (Base Models):                                  │
│   ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                │
│   │  RF  │  │  SVM │  │  KNN │  │  LR  │                │
│   └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘                │
│      │         │         │         │                      │
│      ▼         ▼         ▼         ▼                      │
│   [0.8,0.2] [0.7,0.3] [0.9,0.1] [0.6,0.4]              │
│      │         │         │         │                      │
│      └─────────┴─────────┴─────────┘                      │
│                     │                                      │
│                     ▼                                      │
│   LAYER 1 (Meta-Model):                                  │
│   ┌──────────────────────────────────┐                    │
│   │  Meta-Learner (e.g., LogReg)     │                    │
│   │  Input: predictions from Layer 0 │                    │
│   │  Learns optimal combination      │                    │
│   └──────────────┬───────────────────┘                    │
│                  │                                         │
│                  ▼                                         │
│           Final Prediction                                │
│                                                            │
└──────────────────────────────────────────────────────────┘

KEY: Base models are trained with CROSS-VALIDATION to avoid data leakage!
```

### Code Example

```python
from sklearn.ensemble import StackingClassifier, RandomForestClassifier
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
from sklearn.neighbors import KNeighborsClassifier

# Define base models (Level 0)
base_models = [
    ('rf', RandomForestClassifier(n_estimators=100, random_state=42)),
    ('gb', GradientBoostingClassifier(n_estimators=100, random_state=42)),
    ('svm', SVC(probability=True, random_state=42)),
    ('knn', KNeighborsClassifier(n_neighbors=7))
]

# Define meta-learner (Level 1)
# LogisticRegression is a popular choice — simple, prevents overfitting
meta_learner = LogisticRegression(max_iter=1000, random_state=42)

# Create Stacking Classifier
stacker = StackingClassifier(
    estimators=base_models,
    final_estimator=meta_learner,
    cv=5,                    # 5-fold CV to generate meta-features (prevents leakage!)
    stack_method='auto',     # Uses predict_proba if available, else decision_function
    passthrough=False,       # If True, also passes original features to meta-learner
    n_jobs=-1
)

stacker.fit(X_train, y_train)
print(f"Stacking Accuracy: {stacker.score(X_test, y_test):.4f}")

# Output:
# Stacking Accuracy: 0.9350
```

#### Advanced Stacking: Multi-Layer

```python
# Two-level stacking with passthrough
stacker_advanced = StackingClassifier(
    estimators=[
        ('rf', RandomForestClassifier(n_estimators=200, random_state=42)),
        ('gb', GradientBoostingClassifier(n_estimators=200, learning_rate=0.05, random_state=42)),
        ('svm', SVC(probability=True, kernel='rbf', random_state=42)),
    ],
    final_estimator=LogisticRegression(C=0.1, max_iter=1000),  # Regularized meta-learner
    cv=5,
    passthrough=True,   # Include original features + model predictions as meta-features
    n_jobs=-1
)

stacker_advanced.fit(X_train, y_train)
print(f"Advanced Stacking Accuracy: {stacker_advanced.score(X_test, y_test):.4f}")
```

> **Warning:** Stacking is prone to overfitting if:
> - You use too complex a meta-learner (stick to LogisticRegression or simple models)
> - You don't use CV for generating meta-features (sklearn handles this automatically)
> - Your base models are too similar

---

## BaggingClassifier (Generic Bagging)

### What It Is
While Random Forest is bagging + trees, `BaggingClassifier` lets you bag ANY model.

```python
from sklearn.ensemble import BaggingClassifier
from sklearn.svm import SVC

# Bag 10 SVMs (each trained on different data subsets)
bagged_svm = BaggingClassifier(
    estimator=SVC(),
    n_estimators=10,
    max_samples=0.8,         # Each model sees 80% of data
    max_features=0.8,        # Each model sees 80% of features
    bootstrap=True,          # Sample with replacement
    bootstrap_features=False,# Don't bootstrap features (sample without replacement)
    oob_score=True,
    random_state=42,
    n_jobs=-1
)

bagged_svm.fit(X_train, y_train)
print(f"Bagged SVM Accuracy: {bagged_svm.score(X_test, y_test):.4f}")
print(f"OOB Score: {bagged_svm.oob_score_:.4f}")
```

---

## Common Mistakes

### 1. Using Too Many Trees in Production
```python
# BAD: 10000 trees — slow inference, barely better than 500
rf = RandomForestClassifier(n_estimators=10000)

# GOOD: Find the sweet spot with diminishing returns
# Plot OOB error vs n_estimators to find where it plateaus
```

### 2. Not Tuning Learning Rate + N_Estimators Together
```python
# BAD: High learning rate with many estimators (overfitting)
gb = GradientBoostingClassifier(n_estimators=1000, learning_rate=1.0)

# GOOD: Lower learning rate needs more estimators
# Rule of thumb: lr=0.1 → 100-500 trees, lr=0.01 → 1000-5000 trees
gb = GradientBoostingClassifier(n_estimators=500, learning_rate=0.05)
```

### 3. Deep Trees in Gradient Boosting
```python
# BAD: Deep trees in boosting (overfits quickly)
gb = GradientBoostingClassifier(max_depth=20)

# GOOD: Shallow trees (3-8 depth) — each tree is a "weak learner"
gb = GradientBoostingClassifier(max_depth=4)
```

### 4. Not Using Early Stopping
```python
# BAD: Fixed number of iterations (might overfit or underfit)
gb = GradientBoostingClassifier(n_estimators=1000)

# GOOD: Use HistGradientBoosting with early stopping
from sklearn.ensemble import HistGradientBoostingClassifier
gb = HistGradientBoostingClassifier(
    max_iter=1000,
    early_stopping=True,
    n_iter_no_change=15,
    validation_fraction=0.15
)
```

### 5. Ignoring Feature Importance Bias
```python
# BAD: Trusting impurity-based importance blindly
# High-cardinality features (like IDs) get artificially high importance

# GOOD: Always use permutation importance for final feature selection
from sklearn.inspection import permutation_importance
perm_imp = permutation_importance(model, X_test, y_test, n_repeats=30)
```

---

## Interview Questions

### Conceptual

**Q1: What's the difference between Bagging and Boosting?**
> Bagging trains models independently on random subsets and averages them (reduces variance). Boosting trains models sequentially, each correcting the previous model's errors (reduces bias). Bagging → parallel, Boosting → sequential.

**Q2: Why does Random Forest use `max_features='sqrt'`?**
> To decorrelate trees. If all trees see all features, they'll all split on the same strong features and be highly correlated. Limiting features forces trees to explore different patterns, reducing the correlation term in the variance formula.

**Q3: Can a Random Forest overfit? How do you prevent it?**
> Yes, but it's harder than single trees. Overfitting happens with very deep trees on small datasets. Prevention: increase `min_samples_leaf`, reduce `max_depth`, use more trees (doesn't increase overfitting), reduce `max_features`.

**Q4: What is the Out-of-Bag (OOB) score?**
> Each tree in a Random Forest is trained on ~63.2% of data (bootstrap sample). The remaining ~36.8% (out-of-bag samples) can be used as a free validation set. OOB score ≈ cross-validation score without extra computation.

**Q5: Why are shallow trees used in Gradient Boosting?**
> Each tree is a "weak learner" that captures one small pattern. Deep trees would overfit to training data immediately. Boosting combines many weak patterns to build a strong model — it's the combination that gives power, not individual tree strength.

**Q6: What is the learning rate in boosting and why not set it to 1?**
> Learning rate shrinks each tree's contribution: $F_m = F_{m-1} + \eta \cdot h_m$. With $\eta=1$, each tree has full impact and the model overfits quickly. Smaller $\eta$ (0.01-0.1) means each tree contributes less, requiring more trees but generalizing better (regularization through shrinkage).

**Q7: When would you choose Stacking over simple Voting?**
> Stacking when: (1) you have diverse models with complementary strengths, (2) you have enough data to train a meta-learner without overfitting, (3) a few percent accuracy matters (competitions). Voting when: (1) simplicity matters, (2) limited data, (3) you want easy interpretability.

### Coding

**Q8: How would you find the optimal number of trees for a Random Forest?**
```python
from sklearn.model_selection import cross_val_score

scores = []
tree_range = range(50, 501, 50)
for n in tree_range:
    rf = RandomForestClassifier(n_estimators=n, random_state=42, n_jobs=-1)
    score = cross_val_score(rf, X_train, y_train, cv=5, scoring='accuracy').mean()
    scores.append(score)
    
optimal_n = tree_range[np.argmax(scores)]
print(f"Optimal n_estimators: {optimal_n}")
```

---

## Quick Reference

### When to Use What

| Scenario | Best Ensemble | Why |
|----------|--------------|-----|
| Quick baseline, no tuning | Random Forest | Works well out of the box |
| Maximum accuracy on tabular data | HistGradientBoosting | Best bias-variance trade-off |
| Small dataset (<1000 samples) | Random Forest | Less prone to overfitting |
| Large dataset (>100K samples) | HistGradientBoosting | Fast + accurate |
| Combining different model types | Stacking | Learns optimal combination |
| Need interpretable ensemble | Random Forest + Feature Importance | Built-in importance |
| Noisy data with outliers | Random Forest | Averaging reduces noise impact |

### Hyperparameter Cheat Sheet

| Model | Key Parameters | Quick Tuning |
|-------|---------------|--------------|
| RandomForest | `n_estimators`, `max_depth`, `max_features` | 200 trees, None depth, sqrt features |
| GradientBoosting | `n_estimators`, `learning_rate`, `max_depth` | 200 trees, 0.1 lr, depth 4 |
| HistGradientBoosting | `max_iter`, `learning_rate`, `max_leaf_nodes` | 300 iter, 0.1 lr, 31 leaves |
| AdaBoost | `n_estimators`, `learning_rate` | 200 estimators, 0.1 lr |
| VotingClassifier | `voting`, `weights` | 'soft', equal weights first |
| StackingClassifier | `cv`, `final_estimator` | cv=5, LogisticRegression |

### Import Quick Reference

```python
from sklearn.ensemble import (
    RandomForestClassifier,
    RandomForestRegressor,
    GradientBoostingClassifier,
    GradientBoostingRegressor,
    HistGradientBoostingClassifier,
    HistGradientBoostingRegressor,
    AdaBoostClassifier,
    AdaBoostRegressor,
    BaggingClassifier,
    BaggingRegressor,
    VotingClassifier,
    VotingRegressor,
    StackingClassifier,
    StackingRegressor,
)
```

---

> **Key Takeaway:** For 90% of tabular/structured data problems, your workflow should be:
> 1. Start with Random Forest (baseline)
> 2. Move to HistGradientBoosting (production)
> 3. Stack multiple models only if you need that last 1-2% accuracy
