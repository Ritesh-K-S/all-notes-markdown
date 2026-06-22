# Chapter 04: Optimization Theory for Machine Learning

## Overview
Every machine learning algorithm, at its core, is solving an optimization problem. Training a model means finding the best parameters that minimize (or maximize) some objective function. This chapter covers the mathematics of optimization from intuitive foundations to advanced techniques used in modern deep learning.

---

## Table of Contents
1. [What is Optimization?](#1-what-is-optimization)
2. [Convex vs Non-Convex Optimization](#2-convex-vs-non-convex-optimization)
3. [Gradient Descent — The Foundation](#3-gradient-descent--the-foundation)
4. [Gradient Descent Variants](#4-gradient-descent-variants)
5. [Advanced Optimizers (Adam, RMSProp, etc.)](#5-advanced-optimizers)
6. [Learning Rate Scheduling](#6-learning-rate-scheduling)
7. [Constrained Optimization](#7-constrained-optimization)
8. [Second-Order Methods](#8-second-order-methods)
9. [Convergence Theory](#9-convergence-theory)
10. [Optimization in Practice](#10-optimization-in-practice)

---

## 1. What is Optimization?

### What It Is
Imagine you're blindfolded on a hilly landscape and you want to find the lowest valley. You can only feel the slope under your feet. Optimization is the mathematical process of finding the "best" point — usually the minimum or maximum of some function.

In ML:
- **The landscape** = the loss function (measures how wrong your model is)
- **Your position** = the current model parameters (weights)
- **Finding the valley** = finding parameters that make the model most accurate

### Why It Matters
- **Every ML model** uses optimization during training
- Neural networks with millions of parameters need efficient optimization
- Choice of optimizer can mean the difference between a model that works and one that doesn't
- Understanding optimization helps debug training failures (exploding gradients, stuck training, etc.)

### Mathematical Formulation

$$\min_{\theta} \mathcal{L}(\theta) = \min_{\theta} \frac{1}{N} \sum_{i=1}^{N} \ell(f(x_i; \theta), y_i)$$

Where:
- $\theta$ = model parameters (what we're optimizing)
- $\mathcal{L}$ = loss function (what we're minimizing)
- $f(x_i; \theta)$ = model prediction for input $x_i$
- $y_i$ = true label
- $N$ = number of training examples

### Key Terminology

| Term | Meaning | Analogy |
|------|---------|---------|
| Objective function | The function to minimize/maximize | The landscape you're exploring |
| Global minimum | The absolute lowest point | The deepest valley on Earth |
| Local minimum | A low point (but not the lowest) | A valley in the hills (not the deepest) |
| Saddle point | Flat area that's min in one direction, max in another | A mountain pass |
| Gradient | Direction of steepest ascent | Which way is "uphill" |
| Learning rate | Size of step taken | How far you walk each step |

```python
import numpy as np
import matplotlib.pyplot as plt

# Simple optimization example: Find the minimum of f(x) = (x - 3)^2 + 5
def f(x):
    """Our objective function - a simple parabola"""
    return (x - 3)**2 + 5

def f_derivative(x):
    """The gradient (derivative) of our function"""
    return 2 * (x - 3)

# Visualize the function
x = np.linspace(-2, 8, 100)
plt.figure(figsize=(10, 6))
plt.plot(x, f(x), 'b-', linewidth=2, label='f(x) = (x-3)² + 5')
plt.axvline(x=3, color='r', linestyle='--', label='Minimum at x=3')
plt.xlabel('x (parameter)')
plt.ylabel('f(x) (loss)')
plt.title('Simple Optimization Problem')
plt.legend()
plt.grid(True)
plt.show()

print(f"Minimum value: f(3) = {f(3)}")
print(f"Gradient at minimum: f'(3) = {f_derivative(3)}")  # Should be 0
```

---

## 2. Convex vs Non-Convex Optimization

### What It Is
A **convex function** is like a bowl — no matter where you start sliding, you'll always end up at the same bottom point. A **non-convex function** is like a mountain range with many valleys — where you end up depends on where you start.

### Why It Matters
- **Convex problems** are "easy" — any local minimum IS the global minimum
- **Non-convex problems** (like neural networks) are hard — you might get stuck in bad local minima
- Understanding convexity tells you what guarantees you can expect from your optimizer

### Mathematical Definition

A function $f$ is **convex** if for all $x, y$ and $\lambda \in [0, 1]$:

$$f(\lambda x + (1-\lambda)y) \leq \lambda f(x) + (1-\lambda) f(y)$$

**Intuition**: A line segment between any two points on the function lies above (or on) the function itself.

### Visual Comparison

```
CONVEX:                          NON-CONVEX:
                                      *
    *           *                *   * *   *
     *         *                  * *   * *
      *       *                    *     *
       *     *                          
        *   *                     Local    Local
         * *                      min      min
          *   ← Global min              *
                                        ↑ Global min
```

### Types of Critical Points

```
                    Saddle Point:
Local Maximum:      (min in x, max in y)     Local Minimum:
     *                    *                        *
    * *                  / \                      * *
   *   *               *   *                    *   *
  *     *             /     \                  *     *
                     *       *
```

### Conditions for Convexity

| Condition | Check | Example |
|-----------|-------|---------|
| Second derivative ≥ 0 | $f''(x) \geq 0$ for all $x$ | $f(x) = x^2$ → $f''(x) = 2 > 0$ ✓ |
| Hessian is PSD | $H \succeq 0$ (all eigenvalues ≥ 0) | Multivariate functions |
| Sum of convex is convex | $f + g$ convex if both convex | Regularized loss |
| Linear composition | $f(Ax + b)$ convex if $f$ convex | Affine transformation |

```python
import numpy as np
import matplotlib.pyplot as plt
from mpl_toolkits.mplot3d import Axes3D

# Example 1: Convex function
def convex_func(x):
    """MSE loss is convex for linear regression"""
    return x**2

# Example 2: Non-convex function (typical neural network loss landscape)
def non_convex_func(x):
    """Multiple local minima - like a neural network loss"""
    return x**4 - 5*x**2 + 4*x + 5

# Check convexity using second derivative
def second_derivative_check(f, x, h=1e-5):
    """Numerical second derivative to check convexity"""
    return (f(x + h) - 2*f(x) + f(x - h)) / h**2

x = np.linspace(-3, 3, 1000)

fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# Convex
axes[0].plot(x, convex_func(x), 'b-', linewidth=2)
axes[0].set_title('Convex Function: f(x) = x²\n(One minimum, easy to optimize)')
axes[0].grid(True)

# Non-convex
axes[1].plot(x, non_convex_func(x), 'r-', linewidth=2)
axes[1].set_title('Non-Convex: f(x) = x⁴ - 5x² + 4x + 5\n(Multiple minima, hard to optimize)')
axes[1].grid(True)

plt.tight_layout()
plt.show()

# Verify convexity
test_points = np.linspace(-3, 3, 100)
all_convex = all(second_derivative_check(convex_func, x) >= -1e-10 for x in test_points)
print(f"Is x² convex? {all_convex}")  # True
```

### ML Functions and Their Convexity

| Algorithm | Loss Function | Convex? | Implication |
|-----------|--------------|---------|-------------|
| Linear Regression | MSE | ✅ Yes | Global optimum guaranteed |
| Logistic Regression | Cross-entropy (w.r.t. weights) | ✅ Yes | Global optimum guaranteed |
| SVM | Hinge loss | ✅ Yes | Global optimum guaranteed |
| Neural Networks | Any loss | ❌ No | Local optima, depends on initialization |
| K-means | Within-cluster sum of squares | ❌ No | Results vary by initialization |

---

## 3. Gradient Descent — The Foundation

### What It Is
Gradient Descent is like walking downhill with your eyes closed — you feel which direction is steepest and take a step that way. Repeat until you reach the bottom.

### Why It Matters
- The **backbone of all modern ML training**
- Understanding GD helps you debug models that won't train
- Variants of GD (Adam, SGD with momentum) are just improvements on this basic idea

### How It Works

The update rule:

$$\theta_{t+1} = \theta_t - \eta \nabla \mathcal{L}(\theta_t)$$

Where:
- $\theta_t$ = current parameters
- $\eta$ = learning rate (step size)
- $\nabla \mathcal{L}(\theta_t)$ = gradient of loss at current position
- $\theta_{t+1}$ = updated parameters

**Intuition**: Move in the opposite direction of the gradient (steepest ascent → we go steepest descent).

### Step-by-Step Visualization

```
Step 1: Start at random point         Step 2: Compute gradient (slope)
        ↓                                      ↓
    *                                      * ← slope = -4
     \                                      \
      \  Current position                    \ →  Move right (opposite to negative gradient)
       \                                      \
        \_____*_____                           \_____*_____
              ↑                                      ↑
         Not minimum yet                        Getting closer!

Step 3: Take a step (lr × gradient)    Step 4: Repeat until convergence
        ↓                                      ↓
                                               At minimum!
         \                                     
          \ New position                            *
           *                               ________/ \________
            \_____                         Gradient ≈ 0 → STOP
```

### Complete Implementation from Scratch

```python
import numpy as np
import matplotlib.pyplot as plt

class GradientDescent:
    """Vanilla Gradient Descent optimizer implemented from scratch."""
    
    def __init__(self, learning_rate=0.01, max_iterations=1000, tolerance=1e-8):
        self.lr = learning_rate
        self.max_iter = max_iterations
        self.tol = tolerance
        self.history = []  # Store parameter history for visualization
    
    def optimize(self, gradient_fn, initial_params):
        """
        Find the minimum using gradient descent.
        
        Args:
            gradient_fn: Function that computes gradient at given params
            initial_params: Starting point (numpy array)
        
        Returns:
            Optimal parameters
        """
        params = np.array(initial_params, dtype=float)
        
        for i in range(self.max_iter):
            self.history.append(params.copy())
            
            # Compute gradient at current position
            grad = gradient_fn(params)
            
            # Update rule: move opposite to gradient
            params_new = params - self.lr * grad
            
            # Check convergence (if we barely moved, we're done)
            if np.linalg.norm(params_new - params) < self.tol:
                print(f"Converged at iteration {i}")
                break
            
            params = params_new
        
        self.history.append(params.copy())
        return params


# Example: Minimize f(x, y) = (x-2)^2 + (y-3)^2
# This is a simple bowl with minimum at (2, 3)

def gradient_fn(params):
    """Gradient of f(x,y) = (x-2)^2 + (y-3)^2"""
    x, y = params
    df_dx = 2 * (x - 2)  # Partial derivative w.r.t. x
    df_dy = 2 * (y - 3)  # Partial derivative w.r.t. y
    return np.array([df_dx, df_dy])

# Run optimization
optimizer = GradientDescent(learning_rate=0.1, max_iterations=100)
result = optimizer.optimize(gradient_fn, initial_params=[0.0, 0.0])
print(f"Found minimum at: ({result[0]:.4f}, {result[1]:.4f})")
print(f"Expected minimum: (2.0, 3.0)")

# Visualize the optimization path
history = np.array(optimizer.history)
x_range = np.linspace(-1, 4, 100)
y_range = np.linspace(-1, 5, 100)
X, Y = np.meshgrid(x_range, y_range)
Z = (X - 2)**2 + (Y - 3)**2

plt.figure(figsize=(10, 8))
plt.contour(X, Y, Z, levels=20, cmap='viridis')
plt.colorbar(label='f(x, y)')
plt.plot(history[:, 0], history[:, 1], 'ro-', markersize=4, label='GD Path')
plt.plot(2, 3, 'g*', markersize=15, label='True Minimum')
plt.xlabel('x')
plt.ylabel('y')
plt.title('Gradient Descent Optimization Path')
plt.legend()
plt.grid(True)
plt.show()
```

### Learning Rate Effect

```python
import numpy as np
import matplotlib.pyplot as plt

def f(x):
    return x**2

def grad_f(x):
    return 2*x

def run_gd(lr, start=5.0, steps=20):
    """Run GD with given learning rate and return trajectory"""
    path = [start]
    x = start
    for _ in range(steps):
        x = x - lr * grad_f(x)
        path.append(x)
    return path

# Compare different learning rates
fig, axes = plt.subplots(1, 3, figsize=(15, 4))
learning_rates = [0.01, 0.1, 1.1]
titles = ['Too Small (lr=0.01)\nSlow convergence', 
          'Just Right (lr=0.1)\nSmooth convergence',
          'Too Large (lr=1.1)\nDiverges!']

for ax, lr, title in zip(axes, learning_rates, titles):
    path = run_gd(lr)
    x_plot = np.linspace(-6, 6, 100)
    ax.plot(x_plot, f(x_plot), 'b-', linewidth=2)
    ax.plot(path, [f(x) for x in path], 'ro-', markersize=5)
    ax.set_title(title)
    ax.set_ylim(-1, 40)
    ax.grid(True)

plt.tight_layout()
plt.show()
```

> ⚠️ **Critical Insight**: Learning rate is the single most important hyperparameter. Too small = slow training. Too large = divergence. There's no universal "best" value — it depends on your problem.

---

## 4. Gradient Descent Variants

### What They Are
Different ways to compute the gradient and update parameters, each with different trade-offs between accuracy and speed.

### The Three Main Variants

| Variant | Data Used Per Step | Speed | Noise | Memory |
|---------|-------------------|-------|-------|--------|
| Batch GD | All N samples | Slow | None | High |
| Stochastic GD (SGD) | 1 sample | Fast | High | Low |
| Mini-batch GD | B samples (typically 32-256) | Balanced | Moderate | Moderate |

### Batch Gradient Descent

$$\theta_{t+1} = \theta_t - \eta \frac{1}{N} \sum_{i=1}^{N} \nabla \ell(f(x_i; \theta_t), y_i)$$

- Uses **all** training examples to compute gradient
- **Pro**: Stable, smooth convergence
- **Con**: Extremely slow for large datasets, high memory usage

### Stochastic Gradient Descent (SGD)

$$\theta_{t+1} = \theta_t - \eta \nabla \ell(f(x_i; \theta_t), y_i) \quad \text{(single random sample } i\text{)}$$

- Uses **one** random sample per update
- **Pro**: Very fast updates, can escape local minima (noise helps!)
- **Con**: Very noisy, unstable convergence

### Mini-batch Gradient Descent

$$\theta_{t+1} = \theta_t - \eta \frac{1}{B} \sum_{i \in \mathcal{B}} \nabla \ell(f(x_i; \theta_t), y_i)$$

- Uses a **random subset** (batch) of B samples
- **The standard in practice** — best of both worlds
- GPU-efficient (parallel computation on batches)

```python
import numpy as np

class LinearRegressionGD:
    """Linear Regression trained with different GD variants."""
    
    def __init__(self):
        self.weights = None
        self.bias = None
        self.loss_history = []
    
    def _compute_loss(self, X, y):
        """MSE loss"""
        predictions = X @ self.weights + self.bias
        return np.mean((predictions - y) ** 2)
    
    def _compute_gradients(self, X, y):
        """Compute gradients for weights and bias"""
        N = len(y)
        predictions = X @ self.weights + self.bias
        errors = predictions - y
        
        dw = (2/N) * (X.T @ errors)  # Gradient w.r.t. weights
        db = (2/N) * np.sum(errors)   # Gradient w.r.t. bias
        return dw, db
    
    def fit_batch_gd(self, X, y, lr=0.01, epochs=100):
        """Full Batch Gradient Descent — uses ALL data each step"""
        n_features = X.shape[1]
        self.weights = np.zeros(n_features)
        self.bias = 0.0
        self.loss_history = []
        
        for epoch in range(epochs):
            # Use ALL data to compute gradient
            dw, db = self._compute_gradients(X, y)
            self.weights -= lr * dw
            self.bias -= lr * db
            self.loss_history.append(self._compute_loss(X, y))
        
        return self
    
    def fit_sgd(self, X, y, lr=0.01, epochs=100):
        """Stochastic Gradient Descent — uses ONE sample each step"""
        n_samples, n_features = X.shape
        self.weights = np.zeros(n_features)
        self.bias = 0.0
        self.loss_history = []
        
        for epoch in range(epochs):
            # Shuffle data each epoch
            indices = np.random.permutation(n_samples)
            
            for i in indices:
                # Use SINGLE sample
                xi = X[i:i+1]  # Keep 2D shape
                yi = y[i:i+1]
                dw, db = self._compute_gradients(xi, yi)
                self.weights -= lr * dw
                self.bias -= lr * db
            
            self.loss_history.append(self._compute_loss(X, y))
        
        return self
    
    def fit_minibatch_gd(self, X, y, lr=0.01, epochs=100, batch_size=32):
        """Mini-batch GD — uses a SUBSET of data each step"""
        n_samples, n_features = X.shape
        self.weights = np.zeros(n_features)
        self.bias = 0.0
        self.loss_history = []
        
        for epoch in range(epochs):
            indices = np.random.permutation(n_samples)
            
            for start in range(0, n_samples, batch_size):
                # Use a BATCH of samples
                batch_idx = indices[start:start + batch_size]
                X_batch = X[batch_idx]
                y_batch = y[batch_idx]
                
                dw, db = self._compute_gradients(X_batch, y_batch)
                self.weights -= lr * dw
                self.bias -= lr * db
            
            self.loss_history.append(self._compute_loss(X, y))
        
        return self


# Generate synthetic data
np.random.seed(42)
X = np.random.randn(1000, 3)
true_weights = np.array([2.0, -1.5, 0.5])
y = X @ true_weights + 0.1 * np.random.randn(1000)

# Compare all three variants
model_batch = LinearRegressionGD().fit_batch_gd(X, y, lr=0.01, epochs=50)
model_sgd = LinearRegressionGD().fit_sgd(X, y, lr=0.001, epochs=50)
model_mini = LinearRegressionGD().fit_minibatch_gd(X, y, lr=0.01, epochs=50, batch_size=32)

# Plot convergence comparison
plt.figure(figsize=(10, 6))
plt.plot(model_batch.loss_history, label='Batch GD', linewidth=2)
plt.plot(model_sgd.loss_history, label='SGD', linewidth=2)
plt.plot(model_mini.loss_history, label='Mini-batch GD (B=32)', linewidth=2)
plt.xlabel('Epoch')
plt.ylabel('MSE Loss')
plt.title('Convergence Comparison of GD Variants')
plt.legend()
plt.grid(True)
plt.yscale('log')
plt.show()

print(f"True weights: {true_weights}")
print(f"Batch GD:     {model_batch.weights}")
print(f"SGD:          {model_sgd.weights}")
print(f"Mini-batch:   {model_mini.weights}")
```

### Momentum

**Problem**: SGD oscillates in ravines (narrow valleys).
**Solution**: Add "momentum" — like a ball rolling downhill that accumulates speed.

$$v_t = \gamma v_{t-1} + \eta \nabla \mathcal{L}(\theta_t)$$
$$\theta_{t+1} = \theta_t - v_t$$

Where $\gamma$ is the momentum coefficient (typically 0.9).

```
Without Momentum:              With Momentum:
   ↗ ↙ ↗ ↙ ↗ ↙ →            →→→→→→→→→→→→→
   (oscillates, slow)          (smooth, fast convergence)
```

```python
import numpy as np

class SGDWithMomentum:
    """SGD with Momentum - accelerates convergence in consistent directions."""
    
    def __init__(self, lr=0.01, momentum=0.9):
        self.lr = lr
        self.momentum = momentum
        self.velocity = None
    
    def step(self, params, gradient):
        """Perform one optimization step."""
        if self.velocity is None:
            self.velocity = np.zeros_like(params)
        
        # Accumulate velocity (like a rolling ball)
        self.velocity = self.momentum * self.velocity + self.lr * gradient
        
        # Update parameters
        params -= self.velocity
        return params


# Nesterov Accelerated Gradient (NAG) — "Look ahead" variant
class NesterovMomentum:
    """Nesterov Momentum - looks ahead before computing gradient."""
    
    def __init__(self, lr=0.01, momentum=0.9):
        self.lr = lr
        self.momentum = momentum
        self.velocity = None
    
    def step(self, params, gradient_fn):
        """
        Key insight: Compute gradient at the "look-ahead" position.
        This gives better convergence because we correct before overshooting.
        """
        if self.velocity is None:
            self.velocity = np.zeros_like(params)
        
        # Look ahead: where would momentum take us?
        look_ahead = params - self.momentum * self.velocity
        
        # Compute gradient at that future position
        gradient = gradient_fn(look_ahead)
        
        # Update velocity and parameters
        self.velocity = self.momentum * self.velocity + self.lr * gradient
        params -= self.velocity
        return params
```

---

## 5. Advanced Optimizers

### Why They Matter
Vanilla SGD and even momentum have limitations. Modern optimizers **adapt the learning rate per parameter** — parameters that get frequent updates get smaller learning rates, and rare parameters get larger ones.

### AdaGrad (Adaptive Gradient)

**Idea**: Accumulate squared gradients; divide learning rate by the square root of this sum.

$$g_t = \nabla \mathcal{L}(\theta_t)$$
$$G_t = G_{t-1} + g_t^2 \quad \text{(element-wise square)}$$
$$\theta_{t+1} = \theta_t - \frac{\eta}{\sqrt{G_t + \epsilon}} \cdot g_t$$

**Pro**: Great for sparse data (NLP, recommender systems)  
**Con**: Learning rate monotonically decreases — can stop learning too early

### RMSProp (Root Mean Square Propagation)

**Idea**: Like AdaGrad but uses exponential moving average (doesn't accumulate forever).

$$G_t = \beta G_{t-1} + (1-\beta) g_t^2$$
$$\theta_{t+1} = \theta_t - \frac{\eta}{\sqrt{G_t + \epsilon}} \cdot g_t$$

Where $\beta$ is typically 0.9.

**Pro**: Fixes AdaGrad's dying learning rate problem  
**Con**: Doesn't include momentum

### Adam (Adaptive Moment Estimation)

**The most popular optimizer in deep learning.** Combines momentum (first moment) + RMSProp (second moment) + bias correction.

$$m_t = \beta_1 m_{t-1} + (1 - \beta_1) g_t \quad \text{(First moment: momentum)}$$
$$v_t = \beta_2 v_{t-1} + (1 - \beta_2) g_t^2 \quad \text{(Second moment: squared gradients)}$$
$$\hat{m}_t = \frac{m_t}{1 - \beta_1^t} \quad \text{(Bias correction for first moment)}$$
$$\hat{v}_t = \frac{v_t}{1 - \beta_2^t} \quad \text{(Bias correction for second moment)}$$
$$\theta_{t+1} = \theta_t - \frac{\eta}{\sqrt{\hat{v}_t} + \epsilon} \cdot \hat{m}_t$$

Default hyperparameters: $\beta_1 = 0.9$, $\beta_2 = 0.999$, $\epsilon = 10^{-8}$

```python
import numpy as np

class Adam:
    """
    Adam Optimizer — implemented from scratch.
    The gold standard for deep learning optimization.
    """
    
    def __init__(self, lr=0.001, beta1=0.9, beta2=0.999, epsilon=1e-8):
        self.lr = lr
        self.beta1 = beta1       # Momentum decay rate
        self.beta2 = beta2       # RMSProp decay rate
        self.epsilon = epsilon   # Prevent division by zero
        self.m = None            # First moment (mean of gradients)
        self.v = None            # Second moment (mean of squared gradients)
        self.t = 0              # Time step (for bias correction)
    
    def step(self, params, gradient):
        """Perform one Adam update step."""
        self.t += 1
        
        if self.m is None:
            self.m = np.zeros_like(params)
            self.v = np.zeros_like(params)
        
        # Update biased first moment estimate (momentum)
        self.m = self.beta1 * self.m + (1 - self.beta1) * gradient
        
        # Update biased second moment estimate (RMSProp)
        self.v = self.beta2 * self.v + (1 - self.beta2) * gradient**2
        
        # Bias correction (crucial in early steps when m and v are near zero)
        m_hat = self.m / (1 - self.beta1**self.t)
        v_hat = self.v / (1 - self.beta2**self.t)
        
        # Update parameters
        params -= self.lr * m_hat / (np.sqrt(v_hat) + self.epsilon)
        return params


class AdamW:
    """
    AdamW — Adam with decoupled weight decay.
    Preferred over Adam in modern deep learning (better generalization).
    
    Key insight: L2 regularization ≠ weight decay in Adam!
    AdamW applies weight decay SEPARATELY from the gradient update.
    """
    
    def __init__(self, lr=0.001, beta1=0.9, beta2=0.999, epsilon=1e-8, weight_decay=0.01):
        self.lr = lr
        self.beta1 = beta1
        self.beta2 = beta2
        self.epsilon = epsilon
        self.weight_decay = weight_decay
        self.m = None
        self.v = None
        self.t = 0
    
    def step(self, params, gradient):
        """AdamW update — weight decay applied to params directly."""
        self.t += 1
        
        if self.m is None:
            self.m = np.zeros_like(params)
            self.v = np.zeros_like(params)
        
        # Standard Adam moment updates
        self.m = self.beta1 * self.m + (1 - self.beta1) * gradient
        self.v = self.beta2 * self.v + (1 - self.beta2) * gradient**2
        
        m_hat = self.m / (1 - self.beta1**self.t)
        v_hat = self.v / (1 - self.beta2**self.t)
        
        # DECOUPLED weight decay (applied to params, not gradient)
        params -= self.lr * self.weight_decay * params
        
        # Adam update
        params -= self.lr * m_hat / (np.sqrt(v_hat) + self.epsilon)
        return params


# Comparison: Optimize Rosenbrock function (a challenging test function)
def rosenbrock(params):
    """The Rosenbrock function — a classic optimization benchmark.
    Minimum at (1, 1) with value 0. Has a narrow curved valley."""
    x, y = params
    return (1 - x)**2 + 100*(y - x**2)**2

def rosenbrock_grad(params):
    """Gradient of Rosenbrock function."""
    x, y = params
    dx = -2*(1 - x) + 200*(y - x**2)*(-2*x)
    dy = 200*(y - x**2)
    return np.array([dx, dy])

# Run Adam on Rosenbrock
adam = Adam(lr=0.001)
params = np.array([-1.0, -1.0])
adam_path = [params.copy()]

for _ in range(10000):
    grad = rosenbrock_grad(params)
    params = adam.step(params, grad)
    adam_path.append(params.copy())

adam_path = np.array(adam_path)
print(f"Adam found minimum at: ({params[0]:.4f}, {params[1]:.4f})")
print(f"True minimum: (1.0, 1.0)")
print(f"Final loss: {rosenbrock(params):.6f}")
```

### Optimizer Comparison Table

| Optimizer | Learning Rate | Momentum | Adaptive LR | Best For |
|-----------|:---:|:---:|:---:|----------|
| SGD | Fixed | ❌ | ❌ | Simple problems, research baselines |
| SGD + Momentum | Fixed | ✅ | ❌ | Computer vision (with scheduling) |
| AdaGrad | Decays | ❌ | ✅ | Sparse data (NLP, recommendations) |
| RMSProp | Adaptive | ❌ | ✅ | RNNs, non-stationary problems |
| Adam | Adaptive | ✅ | ✅ | Default choice, most problems |
| AdamW | Adaptive | ✅ | ✅ | Modern deep learning (Transformers) |
| LAMB/LARS | Adaptive | ✅ | ✅ | Very large batch training |

> 💡 **Pro Tip**: In practice, start with **AdamW** (lr=3e-4). If you need the absolute best performance and have time to tune, switch to **SGD + Momentum** with cosine annealing schedule — it often generalizes better.

---

## 6. Learning Rate Scheduling

### What It Is
Instead of keeping the learning rate fixed, we change it during training. Start with a larger LR (explore broadly) and gradually decrease it (fine-tune).

### Why It Matters
- Fixed LR often can't achieve both fast convergence AND precise final solution
- Scheduling is used in virtually all state-of-the-art models
- Warmup prevents instability in early training (especially with Adam)

### Common Schedules

```python
import numpy as np
import matplotlib.pyplot as plt

def step_decay(epoch, initial_lr=0.1, drop_rate=0.5, drop_every=30):
    """Reduce LR by factor every N epochs."""
    return initial_lr * (drop_rate ** (epoch // drop_every))

def exponential_decay(epoch, initial_lr=0.1, decay_rate=0.95):
    """Exponential decay: lr = lr0 * decay^epoch"""
    return initial_lr * (decay_rate ** epoch)

def cosine_annealing(epoch, initial_lr=0.1, total_epochs=100):
    """Cosine annealing: smooth decrease following cosine curve."""
    return initial_lr * 0.5 * (1 + np.cos(np.pi * epoch / total_epochs))

def warmup_cosine(epoch, initial_lr=0.1, warmup_epochs=10, total_epochs=100):
    """Linear warmup followed by cosine decay — the modern standard."""
    if epoch < warmup_epochs:
        # Linear warmup
        return initial_lr * epoch / warmup_epochs
    else:
        # Cosine decay
        progress = (epoch - warmup_epochs) / (total_epochs - warmup_epochs)
        return initial_lr * 0.5 * (1 + np.cos(np.pi * progress))

def one_cycle(epoch, max_lr=0.1, total_epochs=100):
    """One-cycle policy: warmup to max, then decay below initial."""
    if epoch < total_epochs * 0.3:
        # Warmup phase (30% of training)
        return max_lr * epoch / (total_epochs * 0.3)
    else:
        # Decay phase (70% of training)
        progress = (epoch - total_epochs * 0.3) / (total_epochs * 0.7)
        return max_lr * 0.5 * (1 + np.cos(np.pi * progress))

# Visualize all schedules
epochs = np.arange(100)
fig, ax = plt.subplots(figsize=(12, 6))

schedules = {
    'Step Decay': [step_decay(e) for e in epochs],
    'Exponential': [exponential_decay(e) for e in epochs],
    'Cosine Annealing': [cosine_annealing(e) for e in epochs],
    'Warmup + Cosine': [warmup_cosine(e) for e in epochs],
    'One-Cycle': [one_cycle(e) for e in epochs],
}

for name, lrs in schedules.items():
    ax.plot(epochs, lrs, linewidth=2, label=name)

ax.set_xlabel('Epoch')
ax.set_ylabel('Learning Rate')
ax.set_title('Learning Rate Schedules Comparison')
ax.legend()
ax.grid(True)
plt.tight_layout()
plt.show()
```

### When to Use Which Schedule

| Schedule | Use Case | Example |
|----------|----------|---------|
| Step Decay | Classic CNNs (ResNet) | Divide by 10 at epochs 30, 60, 90 |
| Cosine Annealing | General deep learning | Transformers, modern CNNs |
| Warmup + Cosine | Large models, Transformers | BERT, GPT, ViT |
| One-Cycle | Fast training with super-convergence | Transfer learning, quick experiments |
| Reduce on Plateau | When you monitor validation loss | `ReduceLROnPlateau` in PyTorch |

---

## 7. Constrained Optimization

### What It Is
Sometimes we can't just minimize freely — there are **constraints** on what parameter values are valid.

**Example**: "Find the cheapest way to build a bridge, but it must hold 10 tons."

### Why It Matters
- **SVMs** use constrained optimization (maximize margin subject to classification constraints)
- **Regularization** can be viewed as constrained optimization
- Many real-world problems have physical or business constraints

### Types of Constraints

| Type | Form | Example |
|------|------|---------|
| Equality | $h(x) = 0$ | Parameters must sum to 1 (probability) |
| Inequality | $g(x) \leq 0$ | Weights must be non-negative |
| Box constraints | $a \leq x \leq b$ | Learning rate between 0 and 1 |

### Lagrange Multipliers (Equality Constraints)

**Problem**: Minimize $f(x)$ subject to $h(x) = 0$

**Solution**: Introduce a Lagrange multiplier $\lambda$ and solve:

$$\mathcal{L}(x, \lambda) = f(x) + \lambda h(x)$$

Set gradients to zero:
$$\nabla_x \mathcal{L} = 0 \quad \text{and} \quad \nabla_\lambda \mathcal{L} = 0$$

**Intuition**: At the optimum, the gradient of $f$ must be parallel to the gradient of the constraint $h$. Otherwise, you could improve $f$ without violating the constraint.

```
Contours of f(x,y):         Constraint h(x,y) = 0:
                              
    ○  ○  ○                       /
   ○  ○  ○  ○                    /  ← Constraint line
  ○  ○  ●  ○  ○                 / ●  ← Optimum (where gradient of f
   ○  ○  ○  ○                  /        is perpendicular to constraint)
    ○  ○  ○                   /
```

```python
import numpy as np
from scipy.optimize import minimize

# Example: Minimize f(x,y) = x^2 + y^2 subject to x + y = 1
# (Find the point closest to origin on the line x + y = 1)

# Method 1: Using Lagrange multipliers analytically
# L(x, y, λ) = x² + y² + λ(x + y - 1)
# ∂L/∂x = 2x + λ = 0  →  x = -λ/2
# ∂L/∂y = 2y + λ = 0  →  y = -λ/2
# ∂L/∂λ = x + y - 1 = 0  →  -λ/2 - λ/2 = 1  →  λ = -1
# Solution: x = 0.5, y = 0.5

# Method 2: Using scipy
def objective(xy):
    return xy[0]**2 + xy[1]**2

def constraint_eq(xy):
    return xy[0] + xy[1] - 1  # Must equal 0

result = minimize(
    objective,
    x0=[0, 0],  # Starting point
    method='SLSQP',  # Sequential Least Squares Programming
    constraints={'type': 'eq', 'fun': constraint_eq}
)

print(f"Optimal solution: x={result.x[0]:.4f}, y={result.x[1]:.4f}")
print(f"Minimum value: {result.fun:.4f}")
print(f"Constraint satisfied: x+y = {sum(result.x):.4f}")

# Output:
# Optimal solution: x=0.5000, y=0.5000
# Minimum value: 0.5000
# Constraint satisfied: x+y = 1.0000
```

### KKT Conditions (Inequality Constraints)

For problems with inequality constraints $g(x) \leq 0$, we use **Karush-Kuhn-Tucker (KKT)** conditions:

$$\nabla f(x^*) + \sum_j \mu_j \nabla g_j(x^*) = 0 \quad \text{(Stationarity)}$$
$$g_j(x^*) \leq 0 \quad \text{(Primal feasibility)}$$
$$\mu_j \geq 0 \quad \text{(Dual feasibility)}$$
$$\mu_j g_j(x^*) = 0 \quad \text{(Complementary slackness)}$$

**Complementary slackness** is the key insight: either the constraint is active ($g_j = 0$) or the multiplier is zero ($\mu_j = 0$). You can't have both be non-zero.

```python
from scipy.optimize import minimize

# Example: Minimize f(x,y) = (x-1)^2 + (y-2)^2
# Subject to: x + y <= 2 and x >= 0 and y >= 0
# (Find point closest to (1,2) within the triangle)

def objective(xy):
    return (xy[0] - 1)**2 + (xy[1] - 2)**2

constraints = [
    {'type': 'ineq', 'fun': lambda xy: 2 - xy[0] - xy[1]},  # x + y <= 2
    {'type': 'ineq', 'fun': lambda xy: xy[0]},               # x >= 0
    {'type': 'ineq', 'fun': lambda xy: xy[1]},               # y >= 0
]

result = minimize(objective, x0=[0.5, 0.5], method='SLSQP', constraints=constraints)
print(f"Solution: x={result.x[0]:.4f}, y={result.x[1]:.4f}")
print(f"Distance² from (1,2): {result.fun:.4f}")
# Active constraint: x + y = 2, so the point is on the boundary
```

### Connection to Regularization

L2 regularization (Ridge) IS constrained optimization in disguise:

$$\text{Minimize } \mathcal{L}(\theta) \quad \text{s.t. } \|\theta\|_2^2 \leq C$$

is equivalent to:

$$\text{Minimize } \mathcal{L}(\theta) + \lambda \|\theta\|_2^2$$

where $\lambda$ is the Lagrange multiplier corresponding to constraint strength $C$.

---

## 8. Second-Order Methods

### What They Are
While gradient descent uses only first derivatives (slope), second-order methods also use **second derivatives (curvature)** to make smarter steps.

### Why They Matter
- Converge much faster (fewer iterations) for well-behaved problems
- Newton's method converges **quadratically** vs linearly for GD
- Used in small-scale ML (logistic regression, small neural nets)
- Too expensive for large neural networks (Hessian is enormous)

### Newton's Method

$$\theta_{t+1} = \theta_t - H^{-1} \nabla \mathcal{L}(\theta_t)$$

Where $H$ is the **Hessian matrix** (matrix of second derivatives):

$$H_{ij} = \frac{\partial^2 \mathcal{L}}{\partial \theta_i \partial \theta_j}$$

**Intuition**: Instead of just knowing "go left," Newton's method knows "go left by exactly this much because I know the curvature."

```
Gradient Descent:                 Newton's Method:
Takes many small steps            Jumps close to minimum in one step
    \                                 \
     \                                 \
      \   step step step step          \--------→ ● minimum!
       \  ↓    ↓    ↓    ↓            (uses curvature info)
        \_●___●___●___●___●
```

### Cost Comparison

| Method | Per-step cost | Convergence | Memory |
|--------|:---:|:---:|:---:|
| GD | $O(n)$ | Linear | $O(n)$ |
| Newton | $O(n^3)$ | Quadratic | $O(n^2)$ |
| L-BFGS | $O(n \cdot m)$ | Super-linear | $O(n \cdot m)$ |

Where $n$ = number of parameters, $m$ = memory parameter (typically 10-20).

```python
import numpy as np
from scipy.optimize import minimize

# Newton's method from scratch for 1D
def newtons_method_1d(f, f_prime, f_double_prime, x0, tol=1e-8, max_iter=100):
    """Newton's method for 1D optimization."""
    x = x0
    history = [x]
    
    for i in range(max_iter):
        grad = f_prime(x)
        hessian = f_double_prime(x)
        
        if abs(hessian) < 1e-12:
            print("Warning: Hessian near zero, stopping.")
            break
        
        # Newton step: x_new = x - f'(x) / f''(x)
        x_new = x - grad / hessian
        history.append(x_new)
        
        if abs(x_new - x) < tol:
            print(f"Converged in {i+1} iterations")
            break
        x = x_new
    
    return x, history

# Example: f(x) = x^4 - 3x^2 + 2
f = lambda x: x**4 - 3*x**2 + 2
f_prime = lambda x: 4*x**3 - 6*x
f_double_prime = lambda x: 12*x**2 - 6

result, history = newtons_method_1d(f, f_prime, f_double_prime, x0=2.0)
print(f"Minimum at x = {result:.6f}, f(x) = {f(result):.6f}")
print(f"Path: {[f'{h:.4f}' for h in history]}")

# Compare with gradient descent
def gd_1d(f_prime, x0, lr=0.01, max_iter=1000, tol=1e-8):
    x = x0
    history = [x]
    for i in range(max_iter):
        x_new = x - lr * f_prime(x)
        history.append(x_new)
        if abs(x_new - x) < tol:
            print(f"GD converged in {i+1} iterations")
            break
        x = x_new
    return x, history

result_gd, history_gd = gd_1d(f_prime, x0=2.0, lr=0.01)
print(f"\nNewton: {len(history)} steps vs GD: {len(history_gd)} steps")
```

### L-BFGS (Limited-memory BFGS)

The practical compromise: approximates the Hessian inverse using only gradient history.

```python
from scipy.optimize import minimize

# L-BFGS is great for medium-sized problems
def rosenbrock(x):
    return sum(100.0*(x[1:]-x[:-1]**2.0)**2.0 + (1-x[:-1])**2.0)

def rosenbrock_grad(x):
    xm = x[1:-1]
    xm_m1 = x[:-2]
    xm_p1 = x[2:]
    der = np.zeros_like(x)
    der[1:-1] = 200*(xm-xm_m1**2) - 400*(xm_p1 - xm**2)*xm - 2*(1-xm)
    der[0] = -400*x[0]*(x[1]-x[0]**2) - 2*(1-x[0])
    der[-1] = 200*(x[-1]-x[-2]**2)
    return der

# Optimize 100-dimensional Rosenbrock
x0 = np.zeros(100)
result = minimize(rosenbrock, x0, method='L-BFGS-B', jac=rosenbrock_grad)
print(f"L-BFGS converged: {result.success}")
print(f"Iterations: {result.nit}")
print(f"Final loss: {result.fun:.2e}")
```

---

## 9. Convergence Theory

### What It Is
Mathematical guarantees about whether and how fast an optimizer will reach the minimum.

### Convergence Rates

| Rate | Definition | Speed | Example |
|------|-----------|-------|---------|
| Sub-linear | $\|x_t - x^*\| = O(1/t)$ | Slow | SGD on non-smooth problems |
| Linear | $\|x_t - x^*\| = O(\rho^t), \rho < 1$ | Medium | GD on strongly convex |
| Super-linear | Faster than any $\rho^t$ | Fast | L-BFGS |
| Quadratic | $\|x_{t+1} - x^*\| = O(\|x_t - x^*\|^2)$ | Very fast | Newton's method |

### Conditions for GD Convergence

For **convex** functions with **L-Lipschitz** gradients:

$$\|\nabla f(x) - \nabla f(y)\| \leq L \|x - y\|$$

GD converges with learning rate $\eta \leq \frac{1}{L}$:

$$f(x_t) - f(x^*) \leq \frac{L \|x_0 - x^*\|^2}{2t}$$

For **strongly convex** functions (condition number $\kappa = L/\mu$):

$$\|x_t - x^*\|^2 \leq \left(\frac{\kappa - 1}{\kappa + 1}\right)^{2t} \|x_0 - x^*\|^2$$

> 💡 **Key Insight**: The **condition number** $\kappa$ determines how hard a problem is. High $\kappa$ = elongated contours = slow convergence. This is why feature scaling matters!

```python
import numpy as np

# Demonstrate the effect of condition number on convergence
def quadratic_gd(A, b, x0, lr, n_steps):
    """GD on f(x) = 0.5 * x^T A x - b^T x"""
    x = x0.copy()
    losses = []
    x_star = np.linalg.solve(A, b)  # True minimum
    
    for _ in range(n_steps):
        grad = A @ x - b
        x = x - lr * grad
        loss = 0.5 * x @ A @ x - b @ x
        losses.append(loss)
    
    return losses, x

# Well-conditioned problem (κ ≈ 1)
A_good = np.array([[2.0, 0], [0, 2.0]])  # κ = 1
b = np.array([1.0, 1.0])

# Ill-conditioned problem (κ = 100)  
A_bad = np.array([[200.0, 0], [0, 2.0]])  # κ = 100

x0 = np.array([5.0, 5.0])

losses_good, _ = quadratic_gd(A_good, b, x0, lr=0.4, n_steps=50)
losses_bad, _ = quadratic_gd(A_bad, b, x0, lr=0.005, n_steps=50)

import matplotlib.pyplot as plt
plt.figure(figsize=(10, 5))
plt.semilogy(losses_good, label=f'Well-conditioned (κ=1)', linewidth=2)
plt.semilogy(losses_bad, label=f'Ill-conditioned (κ=100)', linewidth=2)
plt.xlabel('Iteration')
plt.ylabel('Loss (log scale)')
plt.title('Effect of Condition Number on Convergence Speed')
plt.legend()
plt.grid(True)
plt.show()
```

---

## 10. Optimization in Practice

### Practical Tips for Training Neural Networks

```python
# Complete practical example: Training with PyTorch optimizers
import torch
import torch.nn as nn
import torch.optim as optim

# Define a simple model
model = nn.Sequential(
    nn.Linear(10, 64),
    nn.ReLU(),
    nn.Linear(64, 32),
    nn.ReLU(),
    nn.Linear(32, 1)
)

# The modern recipe for optimization:
# 1. AdamW optimizer with weight decay
optimizer = optim.AdamW(model.parameters(), lr=3e-4, weight_decay=0.01)

# 2. Cosine annealing with warmup
total_steps = 10000
warmup_steps = 1000

def lr_lambda(step):
    if step < warmup_steps:
        return step / warmup_steps  # Linear warmup
    progress = (step - warmup_steps) / (total_steps - warmup_steps)
    return 0.5 * (1 + np.cos(np.pi * progress))  # Cosine decay

scheduler = optim.lr_scheduler.LambdaLR(optimizer, lr_lambda)

# 3. Gradient clipping (prevent exploding gradients)
max_grad_norm = 1.0

# Training loop
for step in range(total_steps):
    # ... forward pass, compute loss ...
    loss = torch.tensor(0.0, requires_grad=True)  # placeholder
    
    optimizer.zero_grad()
    loss.backward()
    
    # Clip gradients
    torch.nn.utils.clip_grad_norm_(model.parameters(), max_grad_norm)
    
    optimizer.step()
    scheduler.step()
```

### Debugging Optimization Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Loss not decreasing | LR too small or too large | Try 10x larger/smaller LR |
| Loss explodes (NaN) | LR too large / no grad clipping | Reduce LR, add grad clipping |
| Loss oscillates wildly | LR too large / batch too small | Reduce LR, increase batch size |
| Loss plateaus early | Stuck in local min / LR too small | Warmup, increase LR, try different init |
| Train loss ↓ but val loss ↑ | Overfitting | Add regularization, reduce model size |
| Very slow convergence | Ill-conditioning | Use adaptive optimizer (Adam), normalize data |

---

## Common Mistakes

1. **Using same LR for all problems** — Different problems need different learning rates. Always do a LR sweep (try 1e-5, 1e-4, 1e-3, 1e-2, 1e-1).

2. **Not using learning rate warmup** — Large models (Transformers) crash without warmup because early gradients are unreliable.

3. **Forgetting `optimizer.zero_grad()`** — In PyTorch, gradients accumulate by default. Forgetting to zero them gives wrong updates.

4. **Confusing L2 regularization with weight decay** — They're the same for SGD but different for Adam. Use AdamW for proper weight decay.

5. **Batch size too large** — Larger batches need proportionally larger LR (linear scaling rule). Very large batches can hurt generalization.

6. **Not monitoring gradient norms** — If gradients are tiny (vanishing) or huge (exploding), your optimizer can't help.

---

## Interview Questions

**Q1: Explain the difference between SGD, Mini-batch GD, and Batch GD.**
> SGD uses one sample per update (noisy but fast), Batch GD uses all samples (stable but slow), Mini-batch uses a subset (balanced). Mini-batch is standard in practice because it's GPU-efficient and has beneficial noise.

**Q2: Why does Adam sometimes generalize worse than SGD?**
> Adam's adaptive learning rates can converge to sharp minima (which generalize poorly). SGD's noise helps it find flat minima. AdamW fixes some of this by decoupling weight decay. In practice, SGD + momentum + cosine annealing often gives better test accuracy for CNNs.

**Q3: What is the role of the learning rate warmup?**
> Early in training, gradients are unreliable (random weights → random gradients). Large LR + bad gradients = instability. Warmup starts with small LR and increases it, allowing the model to stabilize before taking large steps.

**Q4: Explain KKT conditions and their relevance to SVMs.**
> KKT conditions are necessary for optimality in constrained problems. In SVMs, they tell us that only data points ON the margin boundary (support vectors) have non-zero multipliers. All other points have zero multipliers and don't affect the solution.

**Q5: What is the condition number and why does it matter?**
> The condition number κ = L/μ (ratio of largest to smallest curvature). High κ means elongated loss landscape → slow convergence. Feature scaling reduces κ. Preconditioning (adaptive methods like Adam) effectively reduces the condition number.

---

## Quick Reference

| Concept | Formula/Rule | When to Use |
|---------|-------------|-------------|
| GD Update | $\theta \leftarrow \theta - \eta \nabla \mathcal{L}$ | Always (foundation) |
| Momentum | $v = \gamma v + \eta \nabla \mathcal{L}$; $\theta \leftarrow \theta - v$ | When GD oscillates |
| Adam | Combines momentum + adaptive LR + bias correction | Default for most problems |
| AdamW | Adam + decoupled weight decay | Transformers, modern DL |
| Cosine Schedule | $\eta_t = \eta_0 \cdot 0.5(1 + \cos(\pi t/T))$ | Standard scheduling |
| Gradient Clipping | $g \leftarrow g \cdot \min(1, c/\|g\|)$ | Prevent exploding gradients |
| Convexity Check | $f''(x) \geq 0$ or $H \succeq 0$ | Verify problem is "easy" |
| Lagrangian | $\mathcal{L} = f(x) + \lambda h(x)$ | Equality constraints |
| Condition Number | $\kappa = \lambda_{max} / \lambda_{min}$ | Diagnose slow convergence |

---

*Next: [Chapter 05 - Information Theory](05-Information-Theory.md)*
