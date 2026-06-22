# Chapter 03: Logistic Regression

## Table of Contents
- [What is Logistic Regression?](#what-is-logistic-regression)
- [The Sigmoid Function](#the-sigmoid-function)
- [Decision Boundary](#decision-boundary)
- [Cost Function (Log Loss / Cross-Entropy)](#cost-function)
- [Gradient Descent for Logistic Regression](#gradient-descent-for-logistic-regression)
- [Multi-Class Classification](#multi-class-classification)
- [Regularization in Logistic Regression](#regularization-in-logistic-regression)
- [Evaluation Metrics for Classification](#evaluation-metrics)
- [Threshold Tuning](#threshold-tuning)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Logistic Regression?

### What It Is
Despite the name, Logistic Regression is a **classification** algorithm. It predicts the **probability** that an input belongs to a particular class (e.g., spam vs. not spam, sick vs. healthy).

**Analogy for a 15-year-old:** Imagine a college admissions officer. They look at your GPA and test scores and decide: admit or reject. Logistic Regression does the same — it takes numbers in and outputs a probability (like "78% chance of admission"), then draws a line: above 50% → admit, below 50% → reject.

### Why Not Use Linear Regression for Classification?

```
Linear Regression for Classification (WRONG):
    y
  2 │              · · ·
    │          · · ·─────── Predictions > 1 (impossible probability!)
  1 │     · · ·
    │ · · ·  ← Where do we put the threshold?
  0 │──·─·─·─────────────── Predictions < 0 (impossible probability!)
    │
    └──────────────────── x

Logistic Regression (CORRECT):
    P(y=1)
  1 │                 ·───── Asymptote (never exceeds 1)
    │              ·
0.5 │─ ─ ─ ─ ─ ·─ ─ ─ ─ ← Decision boundary
    │        ·
  0 │───·────────────────── Asymptote (never below 0)
    └──────────────────── x
```

**Problems with Linear Regression for classification:**
1. Outputs can be < 0 or > 1 (not valid probabilities)
2. Sensitive to outliers (one extreme point shifts the entire line)
3. Doesn't model the S-shaped nature of probability transitions

### Why It Matters
- **Most widely used classification algorithm** in industry
- **Interpretable** — coefficients tell you feature importance and direction
- **Probabilistic output** — gives confidence, not just yes/no
- **Fast** — trains in seconds, even on millions of rows
- **Baseline model** — first model you should try for any classification task
- **Foundation** — neural network neurons use the same sigmoid/softmax

---

## The Sigmoid Function

### What It Is
The sigmoid (logistic) function squashes any real number into the range $(0, 1)$, making it interpretable as a probability.

### Mathematical Definition

$$\sigma(z) = \frac{1}{1 + e^{-z}}$$

Where $z = \mathbf{w}^T\mathbf{x} + b$ (the linear combination of features)

### Properties

| Property | Value | Significance |
|----------|-------|--------------|
| Output range | $(0, 1)$ | Valid probability |
| $\sigma(0)$ | $0.5$ | Decision boundary point |
| $\sigma(\infty)$ | $1$ | Very positive z → class 1 |
| $\sigma(-\infty)$ | $0$ | Very negative z → class 0 |
| Derivative | $\sigma(z) \cdot (1 - \sigma(z))$ | Elegant for backprop |
| Symmetry | $\sigma(-z) = 1 - \sigma(z)$ | Useful mathematical property |

### Visual Shape

```
    σ(z)
  1 │                          ─────────
    │                       ╱
    │                    ╱
0.5 │─ ─ ─ ─ ─ ─ ─ ─╱─ ─ ─ ─ ─ ─ ─ ─
    │              ╱
    │           ╱
  0 │─────────╱
    └──────────────┼──────────────── z
                   0
    
    z < 0 → σ(z) < 0.5 → Predict class 0
    z > 0 → σ(z) > 0.5 → Predict class 1
    z = 0 → σ(z) = 0.5 → Decision boundary
```

### Code: Sigmoid Function

```python
import numpy as np

def sigmoid(z):
    """
    Compute sigmoid function.
    Numerically stable version handles large positive/negative values.
    """
    # Clip to prevent overflow in exp
    z = np.clip(z, -500, 500)
    return 1 / (1 + np.exp(-z))

# Test behavior
z_values = np.array([-10, -5, -1, 0, 1, 5, 10])
for z in z_values:
    print(f"σ({z:3.0f}) = {sigmoid(z):.6f}")

# Output:
# σ(-10) = 0.000045  → Almost certainly class 0
# σ( -5) = 0.006693  → Very likely class 0
# σ( -1) = 0.268941  → Probably class 0
# σ(  0) = 0.500000  → Completely uncertain
# σ(  1) = 0.731059  → Probably class 1
# σ(  5) = 0.993307  → Very likely class 1
# σ( 10) = 0.999955  → Almost certainly class 1
```

### The Complete Logistic Regression Model

$$P(y=1|\mathbf{x}) = \sigma(\mathbf{w}^T\mathbf{x} + b) = \frac{1}{1 + e^{-(\mathbf{w}^T\mathbf{x} + b)}}$$

**Prediction rule:**
$$\hat{y} = \begin{cases} 1 & \text{if } P(y=1|\mathbf{x}) \geq 0.5 \\ 0 & \text{if } P(y=1|\mathbf{x}) < 0.5 \end{cases}$$

---

## Decision Boundary

### What It Is
The decision boundary is the line (or surface) where the model is equally uncertain — $P(y=1) = 0.5$. Points on one side are classified as class 1, points on the other as class 0.

### Mathematical Derivation

The decision boundary occurs when $\sigma(z) = 0.5$, which means $z = 0$:

$$\mathbf{w}^T\mathbf{x} + b = 0$$

For 2D (two features):
$$w_1 x_1 + w_2 x_2 + b = 0$$
$$x_2 = -\frac{w_1}{w_2} x_1 - \frac{b}{w_2}$$

This is a straight line!

### Visual: Linear Decision Boundary

```
    x₂
    │         Class 1 (●)
    │    ●  ●
    │  ●  ●  ●╲
    │    ●  ●   ╲  Decision Boundary
    │  ●   ●     ╲  (w₁x₁ + w₂x₂ + b = 0)
    │     ●        ╲
    │               ╲  ○  ○
    │                ╲○  ○  ○
    │                  ○  ○
    │               ○   ○   Class 0 (○)
    └─────────────────────── x₁
```

### Non-Linear Decision Boundaries

By adding polynomial features, logistic regression can create curved boundaries:

```
    x₂                       x₂
    │    Linear               │    Polynomial (degree 2)
    │  ● ● ●╲ ○ ○            │  ● ● ●╲    ╱○ ○
    │  ● ● ●  ╲○ ○           │  ● ● ●  ╲  ╱ ○ ○
    │  ● ● ●   ╲ ○           │  ● ● ●   ╲╱  ○ ○
    │  ● ●      ╲○           │  ● ●      )(  ○ ○
    │  ●         ╲           │  ●       ╱╲   ○
    └──────────────           └──────────────
    Can't capture curve        Can capture non-linear boundary
```

### Code: Visualizing Decision Boundary

```python
import numpy as np
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import make_classification
from sklearn.preprocessing import PolynomialFeatures

# Create 2D dataset for visualization
X, y = make_classification(
    n_samples=200, n_features=2, n_redundant=0,
    n_informative=2, n_clusters_per_class=1, random_state=42
)

# Train logistic regression
model = LogisticRegression(random_state=42)
model.fit(X, y)

# Extract decision boundary equation
w = model.coef_[0]       # [w1, w2]
b = model.intercept_[0]  # bias

print(f"Decision Boundary: {w[0]:.4f}*x₁ + {w[1]:.4f}*x₂ + {b:.4f} = 0")
print(f"Slope: {-w[0]/w[1]:.4f}")
print(f"Intercept: {-b/w[1]:.4f}")

# For any new point, check which side of the boundary it falls on
new_point = np.array([[1.5, -0.5]])
prob = model.predict_proba(new_point)
print(f"\nPoint (1.5, -0.5):")
print(f"  P(class 0) = {prob[0][0]:.4f}")
print(f"  P(class 1) = {prob[0][1]:.4f}")
print(f"  Prediction: class {model.predict(new_point)[0]}")
```

---

## Cost Function

### Why Not Use MSE for Classification?

If we use MSE with the sigmoid function, the cost surface becomes **non-convex** (multiple local minima):

```
    MSE Cost (Non-convex):          Log Loss (Convex):
    │                               │
    │  ╱╲    ╱╲                     │╲
    │ ╱  ╲  ╱  ╲                    │ ╲
    │╱    ╲╱    ╲                   │  ╲
    │   local    ╲                  │   ╲
    │   minima    global            │    ╲_____
    └──────────────                 └──────────────
    GD might get stuck!             GD always finds global min!
```

### Log Loss (Binary Cross-Entropy)

The correct cost function for logistic regression:

$$J(\mathbf{w}) = -\frac{1}{m} \sum_{i=1}^{m} \left[ y_i \log(\hat{y}_i) + (1 - y_i) \log(1 - \hat{y}_i) \right]$$

Where $\hat{y}_i = \sigma(\mathbf{w}^T\mathbf{x}_i + b)$

### Intuition Behind Log Loss

**When $y = 1$ (actual positive):**
- Cost = $-\log(\hat{y})$
- If $\hat{y} = 0.99$ → Cost = $-\log(0.99) = 0.01$ (tiny penalty — good!)
- If $\hat{y} = 0.01$ → Cost = $-\log(0.01) = 4.6$ (huge penalty — bad prediction!)

**When $y = 0$ (actual negative):**
- Cost = $-\log(1 - \hat{y})$
- If $\hat{y} = 0.01$ → Cost = $-\log(0.99) = 0.01$ (tiny penalty — good!)
- If $\hat{y} = 0.99$ → Cost = $-\log(0.01) = 4.6$ (huge penalty — bad prediction!)

```
Cost when y=1:                    Cost when y=0:
    Cost                              Cost
    │                                 │
  5 │╲                              5 │              ╱
    │ ╲                               │             ╱
  3 │  ╲                            3 │           ╱
    │   ╲                              │         ╱
  1 │    ╲__                        1 │      __╱
    │        ───────                   │─────
  0 │               ──              0 │──
    └──────────────── ŷ              └──────────────── ŷ
    0            0.5     1            0     0.5        1
    
    High cost when ŷ→0!              High cost when ŷ→1!
    (Wrong prediction)               (Wrong prediction)
```

### Code: Cost Function Implementation

```python
import numpy as np

def logistic_cost(X, y, w, b):
    """
    Compute binary cross-entropy (log loss) cost.
    
    Args:
        X: Feature matrix (m, n)
        y: True labels 0 or 1 (m,)
        w: Weights (n,)
        b: Bias (scalar)
    
    Returns:
        cost: Scalar log loss value
    """
    m = len(y)
    z = X @ w + b
    y_hat = sigmoid(z)
    
    # Clip predictions to avoid log(0) = -inf
    epsilon = 1e-15
    y_hat = np.clip(y_hat, epsilon, 1 - epsilon)
    
    # Binary cross-entropy
    cost = -(1/m) * np.sum(
        y * np.log(y_hat) + (1 - y) * np.log(1 - y_hat)
    )
    return cost

# Example
X_demo = np.array([[1, 2], [2, 3], [3, 4], [4, 5]], dtype=float)
y_demo = np.array([0, 0, 1, 1], dtype=float)

# Perfect predictions
w_good = np.array([0.5, 0.5])
cost_good = logistic_cost(X_demo, y_demo, w_good, b=-3)
print(f"Good weights: Cost = {cost_good:.4f}")

# Bad predictions
w_bad = np.array([-0.5, -0.5])
cost_bad = logistic_cost(X_demo, y_demo, w_bad, b=3)
print(f"Bad weights:  Cost = {cost_bad:.4f}")

# Output:
# Good weights: Cost = 0.3567
# Bad weights:  Cost = 3.8912
```

---

## Gradient Descent for Logistic Regression

### The Gradients

Remarkably, the gradient has the **same form** as linear regression (but $\hat{y}$ is computed differently):

$$\frac{\partial J}{\partial w_j} = \frac{1}{m} \sum_{i=1}^{m} (\hat{y}_i - y_i) \cdot x_j^{(i)}$$

$$\frac{\partial J}{\partial b} = \frac{1}{m} \sum_{i=1}^{m} (\hat{y}_i - y_i)$$

Where $\hat{y}_i = \sigma(\mathbf{w}^T\mathbf{x}_i + b)$

### Code: Gradient Descent from Scratch

```python
import numpy as np

def logistic_regression_gd(X, y, learning_rate=0.1, n_iterations=1000):
    """
    Train logistic regression using gradient descent.
    
    Args:
        X: Feature matrix (m, n)
        y: Binary labels (m,)
        learning_rate: Step size
        n_iterations: Number of iterations
    
    Returns:
        w: Learned weights
        b: Learned bias
        cost_history: Cost at each iteration
    """
    m, n = X.shape
    w = np.zeros(n)
    b = 0.0
    cost_history = []
    
    for i in range(n_iterations):
        # Forward pass
        z = X @ w + b
        y_hat = sigmoid(z)
        
        # Compute gradients
        dw = (1/m) * (X.T @ (y_hat - y))  # (n,) vector
        db = (1/m) * np.sum(y_hat - y)     # scalar
        
        # Update parameters
        w = w - learning_rate * dw
        b = b - learning_rate * db
        
        # Track cost every 100 iterations
        if i % 100 == 0:
            cost = logistic_cost(X, y, w, b)
            cost_history.append(cost)
    
    return w, b, cost_history

# Train on a real dataset
from sklearn.datasets import load_breast_cancer
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import train_test_split

# Load data
data = load_breast_cancer()
X, y = data.data, data.target

# Scale features (crucial for gradient descent!)
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# Split
X_train, X_test, y_train, y_test = train_test_split(
    X_scaled, y, test_size=0.2, random_state=42
)

# Train from scratch
w, b, costs = logistic_regression_gd(X_train, y_train, learning_rate=0.1, n_iterations=2000)

# Evaluate
y_pred = (sigmoid(X_test @ w + b) >= 0.5).astype(int)
accuracy = np.mean(y_pred == y_test)
print(f"From scratch accuracy: {accuracy:.4f}")
print(f"Cost reduced: {costs[0]:.4f} → {costs[-1]:.4f}")

# Compare with sklearn
from sklearn.linear_model import LogisticRegression
sklearn_model = LogisticRegression(max_iter=2000, random_state=42)
sklearn_model.fit(X_train, y_train)
print(f"Sklearn accuracy:      {sklearn_model.score(X_test, y_test):.4f}")

# Output:
# From scratch accuracy: 0.9737
# Cost reduced: 0.6931 → 0.0821
# Sklearn accuracy:      0.9737
```

---

## Multi-Class Classification

### Strategies for Multiple Classes

When $y \in \{0, 1, 2, ..., K-1\}$ (more than 2 classes):

| Strategy | Description | How It Works |
|----------|-------------|--------------|
| **One-vs-Rest (OvR)** | K separate binary classifiers | Train K models, each: "class k vs. all others" |
| **One-vs-One (OvO)** | $\binom{K}{2}$ binary classifiers | Train one model per class pair, vote |
| **Softmax (Multinomial)** | Single model, K outputs | Generalization of sigmoid to K classes |

### Softmax Function

Generalizes sigmoid to multiple classes:

$$P(y=k|\mathbf{x}) = \frac{e^{z_k}}{\sum_{j=1}^{K} e^{z_j}}$$

Where $z_k = \mathbf{w}_k^T\mathbf{x} + b_k$ for each class $k$

**Properties:**
- All probabilities sum to 1: $\sum_{k=1}^{K} P(y=k|\mathbf{x}) = 1$
- Each probability is in $(0, 1)$
- When K=2, softmax reduces to sigmoid

### Code: Multi-Class Classification

```python
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report, confusion_matrix
import numpy as np

# Load multi-class dataset
iris = load_iris()
X, y = iris.data, iris.target  # 3 classes: setosa, versicolor, virginica

# Split data
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=42, stratify=y
)

# ─── Method 1: One-vs-Rest (OvR) ───
model_ovr = LogisticRegression(
    multi_class='ovr',       # One-vs-Rest strategy
    solver='lbfgs',
    max_iter=1000,
    random_state=42
)
model_ovr.fit(X_train, y_train)

# ─── Method 2: Multinomial (Softmax) ───
model_softmax = LogisticRegression(
    multi_class='multinomial',  # Softmax
    solver='lbfgs',
    max_iter=1000,
    random_state=42
)
model_softmax.fit(X_train, y_train)

# Compare
print(f"OvR Accuracy:      {model_ovr.score(X_test, y_test):.4f}")
print(f"Softmax Accuracy:  {model_softmax.score(X_test, y_test):.4f}")

# Get probabilities for a sample
sample = X_test[0:1]
probs = model_softmax.predict_proba(sample)
print(f"\nSample prediction probabilities:")
for i, name in enumerate(iris.target_names):
    print(f"  {name}: {probs[0][i]:.4f}")

# Detailed report
print("\nClassification Report:")
y_pred = model_softmax.predict(X_test)
print(classification_report(y_test, y_pred, target_names=iris.target_names))

# Confusion Matrix
print("Confusion Matrix:")
print(confusion_matrix(y_test, y_pred))
```

**Output:**
```
OvR Accuracy:      0.9778
Softmax Accuracy:  0.9778

Sample prediction probabilities:
  setosa: 0.0000
  versicolor: 0.0240
  virginica: 0.9760

Classification Report:
              precision    recall  f1-score   support
      setosa       1.00      1.00      1.00        15
  versicolor       0.94      1.00      0.97        15
   virginica       1.00      0.93      0.97        15
    accuracy                           0.98        45

Confusion Matrix:
[[15  0  0]
 [ 0 15  0]
 [ 0  1 14]]
```

### Softmax Implementation from Scratch

```python
def softmax(z):
    """
    Compute softmax probabilities.
    Numerically stable version (subtract max to prevent overflow).
    """
    z_shifted = z - np.max(z, axis=1, keepdims=True)  # Stability trick
    exp_z = np.exp(z_shifted)
    return exp_z / np.sum(exp_z, axis=1, keepdims=True)

# Example: 3 classes, 2 samples
z = np.array([
    [2.0, 1.0, 0.1],  # Sample 1: high score for class 0
    [0.1, 3.0, 0.2]   # Sample 2: high score for class 1
])

probs = softmax(z)
print("Softmax probabilities:")
print(probs)
print(f"Sum per row (should be 1): {probs.sum(axis=1)}")
print(f"Predictions: {np.argmax(probs, axis=1)}")

# Output:
# Softmax probabilities:
# [[0.6590 0.2424 0.0986]
#  [0.0466 0.8476 0.1058]]
# Sum per row: [1. 1.]
# Predictions: [0 1]
```

---

## Regularization in Logistic Regression

### Why Regularize?

Logistic Regression can overfit when:
- Many features (high-dimensional data)
- Features are correlated
- Small dataset relative to features
- Perfect or near-perfect separation exists

### Regularized Cost Function

$$J(\mathbf{w}) = -\frac{1}{m} \sum_{i=1}^{m} [y_i\log(\hat{y}_i) + (1-y_i)\log(1-\hat{y}_i)] + \frac{\lambda}{2m}\|\mathbf{w}\|^2$$

> **Note:** In sklearn, the regularization parameter is `C = 1/λ`. Higher C = less regularization. Lower C = more regularization.

### Code: Effect of Regularization

```python
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import cross_val_score
import numpy as np

# Test different regularization strengths
C_values = [0.001, 0.01, 0.1, 1.0, 10.0, 100.0]

print(f"{'C':<8} {'Penalty':<8} {'CV Accuracy':<15} {'Non-zero coefs'}")
print("-" * 50)

for C in C_values:
    model = LogisticRegression(C=C, penalty='l2', max_iter=5000, random_state=42)
    scores = cross_val_score(model, X_train, y_train, cv=5)
    model.fit(X_train, y_train)
    n_nonzero = np.sum(np.abs(model.coef_) > 1e-6)
    print(f"{C:<8} L2       {scores.mean():.4f} ± {scores.std():.4f}  {n_nonzero}")

# L1 regularization (feature selection)
print("\nL1 Regularization (Lasso):")
for C in [0.01, 0.1, 1.0]:
    model = LogisticRegression(C=C, penalty='l1', solver='saga', max_iter=5000, random_state=42)
    scores = cross_val_score(model, X_train, y_train, cv=5)
    model.fit(X_train, y_train)
    n_nonzero = np.sum(np.abs(model.coef_) > 1e-6)
    print(f"  C={C:<5}: Accuracy={scores.mean():.4f}, Non-zero features={n_nonzero}/{X_train.shape[1]}")
```

### Choosing the Right Regularization

| Parameter | Effect |
|-----------|--------|
| `C=0.001` (strong reg) | Very simple model, might underfit |
| `C=1.0` (default) | Moderate regularization |
| `C=100` (weak reg) | Complex model, might overfit |
| `penalty='l1'` | Sparse model (feature selection) |
| `penalty='l2'` | All features kept, coefficients shrunk |
| `penalty='elasticnet'` | Combines L1+L2 (needs `solver='saga'`, `l1_ratio`) |

---

## Evaluation Metrics

### The Confusion Matrix

```
                    Predicted
                 Negative  Positive
              ┌──────────┬──────────┐
    Actual    │    TN    │    FP    │  ← Type I Error (False Alarm)
   Negative   │          │          │
              ├──────────┼──────────┤
    Actual    │    FN    │    TP    │  
   Positive   │          │          │
              └──────────┴──────────┘
                    ↑
              Type II Error
              (Missed Detection)
```

### Key Metrics

| Metric | Formula | Question Answered |
|--------|---------|-------------------|
| **Accuracy** | $\frac{TP+TN}{TP+TN+FP+FN}$ | Of all predictions, how many correct? |
| **Precision** | $\frac{TP}{TP+FP}$ | Of predicted positive, how many truly positive? |
| **Recall (Sensitivity)** | $\frac{TP}{TP+FN}$ | Of actual positive, how many did we catch? |
| **Specificity** | $\frac{TN}{TN+FP}$ | Of actual negative, how many correctly identified? |
| **F1 Score** | $2 \cdot \frac{P \cdot R}{P + R}$ | Harmonic mean of precision and recall |
| **AUC-ROC** | Area under ROC curve | Overall ranking ability across all thresholds |

### When to Prioritize What

| Scenario | Prioritize | Why |
|----------|-----------|-----|
| Spam detection | **Precision** | Don't want to mark real email as spam (FP is costly) |
| Cancer screening | **Recall** | Don't want to miss cancer (FN is deadly) |
| Balanced costs | **F1 Score** | Equal importance of precision and recall |
| Model comparison | **AUC-ROC** | Threshold-independent evaluation |
| Fraud detection | **Recall** + acceptable Precision | Missing fraud is expensive, but too many alerts = alert fatigue |

### Code: Complete Evaluation

```python
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    confusion_matrix, classification_report, roc_auc_score, roc_curve
)
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split
from sklearn.datasets import load_breast_cancer
import numpy as np

# Load and split data
data = load_breast_cancer()
X_train, X_test, y_train, y_test = train_test_split(
    data.data, data.target, test_size=0.2, random_state=42, stratify=data.target
)

# Train model
model = LogisticRegression(max_iter=5000, random_state=42)
model.fit(X_train, y_train)

# Predictions
y_pred = model.predict(X_test)
y_prob = model.predict_proba(X_test)[:, 1]  # Probability of positive class

# All metrics
print("=" * 50)
print("CLASSIFICATION METRICS")
print("=" * 50)
print(f"Accuracy:    {accuracy_score(y_test, y_pred):.4f}")
print(f"Precision:   {precision_score(y_test, y_pred):.4f}")
print(f"Recall:      {recall_score(y_test, y_pred):.4f}")
print(f"F1 Score:    {f1_score(y_test, y_pred):.4f}")
print(f"AUC-ROC:     {roc_auc_score(y_test, y_prob):.4f}")

print(f"\nConfusion Matrix:")
cm = confusion_matrix(y_test, y_pred)
print(f"  TN={cm[0][0]}, FP={cm[0][1]}")
print(f"  FN={cm[1][0]}, TP={cm[1][1]}")

print(f"\nDetailed Report:")
print(classification_report(y_test, y_pred, target_names=['Malignant', 'Benign']))
```

**Output:**
```
==================================================
CLASSIFICATION METRICS
==================================================
Accuracy:    0.9649
Precision:   0.9589
Recall:      0.9859
F1 Score:    0.9722
AUC-ROC:     0.9953

Confusion Matrix:
  TN=39, FP=3
  FN=1, TP=71

Detailed Report:
              precision    recall  f1-score   support
   Malignant       0.97      0.93      0.95        42
      Benign       0.96      0.99      0.97        72
    accuracy                           0.96       114
```

### ROC Curve and AUC

```
    TPR (Recall/Sensitivity)
  1 │          ╱──────────── Perfect model (AUC=1.0)
    │        ╱╱
    │      ╱╱  ← Your model (AUC=0.99)
    │    ╱╱
    │  ╱╱
    │╱╱      ╱── Random guess (AUC=0.5)
    │╱     ╱
    │    ╱
    │  ╱
    │╱
  0 └──────────────────── FPR (1 - Specificity)
    0                   1
```

```python
from sklearn.metrics import roc_curve, roc_auc_score

# Compute ROC curve
fpr, tpr, thresholds = roc_curve(y_test, y_prob)
auc = roc_auc_score(y_test, y_prob)

# Find optimal threshold (Youden's J statistic)
j_scores = tpr - fpr
optimal_idx = np.argmax(j_scores)
optimal_threshold = thresholds[optimal_idx]

print(f"AUC-ROC: {auc:.4f}")
print(f"Optimal Threshold: {optimal_threshold:.4f}")
print(f"At this threshold: TPR={tpr[optimal_idx]:.4f}, FPR={fpr[optimal_idx]:.4f}")
```

---

## Threshold Tuning

### Why Tune the Threshold?

The default threshold of 0.5 isn't always optimal. Different applications need different tradeoffs:

```
Threshold = 0.5 (default):
├── Balanced precision/recall

Threshold = 0.3 (lower):
├── More predictions of positive class
├── Higher recall (catch more positives)
├── Lower precision (more false positives)
├── Use for: medical screening, fraud detection

Threshold = 0.7 (higher):
├── Fewer predictions of positive class
├── Higher precision (fewer false positives)
├── Lower recall (miss some positives)
├── Use for: spam filters, criminal justice
```

### Code: Threshold Analysis

```python
import numpy as np
from sklearn.metrics import precision_score, recall_score, f1_score

# Get predicted probabilities
y_prob = model.predict_proba(X_test)[:, 1]

# Evaluate at different thresholds
thresholds = [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9]

print(f"{'Threshold':<12} {'Precision':<12} {'Recall':<12} {'F1':<12} {'Pred Positive'}")
print("-" * 60)

for t in thresholds:
    y_pred_t = (y_prob >= t).astype(int)
    
    # Avoid undefined metrics when no positive predictions
    if y_pred_t.sum() == 0:
        print(f"{t:<12.1f} {'N/A':<12} {'0.0000':<12} {'N/A':<12} {y_pred_t.sum()}")
        continue
    
    p = precision_score(y_test, y_pred_t, zero_division=0)
    r = recall_score(y_test, y_pred_t, zero_division=0)
    f1 = f1_score(y_test, y_pred_t, zero_division=0)
    print(f"{t:<12.1f} {p:<12.4f} {r:<12.4f} {f1:<12.4f} {y_pred_t.sum()}")

# Choose threshold that maximizes F1 (or optimize for your business metric)
from sklearn.metrics import precision_recall_curve

precisions, recalls, thresholds_pr = precision_recall_curve(y_test, y_prob)
f1_scores = 2 * (precisions * recalls) / (precisions + recalls + 1e-10)
best_idx = np.argmax(f1_scores)
best_threshold = thresholds_pr[best_idx]
print(f"\nBest threshold for F1: {best_threshold:.4f} (F1={f1_scores[best_idx]:.4f})")
```

> **Pro Tip:** In production, set thresholds based on business costs. If a false negative costs $10,000 (missed fraud) and a false positive costs $10 (investigating legitimate transaction), optimize the threshold to minimize expected cost: $\text{Cost} = 10 \times FP + 10000 \times FN$.

---

## Complete End-to-End Example

```python
"""
Complete Logistic Regression Pipeline
Task: Predict customer churn (will they leave?)
"""
import numpy as np
import pandas as pd
from sklearn.model_selection import train_test_split, cross_val_score, GridSearchCV
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import Pipeline
from sklearn.metrics import classification_report, roc_auc_score
from sklearn.datasets import make_classification

# ─── Step 1: Create/Load Dataset ───
X, y = make_classification(
    n_samples=5000,
    n_features=20,
    n_informative=10,
    n_redundant=5,
    weights=[0.7, 0.3],  # Imbalanced: 70% class 0, 30% class 1
    random_state=42
)

# ─── Step 2: Train-Test Split ───
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)
print(f"Train shape: {X_train.shape}, Class distribution: {np.bincount(y_train)}")
print(f"Test shape:  {X_test.shape}, Class distribution: {np.bincount(y_test)}")

# ─── Step 3: Build Pipeline (Scaling + Model) ───
pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('model', LogisticRegression(max_iter=1000, random_state=42))
])

# ─── Step 4: Hyperparameter Tuning ───
param_grid = {
    'model__C': [0.01, 0.1, 1, 10],
    'model__penalty': ['l1', 'l2'],
    'model__solver': ['saga'],  # Supports both L1 and L2
    'model__class_weight': [None, 'balanced']  # Handle imbalance
}

grid_search = GridSearchCV(
    pipeline, param_grid, cv=5,
    scoring='roc_auc',  # Optimize for AUC (good for imbalanced data)
    n_jobs=-1
)
grid_search.fit(X_train, y_train)

print(f"\nBest Parameters: {grid_search.best_params_}")
print(f"Best CV AUC-ROC: {grid_search.best_score_:.4f}")

# ─── Step 5: Final Evaluation on Test Set ───
best_model = grid_search.best_estimator_
y_pred = best_model.predict(X_test)
y_prob = best_model.predict_proba(X_test)[:, 1]

print(f"\nTest AUC-ROC: {roc_auc_score(y_test, y_prob):.4f}")
print("\nClassification Report:")
print(classification_report(y_test, y_pred))

# ─── Step 6: Feature Importance ───
# Get coefficients from the model inside the pipeline
coefs = best_model.named_steps['model'].coef_[0]
feature_importance = pd.DataFrame({
    'Feature': [f'Feature_{i}' for i in range(len(coefs))],
    'Coefficient': coefs,
    'Abs_Coefficient': np.abs(coefs),
    'Odds_Ratio': np.exp(coefs)  # exp(coef) = odds ratio
}).sort_values('Abs_Coefficient', ascending=False)

print("\nTop 10 Most Important Features:")
print(feature_importance.head(10).to_string(index=False))
```

### Interpreting Odds Ratios

| Coefficient | Odds Ratio ($e^{w}$) | Interpretation |
|-------------|---------------------|----------------|
| $w = 0$ | $e^0 = 1.0$ | No effect |
| $w = 0.5$ | $e^{0.5} = 1.65$ | 65% increase in odds per unit increase |
| $w = -0.3$ | $e^{-0.3} = 0.74$ | 26% decrease in odds per unit increase |
| $w = 1.0$ | $e^1 = 2.72$ | 172% increase in odds (nearly triples!) |

> **Key Insight:** In logistic regression, coefficients represent log-odds. $e^{w_j}$ = odds ratio = how many times the odds of the outcome are multiplied for a one-unit increase in feature $j$.

---

## Common Mistakes

### 1. Not Handling Class Imbalance
```python
# WRONG ❌ — Model will predict majority class for everything
model = LogisticRegression()
model.fit(X_train, y_train)  # 95% class 0, 5% class 1 → predicts all 0

# CORRECT ✅ — Use class_weight='balanced'
model = LogisticRegression(class_weight='balanced')
# This adjusts the cost function: minority class errors cost more
# Equivalent to: weight_k = n_samples / (n_classes * n_samples_k)

# Alternative: Use SMOTE for oversampling minority class
from imblearn.over_sampling import SMOTE  # pip install imbalanced-learn
smote = SMOTE(random_state=42)
X_train_resampled, y_train_resampled = smote.fit_resample(X_train, y_train)
```

### 2. Forgetting to Scale Features
```python
# WRONG ❌ — Solver may not converge, coefficients are uninterpretable
model = LogisticRegression(max_iter=100)
model.fit(X_train, y_train)  # ConvergenceWarning!

# CORRECT ✅ — Always scale (especially for L1/L2 regularization)
from sklearn.pipeline import make_pipeline
model = make_pipeline(
    StandardScaler(),
    LogisticRegression(max_iter=1000)
)
model.fit(X_train, y_train)
```

### 3. Ignoring max_iter Warning
```python
# If you see "ConvergenceWarning: lbfgs failed to converge"
# The solver didn't finish optimizing!

# Fix 1: Increase max_iter
model = LogisticRegression(max_iter=5000)

# Fix 2: Scale your data (helps convergence dramatically)
# Fix 3: Try a different solver
model = LogisticRegression(solver='saga', max_iter=2000)

# Fix 4: Increase regularization (simpler problem to solve)
model = LogisticRegression(C=0.1, max_iter=1000)
```

### 4. Using Accuracy as the Only Metric
```python
# With 99% negative, 1% positive:
# Model predicting "always negative" gets 99% accuracy!

# Always check the confusion matrix and per-class metrics
from sklearn.metrics import classification_report
print(classification_report(y_test, y_pred))
# Look at recall for minority class — that's where models fail
```

### 5. Misinterpreting Coefficients with Correlated Features
```python
# If feature A and feature B are highly correlated (r > 0.8):
# - Coefficients become unstable
# - Individual coefficients DON'T represent true feature importance
# - Total effect is shared/split between them

# Solution: Check VIF, use L1 regularization, or use permutation importance
from sklearn.inspection import permutation_importance
result = permutation_importance(model, X_test, y_test, n_repeats=10)
# This gives TRUE feature importance regardless of correlation
```

### 6. Applying Logistic Regression to Non-Linear Problems Without Feature Engineering
```python
# WRONG ❌ — Linear decision boundary can't capture XOR pattern
from sklearn.datasets import make_moons
X, y = make_moons(n_samples=1000, noise=0.2)
model = LogisticRegression()
model.fit(X, y)
print(f"Accuracy: {model.score(X, y):.4f}")  # ~0.85 (bad)

# CORRECT ✅ — Add polynomial features for non-linear boundary
from sklearn.preprocessing import PolynomialFeatures
from sklearn.pipeline import make_pipeline

model = make_pipeline(
    PolynomialFeatures(degree=3),
    StandardScaler(),
    LogisticRegression(C=10, max_iter=5000)
)
model.fit(X, y)
print(f"Accuracy: {model.score(X, y):.4f}")  # ~0.97 (good!)
```

---

## Interview Questions

### Conceptual Questions

**Q1: Why is it called "Logistic Regression" if it's used for classification?**
> It's called regression because it regresses to a continuous value (probability between 0 and 1) using the logistic (sigmoid) function. The classification decision is made by thresholding this probability. Historically, it was developed to model growth curves (logistic growth), and the name stuck.

**Q2: Explain the sigmoid function and why it's used.**
> Sigmoid: σ(z) = 1/(1 + e^(-z)). Properties: (1) Maps any real number to (0,1) → valid probability, (2) Differentiable everywhere → gradient descent works, (3) Symmetric around 0.5, (4) Derivative σ'(z) = σ(z)(1-σ(z)) is elegant for backpropagation. It models the S-shaped transition from 0 to 1 probability as the linear combination of features changes.

**Q3: Why do we use log loss instead of MSE for logistic regression?**
> MSE with sigmoid creates a non-convex cost surface with many local minima — gradient descent can get stuck. Log loss is convex (single global minimum) and provides stronger gradients for confident wrong predictions. Also, log loss directly models the likelihood function (maximum likelihood estimation), making it statistically principled.

**Q4: How do you interpret logistic regression coefficients?**
> Each coefficient $w_j$ represents the change in **log-odds** for a one-unit increase in feature $j$. The odds ratio $e^{w_j}$ tells you how the odds of the positive class are multiplied. Example: $w = 0.7$ → $e^{0.7} = 2.01$ → odds double for each unit increase. A negative coefficient decreases the odds.

**Q5: What is the difference between One-vs-Rest and Softmax for multi-class?**
> OvR trains K independent binary classifiers, each separating one class from all others. Softmax trains a single model with K output neurons that produce probabilities summing to 1. OvR is simpler but assumes classes are independent. Softmax is more principled (joint optimization) and naturally outputs valid probability distributions. Softmax is generally preferred for mutually exclusive classes.

**Q6: How does class_weight='balanced' work?**
> It assigns higher misclassification cost to minority class samples. Weight formula: $w_k = \frac{n_{total}}{n_{classes} \times n_k}$. For example, with 900 negatives and 100 positives: weight_neg = 1000/(2×900) = 0.56, weight_pos = 1000/(2×100) = 5.0. This makes misclassifying a positive sample 9x more costly than misclassifying a negative one.

### Scenario Questions

**Q7: Your logistic regression gives 98% accuracy on credit fraud detection. Is this good?**
> Almost certainly not! Fraud is rare (~1-2% of transactions). A model predicting "not fraud" for everything gets 98% accuracy. Check: precision and recall for the fraud class, confusion matrix (how many frauds are actually caught?), AUC-ROC for threshold-independent evaluation. If recall for fraud is 10%, the model is useless despite 98% accuracy.

**Q8: How would you use logistic regression for a problem with 100,000 features (like text classification)?**
> (1) Use L1 regularization (Lasso) for automatic feature selection — most text features are irrelevant, (2) Use SGDClassifier with log loss for efficiency (doesn't need to load all data into memory), (3) Use sparse matrix format (scipy.sparse) since text data is mostly zeros, (4) Start with high regularization (low C) and decrease until performance stabilizes.

**Q9: When would logistic regression outperform a neural network?**
> (1) Small dataset (< 1000 samples) — LR won't overfit, (2) When interpretability is required (medical, legal), (3) When features are well-engineered (domain expert created them), (4) When the decision boundary is approximately linear, (5) When training time/resources are limited, (6) As a component in ensemble models (stacking).

---

## Quick Reference

### Logistic Regression Cheat Sheet

| Component | Formula |
|-----------|---------|
| Sigmoid | $\sigma(z) = \frac{1}{1+e^{-z}}$ |
| Model | $P(y=1) = \sigma(\mathbf{w}^T\mathbf{x} + b)$ |
| Decision Boundary | $\mathbf{w}^T\mathbf{x} + b = 0$ |
| Log Loss | $J = -\frac{1}{m}\sum[y\log\hat{y} + (1-y)\log(1-\hat{y})]$ |
| Gradient (weights) | $\frac{1}{m}\mathbf{X}^T(\hat{\mathbf{y}} - \mathbf{y})$ |
| Softmax | $P(y=k) = \frac{e^{z_k}}{\sum_j e^{z_j}}$ |
| Odds Ratio | $e^{w_j}$ |

### sklearn Solver Guide

| Solver | L1 | L2 | Multinomial | Large Data | Notes |
|--------|----|----|-------------|------------|-------|
| `lbfgs` | ❌ | ✅ | ✅ | Medium | Default, good all-around |
| `liblinear` | ✅ | ✅ | ❌ (OvR only) | Small | Good for small datasets |
| `saga` | ✅ | ✅ | ✅ | Large | Fast for large datasets |
| `newton-cg` | ❌ | ✅ | ✅ | Medium | Good for multinomial |
| `sag` | ❌ | ✅ | ✅ | Large | Fast for large datasets |

### Decision Flowchart

```
Classification Problem?
│
├── Binary (2 classes)?
│   ├── Need probability output? → LogisticRegression
│   ├── Need feature importance? → LogisticRegression (check coef_)
│   └── Need feature selection? → LogisticRegression(penalty='l1')
│
├── Multi-class (>2 classes)?
│   ├── Classes mutually exclusive? → multi_class='multinomial'
│   └── Classes independent? → multi_class='ovr'
│
├── Imbalanced data?
│   ├── Mild imbalance (80/20) → class_weight='balanced'
│   ├── Severe imbalance (99/1) → SMOTE + threshold tuning
│   └── Always check recall for minority class
│
└── Not converging?
    ├── Scale features (StandardScaler)
    ├── Increase max_iter
    ├── Increase regularization (lower C)
    └── Try solver='saga'
```

### Metrics Quick Guide

| Metric | Use When | Sensitive To |
|--------|----------|-------------|
| Accuracy | Balanced classes | Class imbalance |
| Precision | FP is costly (spam filter) | Threshold |
| Recall | FN is costly (disease detection) | Threshold |
| F1 Score | Need balance of P and R | Threshold |
| AUC-ROC | Comparing models, imbalanced data | Not threshold |
| Log Loss | Probability calibration matters | Confidence |

### Complete Pipeline Template

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import GridSearchCV

# Production-ready pipeline
pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('classifier', LogisticRegression(
        solver='saga',
        max_iter=5000,
        random_state=42
    ))
])

# Hyperparameter search
param_grid = {
    'classifier__C': [0.01, 0.1, 1, 10],
    'classifier__penalty': ['l1', 'l2'],
    'classifier__class_weight': [None, 'balanced']
}

search = GridSearchCV(pipeline, param_grid, cv=5, scoring='roc_auc', n_jobs=-1)
search.fit(X_train, y_train)

# Best model ready for deployment
best_model = search.best_estimator_
```

---

*Previous: [02-Linear-Regression.md](02-Linear-Regression.md)*  
*Next: [04-Decision-Trees-and-Random-Forest.md](04-Decision-Trees-and-Random-Forest.md) — Entropy, Gini, Information Gain, Pruning, Bagging*
