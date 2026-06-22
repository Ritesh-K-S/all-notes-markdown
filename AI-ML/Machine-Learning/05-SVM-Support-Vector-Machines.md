# Chapter 05: Support Vector Machines (SVM)

## Table of Contents
- [1. Introduction to SVM](#1-introduction-to-svm)
- [2. The Intuition — Maximum Margin Classifier](#2-the-intuition--maximum-margin-classifier)
- [3. Hard Margin vs Soft Margin](#3-hard-margin-vs-soft-margin)
- [4. The Mathematics Behind SVM](#4-the-mathematics-behind-svm)
- [5. The Kernel Trick](#5-the-kernel-trick)
- [6. Popular Kernels Explained](#6-popular-kernels-explained)
- [7. SVM for Multi-Class Classification](#7-svm-for-multi-class-classification)
- [8. SVM for Regression (SVR)](#8-svm-for-regression-svr)
- [9. Practical Implementation](#9-practical-implementation)
- [10. Hyperparameter Tuning](#10-hyperparameter-tuning)
- [11. SVM vs Other Algorithms](#11-svm-vs-other-algorithms)
- [12. Common Mistakes](#12-common-mistakes)
- [13. Interview Questions](#13-interview-questions)
- [14. Quick Reference](#14-quick-reference)

---

## 1. Introduction to SVM

### What It Is
A Support Vector Machine finds the **best boundary** (called a hyperplane) that separates different classes with the **maximum possible gap** (margin) between them.

**Analogy for a 15-year-old**: Imagine you have red and blue balls on a table. You want to place a ruler (line) between them so that the closest balls on either side are as FAR from the ruler as possible. SVM finds that "best ruler position" — and it works even in spaces you can't see (higher dimensions)!

### Why It Matters
- **Excellent for high-dimensional data** (text classification, genomics)
- **Memory efficient** — only stores support vectors, not all data
- **Works well with small-to-medium datasets**
- **Theoretically elegant** — strong mathematical foundations (convex optimization)
- **Still competitive** for many tasks despite deep learning hype

### When to Use SVM

| Use SVM When... | Don't Use SVM When... |
|---|---|
| High-dimensional data (features >> samples) | Very large datasets (>100K samples) |
| Clear margin of separation exists | Lots of noise/overlapping classes |
| Text classification / NLP tasks | Need probability estimates (use calibration) |
| Small-to-medium dataset | Need interpretability |
| Binary classification | Very large feature space without kernel |

---

## 2. The Intuition — Maximum Margin Classifier

### The Key Idea

There are infinitely many lines that can separate two classes. SVM picks the ONE that maximizes the margin.

```
Many possible separating lines:         SVM's choice (maximum margin):

  ○ ○                                    ○ ○
  ○  ╱  ●                               ○     |     ●
   ○╱ ●  ●                               ○    |   ●  ●
   ╱○  ●                                  ○   |  ●
  ╱ ○    ●  ●                              ○  |    ●  ●
                                        ←margin→
                                        (maximized!)
```

### Why Maximum Margin?

- **Better generalization**: The wider the margin, the less likely new data falls on the wrong side
- **Theoretical guarantee**: Larger margin → lower VC dimension → lower generalization error
- **Robustness**: Small perturbations in data don't change the decision boundary

### Support Vectors

The data points **closest** to the decision boundary are called **Support Vectors**. They "support" (define) the hyperplane.

```
         ○                  ← regular points (don't matter)
         ○
         ○ ←─── Support Vector (on the margin boundary)
    ─────────────────── ← margin boundary
    ═══════════════════ ← DECISION BOUNDARY (hyperplane)
    ─────────────────── ← margin boundary
              ● ←────── Support Vector (on the margin boundary)
              ●
              ●         ← regular points (don't matter)
```

> **Critical Insight**: If you remove any non-support-vector point, the decision boundary stays EXACTLY the same. Only support vectors determine the boundary. This is why SVM is memory-efficient — it only needs to "remember" the support vectors.

---

## 3. Hard Margin vs Soft Margin

### Hard Margin SVM
- **Requires**: Data is perfectly linearly separable
- **No misclassification** allowed
- **Problem**: Fails if even ONE point is on the wrong side
- **Extremely sensitive** to outliers

```
Hard Margin (fails with outlier):

  ○ ○                      ○ ○
  ○ ○ ●   ← outlier!      ○ ○ ←── this ● makes it
  ○   ● ●                  ○   ● ●    impossible to
  ○   ● ●                  ○   ● ●    find a clean line!
```

### Soft Margin SVM (C-SVM)

Allows some points to violate the margin or even be misclassified. Controlled by parameter **C**.

$$\text{Objective: Minimize } \frac{1}{2}||w||^2 + C \sum_{i=1}^{n} \xi_i$$

Where:
- $\frac{1}{2}||w||^2$ = inverse of margin (smaller = wider margin)
- $\xi_i$ = slack variable (how much point $i$ violates the margin)
- $C$ = tradeoff parameter

### The C Parameter — Intuition

```
Low C (C=0.01):                    High C (C=1000):
Wide margin, tolerates errors      Narrow margin, few errors

  ○ ○     ● ●                     ○ ○    ● ●
  ○ ○  |  ● ●                     ○ ○|   ● ●
  ○ ●  |  ● ●  ← allows this!    ○  |●  ● ●  ← tries hard to
  ○ ○  |  ● ●                     ○ ○|   ● ●     avoid this
       ↑                               ↑
  Wide margin                      Narrow margin
  (more generalization)            (risk of overfitting)
```

| C Value | Margin | Misclassifications | Bias | Variance |
|---------|--------|-------------------|------|----------|
| Small (0.01) | Wide | More allowed | High | Low |
| Large (1000) | Narrow | Few allowed | Low | High |

> **Pro Tip**: Think of C as a "strictness" parameter. Small C = relaxed teacher (allows mistakes). Large C = strict teacher (punishes every mistake).

---

## 4. The Mathematics Behind SVM

### The Optimization Problem

**Goal**: Find hyperplane $w^T x + b = 0$ that maximizes margin $\frac{2}{||w||}$

**Primal Form** (Hard Margin):

$$\min_{w, b} \frac{1}{2} ||w||^2$$
$$\text{subject to: } y_i(w^T x_i + b) \geq 1, \quad \forall i$$

**Primal Form** (Soft Margin):

$$\min_{w, b, \xi} \frac{1}{2} ||w||^2 + C \sum_{i=1}^{n} \xi_i$$
$$\text{subject to: } y_i(w^T x_i + b) \geq 1 - \xi_i, \quad \xi_i \geq 0, \quad \forall i$$

### Dual Form (via Lagrangian)

Using KKT conditions, we convert to the dual:

$$\max_{\alpha} \sum_{i=1}^{n} \alpha_i - \frac{1}{2} \sum_{i=1}^{n} \sum_{j=1}^{n} \alpha_i \alpha_j y_i y_j (x_i^T x_j)$$

$$\text{subject to: } 0 \leq \alpha_i \leq C, \quad \sum_{i=1}^{n} \alpha_i y_i = 0$$

**Key insight**: The dual form only involves **dot products** $x_i^T x_j$ — this is what enables the Kernel Trick!

### Decision Function

$$f(x) = \text{sign}\left(\sum_{i \in SV} \alpha_i y_i (x_i^T x) + b\right)$$

Where $SV$ = set of support vectors (points where $\alpha_i > 0$).

### Geometric Interpretation

```
                        w^T x + b = +1 (positive margin)
                       /
      ○  ○           / 
      ○    ○        /  ← margin = 2/||w||
        ○    ○     /
    ──────────────/──── w^T x + b = 0 (decision boundary)
               ● /
             ●  /● 
           ●   / ●
              /
             / w^T x + b = -1 (negative margin)
```

---

## 5. The Kernel Trick

### The Problem: Non-Linear Data

Real-world data is rarely linearly separable:

```
  Linear (SVM works directly):    Non-Linear (need kernel!):
  
  ○ ○ ○ | ● ● ●                     ● ● ● ●
  ○ ○ ○ | ● ● ●                   ● ○ ○ ○ ○ ●
  ○ ○ ○ | ● ● ●                   ● ○ ○ ○ ○ ●
                                     ● ● ● ●
                                   (circle inside circle!)
```

### The Intuition

**Analogy**: Imagine red and blue balls mixed on a table (2D) — you can't draw a straight line to separate them. But if you LIFT some balls up (add a 3rd dimension based on distance from center), suddenly you CAN separate them with a flat plane in 3D!

```
2D (not separable):          3D after mapping (separable!):
                                        ● ●
  ● ○ ○ ●                        ●  ─────────  ●  ← plane separates!
  ○ ○ ○ ○                           ○ ○ ○ ○
  ● ○ ○ ●                    
                              z = x² + y² (mapping function)
```

### The Mathematical Trick

Instead of explicitly computing the mapping $\phi(x)$ to higher dimensions, we use a **kernel function** that computes the dot product in the higher-dimensional space directly:

$$K(x_i, x_j) = \phi(x_i)^T \phi(x_j)$$

**Why this works**: The dual form only needs dot products $x_i^T x_j$. We replace these with $K(x_i, x_j)$ — computing the result WITHOUT ever transforming the data!

**Example**: Polynomial mapping for 2D data:
- $x = [x_1, x_2]$
- $\phi(x) = [x_1^2, \sqrt{2}x_1 x_2, x_2^2]$ (maps to 3D)
- Computing $\phi(x_i)^T \phi(x_j)$ = $(x_i^T x_j)^2$ = just squaring the dot product!

The kernel "secretly" works in a potentially infinite-dimensional space without the computational cost.

---

## 6. Popular Kernels Explained

### 6.1 Linear Kernel

$$K(x_i, x_j) = x_i^T x_j$$

- No transformation — equivalent to standard linear SVM
- **Use when**: Data is linearly separable, high-dimensional (text), many features

### 6.2 Polynomial Kernel

$$K(x_i, x_j) = (\gamma \cdot x_i^T x_j + r)^d$$

- Maps to polynomial feature space of degree $d$
- Parameters: $d$ (degree), $\gamma$ (scale), $r$ (coefficient)
- **Use when**: Interaction between features matters

### 6.3 RBF (Radial Basis Function) / Gaussian Kernel

$$K(x_i, x_j) = \exp\left(-\gamma ||x_i - x_j||^2\right)$$

- Maps to INFINITE-dimensional space
- Measures "similarity" — closer points have higher kernel value
- $\gamma$ controls the "reach" of each training point
- **Most popular kernel** — works well in most cases

### Understanding Gamma ($\gamma$) in RBF

```
Small γ (γ=0.01):                  Large γ (γ=100):
Each point has WIDE influence       Each point has NARROW influence
Smooth boundary                     Complex boundary (overfits)

  ○○○     ●●●                       ○○○     ●●●
  ○○ ╲    ●●●                       ○○╲ ╱╲  ●●●
  ○○  ╲   ●●●                       ○○ ╳  ╲ ●●●
       smooth                            wiggly
```

| γ Value | Influence Radius | Boundary | Risk |
|---------|-----------------|----------|------|
| Small | Wide (far points matter) | Smooth | Underfitting |
| Large | Narrow (only close points) | Complex | Overfitting |

### 6.4 Sigmoid Kernel

$$K(x_i, x_j) = \tanh(\gamma \cdot x_i^T x_j + r)$$

- Equivalent to a two-layer neural network
- Rarely used in practice (RBF is almost always better)

### Kernel Selection Guide

```
Start here:
│
├── Is data linearly separable?
│   ├── Yes → Linear Kernel
│   └── Not sure → Try Linear first (fast), then RBF
│
├── Very high dimensional (text, genomics)?
│   └── Linear Kernel (RBF too slow, linear often works)
│
├── Medium dimensions, non-linear patterns?
│   └── RBF Kernel (default choice)
│
└── Need polynomial interactions?
    └── Polynomial Kernel (degree 2-3)
```

---

## 7. SVM for Multi-Class Classification

SVM is inherently binary. For multi-class, two strategies:

### One-vs-Rest (OvR) — Default in sklearn

- Train $K$ binary classifiers (one per class vs all others)
- For class $k$: treat class $k$ as positive, everything else as negative
- Prediction: class with highest decision function value

```
3 classes (A, B, C):
  Classifier 1: A vs (B+C)
  Classifier 2: B vs (A+C)
  Classifier 3: C vs (A+B)
  
  New sample → run all 3 → pick highest confidence
```

### One-vs-One (OvO)

- Train $\frac{K(K-1)}{2}$ binary classifiers (one for each pair)
- Prediction: majority vote
- More classifiers but each trained on less data (faster per classifier)

```
3 classes (A, B, C):
  Classifier 1: A vs B
  Classifier 2: A vs C
  Classifier 3: B vs C
  
  New sample → run all 3 → majority vote
```

| Strategy | # Classifiers | Training Speed | Use When |
|----------|--------------|----------------|----------|
| OvR | $K$ | Faster overall | Default / large datasets |
| OvO | $K(K-1)/2$ | Faster per classifier | Small datasets, kernel SVM |

> **Note**: `sklearn.svm.SVC` uses OvO by default. `sklearn.svm.LinearSVC` uses OvR.

---

## 8. SVM for Regression (SVR)

### How It Works

Instead of maximizing margin between classes, SVR fits a tube of width $\epsilon$ around the data and tries to keep all points INSIDE the tube.

$$\text{Minimize: } \frac{1}{2}||w||^2 + C \sum_{i=1}^{n} (\xi_i + \xi_i^*)$$
$$\text{subject to: } |y_i - (w^T x_i + b)| \leq \epsilon + \xi_i$$

```
                    ●
        ╭─────────────────────╮  ← ε-tube (insensitive zone)
   ●    │   ●    ●    ●      │
        │ ────────────────────│── ← regression line
   ●    │      ●     ●       │
        ╰─────────────────────╯
              ●                    ← points outside tube are penalized
```

**Key parameters**:
- $\epsilon$ = width of the insensitive tube (errors within $\epsilon$ are ignored)
- $C$ = penalty for points outside the tube
- Kernel = same as classification

```python
from sklearn.svm import SVR
import numpy as np
import matplotlib.pyplot as plt

# Generate non-linear data
np.random.seed(42)
X = np.sort(5 * np.random.rand(100, 1), axis=0)
y = np.sin(X).ravel() + np.random.normal(0, 0.1, X.shape[0])

# Fit SVR with different kernels
kernels = ['linear', 'rbf', 'poly']
fig, axes = plt.subplots(1, 3, figsize=(18, 5))

for ax, kernel in zip(axes, kernels):
    svr = SVR(kernel=kernel, C=100, epsilon=0.1, gamma='scale')
    svr.fit(X, y)
    
    X_plot = np.linspace(0, 5, 500).reshape(-1, 1)
    y_pred = svr.predict(X_plot)
    
    ax.scatter(X, y, s=20, alpha=0.5, label='Data')
    ax.plot(X_plot, y_pred, 'r-', linewidth=2, label='SVR')
    ax.set_title(f'SVR (kernel={kernel})')
    ax.legend()
    
    # Show support vectors
    sv_idx = svr.support_
    ax.scatter(X[sv_idx], y[sv_idx], s=100, facecolors='none', 
               edgecolors='red', linewidth=2, label='Support Vectors')

plt.tight_layout()
plt.show()
```

---

## 9. Practical Implementation

### Complete Classification Pipeline

```python
from sklearn.svm import SVC
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.datasets import load_breast_cancer
from sklearn.metrics import classification_report, confusion_matrix
import numpy as np

# Load data
data = load_breast_cancer()
X, y = data.data, data.target
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# CRITICAL: SVM requires feature scaling!
# Use a pipeline to ensure scaling is done correctly
svm_pipeline = Pipeline([
    ('scaler', StandardScaler()),    # Scale features to mean=0, std=1
    ('svm', SVC(
        kernel='rbf',               # RBF kernel (most versatile)
        C=1.0,                      # Regularization parameter
        gamma='scale',              # 1 / (n_features * X.var())
        probability=True,           # Enable probability estimates
        random_state=42
    ))
])

# Train
svm_pipeline.fit(X_train, y_train)

# Evaluate
print(f"Training Accuracy: {svm_pipeline.score(X_train, y_train):.4f}")
print(f"Test Accuracy: {svm_pipeline.score(X_test, y_test):.4f}")

# Cross-validation
cv_scores = cross_val_score(svm_pipeline, X, y, cv=5, scoring='accuracy')
print(f"CV Score: {cv_scores.mean():.4f} ± {cv_scores.std():.4f}")

# Detailed report
y_pred = svm_pipeline.predict(X_test)
print("\nClassification Report:")
print(classification_report(y_test, y_pred, target_names=data.target_names))

# Get probability estimates (useful for threshold tuning)
y_proba = svm_pipeline.predict_proba(X_test)[:5]
print("\nProbability estimates (first 5 samples):")
print(y_proba)
```

### Linear SVM for Large Datasets

```python
from sklearn.svm import LinearSVC
from sklearn.calibration import CalibratedClassifierCV
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline

# LinearSVC is MUCH faster than SVC(kernel='linear') for large datasets
# It uses liblinear instead of libsvm
linear_svm = Pipeline([
    ('scaler', StandardScaler()),
    ('svm', LinearSVC(
        C=1.0,
        loss='hinge',        # Standard SVM loss
        max_iter=10000,      # Increase if convergence warning
        dual=True,           # True when n_samples > n_features
        random_state=42
    ))
])

linear_svm.fit(X_train, y_train)
print(f"LinearSVC Accuracy: {linear_svm.score(X_test, y_test):.4f}")

# LinearSVC doesn't have predict_proba — use calibration wrapper
calibrated_svm = Pipeline([
    ('scaler', StandardScaler()),
    ('svm', CalibratedClassifierCV(
        LinearSVC(C=1.0, random_state=42),
        cv=5
    ))
])
calibrated_svm.fit(X_train, y_train)
probas = calibrated_svm.predict_proba(X_test)[:3]
print(f"Calibrated probabilities:\n{probas}")
```

### Text Classification with SVM (Real-World Use Case)

```python
from sklearn.svm import LinearSVC
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.pipeline import Pipeline
from sklearn.datasets import fetch_20newsgroups
from sklearn.metrics import classification_report

# Load text data
categories = ['sci.med', 'sci.space', 'rec.sport.baseball', 'talk.politics.guns']
train_data = fetch_20newsgroups(subset='train', categories=categories)
test_data = fetch_20newsgroups(subset='test', categories=categories)

# SVM excels at high-dimensional sparse data (text!)
text_clf = Pipeline([
    ('tfidf', TfidfVectorizer(
        max_features=10000,      # Top 10K words
        stop_words='english',
        ngram_range=(1, 2)       # Unigrams and bigrams
    )),
    ('svm', LinearSVC(C=1.0, random_state=42))
])

text_clf.fit(train_data.data, train_data.target)

y_pred = text_clf.predict(test_data.data)
print(classification_report(test_data.target, y_pred, 
                           target_names=train_data.target_names))

# Typically achieves >90% accuracy on text classification!
```

---

## 10. Hyperparameter Tuning

### The C-Gamma Grid Search

```python
from sklearn.model_selection import GridSearchCV
from sklearn.svm import SVC
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
import numpy as np

# Define pipeline (always include scaling!)
pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('svm', SVC(random_state=42))
])

# Parameter grid — logarithmic scale for C and gamma
param_grid = {
    'svm__kernel': ['rbf', 'linear'],
    'svm__C': [0.01, 0.1, 1, 10, 100, 1000],
    'svm__gamma': ['scale', 'auto', 0.001, 0.01, 0.1, 1],
}

# Grid Search with cross-validation
grid_search = GridSearchCV(
    pipe,
    param_grid,
    cv=5,
    scoring='accuracy',
    n_jobs=-1,
    verbose=1,
    refit=True          # Retrain on full data with best params
)

grid_search.fit(X_train, y_train)

print(f"Best Parameters: {grid_search.best_params_}")
print(f"Best CV Score: {grid_search.best_score_:.4f}")
print(f"Test Score: {grid_search.score(X_test, y_test):.4f}")

# Visualize C vs Gamma heatmap (for RBF only)
import matplotlib.pyplot as plt
import pandas as pd

results = pd.DataFrame(grid_search.cv_results_)
rbf_results = results[results['param_svm__kernel'] == 'rbf']
```

### Tuning Strategy Guide

```
Step 1: Try LinearSVC first (fast baseline)
Step 2: If non-linear needed, use SVC(kernel='rbf') 
Step 3: Coarse grid search on C and gamma (log scale: 10^-3 to 10^3)
Step 4: Fine grid search around best values
Step 5: Final model with best params on full training data
```

---

## 11. SVM vs Other Algorithms

| Aspect | SVM | Logistic Regression | Random Forest | Neural Networks |
|--------|-----|--------------------:|:--------------|:----------------|
| **Best for** | High-dim, small data | Linear problems, proba | Tabular, any size | Huge data, complex |
| **Interpretability** | Low (kernel) | High | Medium | Very Low |
| **Feature Scaling** | Required! | Recommended | Not needed | Required |
| **Speed (training)** | Slow (>10K samples) | Fast | Medium | Slow |
| **Speed (prediction)** | Fast (only SVs) | Very Fast | Medium | Varies |
| **Handles non-linearity** | Kernels | No (needs features) | Naturally | Naturally |
| **Probability output** | Indirect (Platt) | Direct | Direct (averaged) | Direct (softmax) |
| **Memory** | Efficient (SVs only) | All weights | All trees | All weights |
| **Hyperparameters** | C, gamma, kernel | C | Many but robust | Many, tricky |

### When SVM Still Wins (2024+)

1. **Text classification** with TF-IDF (high-dim, sparse)
2. **Small datasets** (<10K samples) with clear margins
3. **Bioinformatics** (gene expression, protein classification)
4. **Image classification** with hand-crafted features (HOG, SIFT)
5. When you need a **strong baseline** with minimal tuning

---

## 12. Common Mistakes

### Mistake 1: Forgetting to Scale Features
```python
# BAD — SVM will be dominated by large-scale features!
svm = SVC(kernel='rbf')
svm.fit(X_train, y_train)  # Feature 1: [0-1], Feature 2: [0-100000]

# GOOD — Always scale!
from sklearn.preprocessing import StandardScaler
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)  # Use SAME scaler!
svm.fit(X_train_scaled, y_train)
```

> **This is the #1 SVM mistake.** Trees don't need scaling, but SVM absolutely does.

### Mistake 2: Using SVC on Large Datasets
```python
# BAD — SVC is O(n²) to O(n³) in training time!
svm = SVC(kernel='rbf')
svm.fit(X_100k_samples, y)  # Will take forever

# GOOD — Use LinearSVC or SGDClassifier for large data
from sklearn.linear_model import SGDClassifier
sgd_svm = SGDClassifier(loss='hinge', alpha=0.001, random_state=42)
sgd_svm.fit(X_100k_samples, y)  # Fast! Uses SGD optimization
```

### Mistake 3: Not Tuning Gamma
```python
# BAD — Default gamma might underfit or overfit
svm = SVC(kernel='rbf', C=1.0)  # gamma='scale' is default

# GOOD — Always tune C and gamma together
from sklearn.model_selection import GridSearchCV
param_grid = {'C': [0.1, 1, 10, 100], 'gamma': [0.001, 0.01, 0.1, 1]}
grid = GridSearchCV(SVC(kernel='rbf'), param_grid, cv=5)
grid.fit(X_train_scaled, y_train)
```

### Mistake 4: Using probability=True Unnecessarily
```python
# BAD — probability=True uses Platt scaling (5-fold CV internally), much slower!
svm = SVC(kernel='rbf', probability=True)

# GOOD — Only enable when you actually need probabilities
svm = SVC(kernel='rbf')  # Faster training
# Add probability later only if needed:
# from sklearn.calibration import CalibratedClassifierCV
```

### Mistake 5: Applying Kernel SVM to Very High-Dimensional Sparse Data
```python
# BAD — RBF kernel on text data (10K+ features) is slow and unnecessary
svm = SVC(kernel='rbf')
svm.fit(tfidf_matrix, y)  # Slow and often worse than linear!

# GOOD — Linear kernel for high-dimensional sparse data
svm = LinearSVC(C=1.0)
svm.fit(tfidf_matrix, y)  # Fast and effective
```

### Mistake 6: Data Leakage with Scaling
```python
# BAD — Fitting scaler on all data (leaks test info into training)
scaler.fit(X)  # Sees test data!
X_scaled = scaler.transform(X)

# GOOD — Fit scaler ONLY on training data
scaler.fit(X_train)
X_train_scaled = scaler.transform(X_train)
X_test_scaled = scaler.transform(X_test)

# BEST — Use Pipeline (handles this automatically)
pipe = Pipeline([('scaler', StandardScaler()), ('svm', SVC())])
```

---

## 13. Interview Questions

### Conceptual Questions

**Q1: What are Support Vectors?**
> Support vectors are the data points closest to the decision boundary (on the margin). They are the ONLY points that determine the hyperplane. Removing any non-support-vector point doesn't change the model. In practice, typically 10-30% of training points are support vectors.

**Q2: Explain the Kernel Trick in simple terms.**
> The kernel trick lets SVM work in high-dimensional spaces without actually transforming the data. Since the SVM optimization only needs dot products between pairs of points, we replace $x_i^T x_j$ with $K(x_i, x_j)$ — a function that computes what the dot product WOULD be in the higher-dimensional space. This gives us non-linear boundaries while keeping computation manageable.

**Q3: What happens when C is very large vs very small?**
> Large C: SVM tries hard to classify ALL training points correctly → narrow margin, complex boundary, risk of overfitting. Small C: SVM allows more misclassifications → wider margin, simpler boundary, risk of underfitting. C controls the bias-variance tradeoff.

**Q4: Why does SVM need feature scaling?**
> SVM uses distances (margins, dot products). If Feature A ranges [0,1] and Feature B ranges [0,1000], Feature B dominates the distance calculations. Scaling ensures all features contribute equally to the margin maximization.

**Q5: What's the difference between SVC and LinearSVC in sklearn?**
> `SVC(kernel='linear')` uses libsvm (quadratic programming, O(n²+)). `LinearSVC` uses liblinear (optimized for linear case, much faster for large datasets). For linear kernels with >10K samples, always use LinearSVC.

**Q6: Can SVM do multi-class classification? How?**
> SVM is inherently binary. Multi-class is done via: (1) One-vs-Rest (OvR): K classifiers, each one class vs all others; (2) One-vs-One (OvO): K(K-1)/2 classifiers, each pair. sklearn SVC uses OvO by default, LinearSVC uses OvR.

**Q7: How do you choose between Linear and RBF kernel?**
> Start with linear (fast, works well for high-dim/sparse data like text). If linear is insufficient, try RBF. If n_features >> n_samples, linear is usually enough. If n_samples >> n_features and data is non-linear, RBF is better. Always validate with cross-validation.

**Q8: What is the relationship between SVM and Logistic Regression?**
> Both find a linear decision boundary. LR minimizes log-loss (cross-entropy), SVM minimizes hinge loss. SVM focuses only on points near the boundary (support vectors), LR considers all points. SVM gives hard margins, LR gives calibrated probabilities. Mathematically, SVM with linear kernel and LR with L2 regularization are closely related (both minimize $||w||$ + loss).

**Q9: How does SVM handle imbalanced classes?**
> Use `class_weight='balanced'` which sets $C_k = C \times \frac{n_{samples}}{n_{classes} \times n_{samples_k}}$. This makes misclassifying the minority class more expensive. Alternatively, manually set per-class weights.

**Q10: What's the time complexity of SVM?**
> Training: O(n² · p) to O(n³) for kernel SVM (libsvm), O(n · p) for LinearSVC. Prediction: O(n_sv · p) for kernel SVM (need to compute kernel with all support vectors), O(p) for linear. This is why kernel SVM doesn't scale to very large datasets.

---

## 14. Quick Reference

### SVM Cheat Sheet

| Aspect | Detail |
|--------|--------|
| Type | Supervised (Classification & Regression) |
| Core Idea | Maximum margin hyperplane |
| Feature Scaling | **REQUIRED** (StandardScaler) |
| Key Parameters | C (regularization), gamma (RBF radius), kernel |
| Default Kernel | RBF (most versatile) |
| Training Complexity | O(n²) to O(n³) for SVC; O(n) for LinearSVC |
| Prediction Complexity | O(n_sv · features) |
| Memory | Only stores support vectors |
| Probability Output | Indirect (Platt scaling, slow) |
| Handles Non-linear | Yes (via kernels) |
| Handles Multi-class | OvO (SVC) or OvR (LinearSVC) |

### Parameter Guide

| Parameter | Low Value → | High Value → |
|-----------|-------------|--------------|
| **C** | Wide margin, more errors (underfit) | Narrow margin, fewer errors (overfit) |
| **gamma** | Wide influence, smooth boundary | Narrow influence, complex boundary |
| **epsilon** (SVR) | Tight tube, more SVs | Wide tube, fewer SVs |
| **degree** (poly) | Simpler boundary | More complex boundary |

### When to Use Which SVM Variant

```
Dataset size < 10K samples → SVC (kernel SVM)
Dataset size 10K-100K → LinearSVC or SGDClassifier(loss='hinge')
Dataset size > 100K → SGDClassifier(loss='hinge')
Text/sparse data → LinearSVC (always)
Non-linear, small data → SVC(kernel='rbf')
Need probabilities → SVC(probability=True) or CalibratedClassifierCV
Regression → SVR(kernel='rbf')
```

### Essential Import Patterns

```python
# Classification
from sklearn.svm import SVC           # Kernel SVM (small-medium data)
from sklearn.svm import LinearSVC     # Linear SVM (large data)

# Regression
from sklearn.svm import SVR           # Kernel SVR
from sklearn.svm import LinearSVR     # Linear SVR

# Fast SVM via SGD (very large data)
from sklearn.linear_model import SGDClassifier  # loss='hinge' → SVM

# Always needed with SVM
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
```

---

*Previous: [04 - Decision Trees and Random Forest](04-Decision-Trees-and-Random-Forest.md)*  
*Next: [06 - KNN and Naive Bayes](06-KNN-and-Naive-Bayes.md)*
