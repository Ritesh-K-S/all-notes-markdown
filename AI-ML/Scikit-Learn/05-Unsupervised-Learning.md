# Chapter 05: Unsupervised Learning in Scikit-Learn

## Table of Contents
- [Introduction to Unsupervised Learning](#introduction-to-unsupervised-learning)
- [K-Means Clustering](#k-means-clustering)
- [DBSCAN — Density-Based Clustering](#dbscan--density-based-clustering)
- [Agglomerative (Hierarchical) Clustering](#agglomerative-hierarchical-clustering)
- [PCA — Principal Component Analysis](#pca--principal-component-analysis)
- [t-SNE — Visualization of High-Dimensional Data](#t-sne--visualization-of-high-dimensional-data)
- [Comparison of Clustering Algorithms](#comparison-of-clustering-algorithms)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Introduction to Unsupervised Learning

### What It Is
Supervised learning is like studying with an answer key — you know the right answers and learn to match them. **Unsupervised learning** is like sorting a pile of 1000 photos with no labels — you have to discover the natural groupings yourself.

There are no labels, no "right answers." The algorithm finds hidden patterns, structures, or groupings in the data on its own.

### Why It Matters
- **Customer segmentation** — Group customers by behavior for targeted marketing
- **Anomaly detection** — Find fraudulent transactions or faulty equipment
- **Dimensionality reduction** — Compress 1000 features into 10 meaningful ones
- **Data exploration** — Understand structure before building supervised models
- **Feature engineering** — Create cluster IDs as features for supervised models

### Two Main Types

| Type | Goal | Example Algorithms |
|------|------|-------------------|
| **Clustering** | Group similar data points together | K-Means, DBSCAN, Agglomerative |
| **Dimensionality Reduction** | Reduce features while preserving information | PCA, t-SNE, UMAP |

---

## K-Means Clustering

### What It Is
K-Means divides your data into **K groups** (clusters) where each data point belongs to the cluster with the nearest center (centroid). Think of it like placing K magnets on a table covered in iron filings — each filing moves to the nearest magnet.

### Why It Matters
- Simplest and most widely used clustering algorithm
- Scales well to large datasets (linear complexity)
- Used in customer segmentation, image compression, document grouping
- Often the first algorithm to try for clustering

### How It Works

```
┌──────────────────────────────────────────────────────────────┐
│                    K-MEANS ALGORITHM                          │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: Choose K (number of clusters) and randomly place     │
│          K centroids in the feature space                     │
│                                                                │
│     ·  ·     ·                                                │
│   ·   ★  ·    ·   ★ = random centroids (K=2)                │
│     ·    ·  ·   ★                                            │
│       ·   ·  ·                                                │
│                                                                │
│  Step 2: ASSIGN each point to the nearest centroid            │
│                                                                │
│     A  A     B         A = Cluster 1, B = Cluster 2          │
│   A   ★  A    B                                              │
│     A    B  B   ★                                            │
│       A   B  B                                                │
│                                                                │
│  Step 3: UPDATE centroids to the mean of assigned points     │
│                                                                │
│     A  A     B                                                │
│   A  ★   A    B        ★ centroids move to cluster means     │
│     A    B  B  ★                                             │
│       A   B  B                                                │
│                                                                │
│  Step 4: REPEAT Steps 2-3 until centroids stop moving        │
│          (or max iterations reached)                          │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Mathematical Foundation

**Objective:** Minimize Within-Cluster Sum of Squares (WCSS / Inertia):

$$J = \sum_{k=1}^{K} \sum_{x_i \in C_k} \|x_i - \mu_k\|^2$$

Where:
- $K$ = number of clusters
- $C_k$ = set of points in cluster $k$
- $\mu_k$ = centroid (mean) of cluster $k$
- $\|x_i - \mu_k\|^2$ = squared Euclidean distance

**Update rule for centroids:**
$$\mu_k = \frac{1}{|C_k|} \sum_{x_i \in C_k} x_i$$

### Code Examples

#### Basic K-Means

```python
import numpy as np
import matplotlib.pyplot as plt
from sklearn.cluster import KMeans
from sklearn.datasets import make_blobs

# Generate synthetic data with 4 natural clusters
X, y_true = make_blobs(
    n_samples=500, 
    centers=4,          # 4 true clusters
    cluster_std=0.8,    # How spread out each cluster is
    random_state=42
)

# Apply K-Means
kmeans = KMeans(
    n_clusters=4,       # We choose K=4 (must know or estimate this!)
    init='k-means++',   # Smart initialization (much better than random)
    n_init=10,          # Run 10 times with different seeds, pick best
    max_iter=300,       # Maximum iterations per run
    tol=1e-4,           # Convergence tolerance (stop if centroids move less)
    random_state=42
)

# Fit and predict cluster labels
labels = kmeans.fit_predict(X)

# Results
print(f"Cluster centers shape: {kmeans.cluster_centers_.shape}")  # (4, 2)
print(f"Inertia (WCSS): {kmeans.inertia_:.2f}")
print(f"Number of iterations: {kmeans.n_iter_}")
print(f"Labels: {np.unique(labels)}")  # [0, 1, 2, 3]

# Visualize
plt.figure(figsize=(10, 6))
plt.scatter(X[:, 0], X[:, 1], c=labels, cmap='viridis', s=30, alpha=0.6)
plt.scatter(kmeans.cluster_centers_[:, 0], kmeans.cluster_centers_[:, 1], 
            c='red', marker='X', s=200, edgecolors='black', linewidths=2,
            label='Centroids')
plt.title('K-Means Clustering (K=4)')
plt.legend()
plt.tight_layout()
plt.show()
```

#### Finding the Optimal K — The Elbow Method

```python
# The ELBOW METHOD: plot inertia vs K, look for the "elbow"
inertias = []
K_range = range(1, 11)

for k in K_range:
    km = KMeans(n_clusters=k, random_state=42, n_init=10)
    km.fit(X)
    inertias.append(km.inertia_)

# Plot
plt.figure(figsize=(8, 5))
plt.plot(K_range, inertias, 'bo-', linewidth=2)
plt.xlabel('Number of Clusters (K)')
plt.ylabel('Inertia (WCSS)')
plt.title('Elbow Method — Finding Optimal K')
plt.xticks(K_range)
plt.grid(True)
plt.tight_layout()
plt.show()

# Look for the "elbow" — the point where adding more clusters
# gives diminishing returns in reducing inertia
```

#### Silhouette Score — Better Cluster Evaluation

```python
from sklearn.metrics import silhouette_score, silhouette_samples

# Silhouette Score: measures how similar a point is to its own cluster
# vs neighboring clusters. Range: [-1, 1], higher = better
silhouette_scores = []

for k in range(2, 11):  # silhouette needs at least 2 clusters
    km = KMeans(n_clusters=k, random_state=42, n_init=10)
    labels = km.fit_predict(X)
    score = silhouette_score(X, labels)
    silhouette_scores.append(score)
    print(f"K={k}: Silhouette Score = {score:.4f}")

# Output:
# K=2: Silhouette Score = 0.5816
# K=3: Silhouette Score = 0.5598
# K=4: Silhouette Score = 0.6825  ← Best! (matches true K)
# K=5: Silhouette Score = 0.5432

optimal_k = range(2, 11)[np.argmax(silhouette_scores)]
print(f"\nOptimal K by Silhouette: {optimal_k}")
```

#### Silhouette Score Formula

For a single data point $i$:

$$s(i) = \frac{b(i) - a(i)}{\max(a(i), b(i))}$$

Where:
- $a(i)$ = mean distance to other points in **same** cluster (cohesion)
- $b(i)$ = mean distance to points in **nearest other** cluster (separation)
- $s(i) \approx 1$: point is well-clustered
- $s(i) \approx 0$: point is on the boundary
- $s(i) \approx -1$: point is in the wrong cluster

#### K-Means for Image Compression

```python
from sklearn.datasets import load_sample_image

# Load a sample image
china = load_sample_image("china.jpg")
print(f"Original image shape: {china.shape}")  # (427, 640, 3)

# Reshape to (n_pixels, 3) — each pixel is an RGB triplet
X_img = china.reshape(-1, 3).astype(np.float64) / 255.0

# Compress: reduce 16 million colors to just 16 using K-Means
n_colors = 16
kmeans_img = KMeans(n_clusters=n_colors, random_state=42, n_init=5)
labels_img = kmeans_img.fit_predict(X_img)

# Replace each pixel with its cluster centroid color
compressed = kmeans_img.cluster_centers_[labels_img]
compressed_image = compressed.reshape(china.shape)

# Compare
fig, axes = plt.subplots(1, 2, figsize=(14, 6))
axes[0].imshow(china)
axes[0].set_title(f"Original (16M colors)")
axes[1].imshow(compressed_image)
axes[1].set_title(f"Compressed ({n_colors} colors)")
for ax in axes:
    ax.axis('off')
plt.tight_layout()
plt.show()
```

### K-Means++ Initialization

Standard K-Means picks random centroids — sometimes badly. **K-Means++** picks centroids that are spread out:

1. Pick first centroid randomly
2. For each remaining point, compute distance to nearest centroid
3. Pick next centroid with probability proportional to distance²
4. Repeat until K centroids chosen

```python
# Sklearn uses k-means++ by default!
kmeans = KMeans(n_clusters=4, init='k-means++')  # default

# You can also provide custom initial centroids
custom_centers = np.array([[0, 0], [5, 5], [10, 0], [5, -5]])
kmeans_custom = KMeans(n_clusters=4, init=custom_centers, n_init=1)
```

### MiniBatchKMeans — For Large Datasets

```python
from sklearn.cluster import MiniBatchKMeans

# Regular KMeans: O(n * K * iter * d) — slow for large n
# MiniBatchKMeans: uses random batches → 3-10x faster, nearly as good

mbk = MiniBatchKMeans(
    n_clusters=4,
    batch_size=100,          # Process 100 samples at a time
    max_iter=100,
    random_state=42
)

# For very large datasets
labels = mbk.fit_predict(X)
print(f"MiniBatch Inertia: {mbk.inertia_:.2f}")
```

> **Pro Tip:** Use MiniBatchKMeans when `n_samples > 10,000`. The quality loss is typically <1% but the speed gain is massive.

---

## DBSCAN — Density-Based Clustering

### What It Is
DBSCAN (Density-Based Spatial Clustering of Applications with Noise) finds clusters based on **density** — groups of points that are closely packed together. Points in low-density regions are classified as noise/outliers.

Analogy: Imagine looking at a city at night from an airplane. Dense clusters of lights are cities, scattered lights are villages, and dark areas have no people. DBSCAN finds the "cities" automatically.

### Why It Matters
- **No need to specify K** (number of clusters) — discovers it automatically
- **Finds arbitrarily shaped clusters** (not just spherical like K-Means)
- **Built-in outlier detection** — labels noise points as -1
- Perfect for spatial data, anomaly detection, and irregular cluster shapes

### How It Works

Two key parameters:
- **eps (ε):** The radius of the neighborhood around each point
- **min_samples:** Minimum points within ε to form a dense region

Three types of points:
```
┌──────────────────────────────────────────────────────┐
│              DBSCAN POINT CLASSIFICATION              │
├──────────────────────────────────────────────────────┤
│                                                        │
│  CORE POINT: Has ≥ min_samples points within eps      │
│                                                        │
│        · ·                                            │
│       · ★ ·    ★ = Core point (5 neighbors, min=5)   │
│        · ·                                            │
│                                                        │
│  BORDER POINT: Within eps of a core point, but        │
│  doesn't have min_samples neighbors itself            │
│                                                        │
│        · ·                                            │
│       · ★ · ○   ○ = Border point (near core ★)       │
│        · ·                                            │
│                                                        │
│  NOISE POINT: Not a core or border point              │
│                                                        │
│        · ·                                            │
│       · ★ ·         ×   × = Noise/outlier            │
│        · ·                                            │
│                                                        │
│  CLUSTER FORMATION:                                   │
│  Core points within eps of each other are connected   │
│  → Forms a cluster chain                              │
│                                                        │
│     ★─★─★─★     ★─★─★                               │
│      \         /                                      │
│       ★─★─★─★                                        │
│     [Cluster 1]    [Cluster 2]                        │
│                          × × (Noise)                  │
│                                                        │
└──────────────────────────────────────────────────────┘
```

### Code Examples

#### Basic DBSCAN

```python
from sklearn.cluster import DBSCAN
from sklearn.datasets import make_moons, make_circles
from sklearn.preprocessing import StandardScaler

# Generate non-spherical data (K-Means fails on this!)
X_moons, y_moons = make_moons(n_samples=500, noise=0.1, random_state=42)

# IMPORTANT: Scale data first! DBSCAN uses distance, so scale matters
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_moons)

# Apply DBSCAN
dbscan = DBSCAN(
    eps=0.3,             # Neighborhood radius (THE most important parameter)
    min_samples=5,       # Minimum points to form a dense region
    metric='euclidean',  # Distance metric
    n_jobs=-1
)

labels = dbscan.fit_predict(X_scaled)

# Analyze results
n_clusters = len(set(labels)) - (1 if -1 in labels else 0)
n_noise = list(labels).count(-1)
print(f"Clusters found: {n_clusters}")
print(f"Noise points: {n_noise}")
print(f"Unique labels: {np.unique(labels)}")

# Output:
# Clusters found: 2
# Noise points: 3
# Unique labels: [-1  0  1]  (-1 = noise)

# Visualize
plt.figure(figsize=(10, 6))
plt.scatter(X_moons[labels >= 0, 0], X_moons[labels >= 0, 1], 
            c=labels[labels >= 0], cmap='viridis', s=30, alpha=0.6, label='Clustered')
plt.scatter(X_moons[labels == -1, 0], X_moons[labels == -1, 1],
            c='red', marker='x', s=50, label='Noise')
plt.title('DBSCAN on Moon-Shaped Data')
plt.legend()
plt.tight_layout()
plt.show()
```

#### Finding Optimal eps — The k-Distance Plot

```python
from sklearn.neighbors import NearestNeighbors

# The k-distance plot helps you choose eps
# k = min_samples (typically)

k = 5  # Same as min_samples
nn = NearestNeighbors(n_neighbors=k)
nn.fit(X_scaled)
distances, _ = nn.kneighbors(X_scaled)

# Sort the k-th nearest neighbor distances
k_distances = np.sort(distances[:, k-1])

# Plot — look for the "knee" (sharp change in slope)
plt.figure(figsize=(8, 5))
plt.plot(k_distances)
plt.xlabel('Points (sorted by distance)')
plt.ylabel(f'{k}-th Nearest Neighbor Distance')
plt.title('k-Distance Plot for Choosing eps')
plt.grid(True)
plt.tight_layout()
plt.show()

# The "knee" in the plot is your optimal eps value
```

#### DBSCAN vs K-Means on Complex Shapes

```python
from sklearn.cluster import KMeans, DBSCAN

# Create concentric circles — K-Means can't handle this!
X_circles, y_circles = make_circles(n_samples=500, noise=0.05, factor=0.5, random_state=42)
X_circles_scaled = StandardScaler().fit_transform(X_circles)

fig, axes = plt.subplots(1, 2, figsize=(14, 6))

# K-Means (FAILS on non-convex shapes)
km = KMeans(n_clusters=2, random_state=42)
km_labels = km.fit_predict(X_circles_scaled)
axes[0].scatter(X_circles[:, 0], X_circles[:, 1], c=km_labels, cmap='viridis', s=30)
axes[0].set_title('K-Means (FAILS on circles)')

# DBSCAN (SUCCEEDS on non-convex shapes)
db = DBSCAN(eps=0.3, min_samples=5)
db_labels = db.fit_predict(X_circles_scaled)
axes[1].scatter(X_circles[:, 0], X_circles[:, 1], c=db_labels, cmap='viridis', s=30)
axes[1].set_title('DBSCAN (Handles circles perfectly)')

plt.tight_layout()
plt.show()
```

### When to Choose eps and min_samples

| Parameter | Rule of Thumb | Effect |
|-----------|--------------|--------|
| `eps` too small | Many noise points, fragmented clusters | Under-clustering |
| `eps` too large | Everything merges into one cluster | Over-clustering |
| `min_samples` too small | Noise gets included in clusters | Over-sensitive |
| `min_samples` too large | Small clusters become noise | Under-sensitive |

> **Rule of thumb for min_samples:** Use `min_samples = 2 × n_features` as a starting point. For 2D data, `min_samples=5` is typical.

---

## Agglomerative (Hierarchical) Clustering

### What It Is
Agglomerative clustering works **bottom-up**: start with every point as its own cluster, then repeatedly merge the two closest clusters until you have the desired number.

Think of it like a social network forming: first individuals, then pairs of friends, then friend groups, then larger communities — all visualized in a **dendrogram** (tree diagram).

### Why It Matters
- Produces a **dendrogram** — visual hierarchy of cluster merges
- No need to specify K upfront (cut the dendrogram at any level)
- Works with any distance metric
- Good for small-to-medium datasets with hierarchical structure

### How It Works

```
┌──────────────────────────────────────────────────────┐
│         AGGLOMERATIVE CLUSTERING PROCESS              │
├──────────────────────────────────────────────────────┤
│                                                        │
│  Step 1: Each point is its own cluster                │
│  {A} {B} {C} {D} {E}                                 │
│                                                        │
│  Step 2: Merge two closest clusters                   │
│  {A,B} {C} {D} {E}                                   │
│                                                        │
│  Step 3: Merge next closest                           │
│  {A,B} {C} {D,E}                                     │
│                                                        │
│  Step 4: Merge again                                  │
│  {A,B} {C,D,E}                                       │
│                                                        │
│  Step 5: Final merge                                  │
│  {A,B,C,D,E}                                         │
│                                                        │
│  DENDROGRAM (tree view):                              │
│                                                        │
│  Height │          ┌─────┐                            │
│  (dist) │     ┌────┤     │                            │
│    5    │     │    │     │                            │
│    4    │  ┌──┤    │  ┌──┤                            │
│    3    │  │  │    │  │  │                            │
│    2    │  │  │    │  │  │                            │
│    1    │──┤  │    │──┤  │                            │
│         │  A  B    C  D  E                            │
│         └──────────────────                           │
│                                                        │
│  Cut at height 3 → 2 clusters: {A,B} and {C,D,E}    │
│                                                        │
└──────────────────────────────────────────────────────┘
```

### Linkage Methods (How to Measure Cluster Distance)

| Linkage | Distance Between Clusters | Best For |
|---------|--------------------------|----------|
| **Ward** | Minimizes variance increase when merging | Compact, spherical clusters (default) |
| **Complete** | Maximum distance between points in clusters | Avoiding elongated clusters |
| **Average** | Average distance between all point pairs | Balanced approach |
| **Single** | Minimum distance between points in clusters | Irregular/chain-like shapes |

$$d_{\text{single}}(A, B) = \min_{a \in A, b \in B} \|a - b\|$$
$$d_{\text{complete}}(A, B) = \max_{a \in A, b \in B} \|a - b\|$$
$$d_{\text{average}}(A, B) = \frac{1}{|A||B|} \sum_{a \in A} \sum_{b \in B} \|a - b\|$$

### Code Examples

#### Basic Agglomerative Clustering

```python
from sklearn.cluster import AgglomerativeClustering

# Generate blob data
X, y_true = make_blobs(n_samples=300, centers=4, cluster_std=0.8, random_state=42)

# Agglomerative Clustering
agg = AgglomerativeClustering(
    n_clusters=4,           # Number of clusters to find
    linkage='ward',         # Minimizes variance (most common)
    metric='euclidean'      # Distance metric (ward requires euclidean)
)

labels = agg.fit_predict(X)
print(f"Cluster labels: {np.unique(labels)}")  # [0, 1, 2, 3]

# Visualize
plt.figure(figsize=(8, 6))
plt.scatter(X[:, 0], X[:, 1], c=labels, cmap='viridis', s=30)
plt.title('Agglomerative Clustering (Ward Linkage)')
plt.tight_layout()
plt.show()
```

#### Dendrogram Visualization

```python
from scipy.cluster.hierarchy import dendrogram, linkage

# Compute linkage matrix (scipy, not sklearn)
Z = linkage(X, method='ward')

# Plot dendrogram
plt.figure(figsize=(14, 7))
dendrogram(
    Z,
    truncate_mode='lastp',  # Show only last p merged clusters
    p=20,                   # Show 20 clusters
    leaf_rotation=90,
    leaf_font_size=10,
    show_leaf_counts=True,
    color_threshold=20      # Color clusters below this distance
)
plt.xlabel('Cluster Size')
plt.ylabel('Distance (Ward)')
plt.title('Dendrogram — Cut horizontally to choose number of clusters')
plt.tight_layout()
plt.show()
```

#### Choosing Number of Clusters from Dendrogram

```python
# Find optimal cut by looking at the largest "gap" in merge distances
# The biggest vertical gap in the dendrogram suggests the optimal K

from scipy.cluster.hierarchy import fcluster

# Cut the dendrogram at a specific distance threshold
labels_3 = fcluster(Z, t=3, criterion='maxclust')   # Force 3 clusters
labels_dist = fcluster(Z, t=15, criterion='distance') # Cut at distance=15

print(f"Clusters with maxclust=3: {np.unique(labels_3)}")
print(f"Clusters with distance=15: {len(np.unique(labels_dist))}")
```

> **Pro Tip:** Use the dendrogram to choose K, then use `AgglomerativeClustering(n_clusters=K)` for the actual labels. The dendrogram gives you visual intuition that K-Means doesn't.

---

## PCA — Principal Component Analysis

### What It Is
PCA finds the directions (axes) along which your data varies the most, then projects the data onto those axes. It's like looking at a 3D object and finding the best 2D "shadow" that preserves the most information.

Analogy: Imagine you have a pancake-shaped cloud of 3D points. PCA finds that the "flat" direction carries almost no information, so you can safely project onto the 2 "wide" directions without losing much.

### Why It Matters
- **Dimensionality reduction** — 1000 features → 50 features (speeds up ML models)
- **Visualization** — Project high-dimensional data to 2D/3D for plotting
- **Noise reduction** — Removing low-variance components removes noise
- **Multicollinearity** — PCA components are uncorrelated (fixes collinearity issues)
- **Preprocessing** — Often used before clustering or classification

### How It Works

```
┌──────────────────────────────────────────────────────────────┐
│                      PCA ALGORITHM                            │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Step 1: CENTER the data (subtract mean)                      │
│                                                                │
│  Step 2: Compute COVARIANCE MATRIX                            │
│          Σ = (1/n) · Xᵀ · X                                  │
│                                                                │
│  Step 3: Find EIGENVECTORS and EIGENVALUES of Σ              │
│          Eigenvectors = principal component directions        │
│          Eigenvalues = variance explained by each PC          │
│                                                                │
│  Step 4: Sort by eigenvalue (highest first)                   │
│          PC1 has the MOST variance, PC2 second most, etc.    │
│                                                                │
│  Step 5: Project data onto top k principal components         │
│                                                                │
│  GEOMETRIC INTUITION:                                         │
│                                                                │
│     Original Data            After PCA                        │
│     (correlated)             (uncorrelated)                   │
│                                                                │
│        ·  ·  ·                   ·   ·                        │
│      ·  ·  ·  ·               · · · · ·                      │
│    ·  ·  ·  ·  ·    →         ·   ·                          │
│      ·  ·  ·  ·                                              │
│        ·  ·  ·               PC1 →→→→→→                      │
│     ↗ Direction               (captures most variance)       │
│    of max variance                                            │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### Mathematical Foundation

Given centered data matrix $X \in \mathbb{R}^{n \times d}$:

1. **Covariance matrix:**
$$\Sigma = \frac{1}{n-1} X^T X$$

2. **Eigen decomposition:**
$$\Sigma v_k = \lambda_k v_k$$

Where $v_k$ = eigenvector (principal component direction), $\lambda_k$ = eigenvalue (variance captured).

3. **Projection onto top $k$ components:**
$$Z = X \cdot W_k$$

Where $W_k = [v_1, v_2, \ldots, v_k]$ is the matrix of top $k$ eigenvectors.

4. **Variance explained by component $k$:**
$$\text{Explained Variance Ratio}_k = \frac{\lambda_k}{\sum_{i=1}^{d} \lambda_i}$$

### Code Examples

#### Basic PCA

```python
from sklearn.decomposition import PCA
from sklearn.datasets import load_iris
from sklearn.preprocessing import StandardScaler

# Load data
iris = load_iris()
X = iris.data          # 4 features: sepal/petal length/width
y = iris.target        # 3 species
feature_names = iris.feature_names

# IMPORTANT: Always scale before PCA! (PCA is sensitive to feature scales)
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# Apply PCA — reduce from 4D to 2D
pca = PCA(n_components=2)
X_pca = pca.fit_transform(X_scaled)

print(f"Original shape: {X_scaled.shape}")      # (150, 4)
print(f"Reduced shape: {X_pca.shape}")           # (150, 2)
print(f"Explained variance ratio: {pca.explained_variance_ratio_}")
print(f"Total variance explained: {pca.explained_variance_ratio_.sum():.4f}")

# Output:
# Explained variance ratio: [0.7296 0.2285]
# Total variance explained: 0.9581 → 95.81% of info preserved with just 2 components!

# Visualize
plt.figure(figsize=(10, 7))
for i, name in enumerate(iris.target_names):
    mask = y == i
    plt.scatter(X_pca[mask, 0], X_pca[mask, 1], label=name, s=50, alpha=0.7)
plt.xlabel(f'PC1 ({pca.explained_variance_ratio_[0]:.1%} variance)')
plt.ylabel(f'PC2 ({pca.explained_variance_ratio_[1]:.1%} variance)')
plt.title('PCA — Iris Dataset (4D → 2D)')
plt.legend()
plt.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

#### Choosing Number of Components — Cumulative Variance

```python
# Fit PCA with ALL components to see variance distribution
pca_full = PCA().fit(X_scaled)

# Plot cumulative explained variance
cumulative_variance = np.cumsum(pca_full.explained_variance_ratio_)

plt.figure(figsize=(8, 5))
plt.bar(range(1, len(cumulative_variance) + 1), 
        pca_full.explained_variance_ratio_, alpha=0.6, label='Individual')
plt.step(range(1, len(cumulative_variance) + 1), 
         cumulative_variance, where='mid', color='red', linewidth=2, label='Cumulative')
plt.axhline(y=0.95, color='gray', linestyle='--', label='95% threshold')
plt.xlabel('Principal Component')
plt.ylabel('Explained Variance Ratio')
plt.title('Scree Plot — How Many Components to Keep?')
plt.legend()
plt.xticks(range(1, len(cumulative_variance) + 1))
plt.tight_layout()
plt.show()

# RULE: Keep enough components to explain ≥ 95% variance
n_components_95 = np.argmax(cumulative_variance >= 0.95) + 1
print(f"Components needed for 95% variance: {n_components_95}")
```

#### Using PCA with a Variance Threshold

```python
# Let sklearn choose n_components to explain 95% variance
pca_auto = PCA(n_components=0.95)  # Pass a float between 0 and 1
X_reduced = pca_auto.fit_transform(X_scaled)
print(f"Components chosen: {pca_auto.n_components_}")  # Automatically chosen
print(f"Variance preserved: {pca_auto.explained_variance_ratio_.sum():.4f}")
```

#### PCA Component Loadings (What Each PC Means)

```python
# Loadings: how much each original feature contributes to each PC
loadings = pd.DataFrame(
    pca.components_.T,
    columns=['PC1', 'PC2'],
    index=feature_names
)
print("Feature Loadings:")
print(loadings.round(4))

# Output:
#                    PC1     PC2
# sepal length (cm)  0.5224 -0.3723
# sepal width (cm)  -0.2634 -0.9256
# petal length (cm)  0.5813 -0.0211
# petal width (cm)   0.5656 -0.0654

# Interpretation: PC1 is mainly petal length + petal width (flower size)
#                 PC2 is mainly sepal width (sepal shape)
```

#### PCA for Preprocessing in ML Pipeline

```python
from sklearn.pipeline import Pipeline
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import cross_val_score

# PCA as a preprocessing step — reduces features before classification
pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('pca', PCA(n_components=0.95)),    # Keep 95% variance
    ('classifier', LogisticRegression(max_iter=1000))
])

# Cross-validate the entire pipeline
scores = cross_val_score(pipe, X, y, cv=5, scoring='accuracy')
print(f"Accuracy with PCA: {scores.mean():.4f} ± {scores.std():.4f}")
```

#### Inverse Transform — Reconstructing Data

```python
# PCA can reconstruct (approximately) the original data
pca_2 = PCA(n_components=2)
X_reduced = pca_2.fit_transform(X_scaled)
X_reconstructed = pca_2.inverse_transform(X_reduced)

# Reconstruction error tells you how much info was lost
reconstruction_error = np.mean((X_scaled - X_reconstructed) ** 2)
print(f"Reconstruction Error (MSE): {reconstruction_error:.4f}")
```

> **Pro Tip:** PCA is a linear method. For non-linear dimensionality reduction (e.g., "unrolling" a Swiss roll), use Kernel PCA or t-SNE.

---

## t-SNE — Visualization of High-Dimensional Data

### What It Is
t-SNE (t-distributed Stochastic Neighbor Embedding) is a **non-linear** dimensionality reduction technique designed specifically for **visualization**. It converts high-dimensional data into 2D/3D plots while preserving local neighborhood structure.

Think of it like crumpling a sheet of paper with dots on it: nearby dots stay nearby, but the global layout changes. t-SNE preserves "who is neighbors with whom."

### Why It Matters
- The **gold standard** for visualizing high-dimensional data
- Reveals clusters and structure that PCA can't capture
- Used extensively in biology (single-cell RNA), NLP (word embeddings), and computer vision
- Essential for understanding what your data "looks like"

### How It Differs from PCA

| Aspect | PCA | t-SNE |
|--------|-----|-------|
| Type | Linear | Non-linear |
| Preserves | Global structure (distances) | Local structure (neighborhoods) |
| Speed | Very fast | Slow (O(n²)) |
| Deterministic | Yes | No (random initialization) |
| Use for ML | Yes (preprocessing) | No (visualization only!) |
| Inverse transform | Yes | No |
| Number of components | Any | Typically 2 or 3 |

### How It Works (Simplified)

1. **High-dimensional space:** For each pair of points, compute a probability that they are "neighbors" (Gaussian kernel)
2. **Low-dimensional space:** Place points randomly in 2D, compute same probabilities (Student-t distribution)
3. **Optimize:** Move points in 2D until the two probability distributions match (minimize KL divergence)

$$\text{Cost} = KL(P \| Q) = \sum_{i \neq j} p_{ij} \log \frac{p_{ij}}{q_{ij}}$$

Where $p_{ij}$ = similarity in high-D space, $q_{ij}$ = similarity in low-D space.

The **t-distribution** (heavy tails) in low-D allows moderate distances to model large high-D distances, solving the "crowding problem."

### Code Examples

#### Basic t-SNE

```python
from sklearn.manifold import TSNE
from sklearn.datasets import load_digits

# Load high-dimensional data (8x8 pixel images of digits)
digits = load_digits()
X_digits = digits.data   # 1797 samples × 64 features
y_digits = digits.target  # Digit labels (0-9)

# Apply t-SNE (reduce 64D → 2D)
tsne = TSNE(
    n_components=2,        # Always 2 or 3 for visualization
    perplexity=30,         # Balance between local/global structure (5-50)
    learning_rate='auto',  # Sklearn auto-tunes this (recommended)
    n_iter=1000,           # Optimization iterations (more = better, but slower)
    random_state=42,
    init='pca',            # Initialize with PCA (more stable than random)
    metric='euclidean'     # Distance metric
)

X_tsne = tsne.fit_transform(X_digits)

# Visualize
plt.figure(figsize=(12, 8))
scatter = plt.scatter(X_tsne[:, 0], X_tsne[:, 1], c=y_digits, 
                      cmap='Spectral', s=10, alpha=0.7)
plt.colorbar(scatter, label='Digit')
plt.title('t-SNE Visualization of Handwritten Digits (64D → 2D)')
plt.xlabel('t-SNE Dimension 1')
plt.ylabel('t-SNE Dimension 2')
plt.tight_layout()
plt.show()

# You should see 10 clear clusters — one for each digit!
```

#### Effect of Perplexity

```python
# Perplexity controls the effective number of neighbors considered
# Lower perplexity → more local structure (tight clusters)
# Higher perplexity → more global structure (spread out)

fig, axes = plt.subplots(1, 3, figsize=(18, 5))
perplexities = [5, 30, 100]

for ax, perp in zip(axes, perplexities):
    tsne = TSNE(n_components=2, perplexity=perp, random_state=42, 
                init='pca', learning_rate='auto')
    X_embedded = tsne.fit_transform(X_digits)
    ax.scatter(X_embedded[:, 0], X_embedded[:, 1], c=y_digits, 
               cmap='Spectral', s=5, alpha=0.6)
    ax.set_title(f'Perplexity = {perp}')
    ax.set_xticks([])
    ax.set_yticks([])

plt.suptitle('t-SNE: Effect of Perplexity', fontsize=14)
plt.tight_layout()
plt.show()
```

#### PCA + t-SNE (Best Practice for Large Datasets)

```python
# For large high-dimensional datasets, first reduce with PCA, then apply t-SNE
# This is MUCH faster and often gives better results

from sklearn.decomposition import PCA

# Step 1: PCA to reduce from 64D to 30D (fast)
pca_pre = PCA(n_components=30)
X_pca = pca_pre.fit_transform(X_digits)

# Step 2: t-SNE on the 30D data (much faster than on 64D)
tsne = TSNE(n_components=2, perplexity=30, random_state=42, 
            init='pca', learning_rate='auto')
X_tsne = tsne.fit_transform(X_pca)

# This is the standard pipeline used in practice
```

> **WARNING:** t-SNE Pitfalls to Watch For:
> - **Distances between clusters are meaningless** — don't interpret cluster separation as similarity
> - **Cluster sizes are meaningless** — t-SNE can expand/shrink clusters arbitrarily
> - **Run multiple times** with different random seeds to confirm patterns
> - **NEVER use t-SNE features for ML** — use PCA instead (t-SNE output is non-deterministic and non-generalizable)
> - **Slow for large datasets** — use PCA preprocessing or consider UMAP as alternative

---

## Comparison of Clustering Algorithms

### Side-by-Side Comparison

| Feature | K-Means | DBSCAN | Agglomerative |
|---------|---------|--------|---------------|
| Need to specify K? | Yes | No | Yes (or cut dendrogram) |
| Cluster shape | Spherical only | Arbitrary | Depends on linkage |
| Handles outliers? | No (assigns all points) | Yes (labels noise as -1) | No |
| Scalability | O(nKd) — fast | O(n²) — moderate | O(n²d) — slow for large n |
| Deterministic? | No (depends on init) | Yes | Yes |
| Works well on... | Even-sized, spherical | Irregular shapes, noise | Hierarchical data |
| Scales to | Millions | Thousands-Millions | Thousands |

### When to Use What

```
Is your data large (>100K samples)?
├── YES → K-Means or MiniBatchKMeans
└── NO
    ├── Do you expect outliers/noise?
    │   ├── YES → DBSCAN
    │   └── NO
    │       ├── Do clusters have irregular shapes?
    │       │   ├── YES → DBSCAN
    │       │   └── NO → K-Means or Agglomerative
    │       └── Need hierarchical structure?
    │           ├── YES → Agglomerative + Dendrogram
    │           └── NO → K-Means
```

---

## Common Mistakes

### 1. Not Scaling Before Clustering/PCA
```python
# BAD: Features on different scales dominate distance calculations
kmeans = KMeans(n_clusters=3)
kmeans.fit(X_raw)  # If age=[20-80] and salary=[20000-200000], salary dominates

# GOOD: Always standardize first
from sklearn.preprocessing import StandardScaler
X_scaled = StandardScaler().fit_transform(X_raw)
kmeans.fit(X_scaled)
```

### 2. Assuming K-Means Clusters Are Real
```python
# BAD: K-Means ALWAYS finds K clusters, even if there are none!
X_random = np.random.randn(500, 2)  # Random data with NO structure
kmeans = KMeans(n_clusters=5)
labels = kmeans.fit_predict(X_random)  # Returns 5 "clusters" anyway!

# GOOD: Validate with silhouette score
from sklearn.metrics import silhouette_score
score = silhouette_score(X_random, labels)
print(f"Silhouette: {score:.4f}")  # Low score → clusters aren't meaningful
```

### 3. Using t-SNE Output for ML Models
```python
# BAD: t-SNE is for visualization ONLY
tsne = TSNE(n_components=2)
X_tsne = tsne.fit_transform(X_train)
model.fit(X_tsne, y_train)  # WRONG! t-SNE can't transform new data

# GOOD: Use PCA for feature reduction in ML
pca = PCA(n_components=50)
X_pca = pca.fit_transform(X_train)
X_pca_test = pca.transform(X_test)  # PCA CAN transform new data
model.fit(X_pca, y_train)
```

### 4. Not Using `fit_transform` vs `fit` + `transform`
```python
# t-SNE: ONLY has fit_transform (no separate transform for new data)
tsne.fit_transform(X)     # OK
tsne.transform(X_new)     # ERROR! t-SNE can't transform unseen data

# PCA: has both (supports new data)
pca.fit(X_train)          # Learn components from training data
pca.transform(X_test)     # Apply same transformation to test data
```

### 5. Wrong Perplexity in t-SNE
```python
# BAD: Perplexity > n_samples
tsne = TSNE(perplexity=200)  # If n_samples=150, this fails

# GOOD: perplexity should be 5-50, and < n_samples
tsne = TSNE(perplexity=min(30, len(X) // 3))
```

---

## Interview Questions

### Conceptual

**Q1: What is the difference between PCA and t-SNE?**
> PCA is a linear method that preserves global variance and can transform new data. t-SNE is non-linear, preserves local neighborhoods, and is for visualization only (no transform for new data). Use PCA for preprocessing/ML, t-SNE for visualization.

**Q2: How does K-Means handle outliers? What's a better alternative?**
> K-Means assigns every point to a cluster, so outliers distort centroids. DBSCAN is better — it labels low-density points as noise (-1). Alternatively, use K-Medoids (more robust to outliers) or preprocess outliers before K-Means.

**Q3: Can you run K-Means on categorical data?**
> No. K-Means uses Euclidean distance and means, which require numerical data. For categorical data, use K-Modes or K-Prototypes (mixed data). Alternatively, one-hot encode categories first (but distances become less meaningful).

**Q4: What happens if you apply PCA without scaling?**
> Features with larger scales dominate the principal components. A feature ranging from 0-1000 would overshadow one from 0-1, even if the smaller one is more informative. Always `StandardScaler()` before PCA.

**Q5: DBSCAN says it found 0 clusters. What went wrong?**
> `eps` is too small (every point is noise) or `min_samples` is too large. Use the k-distance plot to find optimal eps. Also check if data is scaled — unscaled features can make all distances huge.

**Q6: What is the "curse of dimensionality" and how does PCA help?**
> In high dimensions, distances between points become nearly equal, making clustering and nearest-neighbor methods fail. PCA reduces dimensions by keeping only the directions with most variance, effectively compressing data to a lower-dimensional space where distances are meaningful again.

**Q7: How do you evaluate clustering when you have no labels?**
> Internal metrics: Silhouette Score (cohesion vs separation), Calinski-Harabasz Index (ratio of between/within cluster variance), Davies-Bouldin Index (average similarity between clusters). Also visual inspection with t-SNE/PCA plots.

### Coding

**Q8: Write code to find the optimal K for K-Means using silhouette score.**
```python
from sklearn.metrics import silhouette_score

best_k, best_score = 2, -1
for k in range(2, 11):
    km = KMeans(n_clusters=k, random_state=42, n_init=10)
    labels = km.fit_predict(X_scaled)
    score = silhouette_score(X_scaled, labels)
    if score > best_score:
        best_k, best_score = k, score
print(f"Optimal K={best_k}, Silhouette={best_score:.4f}")
```

---

## Quick Reference

### Import Cheat Sheet

```python
# Clustering
from sklearn.cluster import KMeans, MiniBatchKMeans
from sklearn.cluster import DBSCAN
from sklearn.cluster import AgglomerativeClustering

# Dimensionality Reduction
from sklearn.decomposition import PCA
from sklearn.manifold import TSNE

# Evaluation
from sklearn.metrics import silhouette_score, silhouette_samples
from sklearn.metrics import calinski_harabasz_score
from sklearn.metrics import davies_bouldin_score

# Preprocessing (ALWAYS before unsupervised learning)
from sklearn.preprocessing import StandardScaler
```

### Algorithm Decision Table

| I Want To... | Use | Key Parameters |
|-------------|-----|----------------|
| Group data into K round clusters | `KMeans(n_clusters=K)` | `n_clusters`, `init`, `n_init` |
| Find clusters of any shape + outliers | `DBSCAN(eps=0.5)` | `eps`, `min_samples` |
| See hierarchical cluster relationships | `AgglomerativeClustering()` | `n_clusters`, `linkage` |
| Reduce dimensions for ML | `PCA(n_components=0.95)` | `n_components` |
| Visualize high-D data in 2D | `TSNE(n_components=2)` | `perplexity`, `n_iter` |
| Fast K-Means on big data | `MiniBatchKMeans()` | `batch_size`, `n_clusters` |

### Key Numbers to Remember

| Metric | Value | Meaning |
|--------|-------|---------|
| Silhouette Score | +1 | Perfect clusters |
| Silhouette Score | 0 | Overlapping clusters |
| Silhouette Score | -1 | Wrong cluster assignment |
| PCA variance threshold | 0.95 | Keep 95% info (standard) |
| t-SNE perplexity | 5-50 | Typical range |
| DBSCAN noise label | -1 | Point is an outlier |
