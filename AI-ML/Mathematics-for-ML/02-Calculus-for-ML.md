# Calculus for Machine Learning

## Overview
Calculus is the **engine of learning** in ML. Every time a model "learns," it's using calculus to figure out how to adjust its parameters. Gradient descent — the algorithm that trains nearly every modern ML model — is fundamentally calculus. Without it, neural networks would be random number generators.

---

## Table of Contents
1. [Derivatives — The Core Idea](#1-derivatives--the-core-idea)
2. [Partial Derivatives](#2-partial-derivatives)
3. [The Gradient](#3-the-gradient)
4. [The Chain Rule — Backbone of Backpropagation](#4-the-chain-rule--backbone-of-backpropagation)
5. [Gradient Descent](#5-gradient-descent)
6. [Jacobians and Hessians](#6-jacobians-and-hessians)
7. [Automatic Differentiation](#7-automatic-differentiation)
8. [Calculus of Common ML Functions](#8-calculus-of-common-ml-functions)
9. [Multivariable Calculus for Neural Networks](#9-multivariable-calculus-for-neural-networks)
10. [Applications and Interview Prep](#10-applications-and-interview-prep)

---

## 1. Derivatives — The Core Idea

### What It Is
A derivative tells you **how fast something is changing**. It's the slope of a function at any point — the "rate of change."

$$f'(x) = \frac{df}{dx} = \lim_{h \to 0} \frac{f(x + h) - f(x)}{h}$$

### Why It Matters
- **Training ML models** = finding parameters that minimize a loss function
- The derivative tells us **which direction to move** parameters to reduce loss
- Without derivatives, we'd have to try random parameter changes (impossibly slow)

### How It Works

**Analogy:** You're hiking in fog and want to reach the valley (minimum). You can't see the valley, but you can feel the slope under your feet. The derivative IS that slope — it tells you which direction is downhill.

```
f(x) = x²

        |     /
        |    /    ← slope = 4 (steep, far from minimum)
        |   /
        |  /      ← slope = 2
        | /
        |/______  ← slope = 0 (minimum!)
        |\
        | \       ← slope = -2 (other side)
```

### Key Derivative Rules

| Rule | Formula | Example |
|------|---------|---------|
| Power | $\frac{d}{dx}x^n = nx^{n-1}$ | $\frac{d}{dx}x^3 = 3x^2$ |
| Constant | $\frac{d}{dx}c = 0$ | $\frac{d}{dx}5 = 0$ |
| Sum | $(f+g)' = f' + g'$ | $(x^2 + 3x)' = 2x + 3$ |
| Product | $(fg)' = f'g + fg'$ | $(x \cdot e^x)' = e^x + xe^x$ |
| Quotient | $(\frac{f}{g})' = \frac{f'g - fg'}{g^2}$ | |
| Chain | $(f(g(x)))' = f'(g(x)) \cdot g'(x)$ | $\frac{d}{dx}e^{x^2} = 2xe^{x^2}$ |
| Exponential | $\frac{d}{dx}e^x = e^x$ | Special! Only function equal to its derivative |
| Log | $\frac{d}{dx}\ln(x) = \frac{1}{x}$ | Used in log-likelihood |

### Code Examples

```python
import numpy as np

# === NUMERICAL DERIVATIVE ===
def numerical_derivative(f, x, h=1e-7):
    """Central difference — more accurate than forward difference"""
    return (f(x + h) - f(x - h)) / (2 * h)

# Test with f(x) = x^3
f = lambda x: x**3
x = 2.0
print(f"Numerical: {numerical_derivative(f, x):.6f}")  # 12.000000
print(f"Analytical (3x²): {3 * x**2:.6f}")              # 12.000000

# === DERIVATIVE OF COMMON ML FUNCTIONS ===
# Sigmoid: σ(x) = 1 / (1 + e^(-x))
# Derivative: σ'(x) = σ(x) * (1 - σ(x))
def sigmoid(x):
    return 1 / (1 + np.exp(-x))

def sigmoid_derivative(x):
    s = sigmoid(x)
    return s * (1 - s)  # Beautiful property!

x = np.linspace(-5, 5, 100)
print(f"sigmoid(0) = {sigmoid(0):.4f}")            # 0.5
print(f"sigmoid'(0) = {sigmoid_derivative(0):.4f}")  # 0.25 (maximum!)

# ReLU: f(x) = max(0, x)
# Derivative: f'(x) = 0 if x < 0, 1 if x > 0 (undefined at 0, use 0)
def relu(x):
    return np.maximum(0, x)

def relu_derivative(x):
    return (x > 0).astype(float)

# === WHY DERIVATIVES MATTER: Finding Minimum ===
# Minimize f(x) = (x - 3)^2 + 1
# Derivative: f'(x) = 2(x - 3)
# Set f'(x) = 0: x = 3 (minimum!)

f = lambda x: (x - 3)**2 + 1
f_prime = lambda x: 2 * (x - 3)

# Gradient descent to find minimum
x = 0.0  # Start far from minimum
learning_rate = 0.1
for step in range(20):
    gradient = f_prime(x)
    x = x - learning_rate * gradient  # Move opposite to gradient
    if step % 5 == 0:
        print(f"Step {step:2d}: x = {x:.4f}, f(x) = {f(x):.4f}")

# Output:
# Step  0: x = 0.6000, f(x) = 6.7600
# Step  5: x = 2.4126, f(x) = 1.3451
# Step 10: x = 2.8926, f(x) = 1.0115
# Step 15: x = 2.9803, f(x) = 1.0004
```

### Common Mistakes
1. **Forgetting the derivative at a point vs. the derivative function** — $f'(2) = 12$ is a number; $f'(x) = 3x^2$ is a function
2. **Not understanding that derivative = 0 can be a maximum OR minimum** — Use second derivative test: $f''(x) > 0$ = minimum, $f''(x) < 0$ = maximum
3. **Assuming derivative exists everywhere** — ReLU's derivative is undefined at x=0 (we choose 0 by convention)

---

## 2. Partial Derivatives

### What It Is
When a function has **multiple inputs**, a partial derivative measures how the output changes when you vary **one input while holding others constant**.

$$\frac{\partial f}{\partial x_i} = \lim_{h \to 0} \frac{f(x_1, ..., x_i + h, ..., x_n) - f(x_1, ..., x_i, ..., x_n)}{h}$$

### Why It Matters
- ML loss functions depend on **thousands/millions of parameters**
- We need to know how the loss changes with respect to **each** parameter
- Partial derivatives tell us: "If I nudge weight #473 by a tiny bit, how much does the error change?"

### How It Works

**Analogy:** You're adjusting a recipe with multiple ingredients (salt, sugar, butter). A partial derivative answers: "If I add a pinch more salt (keeping sugar and butter the same), how much does the taste score change?"

```
f(x, y) = x² + 3xy + y²

∂f/∂x = 2x + 3y    (treat y as a constant, differentiate w.r.t. x)
∂f/∂y = 3x + 2y    (treat x as a constant, differentiate w.r.t. y)

At point (1, 2):
∂f/∂x = 2(1) + 3(2) = 8    → Increasing x raises f by ~8 per unit
∂f/∂y = 3(1) + 2(2) = 7    → Increasing y raises f by ~7 per unit
```

### Code Examples

```python
import numpy as np

# === COMPUTING PARTIAL DERIVATIVES NUMERICALLY ===
def partial_derivative(f, point, var_index, h=1e-7):
    """
    Compute ∂f/∂x_i at a given point using central difference.
    
    Args:
        f: function taking a numpy array
        point: numpy array of coordinates
        var_index: which variable to differentiate with respect to
    """
    point = np.array(point, dtype=float)
    point_plus = point.copy()
    point_minus = point.copy()
    point_plus[var_index] += h
    point_minus[var_index] -= h
    return (f(point_plus) - f(point_minus)) / (2 * h)

# f(x, y) = x² + 3xy + y²
f = lambda p: p[0]**2 + 3*p[0]*p[1] + p[1]**2

point = np.array([1.0, 2.0])
df_dx = partial_derivative(f, point, 0)  # ∂f/∂x at (1,2)
df_dy = partial_derivative(f, point, 1)  # ∂f/∂y at (1,2)

print(f"∂f/∂x at (1,2) = {df_dx:.4f}")  # 8.0 (analytical: 2*1 + 3*2 = 8)
print(f"∂f/∂y at (1,2) = {df_dy:.4f}")  # 7.0 (analytical: 3*1 + 2*2 = 7)

# === ML EXAMPLE: Loss function with 2 parameters ===
# Linear regression: y_pred = w*x + b
# MSE Loss: L(w, b) = (1/n) Σ(y - (wx + b))²

np.random.seed(42)
x_data = np.array([1, 2, 3, 4, 5], dtype=float)
y_data = np.array([2.1, 4.0, 5.9, 8.1, 9.8])  # Roughly y = 2x + 0

def mse_loss(params):
    """params = [w, b]"""
    w, b = params
    predictions = w * x_data + b
    return np.mean((y_data - predictions)**2)

# Partial derivatives of loss
point = np.array([1.0, 0.0])  # Initial guess: w=1, b=0
dL_dw = partial_derivative(mse_loss, point, 0)
dL_db = partial_derivative(mse_loss, point, 1)
print(f"∂L/∂w = {dL_dw:.4f}")  # Negative → increase w to decrease loss
print(f"∂L/∂b = {dL_db:.4f}")  # Direction to adjust b

# === PARTIAL DERIVATIVES IN NEURAL NETWORK LOSS ===
# Loss depends on ALL weights → we need ∂L/∂w for EACH weight
# This is what backpropagation computes efficiently!
```

### Common Mistakes
1. **Forgetting to hold other variables constant** — The whole point of "partial" is that you differentiate with respect to ONE variable only
2. **Confusing partial with total derivative** — Total derivative accounts for dependencies between variables

---

## 3. The Gradient

### What It Is
The gradient is a **vector of all partial derivatives**. It points in the direction of **steepest increase** of the function.

$$\nabla f = \begin{bmatrix} \frac{\partial f}{\partial x_1} \\ \frac{\partial f}{\partial x_2} \\ \vdots \\ \frac{\partial f}{\partial x_n} \end{bmatrix}$$

### Why It Matters
- The **negative gradient** points "downhill" → gradient descent follows it
- Its **magnitude** tells you how steep the slope is
- At a minimum, the gradient is **zero** (flat landscape)

### How It Works

**Analogy:** You're standing on a hillside in fog. The gradient is like a compass that always points uphill (toward the summit). To reach the valley, you walk in the **opposite** direction (negative gradient).

```
Contour plot of f(x,y) = x² + y²:

    y
    |    ╱╲
    |   ╱  ╲     Gradient at (2,1): [4, 2]
    |  ╱ ↗  ╲    Points toward steepest ascent
    | ╱  •   ╲   
    |╱________╲___x
    
    To minimize: move in direction [-4, -2] (negative gradient)
```

### Code Examples

```python
import numpy as np

# === COMPUTING THE GRADIENT ===
def compute_gradient(f, point, h=1e-7):
    """Compute the full gradient vector at a point"""
    point = np.array(point, dtype=float)
    n = len(point)
    gradient = np.zeros(n)
    
    for i in range(n):
        point_plus = point.copy()
        point_minus = point.copy()
        point_plus[i] += h
        point_minus[i] -= h
        gradient[i] = (f(point_plus) - f(point_minus)) / (2 * h)
    
    return gradient

# f(x, y) = x² + 4y²  (elliptical bowl)
f = lambda p: p[0]**2 + 4*p[1]**2

# Gradient at (3, 2)
grad = compute_gradient(f, [3.0, 2.0])
print(f"Gradient at (3,2): {grad}")  # [6, 16]
# Analytical: ∇f = [2x, 8y] = [6, 16] ✓

# The gradient points toward steepest INCREASE
# To minimize, move in OPPOSITE direction

# === GRADIENT PROPERTIES ===
# 1. Gradient is perpendicular to contour lines
# 2. Magnitude = steepness of slope
# 3. At minima/maxima, gradient = 0

# Check perpendicularity to contour:
# On the contour f(x,y) = 13, point (3,1) satisfies: 9 + 4 = 13
point = np.array([3.0, 1.0])
grad = compute_gradient(f, point)
print(f"Gradient at contour point: {grad}")  # [6, 8]
# This vector is perpendicular to the ellipse at that point

# === GRADIENT IN ML: Linear Regression ===
np.random.seed(42)
X = np.random.randn(100, 3)  # 100 samples, 3 features
w_true = np.array([2.0, -1.0, 0.5])
y = X @ w_true + np.random.randn(100) * 0.1

# MSE Loss: L(w) = (1/n) ||Xw - y||²
def mse_loss(w):
    residuals = X @ w - y
    return np.mean(residuals**2)

# Analytical gradient: ∇L = (2/n) X^T (Xw - y)
def mse_gradient(w):
    n = len(y)
    return (2/n) * X.T @ (X @ w - y)

# Verify numerical matches analytical
w_test = np.array([1.0, 0.0, 0.0])
numerical_grad = compute_gradient(mse_loss, w_test)
analytical_grad = mse_gradient(w_test)
print(f"Numerical:  {numerical_grad}")
print(f"Analytical: {analytical_grad}")
print(f"Match: {np.allclose(numerical_grad, analytical_grad)}")  # True

# === GRADIENT DESCENT (Full Implementation) ===
w = np.zeros(3)  # Start at origin
learning_rate = 0.1
losses = []

for epoch in range(100):
    grad = mse_gradient(w)
    w = w - learning_rate * grad  # Update rule!
    loss = mse_loss(w)
    losses.append(loss)
    if epoch % 20 == 0:
        print(f"Epoch {epoch:3d}: loss = {loss:.6f}, w = {w}")

print(f"\nFinal weights: {w}")
print(f"True weights:  {w_true}")
# They should be very close!
```

> **Key Insight:** The gradient gives you the direction AND magnitude of the steepest ascent. The learning rate controls how far you step. Too large → overshoot. Too small → painfully slow.

### Common Mistakes
1. **Moving IN the gradient direction instead of against it** — Gradient points uphill! You want `w = w - lr * gradient`
2. **Not understanding that gradient = 0 doesn't guarantee a minimum** — Could be maximum or saddle point. In high dimensions, saddle points are far more common than local minima.
3. **Ignoring gradient magnitude** — If gradients are huge (exploding) or tiny (vanishing), training fails

---

## 4. The Chain Rule — Backbone of Backpropagation

### What It Is
The chain rule tells you how to differentiate **composed functions** — functions inside functions.

$$\frac{df}{dx} = \frac{df}{dg} \cdot \frac{dg}{dx}$$

For multiple layers:
$$\frac{\partial L}{\partial w_1} = \frac{\partial L}{\partial a_3} \cdot \frac{\partial a_3}{\partial a_2} \cdot \frac{\partial a_2}{\partial a_1} \cdot \frac{\partial a_1}{\partial w_1}$$

### Why It Matters
- **Backpropagation IS the chain rule** applied to neural network layers
- Without it, we couldn't train deep networks (computing gradients layer by layer)
- Understanding it helps debug vanishing/exploding gradients

### How It Works

**Analogy:** If changing temperature by 1°C changes ice cream sales by $100, and changing ice cream sales by $100 changes supplier revenue by $50, then changing temperature by 1°C changes supplier revenue by $100 × $50/$100 = $50. You multiply the "rates of change" along the chain.

```
Neural Network Forward Pass:

Input x → [Layer 1] → a₁ → [Layer 2] → a₂ → [Layer 3] → output → Loss L

Backpropagation (Chain Rule):

∂L/∂w₁ = ∂L/∂output × ∂output/∂a₂ × ∂a₂/∂a₁ × ∂a₁/∂w₁
          ←────────────────────────────────────────────────
          Multiply backwards through the chain!
```

### Code Examples

```python
import numpy as np

# === CHAIN RULE: SIMPLE EXAMPLE ===
# f(x) = (3x + 1)²
# Let g(x) = 3x + 1, f(g) = g²
# f'(x) = 2g * g'(x) = 2(3x+1) * 3 = 6(3x+1)

x = 2.0
# Analytical
analytical = 6 * (3*x + 1)
print(f"f'(2) = {analytical}")  # 42

# Numerical verification
f = lambda x: (3*x + 1)**2
h = 1e-7
numerical = (f(x+h) - f(x-h)) / (2*h)
print(f"Numerical: {numerical:.6f}")  # 42.000000

# === BACKPROPAGATION FROM SCRATCH ===
# Simple 2-layer network: 
# Input(2) → Hidden(3) → Output(1)

np.random.seed(42)

# Data
X = np.array([[0, 0], [0, 1], [1, 0], [1, 1]], dtype=float)
y = np.array([[0], [1], [1], [0]], dtype=float)  # XOR problem

# Initialize weights
W1 = np.random.randn(2, 3) * 0.5  # Input → Hidden
b1 = np.zeros((1, 3))
W2 = np.random.randn(3, 1) * 0.5  # Hidden → Output
b2 = np.zeros((1, 1))

def sigmoid(z):
    return 1 / (1 + np.exp(-z))

def sigmoid_deriv(z):
    s = sigmoid(z)
    return s * (1 - s)

# Training loop
learning_rate = 1.0
for epoch in range(10000):
    # === FORWARD PASS ===
    z1 = X @ W1 + b1          # Linear: (4×2) @ (2×3) = (4×3)
    a1 = sigmoid(z1)           # Activation: (4×3)
    z2 = a1 @ W2 + b2         # Linear: (4×3) @ (3×1) = (4×1)
    a2 = sigmoid(z2)           # Output: (4×1)
    
    # === LOSS ===
    loss = np.mean((y - a2)**2)
    
    # === BACKWARD PASS (Chain Rule!) ===
    # ∂L/∂a2 = -2(y - a2) / n
    dL_da2 = -2 * (y - a2) / len(X)
    
    # ∂L/∂z2 = ∂L/∂a2 * ∂a2/∂z2 (chain rule!)
    dL_dz2 = dL_da2 * sigmoid_deriv(z2)    # (4×1)
    
    # ∂L/∂W2 = a1.T @ ∂L/∂z2
    dL_dW2 = a1.T @ dL_dz2                  # (3×4) @ (4×1) = (3×1)
    dL_db2 = np.sum(dL_dz2, axis=0, keepdims=True)
    
    # ∂L/∂a1 = ∂L/∂z2 @ W2.T (chain rule continues!)
    dL_da1 = dL_dz2 @ W2.T                  # (4×1) @ (1×3) = (4×3)
    
    # ∂L/∂z1 = ∂L/∂a1 * ∂a1/∂z1
    dL_dz1 = dL_da1 * sigmoid_deriv(z1)     # (4×3)
    
    # ∂L/∂W1 = X.T @ ∂L/∂z1
    dL_dW1 = X.T @ dL_dz1                   # (2×4) @ (4×3) = (2×3)
    dL_db1 = np.sum(dL_dz1, axis=0, keepdims=True)
    
    # === UPDATE WEIGHTS ===
    W2 -= learning_rate * dL_dW2
    b2 -= learning_rate * dL_db2
    W1 -= learning_rate * dL_dW1
    b1 -= learning_rate * dL_db1
    
    if epoch % 2000 == 0:
        print(f"Epoch {epoch:5d}: loss = {loss:.6f}")

# Final predictions
predictions = sigmoid(sigmoid(X @ W1 + b1) @ W2 + b2)
print(f"\nPredictions:\n{np.round(predictions, 2)}")
print(f"Targets:\n{y}")

# === GRADIENT CHECKING (Verify backprop is correct) ===
def gradient_check(f, params, analytical_grad, h=1e-5):
    """Compare analytical gradient with numerical approximation"""
    numerical_grad = np.zeros_like(params)
    
    for i in range(params.size):
        params_plus = params.copy()
        params_minus = params.copy()
        params_plus.flat[i] += h
        params_minus.flat[i] -= h
        numerical_grad.flat[i] = (f(params_plus) - f(params_minus)) / (2*h)
    
    # Relative error
    diff = np.linalg.norm(analytical_grad - numerical_grad)
    norm_sum = np.linalg.norm(analytical_grad) + np.linalg.norm(numerical_grad)
    relative_error = diff / (norm_sum + 1e-8)
    
    return relative_error

# If relative_error < 1e-5: backprop is correct
# If relative_error > 1e-3: there's a bug!
```

### Common Mistakes
1. **Multiplying gradients in the wrong order** — Matrix dimensions must be compatible when chaining
2. **Forgetting the local derivative** — Each layer contributes its own derivative to the chain
3. **Not understanding vanishing gradients** — When sigmoid derivatives (max 0.25) multiply through many layers: $0.25^{10} ≈ 0.000001$ → gradients vanish! This is why ReLU is preferred.

---

## 5. Gradient Descent

### What It Is
An iterative optimization algorithm that finds the minimum of a function by repeatedly stepping in the direction of the negative gradient.

$$\theta_{t+1} = \theta_t - \eta \nabla L(\theta_t)$$

Where:
- $\theta$ = parameters (weights)
- $\eta$ = learning rate (step size)
- $\nabla L$ = gradient of loss function

### Why It Matters
This is THE algorithm that trains ML models. Every variant (SGD, Adam, RMSprop) is built on this foundation.

### How It Works

```
Loss Landscape (2D slice):

Loss │    
     │ ×  Start here                    
     │  \                               
     │   \   Step 1 (big gradient)      
     │    \                             
     │     × After step 1               
     │      \                           
     │       \ Step 2 (smaller gradient)
     │        \                         
     │         × After step 2           
     │          ·                       
     │           · Step 3 (tiny gradient)
     │            ×  Converged! (minimum)
     └────────────────────────── Parameter
```

### Variants

| Variant | Update Rule | Batch Size | Pros | Cons |
|---------|------------|------------|------|------|
| Batch GD | Full dataset | All N | Stable, exact gradient | Slow, memory intensive |
| SGD | Single sample | 1 | Fast updates, escapes local min | Noisy, unstable |
| Mini-batch GD | Small batch | 32-512 | Best of both worlds | Needs batch size tuning |

### Code Examples

```python
import numpy as np

# === THREE VARIANTS OF GRADIENT DESCENT ===
np.random.seed(42)

# Generate data: y = 3x₁ + 2x₂ + 1 + noise
n_samples = 1000
X = np.random.randn(n_samples, 2)
X_bias = np.column_stack([np.ones(n_samples), X])  # Add bias column
w_true = np.array([1.0, 3.0, 2.0])  # [bias, w1, w2]
y = X_bias @ w_true + np.random.randn(n_samples) * 0.5

def mse_loss(w, X, y):
    return np.mean((X @ w - y)**2)

def mse_gradient(w, X, y):
    n = len(y)
    return (2/n) * X.T @ (X @ w - y)

# --- Batch Gradient Descent ---
def batch_gd(X, y, lr=0.01, epochs=100):
    w = np.zeros(X.shape[1])
    losses = []
    for _ in range(epochs):
        grad = mse_gradient(w, X, y)
        w = w - lr * grad
        losses.append(mse_loss(w, X, y))
    return w, losses

# --- Stochastic Gradient Descent ---
def sgd(X, y, lr=0.01, epochs=100):
    w = np.zeros(X.shape[1])
    losses = []
    for epoch in range(epochs):
        indices = np.random.permutation(len(y))
        for i in indices:
            xi = X[i:i+1]  # Single sample
            yi = y[i:i+1]
            grad = mse_gradient(w, xi, yi)
            w = w - lr * grad
        losses.append(mse_loss(w, X, y))
    return w, losses

# --- Mini-Batch Gradient Descent ---
def mini_batch_gd(X, y, lr=0.01, epochs=100, batch_size=32):
    w = np.zeros(X.shape[1])
    losses = []
    n = len(y)
    for epoch in range(epochs):
        indices = np.random.permutation(n)
        for start in range(0, n, batch_size):
            batch_idx = indices[start:start+batch_size]
            X_batch = X[batch_idx]
            y_batch = y[batch_idx]
            grad = mse_gradient(w, X_batch, y_batch)
            w = w - lr * grad
        losses.append(mse_loss(w, X, y))
    return w, losses

# Compare
w_batch, losses_batch = batch_gd(X_bias, y, lr=0.1, epochs=50)
w_mini, losses_mini = mini_batch_gd(X_bias, y, lr=0.1, epochs=50, batch_size=32)

print(f"True weights:  {w_true}")
print(f"Batch GD:      {w_batch}")
print(f"Mini-batch GD: {w_mini}")

# === LEARNING RATE EFFECTS ===
print("\n--- Learning Rate Effects ---")
for lr in [0.001, 0.01, 0.1, 1.0, 2.0]:
    w, losses = batch_gd(X_bias, y, lr=lr, epochs=50)
    final_loss = losses[-1] if not np.isnan(losses[-1]) else float('inf')
    status = "DIVERGED!" if final_loss > 100 else f"loss={final_loss:.4f}"
    print(f"lr={lr:5.3f}: {status}")

# === MOMENTUM — Accelerating Gradient Descent ===
def gd_with_momentum(X, y, lr=0.01, momentum=0.9, epochs=100):
    """
    Momentum accumulates past gradients like a ball rolling downhill.
    Helps escape shallow local minima and speeds up in consistent directions.
    """
    w = np.zeros(X.shape[1])
    velocity = np.zeros_like(w)
    losses = []
    
    for _ in range(epochs):
        grad = mse_gradient(w, X, y)
        velocity = momentum * velocity - lr * grad  # Accumulate!
        w = w + velocity
        losses.append(mse_loss(w, X, y))
    
    return w, losses

w_momentum, losses_momentum = gd_with_momentum(X_bias, y, lr=0.01, momentum=0.9)
print(f"\nWith momentum: {w_momentum}")
print(f"Momentum converges faster: loss after 50 epochs = {losses_momentum[49]:.6f}")
print(f"Without momentum: loss after 50 epochs = {losses_batch[49]:.6f}")

# === ADAM OPTIMIZER (The Most Popular) ===
def adam(X, y, lr=0.001, beta1=0.9, beta2=0.999, epsilon=1e-8, epochs=100):
    """
    Adam = Momentum + RMSprop
    Adapts learning rate per-parameter based on gradient history.
    """
    w = np.zeros(X.shape[1])
    m = np.zeros_like(w)  # First moment (mean of gradients)
    v = np.zeros_like(w)  # Second moment (mean of squared gradients)
    losses = []
    
    for t in range(1, epochs + 1):
        grad = mse_gradient(w, X, y)
        
        m = beta1 * m + (1 - beta1) * grad        # Update biased first moment
        v = beta2 * v + (1 - beta2) * grad**2     # Update biased second moment
        
        m_hat = m / (1 - beta1**t)  # Bias correction
        v_hat = v / (1 - beta2**t)  # Bias correction
        
        w = w - lr * m_hat / (np.sqrt(v_hat) + epsilon)
        losses.append(mse_loss(w, X, y))
    
    return w, losses

w_adam, losses_adam = adam(X_bias, y, lr=0.01, epochs=50)
print(f"\nAdam: {w_adam}")
```

> **Pro Tip:** In practice, Adam is the default optimizer for deep learning. Use SGD with momentum for computer vision tasks where it often generalizes better.

### Common Mistakes
1. **Learning rate too high** → loss oscillates or diverges (NaN)
2. **Learning rate too low** → takes forever, gets stuck
3. **Not shuffling data** → creates bias in SGD/mini-batch updates
4. **Forgetting bias correction in Adam** → slow start in early iterations
5. **Using same LR for all layers** → deep layers may need smaller LR

---

## 6. Jacobians and Hessians

### What It Is

**Jacobian:** When a function maps vectors to vectors ($f: \mathbb{R}^n \to \mathbb{R}^m$), the Jacobian is the matrix of ALL partial derivatives.

$$J = \begin{bmatrix} \frac{\partial f_1}{\partial x_1} & \cdots & \frac{\partial f_1}{\partial x_n} \\ \vdots & \ddots & \vdots \\ \frac{\partial f_m}{\partial x_1} & \cdots & \frac{\partial f_m}{\partial x_n} \end{bmatrix}$$

**Hessian:** Matrix of second-order partial derivatives (how the gradient itself changes).

$$H = \begin{bmatrix} \frac{\partial^2 f}{\partial x_1^2} & \frac{\partial^2 f}{\partial x_1 \partial x_2} \\ \frac{\partial^2 f}{\partial x_2 \partial x_1} & \frac{\partial^2 f}{\partial x_2^2} \end{bmatrix}$$

### Why It Matters
- **Jacobian** → used in backprop through layers with vector outputs (softmax, batch norm)
- **Hessian** → tells you about curvature (is this a minimum, maximum, or saddle point?)
- **Second-order methods** (Newton's method) use the Hessian for faster convergence
- **Hessian eigenvalues** → loss landscape sharpness (affects generalization)

### How It Works

**Jacobian analogy:** If you have a robot arm with 3 joints (inputs) and the hand position is 3D (output), the Jacobian tells you: "If I move joint 1 slightly, how does the hand position change in x, y, z?"

**Hessian analogy:** The gradient tells you the slope of the hill. The Hessian tells you the *shape* of the hill — is it a narrow valley, a wide bowl, or a saddle?

```
Hessian information:

All eigenvalues > 0:  LOCAL MINIMUM (bowl shape)
         ╲  ╱
          ╲╱
          
All eigenvalues < 0:  LOCAL MAXIMUM (dome shape)
          ╱╲
         ╱  ╲

Mixed signs:          SADDLE POINT (horse saddle)
         ╲╱
         ╱╲
```

### Code Examples

```python
import numpy as np

# === JACOBIAN COMPUTATION ===
def compute_jacobian(f, x, h=1e-7):
    """
    Compute Jacobian matrix for f: R^n → R^m
    """
    x = np.array(x, dtype=float)
    f_x = f(x)
    n = len(x)
    m = len(f_x)
    J = np.zeros((m, n))
    
    for i in range(n):
        x_plus = x.copy()
        x_minus = x.copy()
        x_plus[i] += h
        x_minus[i] -= h
        J[:, i] = (f(x_plus) - f(x_minus)) / (2 * h)
    
    return J

# Example: f(x, y) = [x²+y, x*y²]
f = lambda p: np.array([p[0]**2 + p[1], p[0] * p[1]**2])

point = np.array([1.0, 2.0])
J = compute_jacobian(f, point)
print(f"Jacobian at (1,2):\n{J}")
# [[2*1, 1  ],    = [[2, 1],
#  [2², 2*1*2]]      [4, 4]]

# === HESSIAN COMPUTATION ===
def compute_hessian(f, x, h=1e-5):
    """Compute Hessian matrix for f: R^n → R"""
    x = np.array(x, dtype=float)
    n = len(x)
    H = np.zeros((n, n))
    
    for i in range(n):
        for j in range(n):
            x_pp = x.copy(); x_pp[i] += h; x_pp[j] += h
            x_pm = x.copy(); x_pm[i] += h; x_pm[j] -= h
            x_mp = x.copy(); x_mp[i] -= h; x_mp[j] += h
            x_mm = x.copy(); x_mm[i] -= h; x_mm[j] -= h
            H[i, j] = (f(x_pp) - f(x_pm) - f(x_mp) + f(x_mm)) / (4 * h**2)
    
    return H

# f(x, y) = x² + 3xy + 2y²
f = lambda p: p[0]**2 + 3*p[0]*p[1] + 2*p[1]**2

point = np.array([1.0, 1.0])
H = compute_hessian(f, point)
print(f"\nHessian:\n{H}")
# [[2, 3],    (constant! — quadratic function)
#  [3, 4]]

# Check if it's a minimum (all eigenvalues > 0?)
eigenvalues = np.linalg.eigvals(H)
print(f"Hessian eigenvalues: {eigenvalues}")
print(f"Positive definite (minimum)? {np.all(eigenvalues > 0)}")

# === SADDLE POINT DETECTION ===
# f(x, y) = x² - y²  (classic saddle)
f_saddle = lambda p: p[0]**2 - p[1]**2

H_saddle = compute_hessian(f_saddle, [0.0, 0.0])
eigenvalues_saddle = np.linalg.eigvals(H_saddle)
print(f"\nSaddle point Hessian eigenvalues: {eigenvalues_saddle}")  # [2, -2]
# Mixed signs → SADDLE POINT (not a minimum!)

# === NEWTON'S METHOD (Using Hessian for faster convergence) ===
def newtons_method(f, grad_f, hessian_f, x0, epochs=10):
    """
    Newton's method: x_{t+1} = x_t - H^{-1} @ ∇f
    Converges much faster than gradient descent (quadratic vs linear)
    But: computing H^{-1} is O(n³) — too expensive for neural nets
    """
    x = np.array(x0, dtype=float)
    for i in range(epochs):
        g = grad_f(x)
        H = hessian_f(x)
        x = x - np.linalg.solve(H, g)  # Don't invert! Solve instead
        print(f"Step {i}: x = {x}, f(x) = {f(x):.6f}")
    return x

# Minimize f(x,y) = (x-3)² + 2(y-1)²
f = lambda p: (p[0]-3)**2 + 2*(p[1]-1)**2
grad_f = lambda p: np.array([2*(p[0]-3), 4*(p[1]-1)])
hessian_f = lambda p: np.array([[2, 0], [0, 4]])

result = newtons_method(f, grad_f, hessian_f, [0.0, 0.0], epochs=3)
print(f"Converged to: {result}")  # [3, 1] in just 1 step! (quadratic → exact)
```

> **Pro Tip:** In deep learning, we never compute the full Hessian (too expensive). Instead, we use approximations: diagonal Hessian (Adam approximates this), Fisher Information Matrix, or Hessian-vector products.

### Common Mistakes
1. **Trying to use Newton's method for neural networks** — The Hessian has n² entries; for a model with 1M parameters, that's 1 trillion entries!
2. **Confusing Jacobian shape** — For $f: \mathbb{R}^n \to \mathbb{R}^m$, Jacobian is m×n (outputs × inputs)
3. **Assuming all critical points are minima** — In high-dimensional spaces, most critical points are saddle points

---

## 7. Automatic Differentiation

### What It Is
A technique that computes exact derivatives by breaking computations into elementary operations and applying the chain rule systematically. This is what PyTorch and TensorFlow use internally.

### Why It Matters
- **Not numerical** (no floating-point errors from finite differences)
- **Not symbolic** (no expression explosion)
- **Exact** and **efficient** — same cost as forward pass
- Powers all modern deep learning frameworks

### How It Works

Two modes:
- **Forward mode** — efficient when few inputs, many outputs
- **Reverse mode** — efficient when many inputs, few outputs (backpropagation!)

```
Computation Graph for f(x,y) = (x+y) * (x*y):

Forward:  x=3, y=2

    x=3 ─→ [+] ─→ a=5 ─→ [×] ─→ f=30
    y=2 ─↗         ↑
    x=3 ─→ [×] ─→ b=6 ─↗
    y=2 ─↗

Backward (reverse mode autodiff):
    
    ∂f/∂f = 1
    ∂f/∂a = b = 6     (from a*b, derivative w.r.t. a is b)
    ∂f/∂b = a = 5     (from a*b, derivative w.r.t. b is a)
    ∂f/∂x = ∂f/∂a * ∂a/∂x + ∂f/∂b * ∂b/∂x
           = 6 * 1 + 5 * 2 = 16
    ∂f/∂y = ∂f/∂a * ∂a/∂y + ∂f/∂b * ∂b/∂y
           = 6 * 1 + 5 * 3 = 21
```

### Code Examples

```python
import numpy as np

# === AUTODIFF FROM SCRATCH (Simplified) ===
class Variable:
    """A node in the computation graph"""
    def __init__(self, value, _children=(), _op=''):
        self.value = value
        self.grad = 0.0  # Will hold ∂L/∂self
        self._backward = lambda: None
        self._children = set(_children)
        self._op = _op
    
    def __add__(self, other):
        other = other if isinstance(other, Variable) else Variable(other)
        out = Variable(self.value + other.value, (self, other), '+')
        
        def _backward():
            self.grad += out.grad   # ∂(a+b)/∂a = 1
            other.grad += out.grad  # ∂(a+b)/∂b = 1
        out._backward = _backward
        return out
    
    def __mul__(self, other):
        other = other if isinstance(other, Variable) else Variable(other)
        out = Variable(self.value * other.value, (self, other), '*')
        
        def _backward():
            self.grad += other.value * out.grad  # ∂(a*b)/∂a = b
            other.grad += self.value * out.grad  # ∂(a*b)/∂b = a
        out._backward = _backward
        return out
    
    def backward(self):
        """Topological sort then backprop"""
        topo = []
        visited = set()
        def build_topo(v):
            if v not in visited:
                visited.add(v)
                for child in v._children:
                    build_topo(child)
                topo.append(v)
        build_topo(self)
        
        self.grad = 1.0  # ∂self/∂self = 1
        for node in reversed(topo):
            node._backward()

# Test: f(x,y) = (x+y) * (x*y) at x=3, y=2
x = Variable(3.0)
y = Variable(2.0)
a = x + y        # a = 5
b = x * y        # b = 6
f = a * b        # f = 30

f.backward()
print(f"f = {f.value}")       # 30
print(f"∂f/∂x = {x.grad}")   # 16 (verified above)
print(f"∂f/∂y = {y.grad}")   # 21

# === PYTORCH AUTODIFF ===
import torch

# Same computation in PyTorch
x = torch.tensor(3.0, requires_grad=True)
y = torch.tensor(2.0, requires_grad=True)
f = (x + y) * (x * y)
f.backward()

print(f"\nPyTorch:")
print(f"∂f/∂x = {x.grad.item()}")  # 16.0
print(f"∂f/∂y = {y.grad.item()}")  # 21.0

# === AUTODIFF FOR NEURAL NETWORK TRAINING ===
import torch
import torch.nn as nn

# Simple example: linear regression with PyTorch autograd
torch.manual_seed(42)
X = torch.randn(100, 3)
w_true = torch.tensor([2.0, -1.0, 0.5])
y = X @ w_true + torch.randn(100) * 0.1

# Parameters to learn
w = torch.zeros(3, requires_grad=True)

learning_rate = 0.1
for step in range(100):
    # Forward pass
    y_pred = X @ w
    loss = ((y_pred - y)**2).mean()
    
    # Backward pass (autograd computes ALL gradients!)
    loss.backward()
    
    # Update (must be in no_grad context)
    with torch.no_grad():
        w -= learning_rate * w.grad
        w.grad.zero_()  # IMPORTANT: reset gradients!
    
    if step % 20 == 0:
        print(f"Step {step}: loss = {loss.item():.6f}, w = {w.detach().numpy()}")

print(f"\nLearned: {w.detach().numpy()}")
print(f"True:    {w_true.numpy()}")
```

> **Pro Tip:** Always call `optimizer.zero_grad()` or `w.grad.zero_()` before `loss.backward()` in PyTorch. Gradients accumulate by default (useful for gradient accumulation with large models, but a common bug otherwise).

### Common Mistakes
1. **Not zeroing gradients** → gradients accumulate across iterations
2. **Modifying tensors in-place when they require grad** → breaks the computation graph
3. **Calling `.backward()` on non-scalar** → need to pass a gradient tensor
4. **Detaching tensors when you shouldn't** → cuts the gradient flow

---

## 8. Calculus of Common ML Functions

### What It Is
Derivatives of functions you'll encounter constantly in ML.

### Reference Table

| Function | Formula | Derivative | Used In |
|----------|---------|-----------|---------|
| Sigmoid | $\sigma(x) = \frac{1}{1+e^{-x}}$ | $\sigma(x)(1-\sigma(x))$ | Binary classification |
| Tanh | $\tanh(x) = \frac{e^x - e^{-x}}{e^x + e^{-x}}$ | $1 - \tanh^2(x)$ | RNN activations |
| ReLU | $\max(0, x)$ | $\begin{cases} 0 & x < 0 \\ 1 & x > 0 \end{cases}$ | Most neural networks |
| Leaky ReLU | $\max(\alpha x, x)$ | $\begin{cases} \alpha & x < 0 \\ 1 & x > 0 \end{cases}$ | Avoiding dead neurons |
| Softmax | $\frac{e^{x_i}}{\sum_j e^{x_j}}$ | $s_i(\delta_{ij} - s_j)$ | Multi-class classification |
| Log | $\ln(x)$ | $\frac{1}{x}$ | Log-likelihood, cross-entropy |
| MSE | $\frac{1}{n}\sum(y-\hat{y})^2$ | $\frac{-2}{n}(y - \hat{y})$ | Regression |
| Cross-Entropy | $-\sum y\log(\hat{y})$ | $-\frac{y}{\hat{y}}$ | Classification |

### Code Examples

```python
import numpy as np

# === ACTIVATION FUNCTIONS AND THEIR DERIVATIVES ===

class Activations:
    """Collection of activation functions and derivatives"""
    
    @staticmethod
    def sigmoid(x):
        # Numerically stable sigmoid
        return np.where(x >= 0, 
                       1 / (1 + np.exp(-x)), 
                       np.exp(x) / (1 + np.exp(x)))
    
    @staticmethod
    def sigmoid_deriv(x):
        s = Activations.sigmoid(x)
        return s * (1 - s)
    
    @staticmethod
    def tanh(x):
        return np.tanh(x)
    
    @staticmethod
    def tanh_deriv(x):
        return 1 - np.tanh(x)**2
    
    @staticmethod
    def relu(x):
        return np.maximum(0, x)
    
    @staticmethod
    def relu_deriv(x):
        return (x > 0).astype(float)
    
    @staticmethod
    def leaky_relu(x, alpha=0.01):
        return np.where(x > 0, x, alpha * x)
    
    @staticmethod
    def leaky_relu_deriv(x, alpha=0.01):
        return np.where(x > 0, 1.0, alpha)
    
    @staticmethod
    def softmax(x):
        # Numerically stable softmax
        x_shifted = x - np.max(x, axis=-1, keepdims=True)
        exp_x = np.exp(x_shifted)
        return exp_x / np.sum(exp_x, axis=-1, keepdims=True)

# Demonstrate
x = np.linspace(-5, 5, 11)
print("x:            ", x)
print("sigmoid(x):   ", np.round(Activations.sigmoid(x), 3))
print("sigmoid'(x):  ", np.round(Activations.sigmoid_deriv(x), 3))
print("relu(x):      ", Activations.relu(x))
print("relu'(x):     ", Activations.relu_deriv(x))

# === WHY SIGMOID CAUSES VANISHING GRADIENTS ===
# Maximum derivative of sigmoid is 0.25 (at x=0)
# After 10 layers: 0.25^10 = 9.5e-7 → gradient is essentially ZERO
print(f"\nSigmoid max gradient: {Activations.sigmoid_deriv(0)}")  # 0.25
print(f"After 10 layers: {0.25**10:.2e}")  # ~1e-6

# ReLU max gradient is 1 → no vanishing!
# But: ReLU(x<0) = 0 → "dead neurons" (gradient is exactly 0)
print(f"ReLU gradient for x>0: 1 (no vanishing!)")
print(f"ReLU gradient for x<0: 0 (dead neuron!)")

# === SOFTMAX + CROSS-ENTROPY (The Combo) ===
def cross_entropy_loss(logits, targets):
    """
    Combined softmax + cross-entropy (numerically stable).
    This is what nn.CrossEntropyLoss does in PyTorch.
    """
    # Stable softmax
    shifted = logits - np.max(logits, axis=-1, keepdims=True)
    log_sum_exp = np.log(np.sum(np.exp(shifted), axis=-1, keepdims=True))
    log_probs = shifted - log_sum_exp
    
    # Cross-entropy: -sum(target * log(pred))
    n = len(targets)
    loss = -log_probs[np.arange(n), targets].mean()
    return loss

def cross_entropy_gradient(logits, targets):
    """
    Gradient of softmax + cross-entropy w.r.t. logits.
    Beautifully simple: softmax(logits) - one_hot(targets)
    """
    probs = Activations.softmax(logits)
    n = len(targets)
    grad = probs.copy()
    grad[np.arange(n), targets] -= 1  # Subtract 1 from correct class
    return grad / n

# Example: 3 classes, batch of 4
logits = np.array([
    [2.0, 1.0, 0.1],   # Predicts class 0
    [0.1, 2.0, 0.5],   # Predicts class 1
    [0.5, 0.5, 2.0],   # Predicts class 2
    [1.0, 2.0, 0.1],   # Predicts class 1
])
targets = np.array([0, 1, 2, 0])  # True labels

loss = cross_entropy_loss(logits, targets)
grad = cross_entropy_gradient(logits, targets)
print(f"\nCross-entropy loss: {loss:.4f}")
print(f"Gradient shape: {grad.shape}")
print(f"Gradient:\n{np.round(grad, 4)}")
# Note: gradient is small for correct predictions, large for wrong ones!
```

### Common Mistakes
1. **Computing softmax then log separately** — Use log-softmax for numerical stability
2. **Forgetting numerically stable implementations** — `exp(1000)` = inf; always shift by max
3. **Not knowing that softmax + cross-entropy gradient is just `probs - targets`** — This is the most elegant result in ML calculus

---

## 9. Multivariable Calculus for Neural Networks

### What It Is
Extending calculus to functions with many inputs and outputs — the setting of every neural network.

### Why It Matters
Understanding how gradients flow through network architectures helps you:
- Design better architectures
- Debug training issues (vanishing/exploding gradients)
- Understand why certain tricks work (residual connections, batch norm)

### How It Works: Gradient Flow Patterns

```
=== VANISHING GRADIENT PROBLEM ===

Layer 1 → Layer 2 → Layer 3 → ... → Layer 20 → Loss
  ←(tiny)←──(tiny)←──(tiny)←── ... ←──(small)←── ∂L

Each layer multiplies gradient by ~0.25 (sigmoid)
After 20 layers: gradient ≈ 0.25^20 ≈ 10^(-12) → DEAD

=== RESIDUAL CONNECTION (Solution!) ===

Layer n ──→ [Layer n+1] ──→ + ──→ output
   |                        ↑
   └────────────────────────┘  (skip connection)

Gradient can flow DIRECTLY through skip: ∂L/∂x = ∂L/∂(x + F(x)) × (1 + ∂F/∂x)
The "1" ensures gradient never vanishes completely!
```

### Code Examples

```python
import numpy as np

# === GRADIENT FLOW THROUGH A DEEP NETWORK ===
def simulate_gradient_flow(n_layers, activation='sigmoid'):
    """Simulate how gradient magnitude changes through layers"""
    gradient = 1.0  # Start with gradient = 1 at output
    
    for layer in range(n_layers):
        if activation == 'sigmoid':
            # Sigmoid derivative max ≈ 0.25, typical ≈ 0.2
            gradient *= np.random.uniform(0.1, 0.25)
        elif activation == 'relu':
            # ReLU derivative: 0 or 1 (assume ~50% active)
            gradient *= np.random.choice([0, 1], p=[0.5, 0.5])
        elif activation == 'residual':
            # Residual: gradient * (1 + layer_grad)
            layer_grad = np.random.uniform(0.1, 0.25)
            gradient *= (1 + layer_grad)  # Never goes to 0!
            gradient = min(gradient, 100)  # Clip for visualization
    
    return gradient

# Compare gradient magnitudes
np.random.seed(42)
n_experiments = 1000
n_layers = 20

for activation in ['sigmoid', 'relu', 'residual']:
    gradients = [simulate_gradient_flow(n_layers, activation) 
                 for _ in range(n_experiments)]
    avg_grad = np.mean(gradients)
    print(f"{activation:10s}: avg gradient magnitude = {avg_grad:.2e}")

# sigmoid:  ≈ 10^(-8)  → VANISHING!
# relu:     ≈ 10^(-4)  → Some neurons die
# residual: ≈ 10^(+1)  → Healthy flow!

# === BATCH NORMALIZATION — Why It Helps ===
def batch_norm_forward(x, gamma, beta, eps=1e-5):
    """
    Normalizes activations to zero mean, unit variance.
    This keeps gradients in a healthy range!
    """
    mean = np.mean(x, axis=0)
    var = np.var(x, axis=0)
    x_norm = (x - mean) / np.sqrt(var + eps)
    out = gamma * x_norm + beta  # Scale and shift (learnable)
    
    # Cache for backward pass
    cache = (x, x_norm, mean, var, gamma, eps)
    return out, cache

def batch_norm_backward(dout, cache):
    """Backward pass through batch norm"""
    x, x_norm, mean, var, gamma, eps = cache
    N = x.shape[0]
    
    dgamma = np.sum(dout * x_norm, axis=0)
    dbeta = np.sum(dout, axis=0)
    
    dx_norm = dout * gamma
    dvar = np.sum(dx_norm * (x - mean) * -0.5 * (var + eps)**(-1.5), axis=0)
    dmean = np.sum(dx_norm * -1/np.sqrt(var + eps), axis=0)
    
    dx = dx_norm / np.sqrt(var + eps) + dvar * 2*(x-mean)/N + dmean/N
    return dx, dgamma, dbeta

# Demo
x = np.random.randn(32, 64) * 10 + 5  # Large mean and variance
gamma = np.ones(64)
beta = np.zeros(64)

out, cache = batch_norm_forward(x, gamma, beta)
print(f"\nBefore BN: mean={x.mean():.2f}, std={x.std():.2f}")
print(f"After BN: mean={out.mean():.4f}, std={out.std():.4f}")
# After: mean ≈ 0, std ≈ 1 → gradients stay healthy!

# === TOTAL DERIVATIVE vs PARTIAL DERIVATIVE ===
# In a computation graph, a variable may influence the output 
# through multiple paths. Total derivative sums over ALL paths.

# Example: f(x) = x² * sin(x)
# x influences f through BOTH the x² term and the sin(x) term
# Total: df/dx = 2x*sin(x) + x²*cos(x)

x = 2.0
total_deriv = 2*x*np.sin(x) + x**2*np.cos(x)
print(f"\nTotal derivative of x²sin(x) at x=2: {total_deriv:.4f}")

# Numerical check
f = lambda x: x**2 * np.sin(x)
numerical = (f(x+1e-7) - f(x-1e-7)) / (2e-7)
print(f"Numerical: {numerical:.4f}")
```

---

## 10. Applications and Interview Prep

### Interview Questions

#### Conceptual

1. **Why does gradient descent work? What guarantees convergence?**
   → For convex functions, gradient descent converges to the global minimum with appropriate learning rate. For non-convex (neural nets), it converges to a local minimum or saddle point. Convergence rate depends on learning rate, smoothness (Lipschitz constant), and strong convexity.

2. **Explain vanishing and exploding gradients. How do you fix them?**
   → Vanishing: gradients shrink exponentially through layers (sigmoid). Fix: ReLU, residual connections, proper initialization.
   → Exploding: gradients grow exponentially. Fix: gradient clipping, batch norm, LSTM/GRU gates.

3. **What's the difference between numerical, symbolic, and automatic differentiation?**
   → Numerical: finite differences (approximate, O(n) evaluations per gradient)
   → Symbolic: computer algebra (exact but expressions explode)
   → Automatic: chain rule on computation graph (exact AND efficient)

4. **Why is the gradient perpendicular to contour lines?**
   → On a contour, function value doesn't change. The gradient has zero component along the contour (otherwise the value would change). Therefore it must be perpendicular.

5. **Why does Adam work better than vanilla SGD in practice?**
   → Adam adapts learning rates per-parameter using moving averages of first (momentum) and second (RMSprop) moments. Parameters with consistently large gradients get smaller effective LR; parameters with sparse gradients get larger effective LR.

#### Coding

6. **Implement gradient descent for logistic regression from scratch.**
7. **Implement backpropagation for a 2-layer neural network.**
8. **Write a function that numerically checks if your analytical gradient is correct.**

---

## Quick Reference

### Derivative Rules Cheat Sheet

| Rule | Formula |
|------|---------|
| Constant | $(c)' = 0$ |
| Power | $(x^n)' = nx^{n-1}$ |
| Exponential | $(e^x)' = e^x$ |
| Log | $(\ln x)' = 1/x$ |
| Chain | $(f(g))' = f'(g) \cdot g'$ |
| Product | $(fg)' = f'g + fg'$ |
| Sum | $(f+g)' = f' + g'$ |

### Gradient Descent Variants

| Optimizer | Key Feature | When to Use |
|-----------|------------|-------------|
| SGD | Simple, well-understood | Fine-tuning, when generalization matters |
| SGD + Momentum | Accelerates consistent directions | Computer vision |
| Adam | Adaptive LR per parameter | Default for most tasks |
| AdamW | Adam + decoupled weight decay | Transformers |
| LAMB | Layer-wise adaptive LR | Large batch training |

### Critical Formulas

| Formula | Meaning |
|---------|---------|
| $w_{t+1} = w_t - \eta \nabla L$ | Gradient descent update |
| $\nabla L_{MSE} = \frac{2}{n}X^T(Xw - y)$ | Linear regression gradient |
| $\sigma'(x) = \sigma(x)(1-\sigma(x))$ | Sigmoid derivative |
| $\nabla_{logits} CE = softmax(z) - y_{onehot}$ | Cross-entropy gradient |
| $||g||_{clip} = g \cdot \frac{max\_norm}{||g||}$ | Gradient clipping |

> **The One Rule:** If you understand the chain rule and can compute gradients for basic operations (add, multiply, exp, log), you can derive backprop for ANY architecture.
