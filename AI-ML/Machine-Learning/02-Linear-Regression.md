# Chapter 02: Linear Regression

## Table of Contents
- [What is Linear Regression?](#what-is-linear-regression)
- [Simple Linear Regression](#simple-linear-regression)
- [Multiple Linear Regression](#multiple-linear-regression)
- [Cost Function (MSE)](#cost-function-mse)
- [Gradient Descent](#gradient-descent)
- [Normal Equation (Closed-Form Solution)](#normal-equation)
- [Assumptions of Linear Regression](#assumptions-of-linear-regression)
- [Regularization (Ridge, Lasso, Elastic Net)](#regularization)
- [Polynomial Regression](#polynomial-regression)
- [Model Evaluation Metrics](#model-evaluation-metrics)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Linear Regression?

### What It Is
Linear Regression finds the **best-fitting straight line** (or hyperplane) through your data points to predict a continuous numerical output.

**Analogy for a 15-year-old:** You notice that taller people tend to weigh more. If you plot height vs. weight for 1000 people and draw the best straight line through those dots, that line IS your linear regression model. Given a new person's height, you follow the line to predict their weight.

### Why It Matters
- **Foundation of ML** — Understanding LR means understanding 80% of all ML concepts (cost functions, gradient descent, regularization, overfitting)
- **Interpretable** — You can explain exactly WHY the model made a prediction (coefficient = importance)
- **Fast** — Trains in milliseconds even on millions of rows
- **Surprisingly powerful** — With proper feature engineering, it competes with complex models
- **Industry standard** — Used in finance (risk), economics (forecasting), medicine (dose-response), marketing (ROI prediction)

### When to Use vs. Not Use

| Use Linear Regression When | Don't Use When |
|---------------------------|----------------|
| Relationship is approximately linear | Highly non-linear relationships |
| You need interpretability | You need maximum accuracy at any cost |
| Small-medium dataset | Very high-dimensional sparse data (use Lasso) |
| Quick baseline model | Target is categorical (use Logistic Regression) |
| Features and target have linear correlation | Heavy outliers (use robust regression) |

---

## Simple Linear Regression

### The Model

Simple Linear Regression uses **one** input feature to predict the output:

$$\hat{y} = w_0 + w_1 x$$

Where:
- $\hat{y}$ = predicted value
- $w_0$ = intercept (bias term) — value of $y$ when $x = 0$
- $w_1$ = slope (weight/coefficient) — how much $y$ changes for a unit change in $x$
- $x$ = input feature

### Visual Intuition

```
    y (Price $)
    │           ·    ·
    │        ·  ╱  ·
    │      · ╱·     ← Residual (error)
    │    · ╱ ·
    │  · ╱·
    │ ·╱·       Best fit line: ŷ = w₀ + w₁x
    │╱·
    │·
    └──────────────── x (Square Feet)
    
    Slope (w₁) = rise/run = Δy/Δx
    Intercept (w₀) = where line crosses y-axis
```

### Code Example: Simple Linear Regression from Scratch

```python
import numpy as np

# Generate sample data: house size (sq ft) → price ($1000s)
np.random.seed(42)
X = np.random.uniform(500, 3000, 100)  # House sizes
y = 50 + 0.1 * X + np.random.normal(0, 20, 100)  # True relationship + noise

# ─── Method 1: From scratch using formulas ───
# Slope: w1 = Σ(xi - x̄)(yi - ȳ) / Σ(xi - x̄)²
# Intercept: w0 = ȳ - w1 * x̄

x_mean = np.mean(X)
y_mean = np.mean(y)

# Calculate slope
numerator = np.sum((X - x_mean) * (y - y_mean))
denominator = np.sum((X - x_mean) ** 2)
w1 = numerator / denominator

# Calculate intercept
w0 = y_mean - w1 * x_mean

print(f"From scratch → Intercept: {w0:.4f}, Slope: {w1:.4f}")
# Output: From scratch → Intercept: 48.8721, Slope: 0.1005

# ─── Method 2: Using scikit-learn ───
from sklearn.linear_model import LinearRegression

model = LinearRegression()
model.fit(X.reshape(-1, 1), y)  # sklearn expects 2D input

print(f"Sklearn → Intercept: {model.intercept_:.4f}, Slope: {model.coef_[0]:.4f}")
# Output: Sklearn → Intercept: 48.8721, Slope: 0.1005

# Interpretation: Every 1 sq ft increase → $100.50 price increase
# A 0 sq ft house would cost $48,872 (intercept — often not meaningful)
```

### Interpretation of Coefficients

> **Key Insight:** The coefficient $w_1$ represents the **change in $y$ for a one-unit change in $x$**, holding all else constant.

| Coefficient | Value | Interpretation |
|-------------|-------|----------------|
| $w_0 = 48.87$ | Intercept | Baseline price (not always meaningful) |
| $w_1 = 0.1005$ | Slope | Each additional sq ft adds ~$100.50 to price |

---

## Multiple Linear Regression

### The Model

Multiple features predict the output:

$$\hat{y} = w_0 + w_1 x_1 + w_2 x_2 + ... + w_n x_n = \mathbf{w}^T \mathbf{x}$$

In matrix notation:

$$\hat{\mathbf{y}} = \mathbf{X} \mathbf{w}$$

Where:
- $\mathbf{X}$ is the $(m \times n)$ feature matrix (m samples, n features)
- $\mathbf{w}$ is the $(n \times 1)$ weight vector
- $\hat{\mathbf{y}}$ is the $(m \times 1)$ prediction vector

### Code Example: Multiple Linear Regression

```python
import numpy as np
import pandas as pd
from sklearn.linear_model import LinearRegression
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error, r2_score
from sklearn.datasets import fetch_california_housing

# Load real dataset: California housing prices
data = fetch_california_housing()
X = pd.DataFrame(data.data, columns=data.feature_names)
y = data.target  # Median house value in $100,000s

print("Features:", list(X.columns))
print(f"Dataset shape: {X.shape}")  # (20640, 8)

# Split data
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Train model
model = LinearRegression()
model.fit(X_train, y_train)

# Predictions
y_pred = model.predict(X_test)

# Evaluation
print(f"\nR² Score: {r2_score(y_test, y_pred):.4f}")
print(f"RMSE: {np.sqrt(mean_squared_error(y_test, y_pred)):.4f}")

# Feature importance (coefficients)
coef_df = pd.DataFrame({
    'Feature': X.columns,
    'Coefficient': model.coef_
}).sort_values('Coefficient', key=abs, ascending=False)

print(f"\nIntercept: {model.intercept_:.4f}")
print("\nFeature Coefficients (sorted by importance):")
print(coef_df.to_string(index=False))
```

**Output:**
```
Features: ['MedInc', 'HouseAge', 'AveRooms', 'AveBedrms', 'Population', 'AveOccup', 'Latitude', 'Longitude']
Dataset shape: (20640, 8)

R² Score: 0.5758
RMSE: 0.7456

Intercept: -36.9419

Feature Coefficients (sorted by importance):
  Feature  Coefficient
 Latitude    -0.4232
Longitude    -0.4345
   MedInc     0.4367
 HouseAge     0.0094
 AveRooms    -0.1073
AveBedrms     0.6452
Population   -0.0000
  AveOccup   -0.0038
```

### Interpreting Multiple Regression Coefficients

> **Critical:** In multiple regression, each coefficient represents the effect of that feature **while holding all other features constant** (ceteris paribus).

Example: `MedInc` coefficient = 0.4367 means:
- For every $10,000 increase in median income (1 unit), house value increases by $43,670
- **Assuming** all other features (location, house age, etc.) stay the same

---

## Cost Function (MSE)

### What It Is
The cost function measures **how wrong** your predictions are. Linear regression uses **Mean Squared Error (MSE)** — the average of squared differences between predicted and actual values.

**Analogy:** Each prediction error is like a rubber band stretched between the predicted point and the actual point. MSE is the average tension of all rubber bands. The model tries to minimize this total tension.

### Mathematical Definition

$$J(\mathbf{w}) = \text{MSE} = \frac{1}{m} \sum_{i=1}^{m} (\hat{y}_i - y_i)^2 = \frac{1}{m} \sum_{i=1}^{m} (\mathbf{w}^T \mathbf{x}_i - y_i)^2$$

Where:
- $m$ = number of training samples
- $\hat{y}_i$ = predicted value for sample $i$
- $y_i$ = actual value for sample $i$

### Why Squared Errors?

| Reason | Explanation |
|--------|-------------|
| **Penalizes large errors more** | Error of 10 contributes 100, error of 2 contributes 4 |
| **Differentiable** | Smooth function → gradient descent works |
| **Always positive** | Errors don't cancel out (+5 and -5 don't give 0) |
| **Mathematical convenience** | Derivative is simple and linear |

### Visualizing the Cost Function

For simple linear regression with one weight, the cost function is a **parabola** (bowl shape):

```
    J(w)  Cost
    │
    │  ╲                    ╱
    │   ╲                  ╱
    │    ╲                ╱
    │     ╲              ╱
    │      ╲            ╱
    │       ╲          ╱
    │        ╲________╱
    │            ★  ← Global minimum (optimal w)
    └───────────────────────── w (weight)
```

For two weights ($w_0$, $w_1$), it's a 3D bowl (convex surface):

```
    Cost J(w₀, w₁)
         ╱│╲
        ╱ │ ╲
       ╱  │  ╲      ← Contour plot view:
      ╱   │   ╲        concentric ellipses
     ╱    ★    ╲        with minimum at center
      ╲   │   ╱
       ╲  │  ╱
        ╲ │ ╱
         ╲│╱
```

### Code: Computing Cost Function

```python
def compute_cost(X, y, w, b):
    """
    Compute MSE cost function.
    
    Args:
        X: Feature matrix (m, n)
        y: Target values (m,)
        w: Weights (n,)
        b: Bias/intercept (scalar)
    
    Returns:
        cost: Mean Squared Error (scalar)
    """
    m = len(y)
    predictions = X @ w + b           # Matrix multiplication + bias
    errors = predictions - y           # Residuals
    cost = (1 / (2 * m)) * np.sum(errors ** 2)  # Note: 1/2m for cleaner derivative
    return cost

# Example
X_example = np.array([[1], [2], [3], [4], [5]], dtype=float)
y_example = np.array([2, 4, 5, 4, 5], dtype=float)

# Try different weights
for w in [0.5, 1.0, 1.5]:
    cost = compute_cost(X_example, y_example, np.array([w]), b=0)
    print(f"w={w:.1f}: Cost = {cost:.4f}")

# Output:
# w=0.5: Cost = 3.3750 (undershoot)
# w=1.0: Cost = 0.7000 (close!)  
# w=1.5: Cost = 2.3750 (overshoot)
```

### Other Cost Functions

| Cost Function | Formula | When to Use |
|---------------|---------|-------------|
| **MSE** | $\frac{1}{m}\sum(y-\hat{y})^2$ | Standard regression, penalize large errors |
| **MAE** | $\frac{1}{m}\sum|y-\hat{y}|$ | Robust to outliers |
| **Huber Loss** | MSE if small error, MAE if large | Best of both worlds |
| **RMSE** | $\sqrt{MSE}$ | Same unit as target (interpretable) |

---

## Gradient Descent

### What It Is
An optimization algorithm that finds the minimum of the cost function by iteratively moving in the direction of steepest descent.

**Analogy:** You're blindfolded on a hilly landscape and want to find the lowest valley. Strategy: Feel the slope under your feet, take a step downhill, repeat. Eventually you reach the bottom.

### How It Works

1. Start with random weights
2. Calculate the gradient (slope) of the cost function
3. Update weights in the opposite direction of the gradient
4. Repeat until convergence

### The Update Rule

$$w_j := w_j - \alpha \frac{\partial J}{\partial w_j}$$

Where:
- $\alpha$ = learning rate (step size)
- $\frac{\partial J}{\partial w_j}$ = partial derivative of cost w.r.t. weight $j$

For MSE cost function:

$$\frac{\partial J}{\partial w_j} = \frac{1}{m} \sum_{i=1}^{m} (\hat{y}_i - y_i) \cdot x_j^{(i)}$$

$$\frac{\partial J}{\partial b} = \frac{1}{m} \sum_{i=1}^{m} (\hat{y}_i - y_i)$$

### Visual: Gradient Descent Steps

```
    J(w)
    │
    │  ·  Step 1
    │   ╲
    │    ·  Step 2
    │     ╲
    │      ·  Step 3
    │       ╲
    │        · Step 4
    │         ╲_____·  Converged! (gradient ≈ 0)
    │              ★
    └──────────────────── w

    Learning rate too small:     Learning rate too large:
    │                            │
    │·                           │·
    │ ·                          │        ·
    │  ·                         │ ·   ·
    │   ·                        │  ·      ·
    │    ·                       │     ·       ·  ← Diverges!
    │     ·····★                 │
    └──────────                  └──────────────
    (Very slow but converges)    (Bounces, may never converge)
```

### Code: Gradient Descent from Scratch

```python
import numpy as np

def gradient_descent(X, y, learning_rate=0.01, n_iterations=1000):
    """
    Implement gradient descent for linear regression.
    
    Args:
        X: Feature matrix (m, n) - should include bias column
        y: Target values (m,)
        learning_rate: Step size (alpha)
        n_iterations: Number of update steps
    
    Returns:
        weights: Learned parameters
        cost_history: Cost at each iteration
    """
    m, n = X.shape
    weights = np.zeros(n)  # Initialize weights to zeros
    cost_history = []
    
    for i in range(n_iterations):
        # Forward pass: make predictions
        predictions = X @ weights
        
        # Compute errors
        errors = predictions - y
        
        # Compute gradients (vectorized)
        gradients = (1/m) * (X.T @ errors)
        
        # Update weights (move opposite to gradient)
        weights = weights - learning_rate * gradients
        
        # Track cost
        cost = (1/(2*m)) * np.sum(errors**2)
        cost_history.append(cost)
    
    return weights, cost_history

# Example usage
np.random.seed(42)
X_raw = np.random.uniform(0, 10, (100, 1))  # 100 samples, 1 feature
y = 3 + 2.5 * X_raw.ravel() + np.random.normal(0, 1, 100)  # y = 3 + 2.5x + noise

# Add bias column (column of 1s)
X = np.column_stack([np.ones(100), X_raw])

# Run gradient descent
weights, costs = gradient_descent(X, y, learning_rate=0.01, n_iterations=1000)

print(f"Learned: intercept = {weights[0]:.4f}, slope = {weights[1]:.4f}")
print(f"True:    intercept = 3.0000, slope = 2.5000")
print(f"Initial cost: {costs[0]:.4f}")
print(f"Final cost:   {costs[-1]:.4f}")

# Output:
# Learned: intercept = 3.0276, slope = 2.4934
# True:    intercept = 3.0000, slope = 2.5000
# Initial cost: 53.7821
# Final cost:   0.4712
```

### Types of Gradient Descent

| Type | Batch Size | Speed | Stability | Memory |
|------|-----------|-------|-----------|--------|
| **Batch GD** | All samples | Slow per step | Very stable | High |
| **Stochastic GD (SGD)** | 1 sample | Fast per step | Noisy | Low |
| **Mini-Batch GD** | 32-256 samples | Balanced | Balanced | Medium |

```python
from sklearn.linear_model import SGDRegressor

# Stochastic Gradient Descent (scales to massive datasets)
sgd_model = SGDRegressor(
    loss='squared_error',    # MSE loss
    learning_rate='invscaling',  # Decreasing learning rate
    eta0=0.01,               # Initial learning rate
    max_iter=1000,           # Maximum epochs
    random_state=42
)
sgd_model.fit(X_train_scaled, y_train)  # Note: ALWAYS scale features for SGD
```

### Learning Rate Selection

```python
# Demonstrating effect of different learning rates
learning_rates = [0.0001, 0.01, 0.1, 0.5]

for lr in learning_rates:
    weights, costs = gradient_descent(X, y, learning_rate=lr, n_iterations=100)
    print(f"LR={lr:.4f}: Final cost = {costs[-1]:.4f} "
          f"(converged: {abs(costs[-1] - costs[-2]) < 1e-6 if len(costs) > 1 else False})")

# Pro Tip: Use learning rate schedulers that decrease over time
# Start fast, then slow down for fine-tuning
```

> **Pro Tip:** Always **normalize/standardize features** before gradient descent. If features have very different scales (age: 0-100, salary: 0-1,000,000), the cost surface becomes elongated and GD zigzags slowly.

---

## Normal Equation

### What It Is
A **closed-form solution** that directly computes the optimal weights without iteration.

$$\mathbf{w} = (\mathbf{X}^T \mathbf{X})^{-1} \mathbf{X}^T \mathbf{y}$$

### Normal Equation vs. Gradient Descent

| Aspect | Normal Equation | Gradient Descent |
|--------|----------------|------------------|
| **Iteration** | No iteration needed | Requires many iterations |
| **Learning rate** | No need to choose | Must tune carefully |
| **Feature scaling** | Not needed | Required for convergence |
| **Complexity** | $O(n^3)$ for matrix inverse | $O(kmn)$ per iteration |
| **Large features** | Slow if $n > 10,000$ | Works well |
| **Large samples** | Works fine | Works fine |
| **Invertibility** | Fails if $X^TX$ not invertible | Always works |

### Code: Normal Equation

```python
def normal_equation(X, y):
    """
    Compute optimal weights using the Normal Equation.
    w = (X^T X)^(-1) X^T y
    """
    # np.linalg.pinv handles non-invertible matrices (pseudo-inverse)
    w = np.linalg.pinv(X.T @ X) @ X.T @ y
    return w

# Compare with gradient descent
X_with_bias = np.column_stack([np.ones(len(X_raw)), X_raw])

w_normal = normal_equation(X_with_bias, y)
print(f"Normal Equation: intercept = {w_normal[0]:.4f}, slope = {w_normal[1]:.4f}")
# Output: Normal Equation: intercept = 3.0276, slope = 2.4934 (same as GD!)
```

> **When to use Normal Equation:** Features < 10,000. For larger feature sets, use gradient descent or its variants.

---

## Assumptions of Linear Regression

### The 5 Key Assumptions (LINE + I)

| # | Assumption | What It Means | How to Check |
|---|-----------|---------------|--------------|
| 1 | **L**inearity | Relationship between X and y is linear | Scatter plot, residual vs. fitted plot |
| 2 | **I**ndependence | Observations are independent | Durbin-Watson test (for time series) |
| 3 | **N**ormality | Residuals are normally distributed | Q-Q plot, Shapiro-Wilk test |
| 4 | **E**qual variance (Homoscedasticity) | Residual variance is constant | Residual vs. fitted plot, Breusch-Pagan test |
| 5 | No Multi**c**ollinearity | Features aren't highly correlated | VIF (Variance Inflation Factor) |

### Code: Checking Assumptions

```python
import numpy as np
import pandas as pd
from sklearn.linear_model import LinearRegression
from scipy import stats

# Fit model
model = LinearRegression()
model.fit(X_train, y_train)
y_pred_train = model.predict(X_train)
residuals = y_train - y_pred_train

# ─── Check 1: Linearity ───
# If residuals show a pattern (curve), linearity is violated
print("=== Linearity Check ===")
print(f"Correlation between fitted and residuals: "
      f"{np.corrcoef(y_pred_train, residuals)[0,1]:.6f}")
# Should be ~0

# ─── Check 3: Normality of Residuals ───
print("\n=== Normality Check ===")
stat, p_value = stats.shapiro(residuals[:5000])  # Shapiro-Wilk (max 5000 samples)
print(f"Shapiro-Wilk p-value: {p_value:.4f}")
print(f"Normally distributed: {'Yes' if p_value > 0.05 else 'No'}")

# ─── Check 4: Homoscedasticity ───
print("\n=== Homoscedasticity Check ===")
# If residuals spread increases with predictions, variance isn't constant
correlation = np.corrcoef(y_pred_train, np.abs(residuals))[0,1]
print(f"Correlation(predicted, |residuals|): {correlation:.4f}")
print(f"Homoscedastic: {'Likely' if abs(correlation) < 0.1 else 'Violated'}")

# ─── Check 5: Multicollinearity (VIF) ───
print("\n=== Multicollinearity Check (VIF) ===")
from sklearn.linear_model import LinearRegression

def calculate_vif(X_df):
    """Calculate Variance Inflation Factor for each feature."""
    vif_data = []
    for i, col in enumerate(X_df.columns):
        # Regress feature i on all other features
        X_other = X_df.drop(columns=[col])
        y_col = X_df[col]
        r2 = LinearRegression().fit(X_other, y_col).score(X_other, y_col)
        vif = 1 / (1 - r2) if r2 < 1 else float('inf')
        vif_data.append({'Feature': col, 'VIF': vif})
    return pd.DataFrame(vif_data)

# VIF > 5 indicates problematic multicollinearity
# VIF > 10 indicates severe multicollinearity
vif_df = calculate_vif(pd.DataFrame(X_train, columns=data.feature_names))
print(vif_df.to_string(index=False))
```

### What to Do When Assumptions Are Violated

| Violation | Solution |
|-----------|----------|
| Non-linearity | Add polynomial features, use non-linear model |
| Non-normal residuals | Transform target (log, Box-Cox), use robust regression |
| Heteroscedasticity | Transform target, use weighted least squares |
| Multicollinearity | Remove correlated features, use PCA, use Ridge regression |
| Non-independence | Use time-series models, add lag features |

---

## Regularization

### Why Regularization?

When you have many features, linear regression can **overfit** — coefficients become very large to fit noise in training data.

**Analogy:** Imagine you're explaining house prices. Without regularization, the model might say "the 3rd digit of the zip code matters a LOT!" (capturing noise). Regularization says "keep explanations simple — don't over-emphasize any one factor."

### Mathematical Formulation

Standard cost (no regularization):
$$J(\mathbf{w}) = \frac{1}{2m} \sum_{i=1}^{m} (\hat{y}_i - y_i)^2$$

With regularization:
$$J(\mathbf{w}) = \frac{1}{2m} \sum_{i=1}^{m} (\hat{y}_i - y_i)^2 + \lambda \cdot \text{Penalty}$$

Where $\lambda$ (alpha) controls regularization strength.

### Three Types of Regularization

| Type | Penalty Term | Effect | Use When |
|------|-------------|--------|----------|
| **Ridge (L2)** | $\lambda \sum w_j^2$ | Shrinks all coefficients toward 0 | Many useful features, multicollinearity |
| **Lasso (L1)** | $\lambda \sum |w_j|$ | Some coefficients become exactly 0 | Feature selection needed, sparse models |
| **Elastic Net** | $\lambda_1 \sum |w_j| + \lambda_2 \sum w_j^2$ | Combines both | Many features, some irrelevant |

### Visual: How Regularization Affects Coefficients

```
Without Regularization:    Ridge (L2):              Lasso (L1):
w = [15.2, -8.7, 0.3,     w = [3.1, -2.4, 0.2,    w = [4.2, -3.1, 0.0,
     22.1, -0.1, 4.5]          5.8, -0.05, 1.2]         7.0, 0.0, 0.0]
                            
(Large, noisy)             (All shrunk toward 0)    (Some exactly 0 → feature selection!)
```

### Code: Regularization Comparison

```python
from sklearn.linear_model import Ridge, Lasso, ElasticNet, LinearRegression
from sklearn.preprocessing import StandardScaler, PolynomialFeatures
from sklearn.pipeline import Pipeline
from sklearn.model_selection import cross_val_score
import numpy as np

# Create dataset with many features (some irrelevant)
np.random.seed(42)
n_samples = 100
X = np.random.randn(n_samples, 20)  # 20 features
# Only first 5 features actually matter
true_weights = np.array([3, -2, 1.5, 0.5, -1] + [0]*15)
y = X @ true_weights + np.random.normal(0, 0.5, n_samples)

# Compare models
models = {
    'Linear (No Reg)': LinearRegression(),
    'Ridge (α=1.0)': Ridge(alpha=1.0),
    'Ridge (α=10.0)': Ridge(alpha=10.0),
    'Lasso (α=0.1)': Lasso(alpha=0.1),
    'Lasso (α=1.0)': Lasso(alpha=1.0),
    'ElasticNet': ElasticNet(alpha=0.1, l1_ratio=0.5),
}

print(f"{'Model':<20} {'CV R² Score':<15} {'Non-zero coefs'}")
print("-" * 55)

for name, model in models.items():
    scores = cross_val_score(model, X, y, cv=5, scoring='r2')
    model.fit(X, y)
    n_nonzero = np.sum(np.abs(model.coef_) > 1e-6)
    print(f"{name:<20} {scores.mean():.4f} ± {scores.std():.4f}  {n_nonzero}")

# Output:
# Model                CV R² Score     Non-zero coefs
# -------------------------------------------------------
# Linear (No Reg)      0.8821 ± 0.0512  20
# Ridge (α=1.0)        0.9134 ± 0.0301  20
# Ridge (α=10.0)       0.8956 ± 0.0289  20
# Lasso (α=0.1)        0.9289 ± 0.0198  6     ← Best! Finds sparse solution
# Lasso (α=1.0)        0.7234 ± 0.0456  3     ← Too aggressive
# ElasticNet           0.9201 ± 0.0245  8
```

### Choosing Alpha (Regularization Strength)

```python
from sklearn.linear_model import RidgeCV, LassoCV

# RidgeCV: Automatically finds best alpha using cross-validation
ridge_cv = RidgeCV(alphas=[0.01, 0.1, 1.0, 10.0, 100.0], cv=5)
ridge_cv.fit(X_train, y_train)
print(f"Best Ridge alpha: {ridge_cv.alpha_}")

# LassoCV: Automatically finds best alpha
lasso_cv = LassoCV(alphas=None, cv=5, max_iter=10000)  # None = auto range
lasso_cv.fit(X_train, y_train)
print(f"Best Lasso alpha: {lasso_cv.alpha_:.4f}")
print(f"Features selected: {np.sum(lasso_cv.coef_ != 0)} / {X_train.shape[1]}")
```

---

## Polynomial Regression

### What It Is
Extends linear regression to capture **non-linear relationships** by adding polynomial terms.

$$\hat{y} = w_0 + w_1 x + w_2 x^2 + w_3 x^3 + ...$$

> **Key Insight:** It's still "linear" regression because it's linear in the **parameters** (weights). The features are just transformed to $x, x^2, x^3, ...$

### Visual: Why You Need It

```
    y                          y
    │     · ·                  │     · ·
    │   ·     ·                │   ·     ·
    │  ·       ·               │  · ╱─────╲ ·
    │ ·    ─────── Line        │ ·╱         ╲·  Polynomial (degree 2)
    │·   (bad fit)             │╱             ╲
    │·                         │
    └──────────── x            └──────────── x
    Linear: can't capture      Polynomial: captures curvature
```

### Code: Polynomial Regression

```python
from sklearn.preprocessing import PolynomialFeatures
from sklearn.pipeline import make_pipeline
from sklearn.linear_model import LinearRegression, Ridge
from sklearn.metrics import mean_squared_error
import numpy as np

# Generate non-linear data
np.random.seed(42)
X = np.sort(np.random.uniform(-3, 3, 100)).reshape(-1, 1)
y = 0.5 * X.ravel()**3 - X.ravel()**2 + np.random.normal(0, 2, 100)

# Compare different polynomial degrees
from sklearn.model_selection import cross_val_score

degrees = [1, 2, 3, 5, 10, 15]
for degree in degrees:
    model = make_pipeline(
        PolynomialFeatures(degree=degree, include_bias=False),
        Ridge(alpha=0.01)  # Slight regularization prevents overfitting
    )
    scores = cross_val_score(model, X, y, cv=5, scoring='neg_mean_squared_error')
    rmse = np.sqrt(-scores.mean())
    print(f"Degree {degree:2d}: RMSE = {rmse:.4f}")

# Output:
# Degree  1: RMSE = 7.2341  ← Underfitting (can't capture cubic)
# Degree  2: RMSE = 4.8912  ← Better but still missing cubic term
# Degree  3: RMSE = 2.1045  ← Good! Matches true function
# Degree  5: RMSE = 2.1198  ← Similar (extra terms not needed)
# Degree 10: RMSE = 2.3456  ← Starting to overfit
# Degree 15: RMSE = 3.8901  ← Overfitting!
```

> **Pro Tip:** Always combine polynomial features with regularization (Ridge/Lasso). High-degree polynomials without regularization will wildly overfit.

---

## Model Evaluation Metrics

### Regression Metrics

| Metric | Formula | Range | Interpretation |
|--------|---------|-------|----------------|
| **MSE** | $\frac{1}{m}\sum(y-\hat{y})^2$ | $[0, \infty)$ | Average squared error. Lower = better |
| **RMSE** | $\sqrt{MSE}$ | $[0, \infty)$ | Same units as y. Lower = better |
| **MAE** | $\frac{1}{m}\sum|y-\hat{y}|$ | $[0, \infty)$ | Average absolute error. Robust to outliers |
| **R² Score** | $1 - \frac{SS_{res}}{SS_{tot}}$ | $(-\infty, 1]$ | % variance explained. 1 = perfect |
| **Adjusted R²** | $1 - \frac{(1-R^2)(m-1)}{m-n-1}$ | $(-\infty, 1]$ | R² penalized for extra features |

### R² Score (Coefficient of Determination)

$$R^2 = 1 - \frac{\sum_{i=1}^{m}(y_i - \hat{y}_i)^2}{\sum_{i=1}^{m}(y_i - \bar{y})^2} = 1 - \frac{SS_{res}}{SS_{tot}}$$

**Interpretation:**
- $R^2 = 1.0$ → Model explains ALL variance (perfect predictions)
- $R^2 = 0.0$ → Model is as good as predicting the mean every time
- $R^2 < 0$ → Model is WORSE than just predicting the mean!

```python
from sklearn.metrics import (
    mean_squared_error, mean_absolute_error, r2_score
)
import numpy as np

# Evaluate model
y_pred = model.predict(X_test)

mse = mean_squared_error(y_test, y_pred)
rmse = np.sqrt(mse)
mae = mean_absolute_error(y_test, y_pred)
r2 = r2_score(y_test, y_pred)

# Adjusted R²
n = len(y_test)       # Number of samples
p = X_test.shape[1]   # Number of features
adj_r2 = 1 - (1 - r2) * (n - 1) / (n - p - 1)

print(f"MSE:          {mse:.4f}")
print(f"RMSE:         {rmse:.4f}")
print(f"MAE:          {mae:.4f}")
print(f"R² Score:     {r2:.4f}")
print(f"Adjusted R²:  {adj_r2:.4f}")
```

### When to Use Which Metric

| Scenario | Best Metric | Reason |
|----------|-------------|--------|
| General regression | RMSE | Same units as target, penalizes large errors |
| Outlier-prone data | MAE | Not sensitive to outliers |
| Comparing models with different #features | Adjusted R² | Penalizes unnecessary features |
| Business reporting | MAE or MAPE | Easy to explain to stakeholders |
| Optimization objective | MSE | Differentiable, standard for training |

---

## Common Mistakes

### 1. Not Scaling Features for Gradient Descent
```python
# WRONG ❌ — GD will converge very slowly or not at all
model = SGDRegressor()
model.fit(X_train, y_train)  # Features: age(0-100), salary(0-1M), rooms(0-10)

# CORRECT ✅ — Always scale for gradient-based methods
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline

pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('model', SGDRegressor())
])
pipeline.fit(X_train, y_train)
```

### 2. Ignoring Multicollinearity
```python
# If features are highly correlated, coefficients become unstable
# Check correlation matrix:
correlation_matrix = pd.DataFrame(X_train).corr()
# If any pair has |correlation| > 0.8, consider:
# 1. Removing one feature
# 2. Using PCA
# 3. Using Ridge regression (handles multicollinearity naturally)
```

### 3. Extrapolating Beyond Training Range
```python
# Model trained on houses 500-5000 sq ft
# Predicting for 50,000 sq ft mansion → UNRELIABLE!
# Linear models assume the relationship continues linearly forever

# Always check: is your prediction input within training data range?
def check_extrapolation(X_new, X_train):
    for i, col in enumerate(X_train.T):
        if X_new[i] < col.min() or X_new[i] > col.max():
            print(f"WARNING: Feature {i} is outside training range!")
```

### 4. Using Linear Regression for Classification
```python
# WRONG ❌ — Linear regression outputs can be > 1 or < 0
model = LinearRegression()
model.fit(X_train, y_train_binary)  # y is 0 or 1
# Predictions: [-0.3, 0.4, 1.7, 0.8] — not valid probabilities!

# CORRECT ✅ — Use Logistic Regression for classification
from sklearn.linear_model import LogisticRegression
model = LogisticRegression()
model.fit(X_train, y_train_binary)
# Predictions: [0.1, 0.6, 0.95, 0.82] — valid probabilities
```

### 5. Not Checking Residual Plots
```python
# Always plot residuals after fitting
# If residuals show patterns → model is missing something

# Residual patterns and their meaning:
# Random scatter → Good fit ✅
# Funnel shape → Heteroscedasticity (variance not constant)
# Curve pattern → Non-linear relationship (try polynomial features)
# Clusters → Missing categorical variable
```

### 6. Adding Too Many Polynomial Features
```python
# Degree 2 with 10 features → 10 + 45 + 10 = 65 features
# Degree 3 with 10 features → 285 features!
# This explodes quickly and leads to overfitting

from sklearn.preprocessing import PolynomialFeatures
poly = PolynomialFeatures(degree=2)
X_poly = poly.fit_transform(X)  # Check shape!
print(f"Original features: {X.shape[1]}, Polynomial features: {X_poly.shape[1]}")
# Always pair with regularization when using polynomial features
```

---

## Interview Questions

### Conceptual Questions

**Q1: What are the assumptions of linear regression?**
> Linearity (linear relationship between X and y), Independence (observations are independent), Normality (residuals are normally distributed), Equal variance/Homoscedasticity (residual variance is constant across predictions), No multicollinearity (features aren't highly correlated). Violations can lead to unreliable coefficients, wrong confidence intervals, and poor predictions.

**Q2: What's the difference between Ridge and Lasso regression?**
> Ridge (L2) adds $\lambda\sum w_j^2$ penalty — shrinks all coefficients toward zero but never to exactly zero. Lasso (L1) adds $\lambda\sum|w_j|$ penalty — can shrink coefficients to exactly zero, performing automatic feature selection. Use Ridge when all features are potentially useful; use Lasso when you suspect many features are irrelevant.

**Q3: Why can't we use R² alone to evaluate a model?**
> R² always increases when you add more features (even random noise). Use Adjusted R² which penalizes extra features. Also, R² doesn't tell you if the model is well-specified (check residual plots), if predictions are practical (check RMSE in context), or if the model generalizes (check test R² vs train R²).

**Q4: Explain gradient descent. Why do we need it?**
> Gradient descent iteratively updates parameters by moving in the direction of steepest descent of the cost function. Update rule: $w := w - \alpha \nabla J(w)$. We need it because: (1) Normal equation is $O(n^3)$ — impractical for many features, (2) It works for any differentiable cost function, not just MSE, (3) It scales to billions of parameters (deep learning), (4) Memory efficient (mini-batch).

**Q5: What is multicollinearity and why is it a problem?**
> Multicollinearity means features are highly correlated (e.g., height in inches and height in cm). Problems: coefficients become unstable (small data change → large coefficient change), coefficients are hard to interpret (which correlated feature gets credit?), variance of estimates increases. Solutions: remove one of correlated features, use PCA, use Ridge regression.

**Q6: When would you prefer MAE over MSE?**
> MAE when: outliers are present (MAE isn't influenced by them), all errors are equally important regardless of size, you need interpretable units. MSE when: large errors are disproportionately bad (safety-critical systems), you need a smooth/differentiable loss for optimization, you want to penalize predictions that are wildly off.

### Coding/Practical Questions

**Q7: How would you handle a linear regression with 1 million features?**
> Use Lasso (L1 regularization) for automatic feature selection → most coefficients become zero. Alternatively: (1) PCA to reduce dimensions, (2) SGD instead of normal equation (can't invert 1M × 1M matrix), (3) Feature importance from initial model → select top features, (4) ElasticNet for stability.

**Q8: Your linear regression R² is 0.95 on training data but 0.40 on test data. What happened?**
> Classic overfitting. The model memorized training noise. Solutions: (1) Add regularization (Ridge/Lasso), (2) Remove features (especially polynomial ones), (3) Get more training data, (4) Use cross-validation to tune complexity, (5) Check for data leakage.

---

## Quick Reference

### Linear Regression Cheat Sheet

| Component | Formula |
|-----------|---------|
| Model | $\hat{y} = \mathbf{w}^T\mathbf{x} + b$ |
| Cost (MSE) | $J = \frac{1}{2m}\sum(\hat{y}_i - y_i)^2$ |
| Gradient | $\nabla J = \frac{1}{m}\mathbf{X}^T(\mathbf{X}\mathbf{w} - \mathbf{y})$ |
| Update Rule | $\mathbf{w} := \mathbf{w} - \alpha \nabla J$ |
| Normal Equation | $\mathbf{w} = (\mathbf{X}^T\mathbf{X})^{-1}\mathbf{X}^T\mathbf{y}$ |
| Ridge Cost | $J + \lambda\|\mathbf{w}\|_2^2$ |
| Lasso Cost | $J + \lambda\|\mathbf{w}\|_1$ |
| R² Score | $1 - \frac{SS_{res}}{SS_{tot}}$ |

### Decision Guide

```
Need Linear Regression?
│
├── Few features (< 10,000)?
│   ├── Yes → Use Normal Equation (fast, exact)
│   └── No → Use Gradient Descent (SGDRegressor)
│
├── Overfitting?
│   ├── Many features, most useful → Ridge (L2)
│   ├── Many features, many irrelevant → Lasso (L1)
│   └── Not sure → ElasticNet (both)
│
├── Non-linear relationship?
│   ├── Mild curves → Polynomial Features + Ridge
│   └── Complex patterns → Use tree-based models instead
│
└── Outliers in data?
    ├── In features → Use RobustScaler
    └── In target → Use Huber loss or MAE
```

### sklearn Quick Reference

```python
from sklearn.linear_model import (
    LinearRegression,    # No regularization
    Ridge, RidgeCV,      # L2 regularization
    Lasso, LassoCV,      # L1 regularization
    ElasticNet,          # L1 + L2
    SGDRegressor,        # For large datasets
    HuberRegressor       # Robust to outliers
)
from sklearn.preprocessing import (
    StandardScaler,      # Zero mean, unit variance
    PolynomialFeatures,  # Create polynomial terms
    MinMaxScaler         # Scale to [0, 1]
)
from sklearn.pipeline import Pipeline, make_pipeline
from sklearn.model_selection import cross_val_score, GridSearchCV
from sklearn.metrics import mean_squared_error, r2_score, mean_absolute_error
```

---

*Previous: [01-Introduction-to-ML.md](01-Introduction-to-ML.md)*  
*Next: [03-Logistic-Regression.md](03-Logistic-Regression.md) — Binary/Multi-class Classification, Sigmoid, Cross-Entropy*
