# Chapter 06: The Math Behind Key ML Algorithms

## Overview
This chapter pulls back the curtain on the most important ML algorithms and derives their math from first principles. Instead of treating algorithms as black boxes, you'll understand **why** each formula exists, **how** the math connects to intuition, and **what** happens under the hood during training. We cover PCA, SVM, Neural Networks, Backpropagation, Logistic Regression, and Naive Bayes — all with full derivations.

---

## Table of Contents
1. [PCA — Principal Component Analysis](#1-pca--principal-component-analysis)
2. [SVM — Support Vector Machines](#2-svm--support-vector-machines)
3. [Logistic Regression — The Math](#3-logistic-regression--the-math)
4. [Naive Bayes — Probabilistic Classification](#4-naive-bayes--probabilistic-classification)
5. [Neural Networks — Forward Pass Math](#5-neural-networks--forward-pass-math)
6. [Backpropagation — Full Derivation](#6-backpropagation--full-derivation)
7. [Batch Normalization — Why It Works](#7-batch-normalization--why-it-works)
8. [Attention Mechanism — The Math](#8-attention-mechanism--the-math)

---

## 1. PCA — Principal Component Analysis

### What It Is
Imagine you have a spreadsheet with 100 columns (features). PCA finds the "most important directions" in your data and lets you keep only the top few, throwing away the rest with minimal information loss. It's like photographing a 3D object — you choose the angle that captures the most shape.

### Why It Matters
- **Dimensionality reduction** — reduce 1000 features to 50 while keeping 95% of the information
- **Visualization** — project high-dimensional data to 2D/3D for plotting
- **Noise removal** — small components are often noise
- **Preprocessing** — decorrelates features, speeds up training
- **Compression** — images, signals, genomics data

### How It Works — Step by Step

**Goal**: Find the direction (unit vector $u$) along which the data has **maximum variance**.

#### Step 1: Center the Data

$$\bar{x} = \frac{1}{N} \sum_{i=1}^{N} x_i, \quad \tilde{x}_i = x_i - \bar{x}$$

#### Step 2: Compute the Covariance Matrix

$$C = \frac{1}{N-1} \tilde{X}^T \tilde{X}$$

Where $\tilde{X}$ is the centered data matrix ($N \times d$).

The covariance matrix $C$ is $d \times d$, symmetric, and positive semi-definite.

#### Step 3: Maximize Variance Along Direction $u$

The variance of the data projected onto unit vector $u$:

$$\text{Var} = u^T C u$$

We want to maximize this subject to $\|u\| = 1$. Using Lagrange multipliers:

$$\mathcal{L}(u, \lambda) = u^T C u - \lambda(u^T u - 1)$$

Setting the gradient to zero:

$$\nabla_u \mathcal{L} = 2Cu - 2\lambda u = 0$$
$$Cu = \lambda u$$

This is an **eigenvalue equation**! The direction of maximum variance is the **eigenvector** with the **largest eigenvalue**.

#### Step 4: The Variance Along Each PC

$$\text{Variance along } u_k = \lambda_k \quad \text{(the eigenvalue itself)}$$

$$\text{Proportion of variance explained by PC } k = \frac{\lambda_k}{\sum_{j=1}^{d} \lambda_j}$$

### Visual Intuition

```
Original 2D Data:                  After PCA:
                                   PC1 (most variance)
    *  *                              ↗
   * ** *  *                     *  ** *  *
  *  * ** *                       * ** *
    * *  *                         * *
   *  *                           *
                                 ↗ PC2 (least variance)

PCA finds the "natural axes" of the data — the directions
along which the data spreads out the most.
```

### Connection to SVD

PCA via eigendecomposition of $C$ is equivalent to SVD of the centered data matrix:

$$\tilde{X} = U \Sigma V^T$$

- **Columns of $V$** = principal components (eigenvectors of $C$)
- **Singular values** $\sigma_k = \sqrt{(N-1)\lambda_k}$
- SVD is numerically more stable and is what sklearn uses internally

```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn.decomposition import PCA

# ============================================
# PCA from scratch — full implementation
# ============================================

class PCAFromScratch:
    """PCA implemented from first principles using eigendecomposition."""
    
    def __init__(self, n_components):
        self.n_components = n_components
        self.components = None       # Principal component directions
        self.eigenvalues = None      # Variance along each PC
        self.mean = None             # Data mean (for centering)
        self.explained_variance_ratio = None
    
    def fit(self, X):
        """
        Compute principal components from data X (N x d).
        
        Math:
        1. Center X
        2. Compute covariance matrix C = X^T X / (N-1)
        3. Eigendecompose C
        4. Sort eigenvectors by eigenvalue (descending)
        5. Keep top k eigenvectors
        """
        N, d = X.shape
        
        # Step 1: Center the data (subtract mean)
        self.mean = np.mean(X, axis=0)
        X_centered = X - self.mean
        
        # Step 2: Covariance matrix (d × d)
        cov_matrix = (X_centered.T @ X_centered) / (N - 1)
        
        # Step 3: Eigendecomposition
        eigenvalues, eigenvectors = np.linalg.eigh(cov_matrix)
        
        # Step 4: Sort by eigenvalue (largest first)
        # np.linalg.eigh returns in ascending order, so reverse
        idx = np.argsort(eigenvalues)[::-1]
        eigenvalues = eigenvalues[idx]
        eigenvectors = eigenvectors[:, idx]
        
        # Step 5: Keep top k components
        self.eigenvalues = eigenvalues[:self.n_components]
        self.components = eigenvectors[:, :self.n_components].T  # (k × d)
        
        # Explained variance ratio
        total_var = np.sum(eigenvalues)
        self.explained_variance_ratio = self.eigenvalues / total_var
        
        return self
    
    def transform(self, X):
        """Project data onto principal components: Z = (X - mean) @ V"""
        X_centered = X - self.mean
        return X_centered @ self.components.T  # (N × k)
    
    def inverse_transform(self, Z):
        """Reconstruct data from reduced representation: X ≈ Z @ V^T + mean"""
        return Z @ self.components + self.mean


# ============================================
# Example: PCA on synthetic 2D data
# ============================================
np.random.seed(42)

# Create correlated 2D data
angle = np.pi / 4  # 45-degree correlation
stretch = np.array([[3, 0], [0, 0.5]])  # Stretch along one axis
rotation = np.array([[np.cos(angle), -np.sin(angle)], 
                      [np.sin(angle), np.cos(angle)]])

X = np.random.randn(200, 2) @ stretch @ rotation.T + np.array([2, 3])

# Fit PCA
pca = PCAFromScratch(n_components=2)
pca.fit(X)

print("=== PCA Results ===")
print(f"Eigenvalues: {pca.eigenvalues}")
print(f"Variance explained: {pca.explained_variance_ratio}")
print(f"PC1 direction: {pca.components[0]}")
print(f"PC2 direction: {pca.components[1]}")

# Visualize
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# Original data with PC directions
axes[0].scatter(X[:, 0], X[:, 1], alpha=0.5, s=10)
mean = pca.mean
for i, (comp, var) in enumerate(zip(pca.components, pca.eigenvalues)):
    axes[0].annotate('', xy=mean + comp * var * 2, xytext=mean,
                     arrowprops=dict(arrowstyle='->', color='red', lw=2))
    axes[0].text(mean[0] + comp[0]*var*2.2, mean[1] + comp[1]*var*2.2, 
                 f'PC{i+1}', fontsize=12, color='red')
axes[0].set_title('Original Data with Principal Components')
axes[0].set_aspect('equal')
axes[0].grid(True)

# Projected data (1D projection onto PC1)
Z = pca.transform(X)
axes[1].scatter(Z[:, 0], Z[:, 1], alpha=0.5, s=10)
axes[1].set_xlabel('PC1 (most variance)')
axes[1].set_ylabel('PC2 (least variance)')
axes[1].set_title('Data in PC Space (Decorrelated!)')
axes[1].grid(True)

plt.tight_layout()
plt.show()

# ============================================
# Scree plot: How many components to keep?
# ============================================
from sklearn.datasets import load_digits

digits = load_digits()
X_digits = digits.data  # 64 features (8x8 pixel images)

pca_full = PCA(n_components=64)
pca_full.fit(X_digits)

cumulative_var = np.cumsum(pca_full.explained_variance_ratio_)

plt.figure(figsize=(10, 5))
plt.bar(range(1, 65), pca_full.explained_variance_ratio_, alpha=0.6, label='Individual')
plt.plot(range(1, 65), cumulative_var, 'ro-', markersize=3, label='Cumulative')
plt.axhline(y=0.95, color='k', linestyle='--', label='95% threshold')
plt.xlabel('Principal Component')
plt.ylabel('Variance Explained')
plt.title('Scree Plot — How Many Components to Keep?')
plt.legend()
plt.grid(True)
plt.show()

n_95 = np.argmax(cumulative_var >= 0.95) + 1
print(f"\nComponents needed for 95% variance: {n_95} out of 64")
print(f"Compression ratio: {64/n_95:.1f}x")
```

### Key Insights About PCA

| Fact | Why It Matters |
|------|---------------|
| PCA finds orthogonal directions | Components are uncorrelated — no redundancy |
| PCA is linear | Can't capture curved/nonlinear structure (use t-SNE, UMAP) |
| PCA maximizes variance | Assumes variance = information (not always true) |
| PCA = SVD | sklearn uses SVD (more numerically stable) |
| Scaling matters! | Always standardize features before PCA |

> ⚠️ **Critical**: PCA is sensitive to feature scales. A feature measured in millions will dominate one measured in decimals. **Always standardize** (z-score) before PCA unless all features share the same units.

---

## 2. SVM — Support Vector Machines

### What It Is
SVM finds the **widest possible highway** (maximum margin) that separates two classes. The data points closest to the highway are "support vectors" — they're the only ones that matter for defining the boundary.

### Why It Matters
- One of the most mathematically elegant ML algorithms
- Works extremely well in high dimensions (even when features > samples)
- The kernel trick lets SVMs handle nonlinear boundaries
- Foundation for understanding optimization with constraints in ML

### The Maximum Margin Principle

For linearly separable data, we want a hyperplane $w^T x + b = 0$ that:

$$\text{Maximize margin} = \frac{2}{\|w\|}$$

subject to:

$$y_i(w^T x_i + b) \geq 1 \quad \forall i$$

Where $y_i \in \{-1, +1\}$ is the class label.

```
           Margin = 2/||w||
      ←─────────────────────→
      
  +   │    +    ║          ║    -    │   -
      │  +      ║          ║      -  │
  +   │    +  ● ║  Highway ║ ●  -    │   -
      │  +      ║          ║    -    │
  +   │         ║          ║         │   -
      
              ● = Support Vectors (on the margin boundary)
              Only these points affect the decision boundary!
```

### Derivation: From Primal to Dual

**Primal problem** (what we want to solve):

$$\min_{w, b} \frac{1}{2} \|w\|^2 \quad \text{s.t. } y_i(w^T x_i + b) \geq 1$$

**Lagrangian** (introduce multipliers $\alpha_i \geq 0$):

$$\mathcal{L}(w, b, \alpha) = \frac{1}{2}\|w\|^2 - \sum_{i=1}^{N} \alpha_i [y_i(w^T x_i + b) - 1]$$

**Setting gradients to zero**:

$$\frac{\partial \mathcal{L}}{\partial w} = 0 \implies w = \sum_{i=1}^{N} \alpha_i y_i x_i$$

$$\frac{\partial \mathcal{L}}{\partial b} = 0 \implies \sum_{i=1}^{N} \alpha_i y_i = 0$$

**Dual problem** (substitute back):

$$\max_{\alpha} \sum_{i=1}^{N} \alpha_i - \frac{1}{2} \sum_{i,j} \alpha_i \alpha_j y_i y_j (x_i^T x_j)$$

subject to $\alpha_i \geq 0$ and $\sum \alpha_i y_i = 0$.

### The Kernel Trick

The dual form only uses dot products $x_i^T x_j$. Replace this with a **kernel function** $K(x_i, x_j)$ to handle nonlinear boundaries without explicitly transforming the data:

$$\max_{\alpha} \sum_{i=1}^{N} \alpha_i - \frac{1}{2} \sum_{i,j} \alpha_i \alpha_j y_i y_j K(x_i, x_j)$$

| Kernel | Formula $K(x, z)$ | Use Case |
|--------|-------------------|----------|
| Linear | $x^T z$ | Linearly separable data |
| Polynomial | $(x^T z + c)^d$ | Polynomial boundaries |
| RBF (Gaussian) | $\exp(-\gamma \|x - z\|^2)$ | Most general, default choice |
| Sigmoid | $\tanh(\kappa x^T z + c)$ | Similar to neural network |

**Why is RBF so powerful?** It implicitly maps data to **infinite-dimensional** space. Every data point becomes a "bump" in feature space, and the SVM finds the optimal combination of bumps.

### Soft-Margin SVM (Real-World Version)

Real data isn't perfectly separable. Add **slack variables** $\xi_i$:

$$\min_{w, b, \xi} \frac{1}{2}\|w\|^2 + C \sum_{i=1}^{N} \xi_i$$

$$\text{s.t. } y_i(w^T x_i + b) \geq 1 - \xi_i, \quad \xi_i \geq 0$$

Where $C$ controls the trade-off:
- Large $C$: Less tolerance for misclassification (might overfit)
- Small $C$: More tolerance (smoother boundary, might underfit)

```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn.svm import SVC
from sklearn.datasets import make_moons, make_circles

# ============================================
# SVM from scratch — Linear case
# ============================================

class LinearSVMScratch:
    """
    Linear SVM using sub-gradient descent on the hinge loss.
    
    Objective: min (1/2)||w||^2 + C * sum(max(0, 1 - y_i(w·x_i + b)))
    """
    
    def __init__(self, C=1.0, lr=0.001, n_iters=1000):
        self.C = C           # Regularization parameter
        self.lr = lr         # Learning rate
        self.n_iters = n_iters
        self.w = None
        self.b = None
    
    def fit(self, X, y):
        """
        Train SVM using sub-gradient descent.
        y must be in {-1, +1}.
        """
        N, d = X.shape
        self.w = np.zeros(d)
        self.b = 0.0
        
        for _ in range(self.n_iters):
            for i in range(N):
                # Check if point violates margin
                margin = y[i] * (X[i] @ self.w + self.b)
                
                if margin >= 1:
                    # Correctly classified with margin — only regularize
                    self.w -= self.lr * self.w  # Gradient of (1/2)||w||^2
                else:
                    # Margin violation — hinge loss gradient
                    self.w -= self.lr * (self.w - self.C * y[i] * X[i])
                    self.b -= self.lr * (-self.C * y[i])
    
    def predict(self, X):
        return np.sign(X @ self.w + self.b)


# ============================================
# Visualize kernel trick
# ============================================

fig, axes = plt.subplots(2, 3, figsize=(15, 10))

# Dataset 1: Linearly separable
X_linear = np.vstack([
    np.random.randn(50, 2) + [2, 2],
    np.random.randn(50, 2) + [-2, -2]
])
y_linear = np.array([1]*50 + [-1]*50)

# Dataset 2: Moons (nonlinear)
X_moons, y_moons = make_moons(100, noise=0.15, random_state=42)
y_moons = 2*y_moons - 1  # Convert to {-1, +1}

# Dataset 3: Circles (nonlinear)
X_circles, y_circles = make_circles(100, noise=0.1, factor=0.3, random_state=42)
y_circles = 2*y_circles - 1

datasets = [
    (X_linear, y_linear, "Linearly Separable"),
    (X_moons, y_moons, "Moons"),
    (X_circles, y_circles, "Circles")
]

kernels = ['linear', 'rbf']

for col, (X, y, name) in enumerate(datasets):
    for row, kernel in enumerate(kernels):
        ax = axes[row, col]
        
        # Fit SVM
        svm = SVC(kernel=kernel, C=1.0, gamma='auto')
        svm.fit(X, y)
        
        # Decision boundary
        xx, yy = np.meshgrid(
            np.linspace(X[:, 0].min()-1, X[:, 0].max()+1, 200),
            np.linspace(X[:, 1].min()-1, X[:, 1].max()+1, 200)
        )
        Z = svm.predict(np.c_[xx.ravel(), yy.ravel()]).reshape(xx.shape)
        
        ax.contourf(xx, yy, Z, alpha=0.3, cmap='RdBu')
        ax.scatter(X[:, 0], X[:, 1], c=y, cmap='RdBu', edgecolors='k', s=30)
        
        # Highlight support vectors
        sv = svm.support_vectors_
        ax.scatter(sv[:, 0], sv[:, 1], s=100, facecolors='none', 
                   edgecolors='yellow', linewidth=2)
        
        n_sv = len(svm.support_vectors_)
        ax.set_title(f'{name}\nKernel={kernel}, SVs={n_sv}')
        ax.grid(True, alpha=0.3)

plt.tight_layout()
plt.suptitle('SVM: Linear vs RBF Kernel\n(Yellow circles = Support Vectors)', 
             y=1.02, fontsize=14)
plt.show()
```

### KKT Conditions in SVM — What They Mean

$$\alpha_i [y_i(w^T x_i + b) - 1] = 0 \quad \text{(Complementary slackness)}$$

This means:
- If $\alpha_i > 0$: point is ON the margin → it's a **support vector**
- If $\alpha_i = 0$: point is outside the margin → it doesn't matter at all
- Only support vectors define the decision boundary!

> 💡 **Key Insight**: You can remove all non-support-vector points from the training set and get the exact same SVM. This is why SVMs are memory-efficient at prediction time.

---

## 3. Logistic Regression — The Math

### What It Is
Logistic regression models the **probability** that an input belongs to a class. Despite the name, it's a classification algorithm, not regression. It's the simplest neural network — a single neuron with a sigmoid activation.

### Why It Matters
- Foundation of neural networks (each neuron is a logistic regression)
- Interpretable — coefficients have direct meaning (log-odds)
- Maximum Likelihood Estimation (MLE) connects stats to ML

### From Linear to Probabilistic

**Problem**: Linear function $z = w^T x + b$ gives values in $(-\infty, +\infty)$. We need probabilities in $[0, 1]$.

**Solution**: Pass through the **sigmoid** (logistic) function:

$$\sigma(z) = \frac{1}{1 + e^{-z}} = \frac{e^z}{e^z + 1}$$

$$P(y=1|x) = \sigma(w^T x + b) = \frac{1}{1 + e^{-(w^T x + b)}}$$

### Properties of the Sigmoid

$$\sigma'(z) = \sigma(z)(1 - \sigma(z)) \quad \text{(elegant derivative!)}$$

```
σ(z):
1.0 |                    ___________
    |                 __/
    |               _/
0.5 |............./ ..............  ← Decision boundary at z=0
    |           _/
    |         _/
0.0 |________/
    -6   -4   -2   0   2   4   6
                   z = w·x + b
```

### MLE Derivation

**Likelihood** (probability of observing the data given parameters):

$$L(w, b) = \prod_{i=1}^{N} \hat{y}_i^{y_i} (1 - \hat{y}_i)^{1 - y_i}$$

Where $\hat{y}_i = \sigma(w^T x_i + b)$.

**Log-likelihood** (easier to work with):

$$\ell(w, b) = \sum_{i=1}^{N} [y_i \log \hat{y}_i + (1 - y_i) \log(1 - \hat{y}_i)]$$

**Negative log-likelihood = Binary Cross-Entropy Loss**:

$$\mathcal{L} = -\frac{1}{N} \sum_{i=1}^{N} [y_i \log \hat{y}_i + (1 - y_i) \log(1 - \hat{y}_i)]$$

### Gradient Derivation

$$\frac{\partial \mathcal{L}}{\partial w_j} = \frac{1}{N} \sum_{i=1}^{N} (\hat{y}_i - y_i) x_{ij}$$

In matrix form:

$$\nabla_w \mathcal{L} = \frac{1}{N} X^T (\hat{y} - y)$$

> 💡 **Beautiful result**: The gradient has the exact same form as linear regression! The sigmoid is "absorbed" into $\hat{y}$.

```python
import numpy as np
from sklearn.datasets import make_classification

class LogisticRegressionScratch:
    """
    Logistic Regression from scratch with full MLE derivation.
    Trained using gradient descent on binary cross-entropy loss.
    """
    
    def __init__(self, lr=0.01, n_iters=1000, reg_lambda=0.0):
        self.lr = lr
        self.n_iters = n_iters
        self.reg_lambda = reg_lambda  # L2 regularization strength
        self.w = None
        self.b = None
        self.loss_history = []
    
    @staticmethod
    def sigmoid(z):
        """Numerically stable sigmoid."""
        return np.where(
            z >= 0,
            1 / (1 + np.exp(-z)),
            np.exp(z) / (1 + np.exp(z))
        )
    
    def fit(self, X, y):
        N, d = X.shape
        self.w = np.zeros(d)
        self.b = 0.0
        
        for iteration in range(self.n_iters):
            # Forward pass: compute predictions
            z = X @ self.w + self.b        # Linear combination (N,)
            y_hat = self.sigmoid(z)         # Probabilities (N,)
            
            # Compute loss: Binary Cross-Entropy + L2 regularization
            # Clip to avoid log(0)
            y_hat_clipped = np.clip(y_hat, 1e-15, 1 - 1e-15)
            bce = -np.mean(y * np.log(y_hat_clipped) + (1-y) * np.log(1-y_hat_clipped))
            reg = (self.reg_lambda / (2*N)) * np.sum(self.w**2)
            loss = bce + reg
            self.loss_history.append(loss)
            
            # Compute gradients (from MLE derivation)
            error = y_hat - y                          # (N,)
            dw = (1/N) * (X.T @ error) + (self.reg_lambda/N) * self.w
            db = (1/N) * np.sum(error)
            
            # Update parameters
            self.w -= self.lr * dw
            self.b -= self.lr * db
        
        return self
    
    def predict_proba(self, X):
        """Return probability of class 1."""
        return self.sigmoid(X @ self.w + self.b)
    
    def predict(self, X, threshold=0.5):
        """Return binary predictions."""
        return (self.predict_proba(X) >= threshold).astype(int)


# Train and evaluate
np.random.seed(42)
X, y = make_classification(n_samples=500, n_features=2, n_redundant=0,
                           n_clusters_per_class=1, random_state=42)

model = LogisticRegressionScratch(lr=0.1, n_iters=500, reg_lambda=0.01)
model.fit(X, y)

# Accuracy
y_pred = model.predict(X)
accuracy = np.mean(y_pred == y)
print(f"Accuracy: {accuracy:.4f}")
print(f"Weights: {model.w}")
print(f"Bias: {model.b:.4f}")
print(f"Final loss: {model.loss_history[-1]:.4f}")

# Interpretation: odds ratio
print(f"\n=== Coefficient Interpretation ===")
for i, w in enumerate(model.w):
    print(f"Feature {i}: coefficient = {w:.3f}, "
          f"odds ratio = {np.exp(w):.3f}")
    print(f"  → 1-unit increase in feature {i} multiplies odds by {np.exp(w):.3f}")
```

### The Log-Odds (Logit) Interpretation

$$\log \frac{P(y=1|x)}{P(y=0|x)} = w^T x + b$$

Each coefficient $w_j$ represents the change in **log-odds** for a 1-unit increase in feature $x_j$:

| $w_j$ | Odds Ratio $e^{w_j}$ | Interpretation |
|--------|---------------------|----------------|
| +0.5 | 1.65 | Feature increases odds by 65% |
| 0 | 1.0 | No effect |
| -1.0 | 0.37 | Feature reduces odds by 63% |

---

## 4. Naive Bayes — Probabilistic Classification

### What It Is
Naive Bayes applies **Bayes' theorem** with the "naive" assumption that features are **conditionally independent** given the class. Despite this unrealistic assumption, it works surprisingly well in practice.

### Why It Matters
- Extremely fast (training = counting frequencies)
- Works well with small datasets and high dimensions (text classification)
- No gradient descent needed — closed-form solution
- Baseline for NLP (spam filtering, sentiment analysis)

### The Math: Bayes' Theorem

$$P(y|x_1, ..., x_d) = \frac{P(x_1, ..., x_d | y) \cdot P(y)}{P(x_1, ..., x_d)}$$

With the **naive independence assumption**:

$$P(x_1, ..., x_d | y) = \prod_{j=1}^{d} P(x_j | y)$$

**Decision rule** (classify to the most likely class):

$$\hat{y} = \arg\max_y P(y) \prod_{j=1}^{d} P(x_j | y)$$

In log space (to avoid numerical underflow from multiplying small numbers):

$$\hat{y} = \arg\max_y \left[\log P(y) + \sum_{j=1}^{d} \log P(x_j | y)\right]$$

### Types of Naive Bayes

| Type | $P(x_j | y)$ | Use Case |
|------|--------------|----------|
| Gaussian NB | $\mathcal{N}(\mu_{jy}, \sigma_{jy}^2)$ | Continuous features |
| Multinomial NB | Multinomial counts | Text (word counts, TF-IDF) |
| Bernoulli NB | Bernoulli (0/1) | Binary features |

```python
import numpy as np
from collections import defaultdict

class GaussianNBScratch:
    """
    Gaussian Naive Bayes from scratch.
    
    Assumes each feature follows a Gaussian distribution per class:
    P(x_j | y=c) = N(mu_{jc}, sigma_{jc}^2)
    """
    
    def __init__(self):
        self.classes = None
        self.priors = {}       # P(y=c)
        self.means = {}        # mu_{jc} for each class c
        self.variances = {}    # sigma_{jc}^2 for each class c
    
    def fit(self, X, y):
        self.classes = np.unique(y)
        N = len(y)
        
        for c in self.classes:
            X_c = X[y == c]  # Data points belonging to class c
            
            # Prior: P(y=c) = count(y=c) / N
            self.priors[c] = len(X_c) / N
            
            # MLE estimates: mean and variance per feature
            self.means[c] = X_c.mean(axis=0)
            self.variances[c] = X_c.var(axis=0) + 1e-9  # Add epsilon for stability
        
        return self
    
    def _log_likelihood(self, x, c):
        """
        Log P(x | y=c) = sum_j log N(x_j; mu_{jc}, sigma_{jc}^2)
        
        Gaussian log-pdf: -0.5 * [log(2π) + log(σ²) + (x-μ)²/σ²]
        """
        mean = self.means[c]
        var = self.variances[c]
        
        log_pdf = -0.5 * (np.log(2 * np.pi * var) + (x - mean)**2 / var)
        return np.sum(log_pdf)  # Sum over features (independence!)
    
    def predict(self, X):
        predictions = []
        for x in X:
            # Compute log P(y=c) + log P(x|y=c) for each class
            posteriors = {}
            for c in self.classes:
                log_prior = np.log(self.priors[c])
                log_likelihood = self._log_likelihood(x, c)
                posteriors[c] = log_prior + log_likelihood
            
            # Pick class with highest posterior
            predictions.append(max(posteriors, key=posteriors.get))
        
        return np.array(predictions)


# Test on Iris dataset
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

iris = load_iris()
X_train, X_test, y_train, y_test = train_test_split(
    iris.data, iris.target, test_size=0.3, random_state=42
)

nb = GaussianNBScratch()
nb.fit(X_train, y_train)
y_pred = nb.predict(X_test)

accuracy = np.mean(y_pred == y_test)
print(f"Gaussian Naive Bayes accuracy: {accuracy:.4f}")

# Show learned parameters
for c in nb.classes:
    print(f"\nClass {c} (prior = {nb.priors[c]:.3f}):")
    for j, name in enumerate(iris.feature_names):
        print(f"  {name}: μ={nb.means[c][j]:.2f}, σ²={nb.variances[c][j]:.2f}")
```

> 💡 **Why does NB work despite the "naive" assumption?** For classification, we only need the **ranking** of $P(y|x)$ to be correct, not the exact values. Even if the independence assumption is wrong, the highest posterior class is often still correct. NB is optimal when features truly are conditionally independent (rare, but the approximation is robust).

---

## 5. Neural Networks — Forward Pass Math

### What It Is
A neural network is a chain of matrix multiplications interleaved with nonlinear functions (activations). Each layer transforms its input into a representation that's progressively more useful for the task.

### Why It Matters
- Understanding the math lets you debug, design, and reason about architectures
- Shape mismatches, gradient issues, and initialization choices all require mathematical understanding

### Single Layer (Dense/Fully Connected)

$$z^{[l]} = W^{[l]} a^{[l-1]} + b^{[l]}$$
$$a^{[l]} = g(z^{[l]})$$

Where:
- $a^{[l-1]}$ = input activations (or the raw input $x$ for $l=1$)
- $W^{[l]}$ = weight matrix of shape $(n^{[l]}, n^{[l-1]})$
- $b^{[l]}$ = bias vector of shape $(n^{[l]}, 1)$
- $z^{[l]}$ = pre-activation (linear combination)
- $g(\cdot)$ = activation function (ReLU, sigmoid, tanh, etc.)
- $a^{[l]}$ = output activations

### Shape Tracking (Most Common Source of Bugs!)

```
Input:     x        (d, 1) = (784, 1)  for MNIST
           ↓
Layer 1:   W1·x + b1    W1: (128, 784), b1: (128, 1) → z1: (128, 1)
           ↓ ReLU
           a1           (128, 1)
           ↓
Layer 2:   W2·a1 + b2   W2: (64, 128),  b2: (64, 1)  → z2: (64, 1)
           ↓ ReLU  
           a2           (64, 1)
           ↓
Output:    W3·a2 + b3   W3: (10, 64),   b3: (10, 1)  → z3: (10, 1)
           ↓ Softmax
           ŷ            (10, 1)  ← probability for each digit

Rule: W^[l] has shape (neurons_in_this_layer, neurons_in_previous_layer)
```

### Common Activation Functions and Their Math

| Activation | Formula | Derivative | When to Use |
|-----------|---------|-----------|-------------|
| Sigmoid | $\frac{1}{1+e^{-z}}$ | $\sigma(z)(1-\sigma(z))$ | Binary output layer |
| Tanh | $\frac{e^z - e^{-z}}{e^z + e^{-z}}$ | $1 - \tanh^2(z)$ | Hidden layers (older networks) |
| ReLU | $\max(0, z)$ | $\begin{cases}1 & z>0\\0 & z\leq 0\end{cases}$ | Default for hidden layers |
| Leaky ReLU | $\max(0.01z, z)$ | $\begin{cases}1 & z>0\\0.01 & z\leq 0\end{cases}$ | When dying ReLU is a problem |
| GELU | $z \cdot \Phi(z)$ | complex | Transformers (BERT, GPT) |
| Softmax | $\frac{e^{z_j}}{\sum_k e^{z_k}}$ | $s_j(\delta_{jk} - s_k)$ | Multi-class output layer |

### Softmax — The Multi-Class Output

$$\text{softmax}(z_j) = \frac{e^{z_j}}{\sum_{k=1}^{C} e^{z_k}}$$

**Properties**:
- Outputs sum to 1 (valid probability distribution)
- Preserves ordering (largest logit → highest probability)
- Numerically: subtract max before computing (prevents overflow)

```python
import numpy as np

def softmax(z):
    """Numerically stable softmax."""
    z_shifted = z - np.max(z)  # Subtract max for numerical stability
    exp_z = np.exp(z_shifted)
    return exp_z / np.sum(exp_z)


class NeuralNetworkScratch:
    """
    A simple feedforward neural network from scratch.
    Supports arbitrary layer sizes with ReLU hidden activations.
    """
    
    def __init__(self, layer_sizes):
        """
        Args:
            layer_sizes: list like [784, 128, 64, 10]
                         input → hidden → hidden → output
        """
        self.n_layers = len(layer_sizes) - 1
        self.weights = []
        self.biases = []
        
        # Xavier/Glorot initialization
        for i in range(self.n_layers):
            n_in = layer_sizes[i]
            n_out = layer_sizes[i + 1]
            # Xavier: std = sqrt(2 / (n_in + n_out))
            std = np.sqrt(2.0 / (n_in + n_out))
            W = np.random.randn(n_out, n_in) * std
            b = np.zeros((n_out, 1))
            self.weights.append(W)
            self.biases.append(b)
    
    @staticmethod
    def relu(z):
        return np.maximum(0, z)
    
    @staticmethod
    def relu_derivative(z):
        return (z > 0).astype(float)
    
    @staticmethod
    def softmax(z):
        z_shifted = z - np.max(z, axis=0, keepdims=True)
        exp_z = np.exp(z_shifted)
        return exp_z / np.sum(exp_z, axis=0, keepdims=True)
    
    def forward(self, X):
        """
        Forward pass through the network.
        
        Args:
            X: Input data, shape (d, N) — each column is a sample
        Returns:
            Output probabilities, shape (C, N)
        """
        self.activations = [X]  # Store for backprop
        self.pre_activations = []
        
        a = X
        for l in range(self.n_layers):
            z = self.weights[l] @ a + self.biases[l]  # Linear
            self.pre_activations.append(z)
            
            if l < self.n_layers - 1:
                a = self.relu(z)       # Hidden: ReLU
            else:
                a = self.softmax(z)    # Output: Softmax
            
            self.activations.append(a)
        
        return a
    
    def compute_loss(self, y_pred, y_true):
        """Cross-entropy loss. y_true is one-hot (C, N)."""
        N = y_true.shape[1]
        y_pred_clipped = np.clip(y_pred, 1e-15, 1 - 1e-15)
        loss = -np.sum(y_true * np.log(y_pred_clipped)) / N
        return loss


# Demo: Forward pass
np.random.seed(42)
net = NeuralNetworkScratch([784, 128, 64, 10])

# Fake batch of 5 MNIST images
X = np.random.randn(784, 5)
output = net.forward(X)

print("=== Forward Pass ===")
print(f"Input shape: {X.shape}")
for l in range(net.n_layers):
    print(f"Layer {l+1}: W{l+1} shape = {net.weights[l].shape}, "
          f"output shape = {net.activations[l+1].shape}")
print(f"\nOutput probabilities (first sample):")
print(f"  {output[:, 0].round(3)}")
print(f"  Sum = {output[:, 0].sum():.6f} (should be 1.0)")
print(f"  Predicted class: {np.argmax(output[:, 0])}")
```

### Weight Initialization — Why It Matters

| Method | Formula | When to Use |
|--------|---------|------------|
| Xavier/Glorot | $W \sim \mathcal{N}(0, \frac{2}{n_{in}+n_{out}})$ | Sigmoid/Tanh activations |
| He/Kaiming | $W \sim \mathcal{N}(0, \frac{2}{n_{in}})$ | ReLU activations |
| LeCun | $W \sim \mathcal{N}(0, \frac{1}{n_{in}})$ | SELU activations |

**Why not just zeros?** All neurons would compute the same thing → identical gradients → they'd all stay identical forever. This is the **symmetry breaking problem**.

**Why not large random values?** Activations explode or saturate → gradients vanish → no learning.

---

## 6. Backpropagation — Full Derivation

### What It Is
Backpropagation is the algorithm for computing gradients of the loss with respect to every weight in a neural network. It's just the **chain rule** applied systematically from the output layer backwards.

### Why It Matters
- The reason deep learning works — without backprop, we can't train networks
- Understanding it helps diagnose vanishing/exploding gradients
- Every modern DL framework (PyTorch, TensorFlow) automates this, but you need to understand it

### The Chain Rule — The Core Idea

For a composite function $L = f(g(h(x)))$:

$$\frac{dL}{dx} = \frac{dL}{df} \cdot \frac{df}{dg} \cdot \frac{dg}{dh} \cdot \frac{dh}{dx}$$

In a neural network, this chain goes from the loss back through every layer.

### Derivation for a 2-Layer Network

Consider: $x \xrightarrow{W_1, b_1} z_1 \xrightarrow{ReLU} a_1 \xrightarrow{W_2, b_2} z_2 \xrightarrow{softmax} \hat{y} \xrightarrow{CE} L$

**Step 1: Output layer gradient**

For softmax + cross-entropy, the combined gradient is beautifully simple:

$$\delta_2 = \frac{\partial L}{\partial z_2} = \hat{y} - y$$

(This is why softmax + cross-entropy is the standard combo — the gradient is just prediction minus truth!)

**Step 2: Gradients for $W_2$ and $b_2$**

$$\frac{\partial L}{\partial W_2} = \delta_2 \cdot a_1^T$$

$$\frac{\partial L}{\partial b_2} = \delta_2$$

**Step 3: Propagate error backward**

$$\delta_1 = (W_2^T \delta_2) \odot g'(z_1)$$

Where $\odot$ is element-wise multiplication and $g'(z_1) = \mathbb{1}(z_1 > 0)$ for ReLU.

**Step 4: Gradients for $W_1$ and $b_1$**

$$\frac{\partial L}{\partial W_1} = \delta_1 \cdot x^T$$

$$\frac{\partial L}{\partial b_1} = \delta_1$$

### General Pattern

For layer $l$, working backwards from the output:

$$\delta^{[l]} = (W^{[l+1]T} \delta^{[l+1]}) \odot g'(z^{[l]})$$
$$\frac{\partial L}{\partial W^{[l]}} = \frac{1}{N} \delta^{[l]} (a^{[l-1]})^T$$
$$\frac{\partial L}{\partial b^{[l]}} = \frac{1}{N} \sum \delta^{[l]}$$

### Visual: The Computation Graph

```
FORWARD PASS (left → right):
x → [W1·x + b1] → z1 → [ReLU] → a1 → [W2·a1 + b2] → z2 → [Softmax] → ŷ → [CE] → L

BACKWARD PASS (right → left):
∂L/∂x ← [∂/∂W1] ← δ1 ← [W2^T] ← δ2 ← [∂/∂z2] ← ∂L/∂ŷ ← [∂/∂ŷ] ← ∂L/∂L=1

Each arrow carries a gradient. We multiply them together (chain rule).
```

```python
import numpy as np

class NeuralNetworkWithBackprop:
    """
    Complete neural network with forward pass, backpropagation, and training.
    Every line of the backprop is derived from the chain rule.
    """
    
    def __init__(self, layer_sizes, lr=0.01):
        self.lr = lr
        self.n_layers = len(layer_sizes) - 1
        self.weights = []
        self.biases = []
        
        for i in range(self.n_layers):
            n_in, n_out = layer_sizes[i], layer_sizes[i+1]
            # He initialization (good for ReLU)
            W = np.random.randn(n_out, n_in) * np.sqrt(2.0 / n_in)
            b = np.zeros((n_out, 1))
            self.weights.append(W)
            self.biases.append(b)
    
    def relu(self, z):
        return np.maximum(0, z)
    
    def relu_deriv(self, z):
        return (z > 0).astype(float)
    
    def softmax(self, z):
        z = z - np.max(z, axis=0, keepdims=True)
        e = np.exp(z)
        return e / np.sum(e, axis=0, keepdims=True)
    
    def forward(self, X):
        """Store all intermediate values for backprop."""
        self.z_cache = []
        self.a_cache = [X]
        
        a = X
        for l in range(self.n_layers):
            z = self.weights[l] @ a + self.biases[l]
            self.z_cache.append(z)
            
            if l < self.n_layers - 1:
                a = self.relu(z)
            else:
                a = self.softmax(z)
            self.a_cache.append(a)
        
        return a
    
    def backward(self, y_onehot):
        """
        Backpropagation: compute gradients for all parameters.
        
        Derivation (for each layer l, working backwards):
        
        Output layer:
            δ_L = ŷ - y     (softmax + cross-entropy combined gradient)
        
        Hidden layer l:
            δ_l = (W_{l+1}^T · δ_{l+1}) ⊙ ReLU'(z_l)
        
        Parameter gradients:
            dW_l = (1/N) · δ_l · a_{l-1}^T
            db_l = (1/N) · sum(δ_l, axis=1)
        """
        N = y_onehot.shape[1]
        grads_w = [None] * self.n_layers
        grads_b = [None] * self.n_layers
        
        # === Output Layer ===
        # Combined softmax + cross-entropy gradient: δ = ŷ - y
        # Why is it so simple? The softmax Jacobian and CE gradient cancel beautifully.
        delta = self.a_cache[-1] - y_onehot  # (C, N)
        
        grads_w[-1] = (1/N) * delta @ self.a_cache[-2].T
        grads_b[-1] = (1/N) * np.sum(delta, axis=1, keepdims=True)
        
        # === Hidden Layers (right to left) ===
        for l in range(self.n_layers - 2, -1, -1):
            # Propagate error backward through weights
            delta = self.weights[l+1].T @ delta
            
            # Element-wise multiply by activation derivative (chain rule)
            delta = delta * self.relu_deriv(self.z_cache[l])
            
            # Compute parameter gradients
            grads_w[l] = (1/N) * delta @ self.a_cache[l].T
            grads_b[l] = (1/N) * np.sum(delta, axis=1, keepdims=True)
        
        return grads_w, grads_b
    
    def update(self, grads_w, grads_b):
        """Gradient descent update."""
        for l in range(self.n_layers):
            self.weights[l] -= self.lr * grads_w[l]
            self.biases[l] -= self.lr * grads_b[l]
    
    def train_step(self, X, y_onehot):
        """One complete training step: forward → backward → update."""
        y_hat = self.forward(X)
        loss = -np.mean(np.sum(y_onehot * np.log(np.clip(y_hat, 1e-15, 1)), axis=0))
        grads_w, grads_b = self.backward(y_onehot)
        self.update(grads_w, grads_b)
        return loss


def to_onehot(y, num_classes):
    """Convert integer labels to one-hot encoding."""
    onehot = np.zeros((num_classes, len(y)))
    onehot[y, np.arange(len(y))] = 1
    return onehot


# ============================================
# Train on a simple dataset
# ============================================
from sklearn.datasets import load_digits
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler

digits = load_digits()
X, y = digits.data, digits.target

# Preprocess
scaler = StandardScaler()
X = scaler.fit_transform(X)

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Transpose for our column-vector convention: (features, samples)
X_train_T = X_train.T  # (64, N_train)
X_test_T = X_test.T
y_train_oh = to_onehot(y_train, 10)

# Build and train network
net = NeuralNetworkWithBackprop([64, 128, 64, 10], lr=0.1)

losses = []
for epoch in range(200):
    loss = net.train_step(X_train_T, y_train_oh)
    losses.append(loss)
    
    if (epoch + 1) % 50 == 0:
        y_pred = np.argmax(net.forward(X_test_T), axis=0)
        acc = np.mean(y_pred == y_test)
        print(f"Epoch {epoch+1}: Loss = {loss:.4f}, Test Acc = {acc:.4f}")

# Final results
y_pred = np.argmax(net.forward(X_test_T), axis=0)
print(f"\nFinal test accuracy: {np.mean(y_pred == y_test):.4f}")

# Gradient checking (verify backprop is correct)
def numerical_gradient(net, X, y_oh, layer, param_type, epsilon=1e-5):
    """Numerical gradient for verification."""
    params = net.weights[layer] if param_type == 'w' else net.biases[layer]
    num_grad = np.zeros_like(params)
    
    for idx in np.ndindex(params.shape):
        old_val = params[idx]
        
        params[idx] = old_val + epsilon
        y_hat = net.forward(X)
        loss_plus = -np.mean(np.sum(y_oh * np.log(np.clip(y_hat, 1e-15, 1)), axis=0))
        
        params[idx] = old_val - epsilon
        y_hat = net.forward(X)
        loss_minus = -np.mean(np.sum(y_oh * np.log(np.clip(y_hat, 1e-15, 1)), axis=0))
        
        num_grad[idx] = (loss_plus - loss_minus) / (2 * epsilon)
        params[idx] = old_val
    
    return num_grad

# Verify on a small network
small_net = NeuralNetworkWithBackprop([4, 5, 3], lr=0.01)
X_small = np.random.randn(4, 2)
y_small = to_onehot(np.array([0, 2]), 3)

small_net.forward(X_small)
grads_w, grads_b = small_net.backward(y_small)

num_grad = numerical_gradient(small_net, X_small, y_small, layer=0, param_type='w')
diff = np.linalg.norm(grads_w[0] - num_grad) / (np.linalg.norm(grads_w[0]) + np.linalg.norm(num_grad))
print(f"\nGradient check (should be < 1e-5): {diff:.2e}")
```

### Vanishing and Exploding Gradients

```
Layer N    Layer N-1    Layer N-2    ...    Layer 1
  δ_N  ←── W_N^T·δ_N ←── W_{N-1}^T·... ←── δ_1

Each backward step MULTIPLIES by W^T.
- If ||W|| < 1: gradients shrink exponentially → VANISHING
- If ||W|| > 1: gradients grow exponentially → EXPLODING
```

**Solutions**:

| Problem | Solution | How It Helps |
|---------|----------|-------------|
| Vanishing | ReLU activation | Gradient is 0 or 1 (no shrinking) |
| Vanishing | Skip connections (ResNet) | Gradient has a direct path |
| Vanishing | Proper initialization (He) | Keeps variance stable |
| Exploding | Gradient clipping | Caps gradient magnitude |
| Exploding | Batch normalization | Stabilizes layer inputs |
| Both | LSTM/GRU (for RNNs) | Gating controls gradient flow |

---

## 7. Batch Normalization — Why It Works

### What It Is
Batch Normalization normalizes the inputs to each layer so they have zero mean and unit variance. It's like standardizing your features, but done **inside the network** at every layer.

### Why It Matters
- Allows much higher learning rates → faster training
- Reduces sensitivity to initialization
- Acts as a mild regularizer (noise from mini-batch statistics)
- Used in virtually all modern CNNs

### The Math

For a mini-batch $B = \{z_1, ..., z_m\}$ (pre-activation values):

**Step 1**: Batch statistics

$$\mu_B = \frac{1}{m} \sum_{i=1}^{m} z_i, \quad \sigma_B^2 = \frac{1}{m} \sum_{i=1}^{m} (z_i - \mu_B)^2$$

**Step 2**: Normalize

$$\hat{z}_i = \frac{z_i - \mu_B}{\sqrt{\sigma_B^2 + \epsilon}}$$

**Step 3**: Scale and shift (learnable parameters)

$$\tilde{z}_i = \gamma \hat{z}_i + \beta$$

Where $\gamma$ (scale) and $\beta$ (shift) are **learned** — this lets the network undo the normalization if it's not helpful.

### Why the Learnable Parameters?

If we only normalized, we'd force every layer to have mean 0 and variance 1, which constrains the network too much. The learnable $\gamma$ and $\beta$ let each layer choose its own optimal scale and shift. If $\gamma = \sigma_B$ and $\beta = \mu_B$, the network recovers the original un-normalized values.

### Training vs Inference

| Phase | Mean & Variance | Why |
|-------|----------------|-----|
| Training | Per mini-batch | Different each batch → acts as regularizer |
| Inference | Running average (from training) | Must be deterministic at test time |

```python
import numpy as np

class BatchNorm:
    """
    Batch Normalization from scratch.
    Applied to pre-activation values z (before the activation function).
    """
    
    def __init__(self, num_features, momentum=0.9, epsilon=1e-5):
        self.gamma = np.ones(num_features)    # Learnable scale
        self.beta = np.zeros(num_features)    # Learnable shift
        self.epsilon = epsilon
        self.momentum = momentum
        
        # Running stats for inference
        self.running_mean = np.zeros(num_features)
        self.running_var = np.ones(num_features)
        
        # Cache for backprop
        self.cache = None
    
    def forward(self, z, training=True):
        """
        Args:
            z: Input, shape (N, D) where N=batch, D=features
            training: If True, use batch stats; if False, use running stats
        """
        if training:
            # Batch statistics
            mu = np.mean(z, axis=0)                    # (D,)
            var = np.var(z, axis=0)                     # (D,)
            
            # Normalize
            z_norm = (z - mu) / np.sqrt(var + self.epsilon)  # (N, D)
            
            # Scale and shift
            out = self.gamma * z_norm + self.beta       # (N, D)
            
            # Update running stats (exponential moving average)
            self.running_mean = self.momentum * self.running_mean + (1 - self.momentum) * mu
            self.running_var = self.momentum * self.running_var + (1 - self.momentum) * var
            
            # Cache for backward pass
            self.cache = (z, z_norm, mu, var)
        else:
            # Use running stats at inference
            z_norm = (z - self.running_mean) / np.sqrt(self.running_var + self.epsilon)
            out = self.gamma * z_norm + self.beta
        
        return out
    
    def backward(self, dout):
        """
        Backprop through batch norm.
        
        This derivation is notoriously tricky — the key insight is that 
        the mean and variance depend on ALL samples in the batch, so
        gradients flow through the statistics too.
        """
        z, z_norm, mu, var = self.cache
        N = z.shape[0]
        std = np.sqrt(var + self.epsilon)
        
        # Gradients of learnable parameters
        dgamma = np.sum(dout * z_norm, axis=0)
        dbeta = np.sum(dout, axis=0)
        
        # Gradient through the normalization
        dz_norm = dout * self.gamma
        
        # The hard part: gradient through mean and variance
        dvar = np.sum(dz_norm * (z - mu) * (-0.5) * (var + self.epsilon)**(-1.5), axis=0)
        dmu = np.sum(dz_norm * (-1/std), axis=0) + dvar * (-2/N) * np.sum(z - mu, axis=0)
        
        # Final gradient w.r.t. input z
        dz = dz_norm / std + dvar * 2 * (z - mu) / N + dmu / N
        
        return dz, dgamma, dbeta


# Quick demo
bn = BatchNorm(num_features=5)
z = np.random.randn(32, 5) * 3 + 7  # Unnormalized: mean≈7, var≈9

z_normed = bn.forward(z, training=True)
print(f"Before BN: mean={z.mean(axis=0).round(2)}, var={z.var(axis=0).round(2)}")
print(f"After BN:  mean={z_normed.mean(axis=0).round(2)}, var={z_normed.var(axis=0).round(2)}")
```

---

## 8. Attention Mechanism — The Math

### What It Is
Attention allows a model to focus on the most relevant parts of the input when producing each part of the output. It's the core innovation behind Transformers (BERT, GPT, etc.).

### Why It Matters
- Powers all modern NLP (ChatGPT, BERT, T5)
- Also used in vision (ViT), speech, and multimodal models
- Replaced RNNs because it allows parallel processing and captures long-range dependencies

### Scaled Dot-Product Attention

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

Where:
- $Q$ = Queries: "What am I looking for?" — shape $(n, d_k)$
- $K$ = Keys: "What do I contain?" — shape $(m, d_k)$
- $V$ = Values: "What information do I have?" — shape $(m, d_v)$
- $d_k$ = dimension of keys (for scaling)

### Step-by-Step Breakdown

```
Step 1: Compute attention scores (how relevant is each key to each query)
    Scores = Q · K^T                    Shape: (n, m)
    
Step 2: Scale (prevent softmax saturation)
    Scores = Scores / √d_k              Shape: (n, m)
    
Step 3: Normalize with softmax (make weights sum to 1)
    Weights = softmax(Scores, dim=-1)   Shape: (n, m)
    
Step 4: Weighted sum of values
    Output = Weights · V                Shape: (n, d_v)
```

### Why Scale by $\sqrt{d_k}$?

Without scaling, dot products grow with dimension $d_k$:

$$\text{Var}(q \cdot k) = d_k \cdot \text{Var}(q_i) \cdot \text{Var}(k_j) = d_k$$

Large dot products → softmax becomes very peaked (like argmax) → gradients vanish. Dividing by $\sqrt{d_k}$ keeps variance ≈ 1.

### Multi-Head Attention

Instead of one attention function, run $h$ attention heads in parallel with different learned projections:

$$\text{head}_i = \text{Attention}(QW_i^Q, KW_i^K, VW_i^V)$$
$$\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, ..., \text{head}_h) W^O$$

Where $W_i^Q \in \mathbb{R}^{d_{model} \times d_k}$, etc.

**Why multiple heads?** Each head can attend to different types of relationships (syntax, semantics, position, etc.).

```python
import numpy as np

def scaled_dot_product_attention(Q, K, V, mask=None):
    """
    Scaled Dot-Product Attention from "Attention Is All You Need".
    
    Args:
        Q: Queries (batch, n_queries, d_k)
        K: Keys    (batch, n_keys, d_k)
        V: Values  (batch, n_keys, d_v)
        mask: Optional mask (for causal/padding attention)
    
    Returns:
        output: Weighted values (batch, n_queries, d_v)
        attention_weights: The attention pattern (batch, n_queries, n_keys)
    """
    d_k = Q.shape[-1]
    
    # Step 1: Compute raw scores
    scores = Q @ K.transpose(0, 2, 1)  # (batch, n_q, n_k)
    
    # Step 2: Scale
    scores = scores / np.sqrt(d_k)
    
    # Step 3: Apply mask (if any)
    if mask is not None:
        scores = np.where(mask, scores, -1e9)  # -inf where masked
    
    # Step 4: Softmax to get attention weights
    scores_shifted = scores - np.max(scores, axis=-1, keepdims=True)
    exp_scores = np.exp(scores_shifted)
    attention_weights = exp_scores / np.sum(exp_scores, axis=-1, keepdims=True)
    
    # Step 5: Weighted sum of values
    output = attention_weights @ V  # (batch, n_q, d_v)
    
    return output, attention_weights


class MultiHeadAttention:
    """
    Multi-Head Attention from scratch.
    
    Each head learns different attention patterns:
    - Head 1 might focus on adjacent words
    - Head 2 might focus on subject-verb agreement
    - Head 3 might focus on coreference
    """
    
    def __init__(self, d_model, n_heads):
        assert d_model % n_heads == 0, "d_model must be divisible by n_heads"
        
        self.d_model = d_model
        self.n_heads = n_heads
        self.d_k = d_model // n_heads  # Dimension per head
        
        # Projection matrices (Xavier init)
        scale = np.sqrt(2.0 / d_model)
        self.W_Q = np.random.randn(d_model, d_model) * scale
        self.W_K = np.random.randn(d_model, d_model) * scale
        self.W_V = np.random.randn(d_model, d_model) * scale
        self.W_O = np.random.randn(d_model, d_model) * scale
    
    def forward(self, Q, K, V, mask=None):
        """
        Args:
            Q, K, V: shape (batch, seq_len, d_model)
        Returns:
            output: shape (batch, seq_len, d_model)
        """
        batch_size = Q.shape[0]
        
        # Project to Q, K, V spaces
        Q_proj = Q @ self.W_Q  # (batch, seq_len, d_model)
        K_proj = K @ self.W_K
        V_proj = V @ self.W_V
        
        # Reshape for multi-head: split d_model into n_heads × d_k
        # (batch, seq_len, d_model) → (batch, n_heads, seq_len, d_k)
        Q_heads = Q_proj.reshape(batch_size, -1, self.n_heads, self.d_k).transpose(0, 2, 1, 3)
        K_heads = K_proj.reshape(batch_size, -1, self.n_heads, self.d_k).transpose(0, 2, 1, 3)
        V_heads = V_proj.reshape(batch_size, -1, self.n_heads, self.d_k).transpose(0, 2, 1, 3)
        
        # Apply attention to each head
        # Reshape to (batch*n_heads, seq_len, d_k) for our attention function
        B_H = batch_size * self.n_heads
        Q_flat = Q_heads.reshape(B_H, -1, self.d_k)
        K_flat = K_heads.reshape(B_H, -1, self.d_k)
        V_flat = V_heads.reshape(B_H, -1, self.d_k)
        
        attn_out, attn_weights = scaled_dot_product_attention(Q_flat, K_flat, V_flat)
        
        # Reshape back: (batch*n_heads, seq_len, d_k) → (batch, seq_len, d_model)
        attn_out = attn_out.reshape(batch_size, self.n_heads, -1, self.d_k)
        attn_out = attn_out.transpose(0, 2, 1, 3).reshape(batch_size, -1, self.d_model)
        
        # Final projection
        output = attn_out @ self.W_O
        
        return output


# Demo: Self-attention on a sequence
np.random.seed(42)
batch_size = 2
seq_len = 5
d_model = 64
n_heads = 8

# Random embeddings (in practice, these come from an embedding layer)
X = np.random.randn(batch_size, seq_len, d_model)

# Self-attention: Q = K = V = X (the input attends to itself)
mha = MultiHeadAttention(d_model=d_model, n_heads=n_heads)
output = mha.forward(X, X, X)

print(f"Input shape:  {X.shape}")
print(f"Output shape: {output.shape}")
print(f"d_k per head: {d_model // n_heads}")

# Visualize attention weights for a single head
Q_single = X[0:1] @ mha.W_Q  # (1, 5, 64)
K_single = X[0:1] @ mha.W_K

# Just take the first d_k dimensions (first head)
d_k = d_model // n_heads
Q_h1 = Q_single[:, :, :d_k]
K_h1 = K_single[:, :, :d_k]
V_h1 = X[0:1, :, :d_k]

_, weights = scaled_dot_product_attention(Q_h1, K_h1, V_h1)

import matplotlib.pyplot as plt
plt.figure(figsize=(6, 5))
plt.imshow(weights[0], cmap='Blues', aspect='auto')
plt.colorbar(label='Attention Weight')
plt.xlabel('Key Position')
plt.ylabel('Query Position')
plt.title('Self-Attention Weights (Head 1)')
tokens = [f'Pos {i}' for i in range(seq_len)]
plt.xticks(range(seq_len), tokens)
plt.yticks(range(seq_len), tokens)
plt.show()

print("\nAttention weights (Head 1):")
print(weights[0].round(3))
print(f"Row sums: {weights[0].sum(axis=1).round(6)}")  # Should all be 1.0
```

### Self-Attention vs Cross-Attention

| Type | Q source | K, V source | Use Case |
|------|----------|-------------|----------|
| Self-attention | Same sequence | Same sequence | BERT, GPT (within a sentence) |
| Cross-attention | Decoder | Encoder output | Machine translation, image captioning |
| Causal self-attention | Same sequence (masked) | Same sequence | GPT (can only look at past tokens) |

---

## Common Mistakes

1. **PCA without standardization** — Features on different scales will dominate PCA. Always use `StandardScaler` before PCA (unless features are already on the same scale).

2. **Confusing SVM primal and dual** — The dual form is where the kernel trick happens. Most implementations solve the dual. The primal gives weights directly; the dual gives support vector coefficients.

3. **Forgetting sigmoid numerical stability** — Computing `1/(1+exp(-z))` overflows for large negative z. Use the stable version: `exp(z)/(1+exp(z))` when z < 0.

4. **Wrong shapes in backprop** — The gradient of a matrix product $Y = W \cdot X$ is $dW = dY \cdot X^T$ and $dX = W^T \cdot dY$. Getting the transposes wrong is the #1 backprop bug.

5. **BatchNorm at inference time** — Must use running statistics, not batch statistics. In PyTorch, call `model.eval()` before inference.

6. **Not understanding attention scaling** — Without $\sqrt{d_k}$ scaling, attention becomes nearly one-hot for large dimensions, killing gradients.

---

## Interview Questions

**Q1: Derive PCA. Why does it use eigenvectors of the covariance matrix?**
> We want to find direction $u$ maximizing projected variance $u^T C u$ subject to $\|u\|=1$. Using Lagrange multipliers: $Cu = \lambda u$. This is the eigenvalue equation. The eigenvector with the largest eigenvalue is the direction of maximum variance. Each eigenvalue equals the variance along its eigenvector.

**Q2: Explain the kernel trick in SVMs. Why does it work?**
> The SVM dual form only uses dot products $x_i^T x_j$. We can replace these with $K(x_i, x_j) = \phi(x_i)^T \phi(x_j)$ where $\phi$ maps to a higher-dimensional space. The trick: we never compute $\phi$ explicitly — the kernel function gives us the dot product directly. RBF kernel implicitly maps to infinite dimensions.

**Q3: Derive the backpropagation update rule. Why is softmax + cross-entropy a good combination?**
> The gradient of cross-entropy loss w.r.t. softmax logits simplifies to $\hat{y} - y$ (prediction minus truth). This means the gradient is proportional to the error — large errors → large gradients → fast learning. Other combinations (e.g., MSE + sigmoid) have vanishing gradient problems.

**Q4: What is Batch Normalization and why does it help training?**
> BN normalizes layer inputs to zero mean and unit variance, then applies learnable scale/shift. It helps by: (1) allowing higher learning rates (inputs are well-scaled), (2) reducing internal covariate shift (layer inputs don't change distribution during training), (3) acting as regularization (batch statistics are noisy). At inference, uses running averages from training.

**Q5: Explain multi-head attention. Why multiple heads instead of one big attention?**
> Each head projects Q, K, V to a lower-dimensional subspace ($d_k = d_{model}/h$) and computes independent attention. Different heads can capture different relationship types (position, syntax, semantics). Total computation is similar to single-head (since dimensions are split), but representation is richer.

---

## Quick Reference

| Algorithm | Core Math | Key Insight |
|-----------|----------|-------------|
| PCA | Eigendecomposition of $C = X^TX/(N-1)$ | Eigenvectors = directions of max variance |
| SVM | Maximize $\frac{2}{\|w\|}$ s.t. $y_i(w^Tx_i+b) \geq 1$ | Only support vectors matter |
| Logistic Reg | $P(y=1) = \sigma(w^Tx+b)$, MLE → cross-entropy | Gradient = $(\hat{y}-y) \cdot x$ |
| Naive Bayes | $\hat{y} = \arg\max_c P(c)\prod P(x_j|c)$ | Independence makes computation tractable |
| Forward Pass | $a^{[l]} = g(W^{[l]}a^{[l-1]} + b^{[l]})$ | Matrix multiply + nonlinearity |
| Backprop | $\delta^{[l]} = (W^{[l+1]T}\delta^{[l+1]}) \odot g'(z^{[l]})$ | Chain rule, layer by layer |
| Batch Norm | $\tilde{z} = \gamma\frac{z-\mu}{\sqrt{\sigma^2+\epsilon}} + \beta$ | Normalize → scale → shift |
| Attention | $\text{softmax}(\frac{QK^T}{\sqrt{d_k}})V$ | Weighted lookup by relevance |

---

*Previous: [Chapter 05 - Information Theory](05-Information-Theory.md) | Back to [Index](00-INDEX.md)*
