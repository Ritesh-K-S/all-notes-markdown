# Chapter 09: Dimensionality Reduction — Complete Guide

## Table of Contents
- [What is Dimensionality Reduction?](#what-is-dimensionality-reduction)
- [Why Dimensionality Reduction Matters](#why-dimensionality-reduction-matters)
- [The Curse of Dimensionality](#the-curse-of-dimensionality)
- [PCA (Principal Component Analysis)](#pca-principal-component-analysis)
- [t-SNE](#t-sne)
- [UMAP](#umap)
- [LDA (Linear Discriminant Analysis)](#lda-linear-discriminant-analysis)
- [Feature Selection Methods](#feature-selection-methods)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Dimensionality Reduction?

### Simple Explanation (Like Explaining to a 15-Year-Old)

Imagine you have a photo (millions of pixels/dimensions). But if you squint, you can still recognize what's in it with far less detail. Dimensionality reduction does exactly that — it compresses your data from many features to fewer features while keeping the important information.

It's like taking a 3D object and casting its **shadow** on a 2D wall. You lose some information, but if you choose the right angle, the shadow still tells you a lot about the object.

### Formal Definition

Dimensionality reduction transforms data from a high-dimensional space $\mathbb{R}^d$ to a lower-dimensional space $\mathbb{R}^k$ (where $k \ll d$) while preserving as much meaningful structure as possible.

$$f: \mathbb{R}^d \rightarrow \mathbb{R}^k \quad \text{where } k \ll d$$

### Two Types

```
DIMENSIONALITY REDUCTION
├── FEATURE EXTRACTION (create new features)
│   ├── PCA (linear, preserves variance)
│   ├── t-SNE (non-linear, preserves local structure)
│   ├── UMAP (non-linear, preserves global + local)
│   ├── LDA (supervised, maximizes class separation)
│   └── Autoencoders (neural network based)
│
└── FEATURE SELECTION (pick existing features)
    ├── Filter Methods (statistical tests)
    ├── Wrapper Methods (model-based search)
    └── Embedded Methods (built into model training)
```

---

## Why Dimensionality Reduction Matters

### The Problems with High Dimensions

| Problem | Explanation |
|---------|-------------|
| **Overfitting** | More features = more parameters = model memorizes noise |
| **Computational cost** | Training time scales with feature count |
| **Visualization** | Can't plot >3D data without reduction |
| **Distance meaningless** | In high-D, all points are ~equidistant (curse of dimensionality) |
| **Multicollinearity** | Correlated features confuse linear models |
| **Storage** | More features = more memory/disk |

### When to Use

- **Visualization:** Project 100D data to 2D for plotting (t-SNE/UMAP)
- **Preprocessing:** Reduce features before feeding to ML model (PCA)
- **Noise removal:** Throw away low-variance components
- **Speed up training:** Fewer features = faster models
- **Avoid overfitting:** Fewer features = simpler model

---

## The Curse of Dimensionality

### What It Is

As dimensions increase, the volume of space increases exponentially, making data incredibly sparse. Counterintuitive things happen:

### Key Effects

```
In 2D:  Points fill a square nicely
        ● ● ● ● ●
        ● ● ● ● ●
        ● ● ● ● ●
        (25 points fill it)

In 100D: Same 25 points are like grains of sand in a warehouse
         All points are nearly equidistant!
         Distance-based methods (KNN, K-Means) break down
```

**Mathematical proof that distances converge:**

For uniform random points in $d$ dimensions:

$$\lim_{d \to \infty} \frac{d_{max} - d_{min}}{d_{min}} \to 0$$

The ratio of maximum to minimum pairwise distance approaches 1 — all distances become the same!

### Practical Impact

```python
import numpy as np
from sklearn.metrics.pairwise import euclidean_distances

# Demonstrate curse of dimensionality
np.random.seed(42)
n_samples = 100

for d in [2, 10, 50, 100, 500, 1000]:
    X = np.random.uniform(0, 1, size=(n_samples, d))
    distances = euclidean_distances(X)
    
    # Remove diagonal (self-distances)
    mask = ~np.eye(n_samples, dtype=bool)
    dist_values = distances[mask]
    
    mean_dist = dist_values.mean()
    std_dist = dist_values.std()
    ratio = std_dist / mean_dist  # Coefficient of variation
    
    print(f"d={d:4d}: Mean distance={mean_dist:.2f}, "
          f"Std={std_dist:.4f}, CV={ratio:.4f}")

# Output shows: as d increases, distances converge (CV → 0)
# d=   2: Mean distance=0.52, Std=0.2423, CV=0.4626
# d=  10: Mean distance=1.29, Std=0.2058, CV=0.1596
# d= 100: Mean distance=4.09, Std=0.2030, CV=0.0496
# d=1000: Mean distance=12.92, Std=0.2032, CV=0.0157  ← All distances ~same!
```

---

## PCA (Principal Component Analysis)

### What It Is

PCA finds the directions (principal components) along which your data varies the most. It then projects data onto these directions, keeping the most informative ones.

### Intuition — The Shadow Analogy

```
3D cloud of points:              Best 2D projection (PCA):

      ↗ PC1 (most variance)         ↗ PC1
     /                              /
    ●  ●  ●                        ●  ●  ●
   ● ●  ● ●  ●     Project →     ● ●  ● ●  ●
    ●  ●  ●                        ●  ●  ●
   ↑ PC2                           ↑ PC2
  (second most variance)

PCA finds the "best angle" for the shadow
that preserves the most spread (variance)
```

### How It Works — Step by Step

1. **Standardize** the data (zero mean, unit variance)
2. Compute the **covariance matrix** $C = \frac{1}{n-1}X^TX$
3. Compute **eigenvalues** and **eigenvectors** of $C$
4. Sort eigenvectors by decreasing eigenvalue
5. Select top $k$ eigenvectors → these are your principal components
6. **Transform:** $X_{reduced} = X \cdot W_k$ where $W_k$ = matrix of top-k eigenvectors

### The Math

**Covariance Matrix:**
$$C = \frac{1}{n-1} X^T X \quad (d \times d \text{ matrix})$$

**Eigendecomposition:**
$$C \cdot v_i = \lambda_i \cdot v_i$$

Where $v_i$ = eigenvector (direction/principal component) and $\lambda_i$ = eigenvalue (variance explained along that direction).

**Variance explained by component $i$:**
$$\text{explained\_variance\_ratio}_i = \frac{\lambda_i}{\sum_{j=1}^{d} \lambda_j}$$

### Code Example — PCA from Scratch + Scikit-learn

```python
import numpy as np
from sklearn.decomposition import PCA
from sklearn.preprocessing import StandardScaler
from sklearn.datasets import load_iris, fetch_openml
import time

# ============= PCA FROM SCRATCH =============
class PCAFromScratch:
    def __init__(self, n_components):
        self.n_components = n_components
    
    def fit(self, X):
        # Step 1: Center the data (standardize)
        self.mean_ = X.mean(axis=0)
        X_centered = X - self.mean_
        
        # Step 2: Compute covariance matrix
        n_samples = X.shape[0]
        cov_matrix = (X_centered.T @ X_centered) / (n_samples - 1)
        
        # Step 3: Eigendecomposition
        eigenvalues, eigenvectors = np.linalg.eigh(cov_matrix)
        
        # Step 4: Sort by decreasing eigenvalue
        sorted_idx = np.argsort(eigenvalues)[::-1]
        eigenvalues = eigenvalues[sorted_idx]
        eigenvectors = eigenvectors[:, sorted_idx]
        
        # Step 5: Select top-k components
        self.components_ = eigenvectors[:, :self.n_components].T
        self.explained_variance_ = eigenvalues[:self.n_components]
        self.explained_variance_ratio_ = (
            eigenvalues[:self.n_components] / eigenvalues.sum()
        )
        return self
    
    def transform(self, X):
        X_centered = X - self.mean_
        return X_centered @ self.components_.T
    
    def fit_transform(self, X):
        self.fit(X)
        return self.transform(X)

# Load Iris dataset
iris = load_iris()
X = iris.data  # 4 features
y = iris.target

# Standardize
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# Our PCA
pca_scratch = PCAFromScratch(n_components=2)
X_pca_scratch = pca_scratch.fit_transform(X_scaled)
print("Custom PCA:")
print(f"  Explained variance ratio: {pca_scratch.explained_variance_ratio_}")
print(f"  Total variance explained: {sum(pca_scratch.explained_variance_ratio_):.4f}")

# ============= SCIKIT-LEARN PCA =============
pca = PCA(n_components=2)
X_pca = pca.fit_transform(X_scaled)

print(f"\nSklearn PCA:")
print(f"  Explained variance ratio: {pca.explained_variance_ratio_}")
print(f"  Total variance explained: {sum(pca.explained_variance_ratio_):.4f}")
print(f"  Components shape: {pca.components_.shape}")  # (2, 4) — 2 PCs, 4 original features

# What each PC represents (loadings)
print("\nComponent Loadings (which features contribute to each PC):")
feature_names = iris.feature_names
for i, comp in enumerate(pca.components_):
    print(f"  PC{i+1}: ", end="")
    for name, loading in zip(feature_names, comp):
        print(f"{name}={loading:.3f}  ", end="")
    print()
```

### Choosing the Right Number of Components

```python
# Method 1: Cumulative explained variance
pca_full = PCA()  # Fit all components
pca_full.fit(X_scaled)

cumulative_variance = np.cumsum(pca_full.explained_variance_ratio_)
print("Cumulative variance explained:")
for i, var in enumerate(cumulative_variance):
    print(f"  {i+1} components: {var:.4f} ({var*100:.1f}%)")

# Find number of components for 95% variance
n_components_95 = np.argmax(cumulative_variance >= 0.95) + 1
print(f"\nComponents needed for 95% variance: {n_components_95}")

# Method 2: Directly specify variance threshold
pca_95 = PCA(n_components=0.95)  # Keep enough for 95% variance
X_pca_95 = pca_95.fit_transform(X_scaled)
print(f"Sklearn auto-selected {pca_95.n_components_} components for 95% variance")

# Method 3: Scree plot (look for "elbow")
print("\nScree Plot Data (Eigenvalues):")
for i, (ev, evr) in enumerate(zip(pca_full.explained_variance_, 
                                     pca_full.explained_variance_ratio_)):
    bar = "█" * int(evr * 50)
    print(f"  PC{i+1}: {evr:.4f} {bar}")
```

### PCA for Noise Reduction

```python
from sklearn.datasets import load_digits
import numpy as np

# Load handwritten digits (64 dimensions = 8x8 pixels)
digits = load_digits()
X_digits = digits.data  # (1797, 64)

# Add noise
np.random.seed(42)
X_noisy = X_digits + np.random.normal(0, 2, X_digits.shape)

# Denoise using PCA
pca_denoise = PCA(n_components=20)  # Keep only 20 components (remove noise)
X_denoised = pca_denoise.inverse_transform(pca_denoise.fit_transform(X_noisy))

# Compare reconstruction error
from sklearn.metrics import mean_squared_error
print(f"MSE (noisy vs original): {mean_squared_error(X_digits, X_noisy):.4f}")
print(f"MSE (denoised vs original): {mean_squared_error(X_digits, X_denoised):.4f}")
# Denoised should be closer to original!
print(f"Variance explained by 20 PCs: {sum(pca_denoise.explained_variance_ratio_):.4f}")
```

### Kernel PCA (Non-linear PCA)

```python
from sklearn.decomposition import KernelPCA
from sklearn.datasets import make_circles

# PCA fails on non-linear structures (like circles)
X_circles, y_circles = make_circles(n_samples=500, noise=0.05, factor=0.3, random_state=42)

# Standard PCA — can't separate circles
pca_linear = PCA(n_components=2)
X_pca_linear = pca_linear.fit_transform(X_circles)
print("Linear PCA on circles — won't separate them well")

# Kernel PCA — uses kernel trick to find non-linear structure
kpca = KernelPCA(
    n_components=2,
    kernel='rbf',          # Options: 'linear', 'poly', 'rbf', 'sigmoid', 'cosine'
    gamma=10,              # RBF kernel parameter
    fit_inverse_transform=True  # Enable approximate inverse transform
)
X_kpca = kpca.fit_transform(X_circles)
print(f"Kernel PCA on circles — first component separates them!")
print(f"  Component 1 range (inner circle): "
      f"[{X_kpca[y_circles==0, 0].min():.2f}, {X_kpca[y_circles==0, 0].max():.2f}]")
print(f"  Component 1 range (outer circle): "
      f"[{X_kpca[y_circles==1, 0].min():.2f}, {X_kpca[y_circles==1, 0].max():.2f}]")
```

> **Pro Tip:** PCA components are **orthogonal** (uncorrelated). This is great as preprocessing for models that assume independence. But PCA is **linear** — for non-linear relationships, use Kernel PCA, t-SNE, or UMAP.

---

## t-SNE

### What It Is

**t-distributed Stochastic Neighbor Embedding** — a non-linear dimensionality reduction technique specifically designed for **visualization** of high-dimensional data in 2D/3D. Created by Laurens van der Maaten and Geoffrey Hinton (2008).

### How It Works — Intuition

Imagine you have a room full of people with various relationships. t-SNE tries to seat them at a restaurant table (2D) such that people who were close in the room remain close at the table, and people who were far apart remain far apart.

### The Algorithm (Simplified)

1. **In high-D:** Compute pairwise similarities using Gaussian kernels:
$$p_{j|i} = \frac{\exp(-\|x_i - x_j\|^2 / 2\sigma_i^2)}{\sum_{k \neq i} \exp(-\|x_i - x_k\|^2 / 2\sigma_i^2)}$$

2. **In low-D:** Compute pairwise similarities using Student's t-distribution (heavier tails):
$$q_{ij} = \frac{(1 + \|y_i - y_j\|^2)^{-1}}{\sum_{k \neq l} (1 + \|y_k - y_l\|^2)^{-1}}$$

3. **Minimize** the KL divergence between $P$ and $Q$:
$$KL(P \| Q) = \sum_{i \neq j} p_{ij} \log \frac{p_{ij}}{q_{ij}}$$

### Why Student's t-distribution?

```
Gaussian (high-D):          Student's t (low-D):
    
    │  ╱╲                      │  ╱──╲
    │ ╱  ╲                     │ ╱    ╲
    │╱    ╲                    │╱      ╲──── (heavy tails)
    └──────────                └──────────

The t-distribution's heavy tails allow moderate distances in high-D
to become larger distances in low-D → better separation of clusters!
```

### Code Example — t-SNE

```python
from sklearn.manifold import TSNE
from sklearn.datasets import load_digits, fetch_openml
from sklearn.preprocessing import StandardScaler
import numpy as np
import time

# Load digits dataset (64 dimensions → 2D for visualization)
digits = load_digits()
X_digits = digits.data
y_digits = digits.target

# Scale data
X_scaled = StandardScaler().fit_transform(X_digits)

# ============= BASIC t-SNE =============
tsne = TSNE(
    n_components=2,         # Output dimensions (usually 2 or 3)
    perplexity=30,          # Balance between local and global structure (5-50)
    learning_rate='auto',   # Step size for gradient descent (or 'auto')
    n_iter=1000,            # Number of iterations
    init='pca',             # Initialization ('random' or 'pca' — pca is more stable)
    random_state=42,
    metric='euclidean'
)

start = time.time()
X_tsne = tsne.fit_transform(X_scaled)
print(f"t-SNE time: {time.time() - start:.2f}s")
print(f"Final KL divergence: {tsne.kl_divergence_:.4f}")
print(f"Output shape: {X_tsne.shape}")

# ============= PERPLEXITY EXPERIMENT =============
# Perplexity ~ "effective number of neighbors"
# Low perplexity (5-10): focuses on very local structure
# High perplexity (30-50): captures more global structure
# Rule of thumb: perplexity should be < n_samples / 3

print("\nPerplexity comparison:")
for perp in [5, 15, 30, 50, 100]:
    tsne_test = TSNE(n_components=2, perplexity=perp, random_state=42,
                      init='pca', n_iter=500)
    X_test = tsne_test.fit_transform(X_scaled)
    print(f"  Perplexity={perp:3d}: KL divergence={tsne_test.kl_divergence_:.4f}")

# ============= IMPORTANT CAVEATS =============
# 1. t-SNE is NON-PARAMETRIC — can't transform new data
# 2. Cluster sizes in t-SNE plot DON'T represent actual cluster sizes
# 3. Distances between clusters are NOT meaningful
# 4. Different runs give different results (use random_state)
# 5. Perplexity matters A LOT — always try multiple values
```

### t-SNE Best Practices

```python
# Pro Tips for better t-SNE results:

# 1. Pre-reduce with PCA (much faster, often better)
from sklearn.decomposition import PCA

# First reduce to 50 dimensions with PCA, then t-SNE to 2D
pca_pre = PCA(n_components=50)
X_pca_50 = pca_pre.fit_transform(X_scaled)

tsne_fast = TSNE(n_components=2, perplexity=30, random_state=42, init='pca')
start = time.time()
X_tsne_fast = tsne_fast.fit_transform(X_pca_50)
print(f"PCA+t-SNE time: {time.time() - start:.2f}s (much faster!)")

# 2. For large datasets, use OpenTSNE or RAPIDS cuML (GPU-accelerated)
# pip install openTSNE
# from openTSNE import TSNE as OpenTSNE

# 3. Always try multiple perplexity values
# 4. Run for enough iterations (at least 1000, ideally until KL converges)
# 5. Use init='pca' for reproducible results
```

> **Critical Warning:** t-SNE is for **VISUALIZATION ONLY**. Never use t-SNE components as features for a downstream ML model. The embedding is non-parametric and distances are not preserved globally.

---

## UMAP

### What It Is

**Uniform Manifold Approximation and Projection** — a newer (2018) alternative to t-SNE that is:
- **Much faster** (scales better to large datasets)
- **Preserves more global structure** (cluster-to-cluster relationships)
- **Can transform new data** (parametric option)
- **Works for general dimensionality reduction** (not just visualization)

### How It Works — Simplified

1. Build a weighted graph of nearest neighbors in high-D
2. Optimize a low-D layout to match the graph topology
3. Uses cross-entropy loss (vs. KL divergence in t-SNE)

### UMAP vs t-SNE

| Feature | t-SNE | UMAP |
|---------|-------|------|
| Speed | Slow ($O(n^2)$) | Fast ($O(n \log n)$) |
| Global structure | Poor | Good |
| New data transform | No | Yes |
| Deterministic | No | More stable |
| Use for ML features | No | Yes (carefully) |
| Dimensionality | 2-3D only | Any dimension |

### Code Example — UMAP

```python
# pip install umap-learn
import umap
from sklearn.datasets import load_digits
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import train_test_split
import numpy as np
import time

# Load data
digits = load_digits()
X = StandardScaler().fit_transform(digits.data)
y = digits.target

# ============= BASIC UMAP =============
reducer = umap.UMAP(
    n_components=2,          # Output dimensions
    n_neighbors=15,          # Size of local neighborhood (like perplexity)
    min_dist=0.1,            # How tightly UMAP packs points (0=clumpy, 1=spread)
    metric='euclidean',      # Distance metric (supports many!)
    random_state=42,
    n_jobs=-1
)

start = time.time()
X_umap = reducer.fit_transform(X)
print(f"UMAP time: {time.time() - start:.2f}s")
print(f"Output shape: {X_umap.shape}")

# ============= UMAP PARAMETERS EXPLORATION =============
# n_neighbors: controls local vs global structure
# - Small (5-15): more local structure, tighter clusters
# - Large (50-200): more global structure, broader view

# min_dist: controls how tightly points are packed
# - 0.0: points can be on top of each other (tight clusters)
# - 0.5-1.0: more uniform distribution

print("\nParameter exploration:")
for n_neigh in [5, 15, 50, 100]:
    for min_d in [0.0, 0.1, 0.5]:
        reducer_test = umap.UMAP(
            n_neighbors=n_neigh, min_dist=min_d, 
            random_state=42, n_components=2
        )
        X_test = reducer_test.fit_transform(X)
        print(f"  n_neighbors={n_neigh:3d}, min_dist={min_d:.1f}: "
              f"spread=[{X_test[:,0].std():.2f}, {X_test[:,1].std():.2f}]")

# ============= UMAP CAN TRANSFORM NEW DATA =============
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

reducer_trainable = umap.UMAP(n_components=2, random_state=42)
X_train_umap = reducer_trainable.fit_transform(X_train)

# Transform test data using the same learned embedding
X_test_umap = reducer_trainable.transform(X_test)
print(f"\nTrain UMAP shape: {X_train_umap.shape}")
print(f"Test UMAP shape: {X_test_umap.shape}")
# This is NOT possible with t-SNE!

# ============= UMAP AS PREPROCESSING FOR CLASSIFICATION =============
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score

# Reduce to 10 dimensions (not just 2!)
reducer_10d = umap.UMAP(n_components=10, random_state=42)
X_train_10d = reducer_10d.fit_transform(X_train)
X_test_10d = reducer_10d.transform(X_test)

# Train classifier on UMAP features
rf = RandomForestClassifier(n_estimators=100, random_state=42)
rf.fit(X_train_10d, y_train)
print(f"\nRF on UMAP (10D): {accuracy_score(y_test, rf.predict(X_test_10d)):.4f}")

# Compare with RF on original features
rf_orig = RandomForestClassifier(n_estimators=100, random_state=42)
rf_orig.fit(X_train, y_train)
print(f"RF on original (64D): {accuracy_score(y_test, rf_orig.predict(X_test)):.4f}")
```

### UMAP with Different Metrics

```python
# UMAP supports many distance metrics — great for different data types!
metrics_to_try = ['euclidean', 'manhattan', 'cosine', 'correlation']

for metric in metrics_to_try:
    reducer_m = umap.UMAP(n_components=2, metric=metric, random_state=42)
    X_m = reducer_m.fit_transform(X)
    print(f"Metric={metric:<12}: range X=[{X_m[:,0].min():.1f}, {X_m[:,0].max():.1f}], "
          f"Y=[{X_m[:,1].min():.1f}, {X_m[:,1].max():.1f}]")

# Special: UMAP with supervised information (semi-supervised)
reducer_supervised = umap.UMAP(n_components=2, random_state=42)
X_supervised = reducer_supervised.fit_transform(X, y=y)  # Pass labels!
# This creates embeddings that respect class boundaries
print(f"\nSupervised UMAP produces better-separated clusters")
```

---

## LDA (Linear Discriminant Analysis) {#lda-linear-discriminant-analysis}

### What It Is

LDA is a **supervised** dimensionality reduction technique. Unlike PCA (which maximizes variance regardless of labels), LDA finds projections that **maximize class separation**.

### Intuition

```
PCA: Finds direction of maximum spread (variance)
     → Might project classes on top of each other!

LDA: Finds direction that best separates classes
     → Maximizes between-class distance, minimizes within-class spread

          PCA direction                    LDA direction
              ↗                               →
    ●●●●  ○○○○                       ●●●●        ○○○○
    ●●●●  ○○○○    Projected →        ●●○●○○●○     (mixed!)
    ●●●●  ○○○○                       
                                      ●●●●        ○○○○     (separated!)
                                          LDA direction →
```

### The Math

**Objective:** Maximize the ratio of between-class scatter to within-class scatter:

$$J(w) = \frac{w^T S_B w}{w^T S_W w}$$

**Within-class scatter matrix:**
$$S_W = \sum_{c=1}^{C} \sum_{x_i \in C_c} (x_i - \mu_c)(x_i - \mu_c)^T$$

**Between-class scatter matrix:**
$$S_B = \sum_{c=1}^{C} n_c (\mu_c - \mu)(\mu_c - \mu)^T$$

**Solution:** Eigenvectors of $S_W^{-1} S_B$ corresponding to largest eigenvalues.

**Key constraint:** LDA can produce at most $C-1$ components (where $C$ = number of classes).

### Code Example — LDA

```python
from sklearn.discriminant_analysis import LinearDiscriminantAnalysis
from sklearn.datasets import load_iris, load_wine
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score
from sklearn.preprocessing import StandardScaler
import numpy as np

# Load Wine dataset (13 features, 3 classes)
wine = load_wine()
X = wine.data
y = wine.target

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Scale features
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)

# ============= LDA FOR DIMENSIONALITY REDUCTION =============
# 3 classes → max 2 LDA components
lda = LinearDiscriminantAnalysis(n_components=2)
X_train_lda = lda.fit_transform(X_train_scaled, y_train)  # NOTE: needs labels!
X_test_lda = lda.transform(X_test_scaled)

print(f"Original dimensions: {X_train_scaled.shape[1]}")
print(f"LDA dimensions: {X_train_lda.shape[1]}")
print(f"Explained variance ratio: {lda.explained_variance_ratio_}")

# ============= LDA AS CLASSIFIER =============
# LDA can also be used directly as a classifier!
lda_clf = LinearDiscriminantAnalysis()
lda_clf.fit(X_train_scaled, y_train)
lda_pred = lda_clf.predict(X_test_scaled)
print(f"\nLDA as classifier accuracy: {accuracy_score(y_test, lda_pred):.4f}")

# ============= PCA vs LDA COMPARISON =============
from sklearn.decomposition import PCA
from sklearn.neighbors import KNeighborsClassifier

# Reduce to 2D with PCA
pca = PCA(n_components=2)
X_train_pca = pca.fit_transform(X_train_scaled)
X_test_pca = pca.transform(X_test_scaled)

# Reduce to 2D with LDA
lda_2d = LinearDiscriminantAnalysis(n_components=2)
X_train_lda2 = lda_2d.fit_transform(X_train_scaled, y_train)
X_test_lda2 = lda_2d.transform(X_test_scaled)

# Compare KNN accuracy on reduced features
for name, X_tr, X_te in [('PCA-2D', X_train_pca, X_test_pca),
                           ('LDA-2D', X_train_lda2, X_test_lda2),
                           ('Original-13D', X_train_scaled, X_test_scaled)]:
    knn = KNeighborsClassifier(n_neighbors=5)
    knn.fit(X_tr, y_train)
    acc = accuracy_score(y_test, knn.predict(X_te))
    print(f"  KNN on {name}: {acc:.4f}")
# LDA typically wins because it's optimized for class separation!
```

### PCA vs LDA — When to Use Which

| Aspect | PCA | LDA |
|--------|-----|-----|
| Type | Unsupervised | Supervised |
| Objective | Max variance | Max class separation |
| Labels needed? | No | Yes |
| Max components | min(n_samples, n_features) | n_classes - 1 |
| Best for | General reduction, visualization | Classification preprocessing |
| Assumption | Linear variance structure | Classes are Gaussian, equal covariance |

---

## Feature Selection Methods

### Overview

Unlike feature extraction (PCA, t-SNE), feature selection picks a **subset** of the original features. Advantages: interpretability, no information loss on selected features.

```
FEATURE SELECTION METHODS
├── FILTER METHODS (fast, independent of model)
│   ├── Variance Threshold
│   ├── Correlation Analysis  
│   ├── Mutual Information
│   ├── Chi-squared Test
│   └── ANOVA F-test
│
├── WRAPPER METHODS (use model to evaluate subsets)
│   ├── Forward Selection
│   ├── Backward Elimination
│   └── Recursive Feature Elimination (RFE)
│
└── EMBEDDED METHODS (built into model training)
    ├── L1 Regularization (Lasso)
    ├── Tree-based Feature Importance
    └── Permutation Importance
```

### Code Example — All Feature Selection Methods

```python
import numpy as np
import pandas as pd
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.ensemble import RandomForestClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import accuracy_score

# Generate dataset with informative and irrelevant features
X, y = make_classification(
    n_samples=1000, n_features=30,
    n_informative=10,        # 10 actually useful features
    n_redundant=5,           # 5 are linear combos of informative
    n_repeated=0,
    n_classes=2,
    random_state=42
)

feature_names = [f'feature_{i}' for i in range(30)]
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# ============= FILTER METHOD 1: Variance Threshold =============
from sklearn.feature_selection import VarianceThreshold

# Remove features with near-zero variance
selector_var = VarianceThreshold(threshold=0.1)
X_var = selector_var.fit_transform(X_train)
print(f"Variance Threshold: {X_train.shape[1]} → {X_var.shape[1]} features")

# ============= FILTER METHOD 2: Mutual Information =============
from sklearn.feature_selection import mutual_info_classif, SelectKBest

mi_scores = mutual_info_classif(X_train, y_train, random_state=42)
top_mi_features = np.argsort(mi_scores)[::-1][:10]
print(f"\nTop 10 features by Mutual Information: {top_mi_features}")

# Select top K features
selector_mi = SelectKBest(score_func=mutual_info_classif, k=10)
X_mi = selector_mi.fit_transform(X_train, y_train)
print(f"MI Selection: {X_train.shape[1]} → {X_mi.shape[1]} features")

# ============= FILTER METHOD 3: ANOVA F-test =============
from sklearn.feature_selection import f_classif

selector_anova = SelectKBest(score_func=f_classif, k=10)
X_anova = selector_anova.fit_transform(X_train, y_train)
f_scores = selector_anova.scores_
p_values = selector_anova.pvalues_
print(f"\nANOVA F-test: features with p < 0.01: "
      f"{np.sum(p_values < 0.01)} features")

# ============= FILTER METHOD 4: Correlation-based =============
# Remove highly correlated features (redundant)
def remove_correlated_features(X, threshold=0.9):
    """Remove features with correlation > threshold."""
    corr_matrix = np.abs(np.corrcoef(X.T))
    upper = np.triu(corr_matrix, k=1)
    to_drop = [i for i in range(upper.shape[1]) if any(upper[:, i] > threshold)]
    return to_drop

correlated_features = remove_correlated_features(X_train, threshold=0.9)
print(f"\nHighly correlated features to remove: {correlated_features}")

# ============= WRAPPER METHOD: RFE =============
from sklearn.feature_selection import RFE, RFECV

# Recursive Feature Elimination
rfe = RFE(
    estimator=RandomForestClassifier(n_estimators=50, random_state=42),
    n_features_to_select=10,
    step=1  # Remove 1 feature at a time
)
rfe.fit(X_train, y_train)
rfe_features = np.where(rfe.support_)[0]
print(f"\nRFE selected features: {rfe_features}")
print(f"RFE ranking: {rfe.ranking_}")

# RFE with Cross-Validation (auto-select optimal number of features)
rfecv = RFECV(
    estimator=RandomForestClassifier(n_estimators=50, random_state=42),
    step=1,
    cv=5,
    scoring='accuracy',
    min_features_to_select=5,
    n_jobs=-1
)
rfecv.fit(X_train, y_train)
print(f"\nRFECV optimal features: {rfecv.n_features_}")
print(f"RFECV CV score: {rfecv.cv_results_['mean_test_score'].max():.4f}")

# ============= EMBEDDED METHOD: L1 (Lasso) =============
from sklearn.feature_selection import SelectFromModel

# L1 regularization drives unimportant feature weights to exactly 0
lasso = LogisticRegression(penalty='l1', C=0.1, solver='saga', max_iter=5000, random_state=42)
lasso.fit(StandardScaler().fit_transform(X_train), y_train)

# Features with non-zero coefficients are selected
selected_l1 = np.where(lasso.coef_[0] != 0)[0]
print(f"\nL1 (Lasso) selected features: {len(selected_l1)} features")
print(f"Non-zero features: {selected_l1}")

# Using SelectFromModel
selector_l1 = SelectFromModel(lasso, prefit=True)
X_l1 = selector_l1.transform(X_train)
print(f"L1 Selection: {X_train.shape[1]} → {X_l1.shape[1]} features")

# ============= EMBEDDED METHOD: Tree-based Importance =============
rf = RandomForestClassifier(n_estimators=200, random_state=42)
rf.fit(X_train, y_train)

importances = rf.feature_importances_
top_tree_features = np.argsort(importances)[::-1][:10]
print(f"\nTree-based top 10 features: {top_tree_features}")

# Permutation importance (more reliable!)
from sklearn.inspection import permutation_importance

perm_imp = permutation_importance(rf, X_test, y_test, n_repeats=10, random_state=42)
top_perm_features = np.argsort(perm_imp.importances_mean)[::-1][:10]
print(f"Permutation importance top 10: {top_perm_features}")

# ============= COMPARE ALL METHODS =============
print("\n" + "="*60)
print("COMPARISON: Accuracy with selected features")
print("="*60)

methods = {
    'All 30 features': X_train,
    'MI (top 10)': selector_mi.transform(X_train),
    'ANOVA (top 10)': selector_anova.transform(X_train),
    'RFE (10)': X_train[:, rfe_features],
    'L1 selected': X_l1,
    'Tree top 10': X_train[:, top_tree_features],
}

for name, X_sel in methods.items():
    rf_eval = RandomForestClassifier(n_estimators=100, random_state=42)
    scores = cross_val_score(rf_eval, X_sel, y_train, cv=5, scoring='accuracy')
    print(f"  {name:<20}: {scores.mean():.4f} ± {scores.std():.4f}")
```

### Automated Feature Selection Pipeline

```python
from sklearn.pipeline import Pipeline
from sklearn.feature_selection import SelectKBest, f_classif
from sklearn.ensemble import GradientBoostingClassifier

# Production-ready pipeline with feature selection
pipeline = Pipeline([
    ('scaler', StandardScaler()),
    ('feature_selection', SelectKBest(score_func=f_classif, k=15)),
    ('classifier', GradientBoostingClassifier(n_estimators=100, random_state=42))
])

# Cross-validate the entire pipeline (proper evaluation!)
from sklearn.model_selection import cross_val_score
scores = cross_val_score(pipeline, X_train, y_train, cv=5, scoring='accuracy')
print(f"Pipeline with feature selection: {scores.mean():.4f} ± {scores.std():.4f}")
```

---

## Common Mistakes

### 1. Applying PCA Without Scaling

```python
# ❌ BAD: Features with large scale dominate PCA
pca_bad = PCA(n_components=2)
X_pca_bad = pca_bad.fit_transform(X)  # Raw data, unscaled

# ✅ GOOD: Always standardize before PCA
X_scaled = StandardScaler().fit_transform(X)
pca_good = PCA(n_components=2)
X_pca_good = pca_good.fit_transform(X_scaled)
```

### 2. Using t-SNE/UMAP Embeddings as ML Features

```python
# ❌ BAD: t-SNE features for training a model
tsne = TSNE(n_components=2)
X_tsne = tsne.fit_transform(X_train)
model.fit(X_tsne, y_train)  # Can't transform new data! Overfitting!

# ✅ GOOD: Use PCA or UMAP (which can transform new data)
umap_reducer = umap.UMAP(n_components=10)
X_train_umap = umap_reducer.fit_transform(X_train)
X_test_umap = umap_reducer.transform(X_test)  # ← This works!
model.fit(X_train_umap, y_train)
model.predict(X_test_umap)
```

### 3. Interpreting t-SNE Cluster Distances/Sizes

```python
# ❌ BAD: "Cluster A is bigger than Cluster B in t-SNE plot"
# ❌ BAD: "Cluster A is far from Cluster B, so they're very different"

# ✅ TRUTH: t-SNE cluster sizes and inter-cluster distances are NOT meaningful
# Only the local neighborhood structure (which points are near each other) matters
# Always try different perplexity values to verify structure
```

### 4. Feature Selection on Full Dataset (Data Leakage)

```python
# ❌ BAD: Feature selection on all data, then split
selector = SelectKBest(k=10)
X_selected = selector.fit_transform(X, y)  # Uses test data info!
X_train, X_test = train_test_split(X_selected, ...)

# ✅ GOOD: Feature selection only on training data
X_train, X_test, y_train, y_test = train_test_split(X, y, ...)
selector = SelectKBest(k=10)
X_train_selected = selector.fit_transform(X_train, y_train)
X_test_selected = selector.transform(X_test)  # Only transform!
```

### 5. Keeping Too Many / Too Few Components

```python
# ❌ BAD: Arbitrary number of components
pca = PCA(n_components=5)  # Why 5? No justification

# ✅ GOOD: Use explained variance to decide
pca = PCA(n_components=0.95)  # Keep 95% of variance
# Or use cross-validation with downstream task
```

### 6. Forgetting That LDA is Limited to C-1 Components

```python
# ❌ BAD: Trying to get 10 LDA components with 3 classes
lda = LinearDiscriminantAnalysis(n_components=10)  # Error! Max is 2 (3 classes - 1)

# ✅ GOOD: 
lda = LinearDiscriminantAnalysis(n_components=2)  # Max for 3 classes
```

---

## Interview Questions

### Conceptual Questions

**Q1: What's the difference between PCA and LDA?**
> PCA is unsupervised — it finds directions of maximum variance regardless of labels. LDA is supervised — it finds directions that maximize separation between classes. PCA can extract up to $d$ components, LDA is limited to $C-1$ where $C$ is the number of classes.

**Q2: Why does PCA require features to be standardized?**
> PCA maximizes variance. If one feature has a much larger scale (e.g., salary in thousands vs. age in decades), it will dominate the first PC regardless of its actual importance. Standardization ensures all features contribute equally.

**Q3: Can you apply PCA to categorical data?**
> No, standard PCA requires continuous numerical data. For categorical data, use Multiple Correspondence Analysis (MCA) or encode categoricals first. For mixed data, use Factor Analysis of Mixed Data (FAMD).

**Q4: Explain the relationship between PCA and SVD.**
> PCA uses eigendecomposition of the covariance matrix. SVD decomposes the data matrix directly: $X = U\Sigma V^T$. The right singular vectors ($V$) are the principal components, and $\sigma_i^2 / (n-1)$ are the eigenvalues. Sklearn's PCA uses SVD internally (more numerically stable).

**Q5: Why does t-SNE use Student's t-distribution in low-D?**
> The "crowding problem": when projecting many points from high-D to low-D, there isn't enough room to maintain all distances. The t-distribution's heavy tails create more space for moderately distant points to spread out in low-D, preventing clusters from collapsing.

**Q6: What are the limitations of t-SNE?**
> (1) Non-parametric — can't transform new data, (2) Stochastic — different runs give different results, (3) Cluster sizes and inter-cluster distances are meaningless, (4) Slow ($O(n^2)$), (5) Results depend heavily on perplexity, (6) Only for visualization, not for downstream ML.

**Q7: When would you use UMAP over t-SNE?**
> When you need: (1) faster computation, (2) ability to transform new data, (3) preservation of global structure, (4) reduction to >3 dimensions for ML, (5) handling larger datasets.

**Q8: How do you decide between feature selection and feature extraction?**
> Use feature **selection** when: interpretability matters, you need to explain which features are important, features have physical meaning. Use feature **extraction** when: you want maximum information compression, features are highly correlated, visualization is the goal.

**Q9: What is the curse of dimensionality and how does it affect ML?**
> In high dimensions: (1) all points become equidistant (distance metrics fail), (2) data becomes sparse (need exponentially more samples), (3) models overfit easily (too many parameters). It affects KNN, K-Means, and any distance-based method the most.

**Q10: How do you validate that dimensionality reduction helped?**
> Compare downstream task performance (accuracy, AUC) with and without reduction. Also check: (1) training time improvement, (2) overfitting reduction (train-test gap), (3) reconstruction error (for PCA), (4) trustworthiness/continuity metrics (for embeddings).

---

## Quick Reference

### Algorithm Comparison Table

| Method | Type | Linear? | Supervised? | Preserves | Best For |
|--------|------|---------|-------------|-----------|----------|
| PCA | Extraction | Yes | No | Global variance | General reduction, denoising |
| Kernel PCA | Extraction | No | No | Non-linear variance | Non-linear manifolds |
| t-SNE | Extraction | No | No | Local structure | 2D/3D visualization |
| UMAP | Extraction | No | No (or semi) | Local + global | Visualization + ML |
| LDA | Extraction | Yes | Yes | Class separation | Classification preprocessing |
| Variance Filter | Selection | N/A | No | N/A | Remove constant features |
| Mutual Info | Selection | N/A | Yes | N/A | Non-linear feature ranking |
| RFE | Selection | N/A | Yes | N/A | Optimal feature subset |
| L1/Lasso | Selection | N/A | Yes | N/A | Automatic feature selection |

### Decision Flowchart

```
Need dimensionality reduction?
│
├── Goal: VISUALIZATION (2D/3D)?
│   ├── Dataset < 5000 points → t-SNE
│   └── Dataset > 5000 points → UMAP
│
├── Goal: PREPROCESSING for ML?
│   ├── Have labels?
│   │   ├── Yes → LDA (if C is small) or Supervised UMAP
│   │   └── No → PCA (linear) or UMAP (non-linear)
│   ├── Need interpretability?
│   │   ├── Yes → Feature Selection (MI, RFE, L1)
│   │   └── No → PCA or UMAP
│   └── Data is linear?
│       ├── Yes → PCA
│       └── No → Kernel PCA or UMAP
│
└── Goal: NOISE REMOVAL?
    └── PCA (keep top K components, discard rest)
```

### Key Parameters

| Algorithm | Key Parameter | How to Choose |
|-----------|---------------|---------------|
| PCA | `n_components` | 95% explained variance, or scree plot elbow |
| t-SNE | `perplexity` | Try 5, 15, 30, 50; typical: 30 |
| UMAP | `n_neighbors` | 5-50; larger = more global; typical: 15 |
| UMAP | `min_dist` | 0-1; smaller = tighter clusters; typical: 0.1 |
| LDA | `n_components` | Max is n_classes - 1 |
| SelectKBest | `k` | Cross-validate or use RFECV |

### Key Takeaways

- **Always scale** before PCA/t-SNE (not needed for tree-based feature importance)
- **PCA** is your go-to for linear dimensionality reduction and noise removal
- **t-SNE** is visualization-only; never use its output as ML features
- **UMAP** is the modern choice — faster, more versatile, can transform new data
- **LDA** is powerful when you have labels and want class-aware reduction
- **Feature selection** preserves interpretability; **extraction** preserves information
- **Combine methods:** PCA for preprocessing → t-SNE/UMAP for visualization
- **Validate** dimensionality reduction with downstream task performance
- The **curse of dimensionality** is real — always question if you need all features

---

*Previous: [08-Clustering-Algorithms.md](08-Clustering-Algorithms.md) | Next: [10-Model-Evaluation-and-Tuning.md](10-Model-Evaluation-and-Tuning.md)*
