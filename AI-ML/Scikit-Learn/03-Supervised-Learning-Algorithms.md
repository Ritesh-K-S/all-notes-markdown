# Supervised Learning Algorithms in Scikit-Learn

## Table of Contents
- [Supervised Learning Overview](#supervised-learning-overview)
- [Linear Regression](#linear-regression)
- [Logistic Regression](#logistic-regression)
- [Support Vector Machines (SVM/SVC)](#support-vector-machines)
- [Decision Trees](#decision-trees)
- [K-Nearest Neighbors (KNN)](#k-nearest-neighbors)
- [Naive Bayes](#naive-bayes)
- [Comparing All Algorithms](#comparing-all-algorithms)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Supervised Learning Overview

### What It Is
Supervised learning is like learning with a **teacher**. You're given questions (features X) along with the correct answers (target y), and you learn the pattern so you can answer new questions on your own.

### Two Types

```
Supervised Learning
├── Regression       → Predict a NUMBER (continuous)
│   "How much?"      → House price, temperature, stock price
│
└── Classification   → Predict a CATEGORY (discrete)
    "Which one?"     → Spam/Not spam, Cat/Dog, Disease type
```

### The Universal Sklearn Workflow

```python
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, mean_squared_error

# Step 1: Load & split data
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Step 2: Create model
model = SomeAlgorithm(hyperparameters)

# Step 3: Train
model.fit(X_train, y_train)

# Step 4: Predict
y_pred = model.predict(X_test)

# Step 5: Evaluate
score = model.score(X_test, y_test)
```

---

## Linear Regression

### What It Is
Fitting a **straight line** (or plane/hyperplane in higher dimensions) through your data to predict a continuous number. Like drawing the best "trend line" through a scatter plot.

### Why It Matters
- Simplest and most interpretable regression model
- Baseline for ALL regression problems — always try this first
- Many advanced models are extensions of linear regression (Ridge, Lasso, ElasticNet)
- Used extensively in economics, social sciences, and A/B testing

### How It Works

The model finds the line $\hat{y} = w_1 x_1 + w_2 x_2 + \ldots + w_n x_n + b$ that minimizes the **sum of squared errors**.

$$\text{Cost Function (MSE)} = \frac{1}{n} \sum_{i=1}^{n} (y_i - \hat{y}_i)^2$$

```
                y
                │     *
                │   *  ·····→ Error (residual)
                │  ──────────── Best fit line
                │ *
                │*
                └──────────────── x
                
Goal: Minimize the total squared distance 
      from all points to the line
```

The solution is found analytically using the **Normal Equation**:

$$\mathbf{w} = (\mathbf{X}^T \mathbf{X})^{-1} \mathbf{X}^T \mathbf{y}$$

### Code Example

```python
import numpy as np
from sklearn.linear_model import LinearRegression
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error, r2_score, mean_absolute_error
from sklearn.datasets import make_regression

# ============================================
# Generate sample regression data
# ============================================
X, y = make_regression(
    n_samples=500,
    n_features=5,
    n_informative=3,    # Only 3 features actually matter
    noise=15.0,         # Add some noise
    random_state=42
)

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# ============================================
# Train Linear Regression
# ============================================
model = LinearRegression(
    fit_intercept=True,   # Include bias term (almost always True)
    # n_jobs=-1           # Parallelize for large datasets
)

model.fit(X_train, y_train)

# ============================================
# Inspect learned parameters
# ============================================
print("Coefficients (weights):", model.coef_.round(2))
# e.g., [48.12, 0.31, 73.55, 0.89, 35.22]
# Large weight → important feature, Near-zero → irrelevant

print("Intercept (bias):", model.intercept_.round(2))
# e.g., 0.42

# Feature importance (by magnitude of coefficients)
feature_importance = np.abs(model.coef_)
for i, imp in enumerate(feature_importance):
    print(f"  Feature {i}: {imp:.2f}")

# ============================================
# Evaluate
# ============================================
y_pred = model.predict(X_test)

mse = mean_squared_error(y_test, y_pred)
rmse = np.sqrt(mse)
mae = mean_absolute_error(y_test, y_pred)
r2 = r2_score(y_test, y_pred)

print(f"\nMSE:  {mse:.2f}")
print(f"RMSE: {rmse:.2f}")   # Same units as y
print(f"MAE:  {mae:.2f}")    # Average absolute error
print(f"R²:   {r2:.4f}")     # 1.0 = perfect, 0 = predicts mean
# R² = "what fraction of variance does the model explain?"

# ============================================
# Prediction on new data
# ============================================
X_new = np.array([[0.5, -1.2, 0.3, 2.1, -0.8]])
y_new = model.predict(X_new)
print(f"\nPrediction for new sample: {y_new[0]:.2f}")
```

### Regularized Variants — Ridge, Lasso, ElasticNet

When plain Linear Regression **overfits** (too many features, too little data), add regularization:

$$\text{Ridge Cost} = MSE + \alpha \sum w_i^2 \quad \text{(L2 penalty)}$$
$$\text{Lasso Cost} = MSE + \alpha \sum |w_i| \quad \text{(L1 penalty)}$$
$$\text{ElasticNet Cost} = MSE + \alpha_1 \sum |w_i| + \alpha_2 \sum w_i^2$$

```python
from sklearn.linear_model import Ridge, Lasso, ElasticNet

# ============================================
# Ridge Regression (L2) — shrinks coefficients
# ============================================
ridge = Ridge(alpha=1.0)  # alpha controls regularization strength
ridge.fit(X_train, y_train)
print(f"Ridge R²: {ridge.score(X_test, y_test):.4f}")
print(f"Ridge coefficients: {ridge.coef_.round(2)}")
# Coefficients are smaller than plain LinearRegression

# ============================================
# Lasso Regression (L1) — sets some coefficients to ZERO
# ============================================
lasso = Lasso(alpha=1.0)
lasso.fit(X_train, y_train)
print(f"\nLasso R²: {lasso.score(X_test, y_test):.4f}")
print(f"Lasso coefficients: {lasso.coef_.round(2)}")
# Some coefficients become exactly 0 → automatic feature selection!
print(f"Non-zero features: {np.sum(lasso.coef_ != 0)} / {len(lasso.coef_)}")

# ============================================
# ElasticNet (L1 + L2) — best of both worlds
# ============================================
elastic = ElasticNet(alpha=1.0, l1_ratio=0.5)  # 0.5 = equal L1 and L2
elastic.fit(X_train, y_train)
print(f"\nElasticNet R²: {elastic.score(X_test, y_test):.4f}")
```

### When to Use Which

| Model | Use When | Strengths | Weaknesses |
|-------|----------|-----------|------------|
| LinearRegression | Few features, no overfitting | Fast, interpretable | Overfits with many features |
| Ridge | Many features, all somewhat useful | Handles multicollinearity | Keeps all features |
| Lasso | Many features, want auto selection | Automatic feature selection | Picks one from correlated group |
| ElasticNet | Many correlated features | Groups correlated features | Two hyperparameters to tune |

> **Pro Tip**: Use `RidgeCV` / `LassoCV` / `ElasticNetCV` which automatically find the best `alpha` using cross-validation.

```python
from sklearn.linear_model import RidgeCV, LassoCV

ridge_cv = RidgeCV(alphas=[0.01, 0.1, 1.0, 10.0, 100.0], cv=5)
ridge_cv.fit(X_train, y_train)
print(f"Best alpha: {ridge_cv.alpha_}")
```

---

## Logistic Regression

### What It Is
Despite its name, Logistic Regression is a **classification** algorithm. It predicts the **probability** of a sample belonging to a class. Think of it as drawing a decision boundary line and asking "which side of the line does this point fall on?"

### Why It Matters
- **Default baseline** for classification — try it before anything fancy
- Outputs **probabilities**, not just class labels
- Highly **interpretable** — coefficients tell you feature influence
- Works well even with relatively small datasets
- Used extensively in medicine, finance, and marketing

### How It Works

Logistic Regression applies the **sigmoid function** to a linear combination:

$$P(y=1|x) = \sigma(w^T x + b) = \frac{1}{1 + e^{-(w^T x + b)}}$$

```
Probability
1.0 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─────────
                              ╱
                            ╱
0.5 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ╱─ ─ ─ ─ ─ ─
                        ╱
                      ╱
0.0 ──────────── ─ ╱─ ─ ─ ─ ─ ─ ─ ─ ─
                        0
              w^T x + b (linear output)

If P ≥ 0.5 → Predict Class 1
If P < 0.5 → Predict Class 0
(0.5 is the default threshold)
```

**Cost Function** (Binary Cross-Entropy / Log Loss):

$$J(w) = -\frac{1}{n}\sum_{i=1}^{n} [y_i \log(\hat{p}_i) + (1-y_i) \log(1-\hat{p}_i)]$$

### Code Example — Binary Classification

```python
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    classification_report, confusion_matrix, roc_auc_score
)

# ============================================
# Generate binary classification data
# ============================================
X, y = make_classification(
    n_samples=1000,
    n_features=10,
    n_informative=5,
    n_classes=2,
    random_state=42
)

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y  # Preserve class ratios
)

# ============================================
# Train Logistic Regression
# ============================================
model = LogisticRegression(
    C=1.0,               # Inverse of regularization (higher = less reg)
    penalty='l2',        # 'l1', 'l2', 'elasticnet', None
    solver='lbfgs',      # Optimization algorithm
    max_iter=200,        # Max iterations for convergence
    random_state=42,
    class_weight=None    # Set 'balanced' for imbalanced datasets
)

model.fit(X_train, y_train)

# ============================================
# Make predictions
# ============================================
y_pred = model.predict(X_test)           # Class labels: [0, 1, 0, ...]
y_proba = model.predict_proba(X_test)    # Probabilities: [[0.8, 0.2], ...]
y_scores = model.decision_function(X_test)  # Raw scores (before sigmoid)

print(f"Predictions (first 5): {y_pred[:5]}")
print(f"Probabilities (first 3):\n{y_proba[:3].round(3)}")
# [[0.892, 0.108]   → 89.2% chance of class 0
#  [0.234, 0.766]   → 76.6% chance of class 1
#  [0.543, 0.457]]  → 54.3% chance of class 0 (uncertain!)

# ============================================
# Evaluate — Full Classification Report
# ============================================
print("\n" + "="*50)
print("CLASSIFICATION REPORT")
print("="*50)
print(classification_report(y_test, y_pred))
#               precision    recall  f1-score   support
#            0       0.88      0.85      0.86       100
#            1       0.86      0.89      0.87       100
#     accuracy                           0.87       200

# Confusion Matrix
cm = confusion_matrix(y_test, y_pred)
print(f"Confusion Matrix:\n{cm}")
#         Predicted
#          0    1
# Actual 0 [85   15]   ← 85 correct, 15 false positives
#        1 [11   89]   ← 89 correct, 11 false negatives

# ROC-AUC (uses probabilities, not predictions)
auc = roc_auc_score(y_test, y_proba[:, 1])
print(f"\nROC-AUC: {auc:.4f}")

# ============================================
# Interpret coefficients
# ============================================
print("\nFeature coefficients:")
for i, coef in enumerate(model.coef_[0]):
    direction = "↑ increases" if coef > 0 else "↓ decreases"
    print(f"  Feature {i}: {coef:+.3f} ({direction} P(class=1))")

print(f"Intercept: {model.intercept_[0]:.3f}")
```

### Multiclass Classification

```python
from sklearn.datasets import load_iris

# Logistic Regression handles multiclass automatically
iris = load_iris()
X_train, X_test, y_train, y_test = train_test_split(
    iris.data, iris.target, test_size=0.2, random_state=42, stratify=iris.target
)

# multi_class='auto' picks the right strategy
model = LogisticRegression(
    multi_class='multinomial',  # or 'ovr' (one-vs-rest)
    solver='lbfgs',
    max_iter=200
)

model.fit(X_train, y_train)
print(f"Accuracy: {model.score(X_test, y_test):.3f}")

# Probabilities for all 3 classes
y_proba = model.predict_proba(X_test[:3])
print(f"\nProbabilities (3 classes):")
for i, probs in enumerate(y_proba):
    print(f"  Sample {i}: {probs.round(3)} → predicted: {iris.target_names[probs.argmax()]}")
```

### Multiclass Strategies

| Strategy | How It Works | When to Use |
|----------|-------------|-------------|
| `ovr` (One-vs-Rest) | Trains N binary classifiers | Default for liblinear solver |
| `multinomial` | Single multiclass loss | Better when classes are mutually exclusive |

### Handling Imbalanced Classes

```python
# When one class has much more data (e.g., 95% negative, 5% positive)

# Method 1: class_weight='balanced'
model = LogisticRegression(class_weight='balanced')  
# Automatically adjusts weights inversely proportional to class frequency

# Method 2: Custom weights
model = LogisticRegression(class_weight={0: 1, 1: 10})
# Penalize misclassifying class 1 ten times more

# Method 3: Adjust decision threshold
y_proba = model.predict_proba(X_test)[:, 1]
threshold = 0.3  # Lower threshold → catch more positives (higher recall)
y_pred_custom = (y_proba >= threshold).astype(int)
```

### Solver Selection Guide

| Solver | L1 | L2 | Multinomial | Large Dataset | Notes |
|--------|----|----|-------------|---------------|-------|
| `lbfgs` | ❌ | ✅ | ✅ | ✅ | **Default**. Best general choice |
| `liblinear` | ✅ | ✅ | ❌ (OVR only) | ❌ | Good for small datasets |
| `saga` | ✅ | ✅ | ✅ | ✅ | Best for very large datasets |
| `newton-cg` | ❌ | ✅ | ✅ | ✅ | Good for multinomial |

> **Pro Tip**: If you get `ConvergenceWarning`, increase `max_iter` (try 500 or 1000) or scale your data first.

---

## Support Vector Machines

### What It Is
SVM finds the **best boundary** (hyperplane) that separates classes with the **maximum margin** — the widest possible gap between classes. Imagine placing a thick ruler between two groups of dots — SVM finds where the ruler fits best.

### Why It Matters
- **Excellent for small-to-medium datasets** with clear margins
- Works in **high-dimensional spaces** (even more features than samples)
- The **kernel trick** makes it handle non-linear boundaries without explicitly computing transformations
- Strong theoretical foundation from optimization theory

### How It Works

```
                Linear SVM
                ──────────
                
    Class 0         │         Class 1
    o               │              x
      o      o      │         x       x
        o  ←margin→ │ ←margin→   x
    o    ●──────────│──────────●    x
      o      o      │         x       x
    o               │              x
                    │
              Decision Boundary
              (maximize margin)
              
    ● = Support Vectors (closest points to boundary)
    Only these points matter for defining the boundary!
```

**Optimization objective:**

$$\min_{w, b} \frac{1}{2} \|w\|^2 \quad \text{subject to} \quad y_i(w^T x_i + b) \geq 1$$

With soft margin (allowing some misclassifications):

$$\min_{w, b} \frac{1}{2} \|w\|^2 + C \sum_{i=1}^{n} \xi_i$$

Where $C$ controls the trade-off: **high C** = fewer errors (overfit risk), **low C** = wider margin (underfit risk).

### The Kernel Trick

When data isn't linearly separable, SVM maps it to a higher dimension using **kernels**:

```
Original Space (not separable):     After Kernel (separable!):
                                    
    o o x x o o                        x     x
    o x x x x o                      x   x   x
    o o x x o o                     o  o  o  o  o
                                      o   o   o
Can't draw a line!                  Can draw a plane!
```

### Code Example — SVM Classification

```python
from sklearn.svm import SVC, LinearSVC
from sklearn.datasets import make_classification, make_moons
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import train_test_split
from sklearn.pipeline import make_pipeline
from sklearn.metrics import classification_report

# ============================================
# Linear SVM
# ============================================
X, y = make_classification(n_samples=500, n_features=20, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=42)

# ALWAYS scale data for SVM!
svm_pipe = make_pipeline(
    StandardScaler(),
    SVC(
        kernel='linear',
        C=1.0,              # Regularization (lower = more regularization)
        random_state=42
    )
)

svm_pipe.fit(X_train, y_train)
print(f"Linear SVM accuracy: {svm_pipe.score(X_test, y_test):.3f}")

# ============================================
# Non-linear SVM with RBF kernel
# ============================================
# Make non-linearly separable data
X_moons, y_moons = make_moons(n_samples=500, noise=0.2, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X_moons, y_moons, random_state=42)

rbf_svm = make_pipeline(
    StandardScaler(),
    SVC(
        kernel='rbf',       # Radial Basis Function (Gaussian)
        C=10.0,             # Higher C = tighter fit
        gamma='scale',      # 'scale' (default), 'auto', or float
        random_state=42
    )
)

rbf_svm.fit(X_train, y_train)
print(f"RBF SVM accuracy: {rbf_svm.score(X_test, y_test):.3f}")

# ============================================
# Getting probabilities from SVM
# ============================================
# SVM doesn't naturally output probabilities
# Must set probability=True (uses Platt scaling — slower!)
svm_proba = make_pipeline(
    StandardScaler(),
    SVC(kernel='rbf', C=10.0, probability=True, random_state=42)
)
svm_proba.fit(X_train, y_train)
probabilities = svm_proba.predict_proba(X_test[:3])
print(f"\nProbabilities:\n{probabilities.round(3)}")

# ============================================
# Support Vectors
# ============================================
svc = SVC(kernel='rbf', C=10.0)
svc.fit(StandardScaler().fit_transform(X_train), y_train)
print(f"\nSupport vectors: {svc.n_support_}")  # Count per class
print(f"Total support vectors: {len(svc.support_vectors_)} / {len(X_train)} training samples")
# Fewer support vectors = simpler, more generalizable model
```

### SVM for Regression (SVR)

```python
from sklearn.svm import SVR
from sklearn.datasets import make_regression

X, y = make_regression(n_samples=200, n_features=5, noise=10, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=42)

svr = make_pipeline(
    StandardScaler(),
    SVR(
        kernel='rbf',
        C=100.0,
        epsilon=0.1  # Tube width (no penalty for errors within epsilon)
    )
)

svr.fit(X_train, y_train)
print(f"SVR R²: {svr.score(X_test, y_test):.3f}")
```

### Kernel Comparison

| Kernel | Formula | Use When | Parameters |
|--------|---------|----------|-----------|
| `linear` | $K(x,z) = x^T z$ | Linearly separable, many features | C |
| `rbf` | $K(x,z) = e^{-\gamma\|x-z\|^2}$ | **Default**. Non-linear, medium data | C, gamma |
| `poly` | $K(x,z) = (\gamma x^T z + r)^d$ | Polynomial relationships | C, gamma, degree |
| `sigmoid` | $K(x,z) = \tanh(\gamma x^T z + r)$ | Similar to neural network | C, gamma |

### Key Parameters to Tune

```python
from sklearn.model_selection import GridSearchCV

param_grid = {
    'svc__C': [0.1, 1, 10, 100],
    'svc__gamma': ['scale', 'auto', 0.01, 0.1, 1],
    'svc__kernel': ['rbf', 'linear']
}

grid = GridSearchCV(svm_pipe, param_grid, cv=5, scoring='accuracy')
grid.fit(X_train, y_train)
print(f"Best: {grid.best_params_} → {grid.best_score_:.3f}")
```

> **Pro Tip**: For large datasets (>10,000 samples), use `LinearSVC` instead of `SVC(kernel='linear')` — it's **much** faster (uses liblinear instead of libsvm).

---

## Decision Trees

### What It Is
A Decision Tree makes predictions by asking a series of **yes/no questions** about the data, like a flowchart. Think of the game "20 Questions" — each question splits possibilities in half until you arrive at an answer.

### Why It Matters
- **Most interpretable** ML model — you can literally draw and explain it
- Foundation for **Random Forest**, **XGBoost**, **LightGBM** (the most powerful models)
- Handles **both numeric and categorical** features naturally
- No feature scaling needed
- Captures **non-linear** relationships and **feature interactions**

### How It Works

```
                    Is income > 50K?
                   /              \
                 YES               NO
                /                    \
        Age > 30?              Education > Bachelor?
        /      \                /          \
      YES      NO            YES           NO
      /          \           /               \
  Approve    Credit > 700?  Review         Reject
             /        \
           YES        NO
           /            \
        Approve       Reject
```

**Splitting Criteria:**

For classification, the tree chooses splits that maximize **information gain** (reduce impurity):

**Gini Impurity** (default in sklearn):
$$Gini = 1 - \sum_{i=1}^{C} p_i^2$$

**Entropy** (Information Gain):
$$Entropy = -\sum_{i=1}^{C} p_i \log_2(p_i)$$

Where $p_i$ is the proportion of class $i$ in the node.

```
Pure node:     Gini = 0      (all same class)
Impure node:   Gini = 0.5    (50/50 split for binary)
```

### Code Example — Decision Tree Classifier

```python
from sklearn.tree import DecisionTreeClassifier, export_text, plot_tree
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
import matplotlib.pyplot as plt

# ============================================
# Train Decision Tree
# ============================================
iris = load_iris()
X_train, X_test, y_train, y_test = train_test_split(
    iris.data, iris.target, test_size=0.2, random_state=42, stratify=iris.target
)

tree = DecisionTreeClassifier(
    max_depth=3,              # Maximum tree depth (controls overfitting)
    min_samples_split=10,     # Min samples to split a node
    min_samples_leaf=5,       # Min samples in a leaf
    criterion='gini',         # 'gini' or 'entropy'
    random_state=42
)

tree.fit(X_train, y_train)

# ============================================
# Evaluate
# ============================================
print(f"Training accuracy: {tree.score(X_train, y_train):.3f}")
print(f"Test accuracy: {tree.score(X_test, y_test):.3f}")
# If training >> test, the tree is overfitting!

# ============================================
# Visualize the tree (text)
# ============================================
tree_rules = export_text(tree, feature_names=iris.feature_names)
print("\nDecision Rules:")
print(tree_rules)
# |--- petal width (cm) <= 0.80
# |   |--- class: 0  (setosa)
# |--- petal width (cm) >  0.80
# |   |--- petal width (cm) <= 1.75
# |   |   |--- class: 1  (versicolor)
# |   |--- petal width (cm) >  1.75
# |   |   |--- class: 2  (virginica)

# ============================================
# Visualize the tree (graphical)
# ============================================
plt.figure(figsize=(20, 10))
plot_tree(
    tree,
    feature_names=iris.feature_names,
    class_names=iris.target_names,
    filled=True,           # Color nodes by class
    rounded=True,
    fontsize=10
)
plt.title("Decision Tree for Iris Classification")
plt.tight_layout()
plt.savefig('decision_tree.png', dpi=150)
plt.show()

# ============================================
# Feature Importance
# ============================================
importances = tree.feature_importances_
for name, imp in zip(iris.feature_names, importances):
    print(f"  {name}: {imp:.4f}")
# petal width might be ~0.90 — dominates the decision!

# ============================================
# Tree properties
# ============================================
print(f"\nTree depth: {tree.get_depth()}")
print(f"Number of leaves: {tree.get_n_leaves()}")
print(f"Number of features used: {tree.n_features_in_}")
```

### Decision Tree Regressor

```python
from sklearn.tree import DecisionTreeRegressor
from sklearn.datasets import make_regression

X, y = make_regression(n_samples=500, n_features=5, noise=20, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=42)

tree_reg = DecisionTreeRegressor(
    max_depth=5,
    min_samples_leaf=10,
    random_state=42
)

tree_reg.fit(X_train, y_train)
print(f"R² score: {tree_reg.score(X_test, y_test):.3f}")
# Uses MSE as splitting criterion by default
```

### Controlling Overfitting (Critical!)

Decision Trees overfit **very easily**. An unpruned tree can memorize the training data.

```python
# ❌ OVERFITTING — no constraints
tree_overfit = DecisionTreeClassifier(random_state=42)  # No limits
tree_overfit.fit(X_train, y_train)
print(f"Train: {tree_overfit.score(X_train, y_train):.3f}")  # 1.000 (memorized!)
print(f"Test:  {tree_overfit.score(X_test, y_test):.3f}")    # 0.867 (poor generalization)

# ✅ CONTROLLED — with pruning parameters
tree_pruned = DecisionTreeClassifier(
    max_depth=4,               # Limit tree depth
    min_samples_split=20,      # Need ≥20 samples to split
    min_samples_leaf=10,       # Each leaf must have ≥10 samples
    max_features='sqrt',       # Only consider sqrt(n) features per split
    random_state=42
)
tree_pruned.fit(X_train, y_train)
print(f"Train: {tree_pruned.score(X_train, y_train):.3f}")   # 0.958
print(f"Test:  {tree_pruned.score(X_test, y_test):.3f}")     # 0.933 (better!)
```

### Cost-Complexity Pruning (Post-Pruning)

```python
# Find the best ccp_alpha using cross-validation
path = tree_overfit.cost_complexity_pruning_path(X_train, y_train)
ccp_alphas = path.ccp_alphas

# Test each alpha
from sklearn.model_selection import cross_val_score

best_alpha = 0
best_score = 0

for alpha in ccp_alphas[::10]:  # Sample every 10th
    tree_temp = DecisionTreeClassifier(ccp_alpha=alpha, random_state=42)
    scores = cross_val_score(tree_temp, X_train, y_train, cv=5)
    if scores.mean() > best_score:
        best_score = scores.mean()
        best_alpha = alpha

print(f"Best ccp_alpha: {best_alpha:.6f}")
print(f"Best CV score: {best_score:.3f}")

tree_final = DecisionTreeClassifier(ccp_alpha=best_alpha, random_state=42)
tree_final.fit(X_train, y_train)
```

> **Pro Tip**: Decision Trees alone are weak learners. Their real power comes in **ensembles** — Random Forest (bagging many trees) and Gradient Boosting (chaining weak trees).

---

## K-Nearest Neighbors

### What It Is
KNN makes predictions by finding the **K closest training points** to a new data point and taking a vote (classification) or average (regression). It's like asking your K nearest neighbors what they think — the majority wins.

### Why It Matters
- **Simplest algorithm** to understand — no math beyond distance
- **No training phase** — it just stores the data (lazy learner)
- Works well for **recommendation systems** and **anomaly detection**
- Great **baseline** for spatial/distance-based problems

### How It Works

```
     New point: ?
     
     K=3: look at 3 nearest
     
        o  o     
     o        x  ←(nearest 1: class x)
        o  ?  x  ←(nearest 2: class x)
     o     x     ←(nearest 3: class x)
        o        
     
     Vote: 3 x's, 0 o's → Predict: x
     
     K=7: look at 7 nearest
     
     Vote: 4 o's, 3 x's → Predict: o
     
     K matters A LOT!
```

**Distance Metrics:**

| Metric | Formula | Use When |
|--------|---------|----------|
| Euclidean (default) | $\sqrt{\sum(x_i - y_i)^2}$ | Continuous features |
| Manhattan | $\sum|x_i - y_i|$ | Grid-like data, high dimensions |
| Minkowski | $(\sum|x_i - y_i|^p)^{1/p}$ | Generalization (p=1: Manhattan, p=2: Euclidean) |

### Code Example

```python
from sklearn.neighbors import KNeighborsClassifier, KNeighborsRegressor
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import make_pipeline
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split, cross_val_score
import numpy as np

# ============================================
# KNN Classifier
# ============================================
iris = load_iris()
X_train, X_test, y_train, y_test = train_test_split(
    iris.data, iris.target, test_size=0.2, random_state=42, stratify=iris.target
)

# MUST scale for KNN! (distance-based)
knn_pipe = make_pipeline(
    StandardScaler(),
    KNeighborsClassifier(
        n_neighbors=5,         # K value
        weights='uniform',     # 'uniform' (equal vote) or 'distance' (closer = more weight)
        metric='minkowski',    # Distance metric
        p=2,                   # p=2: Euclidean, p=1: Manhattan
        n_jobs=-1              # Parallelize distance computations
    )
)

knn_pipe.fit(X_train, y_train)
print(f"KNN accuracy: {knn_pipe.score(X_test, y_test):.3f}")

# Probability estimates
y_proba = knn_pipe.predict_proba(X_test[:3])
print(f"Probabilities:\n{y_proba.round(3)}")
# Based on proportion of neighbors in each class

# ============================================
# Finding the best K
# ============================================
k_values = range(1, 31)
cv_scores = []

for k in k_values:
    knn = make_pipeline(
        StandardScaler(),
        KNeighborsClassifier(n_neighbors=k)
    )
    scores = cross_val_score(knn, X_train, y_train, cv=5, scoring='accuracy')
    cv_scores.append(scores.mean())

best_k = k_values[np.argmax(cv_scores)]
print(f"\nBest K: {best_k} with accuracy: {max(cv_scores):.3f}")

# ============================================
# Weighted KNN (closer neighbors matter more)
# ============================================
knn_weighted = make_pipeline(
    StandardScaler(),
    KNeighborsClassifier(n_neighbors=10, weights='distance')
)
knn_weighted.fit(X_train, y_train)
print(f"Weighted KNN accuracy: {knn_weighted.score(X_test, y_test):.3f}")

# ============================================
# KNN Regressor
# ============================================
from sklearn.datasets import make_regression

X_reg, y_reg = make_regression(n_samples=200, n_features=5, noise=10, random_state=42)
X_tr, X_te, y_tr, y_te = train_test_split(X_reg, y_reg, random_state=42)

knn_reg = make_pipeline(
    StandardScaler(),
    KNeighborsRegressor(n_neighbors=5, weights='distance')
)
knn_reg.fit(X_tr, y_tr)
print(f"\nKNN Regressor R²: {knn_reg.score(X_te, y_te):.3f}")
```

### Choosing K — The Bias-Variance Tradeoff

```
Error
  │
  │  \                          ____
  │   \   Training Error      /
  │    \___________________  /
  │                         /
  │     ___     Test Error /
  │    /   \____    ______/
  │   /         \__/
  │  /
  └──────────────────────────── K
    1     5    10    15    20

K too small (1-3):  High variance, overfitting (memorizes noise)
K too large (>20):  High bias, underfitting (too smooth)
Sweet spot:         Usually between 3-15 (use cross-validation)
```

| K Value | Behavior | Risk |
|---------|----------|------|
| K = 1 | Predicts same as nearest point | Overfitting, noisy |
| K = small | Complex boundary, captures details | Sensitive to outliers |
| K = large | Smooth boundary, general trends | May miss local patterns |
| K = N | Predicts majority class for everything | Useless |

> **Pro Tip**: Use odd K for binary classification to avoid ties. For multiclass, use `weights='distance'` to break ties intelligently.

### KNN Limitations

- **Slow at prediction time**: Must compute distance to ALL training points. $O(n \cdot d)$ per prediction
- **Curse of dimensionality**: In high dimensions, all points become equally distant
- **Memory-heavy**: Stores entire training set
- **Doesn't work with categorical features** (without encoding)

For large datasets, consider `BallTree` or `KDTree` algorithms:

```python
knn = KNeighborsClassifier(
    n_neighbors=5,
    algorithm='ball_tree'  # 'auto', 'ball_tree', 'kd_tree', 'brute'
)
```

---

## Naive Bayes

### What It Is
Naive Bayes applies **Bayes' Theorem** with the "naive" assumption that all features are **independent** of each other. It calculates the probability of each class given the features and picks the most probable class. Think of it as a fast probability calculator that treats each feature as an independent piece of evidence.

### Why It Matters
- **Extremely fast** — both training and prediction are nearly instant
- Works brilliantly for **text classification** (spam detection, sentiment analysis)
- Handles **high-dimensional** data well (thousands of features)
- Works surprisingly well even when the independence assumption is wrong
- Excellent with **small training sets**

### How It Works

**Bayes' Theorem:**

$$P(y|X) = \frac{P(X|y) \cdot P(y)}{P(X)}$$

The "naive" assumption: features are independent given the class:

$$P(X|y) = P(x_1|y) \cdot P(x_2|y) \cdot \ldots \cdot P(x_n|y)$$

**Prediction:**

$$\hat{y} = \arg\max_{y} P(y) \prod_{i=1}^{n} P(x_i|y)$$

```
Example: Is this email spam?

Features: contains("free"), contains("money"), contains("meeting")

P(spam) × P("free"|spam) × P("money"|spam) × P("meeting"|spam) = 0.72
P(ham)  × P("free"|ham)  × P("money"|ham)  × P("meeting"|ham)  = 0.28

→ Predict: spam (72% > 28%)
```

### Three Variants

| Variant | Assumes Feature Distribution | Best For |
|---------|----------------------------|----------|
| `GaussianNB` | Features are normally distributed | Continuous features |
| `MultinomialNB` | Features are counts/frequencies | Text (word counts, TF-IDF) |
| `BernoulliNB` | Features are binary (0/1) | Binary features, short texts |

### Code Example — All Three Variants

```python
from sklearn.naive_bayes import GaussianNB, MultinomialNB, BernoulliNB
from sklearn.datasets import make_classification, load_iris
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report
import numpy as np

# ============================================
# GaussianNB — for continuous features
# ============================================
iris = load_iris()
X_train, X_test, y_train, y_test = train_test_split(
    iris.data, iris.target, test_size=0.2, random_state=42, stratify=iris.target
)

gnb = GaussianNB(
    var_smoothing=1e-9  # Smoothing (adds small value to variance for stability)
)
gnb.fit(X_train, y_train)

print("=== GaussianNB ===")
print(f"Accuracy: {gnb.score(X_test, y_test):.3f}")

# Inspect learned parameters
print(f"Class priors: {gnb.class_prior_}")     # P(y) for each class
print(f"Class means shape: {gnb.theta_.shape}")  # Mean of each feature per class
print(f"Class variances shape: {gnb.var_.shape}")  # Variance of each feature per class

# ============================================
# MultinomialNB — for text classification (count data)
# ============================================
from sklearn.feature_extraction.text import CountVectorizer, TfidfVectorizer

# Simulate text data
texts = [
    "free money click here", "meeting tomorrow office",
    "win prize free", "project deadline report",
    "free cash lottery", "budget review quarterly",
    "cheap pills discount", "team sync standup daily",
    "winner congratulations free", "code review pull request"
]
labels = [1, 0, 1, 0, 1, 0, 1, 0, 1, 0]  # 1=spam, 0=ham

vectorizer = CountVectorizer()
X_text = vectorizer.fit_transform(texts)

mnb = MultinomialNB(
    alpha=1.0  # Laplace smoothing (prevents zero probability)
)
mnb.fit(X_text, labels)

# Predict new email
new_email = vectorizer.transform(["free money win prize"])
pred = mnb.predict(new_email)
prob = mnb.predict_proba(new_email)
print(f"\n=== MultinomialNB ===")
print(f"'{' '.join(vectorizer.inverse_transform(new_email)[0])}' → {'spam' if pred[0] else 'ham'}")
print(f"Probabilities: ham={prob[0][0]:.3f}, spam={prob[0][1]:.3f}")

# Most informative features
feature_names = vectorizer.get_feature_names_out()
for cls in range(2):
    top_idx = mnb.feature_log_prob_[cls].argsort()[-5:]
    top_words = feature_names[top_idx]
    print(f"Top words for class {cls}: {top_words}")

# ============================================
# BernoulliNB — for binary features
# ============================================
# Great for binary document features (word present/absent)
from sklearn.preprocessing import Binarizer

bnb = BernoulliNB(alpha=1.0)
X_binary = (X_text > 0).astype(int)  # Convert counts to binary
bnb.fit(X_binary, labels)
print(f"\n=== BernoulliNB ===")
print(f"Accuracy on training: {bnb.score(X_binary, labels):.3f}")
```

### Laplace Smoothing (alpha)

```python
# Problem: If a word never appears with a class, P(word|class) = 0
# Then the ENTIRE product becomes 0! One unseen word kills everything.

# Solution: Add alpha to all counts (Laplace smoothing)
# P(word|class) = (count(word, class) + alpha) / (total_words_in_class + alpha * vocab_size)

mnb_no_smooth = MultinomialNB(alpha=0)    # No smoothing (risky)
mnb_smooth = MultinomialNB(alpha=1.0)     # Laplace smoothing (default)
mnb_heavy = MultinomialNB(alpha=10.0)     # Heavy smoothing (more uniform)
```

### Partial Fit — Online/Incremental Learning

```python
# Naive Bayes supports partial_fit() — learn from batches!
# Perfect for streaming data or data that doesn't fit in memory

gnb = GaussianNB()

# First batch — must specify all classes
gnb.partial_fit(X_batch1, y_batch1, classes=[0, 1, 2])

# Subsequent batches
gnb.partial_fit(X_batch2, y_batch2)
gnb.partial_fit(X_batch3, y_batch3)
# Model incrementally improves!
```

> **Pro Tip**: Naive Bayes probabilities are **poorly calibrated** — they tend to be pushed toward 0 or 1. Use `CalibratedClassifierCV` if you need accurate probability estimates.

---

## Comparing All Algorithms

### Head-to-Head Comparison

```python
import numpy as np
from sklearn.datasets import make_classification
from sklearn.model_selection import cross_val_score, train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import make_pipeline

# Linear models
from sklearn.linear_model import LogisticRegression, Ridge
# SVM
from sklearn.svm import SVC
# Tree
from sklearn.tree import DecisionTreeClassifier
# KNN
from sklearn.neighbors import KNeighborsClassifier
# Naive Bayes
from sklearn.naive_bayes import GaussianNB

# Generate dataset
X, y = make_classification(
    n_samples=2000, n_features=20, n_informative=10,
    n_redundant=5, n_classes=2, random_state=42
)

models = {
    'Logistic Regression': make_pipeline(StandardScaler(), LogisticRegression(max_iter=200)),
    'SVM (RBF)': make_pipeline(StandardScaler(), SVC(kernel='rbf')),
    'SVM (Linear)': make_pipeline(StandardScaler(), SVC(kernel='linear')),
    'Decision Tree': DecisionTreeClassifier(max_depth=5, random_state=42),
    'KNN (K=5)': make_pipeline(StandardScaler(), KNeighborsClassifier(n_neighbors=5)),
    'KNN (K=10)': make_pipeline(StandardScaler(), KNeighborsClassifier(n_neighbors=10)),
    'Gaussian NB': GaussianNB(),
}

print(f"{'Model':<25} {'Mean Accuracy':>15} {'Std':>8}")
print("-" * 50)

for name, model in models.items():
    scores = cross_val_score(model, X, y, cv=5, scoring='accuracy')
    print(f"{name:<25} {scores.mean():>15.3f} {scores.std():>8.3f}")
```

### Algorithm Selection Guide

| Criterion | LogReg | SVM | Tree | KNN | NB |
|-----------|--------|-----|------|-----|-----|
| **Speed (train)** | Fast | Slow (large data) | Fast | None (lazy) | Very Fast |
| **Speed (predict)** | Very Fast | Fast | Very Fast | Slow | Very Fast |
| **Interpretability** | High | Low | Very High | Medium | Medium |
| **Needs Scaling** | With reg | Yes | No | Yes | No |
| **Handles Non-linear** | No* | Yes (kernel) | Yes | Yes | No |
| **Handles Missing Values** | No | No | No | No | No |
| **Handles Outliers** | Sensitive | Somewhat | Robust | Sensitive | Robust |
| **Large Datasets** | Great | Poor** | Great | Poor | Great |
| **High Dimensions** | Great | Great | Poor | Poor | Great |
| **Small Datasets** | Good | Great | Poor | Good | Great |

\* Unless you add polynomial features  
\** `LinearSVC` handles large datasets well

### When to Use What — Decision Flowchart

```
                 START
                   │
         Is it classification?
              /         \
            YES          NO (regression)
             │            │
    How much data?    Linear relationship?
    /        \          /        \
  Small     Large    YES         NO
  (<5K)     (>50K)    │           │
    │         │    LinReg/      Tree/SVR
    │         │    Ridge/Lasso
    │         │
  SVM/NB   LogReg/
    │      LinearSVC
    │
  Text data?
  /       \
YES        NO
 │          │
 NB      Need interpret?
          /       \
        YES        NO
         │          │
       Tree     SVM/KNN
```

---

## Common Mistakes

### 1. Not Scaling Before Distance-Based Algorithms

```python
# ❌ WRONG
knn = KNeighborsClassifier()
knn.fit(X_train, y_train)  # Features at different scales!

# ✅ CORRECT
knn = make_pipeline(StandardScaler(), KNeighborsClassifier())
knn.fit(X_train, y_train)
```

### 2. Using Accuracy on Imbalanced Data

```python
# ❌ WRONG — 95% accuracy on 95/5 split is meaningless
accuracy = model.score(X_test, y_test)

# ✅ CORRECT
from sklearn.metrics import f1_score, classification_report
print(classification_report(y_test, y_pred))
# Look at precision, recall, and F1 per class
```

### 3. Not Pruning Decision Trees

```python
# ❌ WRONG — tree memorizes training data
tree = DecisionTreeClassifier()  # No constraints = 100% train accuracy, poor test

# ✅ CORRECT — always limit tree complexity
tree = DecisionTreeClassifier(max_depth=5, min_samples_leaf=10, random_state=42)
```

### 4. Using KNN Without Considering K

```python
# ❌ WRONG — K=1 overfits to noise
knn = KNeighborsClassifier(n_neighbors=1)

# ✅ CORRECT — tune K with cross-validation
from sklearn.model_selection import GridSearchCV
grid = GridSearchCV(
    KNeighborsClassifier(),
    {'n_neighbors': range(1, 31, 2)},
    cv=5
)
grid.fit(X_train, y_train)
print(f"Best K: {grid.best_params_}")
```

### 5. Forgetting max_iter for Logistic Regression

```python
# ❌ WRONG — ConvergenceWarning
model = LogisticRegression()  # Default max_iter=100 often too low

# ✅ CORRECT
model = LogisticRegression(max_iter=500)  # Or scale your data first
```

### 6. Using predict_proba with SVM Without Enabling It

```python
# ❌ WRONG
svm = SVC()
svm.fit(X_train, y_train)
# svm.predict_proba(X_test)  # AttributeError!

# ✅ CORRECT
svm = SVC(probability=True)  # Enables Platt scaling
svm.fit(X_train, y_train)
svm.predict_proba(X_test)  # Works but is slower
```

---

## Interview Questions

### Conceptual

**Q1: Explain the bias-variance tradeoff using Decision Trees and KNN as examples.**
> **Decision Trees**: Unpruned tree = low bias, high variance (overfits). Setting max_depth reduces variance but increases bias.  
> **KNN**: K=1 = low bias, high variance (overfits to nearest point). Large K = high bias, low variance (too smooth).  
> Both need a sweet spot found via cross-validation.

**Q2: When would you choose Logistic Regression over SVM?**
> Use Logistic Regression when: (1) you need probability outputs, (2) you need interpretable coefficients, (3) you have a large dataset (faster), (4) you want to understand feature contributions. Use SVM when: (1) you have a small dataset with clear margins, (2) data is non-linearly separable (kernel trick), (3) high-dimensional spaces.

**Q3: Why is Naive Bayes called "naive"?**
> Because it assumes all features are conditionally independent given the class. In reality, features are almost always correlated (e.g., height and weight). Despite this unrealistic assumption, NB works surprisingly well in practice, especially for text classification.

**Q4: What are support vectors? Why do only they matter?**
> Support vectors are the training points closest to the decision boundary. Only these points define the boundary position. If you removed all other points, the boundary wouldn't change. This makes SVM memory-efficient and robust — most training data is irrelevant to the model.

**Q5: Compare L1 (Lasso) and L2 (Ridge) regularization.**
> **L1 (Lasso)**: Adds $\alpha\sum|w_i|$ to cost. Drives some coefficients to exactly zero → automatic feature selection. Produces sparse models. Good when only a few features matter.  
> **L2 (Ridge)**: Adds $\alpha\sum w_i^2$ to cost. Shrinks all coefficients toward zero but never to exactly zero. Better when all features contribute. Handles multicollinearity.

**Q6: How does KNN handle the curse of dimensionality?**
> In high dimensions, all points become nearly equidistant, making the concept of "nearest neighbor" meaningless. Solutions: (1) reduce dimensions with PCA first, (2) use only informative features, (3) use Manhattan distance instead of Euclidean (more robust in high-D), (4) consider other algorithms entirely.

**Q7: Can Decision Trees handle missing values?**
> Sklearn's implementation cannot — you must impute first. However, some implementations (like XGBoost) handle missing values natively by learning the best direction to send missing values at each split.

### Coding

**Q8: Write code to find the best algorithm and hyperparameters for a classification problem.**

```python
from sklearn.model_selection import GridSearchCV, cross_val_score
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
from sklearn.tree import DecisionTreeClassifier
from sklearn.neighbors import KNeighborsClassifier
from sklearn.naive_bayes import GaussianNB
from sklearn.datasets import make_classification

X, y = make_classification(n_samples=1000, n_features=20, random_state=42)

# Define model + hyperparameter space
configs = [
    {
        'name': 'LogisticRegression',
        'pipeline': make_pipeline(StandardScaler(), LogisticRegression(max_iter=500)),
        'params': {'logisticregression__C': [0.01, 0.1, 1, 10]}
    },
    {
        'name': 'SVM',
        'pipeline': make_pipeline(StandardScaler(), SVC()),
        'params': {'svc__C': [0.1, 1, 10], 'svc__kernel': ['rbf', 'linear']}
    },
    {
        'name': 'DecisionTree',
        'pipeline': DecisionTreeClassifier(random_state=42),
        'params': {'max_depth': [3, 5, 10, None], 'min_samples_leaf': [1, 5, 10]}
    },
    {
        'name': 'KNN',
        'pipeline': make_pipeline(StandardScaler(), KNeighborsClassifier()),
        'params': {'kneighborsclassifier__n_neighbors': [3, 5, 7, 11]}
    },
]

best_overall = {'name': '', 'score': 0, 'params': {}}

for config in configs:
    grid = GridSearchCV(config['pipeline'], config['params'], cv=5, scoring='f1')
    grid.fit(X, y)
    
    print(f"{config['name']:25s} Best F1: {grid.best_score_:.3f} Params: {grid.best_params_}")
    
    if grid.best_score_ > best_overall['score']:
        best_overall = {
            'name': config['name'],
            'score': grid.best_score_,
            'params': grid.best_params_
        }

print(f"\n🏆 Best model: {best_overall['name']} with F1={best_overall['score']:.3f}")
```

---

## Quick Reference

### Algorithm Cheat Sheet

| Algorithm | sklearn Class | Type | Key Params | score() Returns |
|-----------|--------------|------|------------|-----------------|
| Linear Regression | `LinearRegression()` | Regression | `fit_intercept` | R² |
| Ridge | `Ridge()` | Regression | `alpha` | R² |
| Lasso | `Lasso()` | Regression | `alpha` | R² |
| Logistic Regression | `LogisticRegression()` | Classification | `C`, `penalty`, `solver` | Accuracy |
| SVM Classifier | `SVC()` | Classification | `C`, `kernel`, `gamma` | Accuracy |
| SVM Regressor | `SVR()` | Regression | `C`, `kernel`, `epsilon` | R² |
| Decision Tree (C) | `DecisionTreeClassifier()` | Classification | `max_depth`, `min_samples_leaf` | Accuracy |
| Decision Tree (R) | `DecisionTreeRegressor()` | Regression | `max_depth`, `min_samples_leaf` | R² |
| KNN Classifier | `KNeighborsClassifier()` | Classification | `n_neighbors`, `weights` | Accuracy |
| KNN Regressor | `KNeighborsRegressor()` | Regression | `n_neighbors`, `weights` | R² |
| Gaussian NB | `GaussianNB()` | Classification | `var_smoothing` | Accuracy |
| Multinomial NB | `MultinomialNB()` | Classification | `alpha` | Accuracy |

### Import Cheat Sheet

```python
# Linear Models
from sklearn.linear_model import (
    LinearRegression, Ridge, Lasso, ElasticNet,
    LogisticRegression, RidgeCV, LassoCV
)

# SVM
from sklearn.svm import SVC, SVR, LinearSVC, LinearSVR

# Trees
from sklearn.tree import (
    DecisionTreeClassifier, DecisionTreeRegressor,
    export_text, plot_tree
)

# KNN
from sklearn.neighbors import KNeighborsClassifier, KNeighborsRegressor

# Naive Bayes
from sklearn.naive_bayes import GaussianNB, MultinomialNB, BernoulliNB

# Metrics
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    mean_squared_error, r2_score, mean_absolute_error,
    classification_report, confusion_matrix, roc_auc_score
)
```

### Hyperparameter Tuning Priority

| Algorithm | Tune First | Tune Second | Tune Third |
|-----------|-----------|-------------|------------|
| Logistic Regression | `C` | `penalty` | `solver` |
| SVM | `C` | `kernel` | `gamma` |
| Decision Tree | `max_depth` | `min_samples_leaf` | `criterion` |
| KNN | `n_neighbors` | `weights` | `metric` |
| Naive Bayes | `alpha` (MNB) | `var_smoothing` (GNB) | — |
| Ridge/Lasso | `alpha` | — | — |

### Metrics Cheat Sheet

| Metric | Formula | Use For | Range |
|--------|---------|---------|-------|
| Accuracy | $\frac{TP+TN}{Total}$ | Balanced classification | [0, 1] |
| Precision | $\frac{TP}{TP+FP}$ | Minimize false positives (spam) | [0, 1] |
| Recall | $\frac{TP}{TP+FN}$ | Minimize false negatives (disease) | [0, 1] |
| F1-Score | $\frac{2 \cdot P \cdot R}{P + R}$ | Balance precision & recall | [0, 1] |
| ROC-AUC | Area under ROC curve | Compare models overall | [0, 1] |
| MSE | $\frac{1}{n}\sum(y-\hat{y})^2$ | Regression (penalizes outliers) | [0, ∞) |
| RMSE | $\sqrt{MSE}$ | Regression (same units as y) | [0, ∞) |
| MAE | $\frac{1}{n}\sum|y-\hat{y}|$ | Regression (robust to outliers) | [0, ∞) |
| R² | $1 - \frac{SS_{res}}{SS_{tot}}$ | Regression (% variance explained) | (-∞, 1] |

---

*Previous: [02-Preprocessing-and-Pipelines](02-Preprocessing-and-Pipelines.md)*
*Next: [04-Ensemble-Methods-in-Sklearn](04-Ensemble-Methods-in-Sklearn.md)*
