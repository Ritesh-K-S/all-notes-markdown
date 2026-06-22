# Chapter 08: Clustering Algorithms — Complete Guide

## Table of Contents
- [What is Clustering?](#what-is-clustering)
- [Why Clustering Matters](#why-clustering-matters)
- [K-Means Clustering](#k-means-clustering)
- [K-Means++ Initialization](#k-means-initialization)
- [Hierarchical Clustering](#hierarchical-clustering)
- [DBSCAN](#dbscan)
- [Gaussian Mixture Models (GMM)](#gaussian-mixture-models)
- [Other Clustering Methods](#other-clustering-methods)
- [Cluster Evaluation Metrics](#cluster-evaluation-metrics)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Clustering?

### Simple Explanation (Like Explaining to a 15-Year-Old)

Imagine you dump 1000 Lego pieces on the floor and ask someone to organize them into groups **without telling them the categories**. They might group by color, shape, size, or some combination. That's clustering — finding natural groups in data without being told what those groups are.

**Clustering is unsupervised learning:** there are no labels, no "right answers." The algorithm discovers structure on its own.

### Formal Definition

Clustering partitions a dataset $X = \{x_1, x_2, ..., x_n\}$ into $K$ groups $C = \{C_1, C_2, ..., C_K\}$ such that:
- Points within a cluster are similar (high **intra-cluster similarity**)
- Points in different clusters are dissimilar (low **inter-cluster similarity**)

$$\text{Objective: } \min_C \sum_{k=1}^{K} \sum_{x_i \in C_k} d(x_i, \mu_k)$$

Where $\mu_k$ is the centroid of cluster $k$ and $d$ is a distance function.

---

## Why Clustering Matters

### Real-World Applications

| Domain | Application | What It Does |
|--------|-------------|--------------|
| Marketing | Customer Segmentation | Group customers by behavior for targeted campaigns |
| Biology | Gene Expression | Find groups of genes with similar expression patterns |
| E-commerce | Product Recommendations | Group similar products together |
| Cybersecurity | Anomaly Detection | Points not belonging to any cluster = anomalies |
| Image Processing | Image Segmentation | Group pixels by color/texture |
| Document Analysis | Topic Modeling | Group similar documents together |
| Social Networks | Community Detection | Find groups of connected users |
| Healthcare | Patient Stratification | Group patients by symptom profiles |

### When to Use Which Algorithm

```
                    ┌─────────────────────────────┐
                    │ Do you know the number       │
                    │ of clusters (K)?             │
                    └──────────┬──────────────────┘
                         ╱           ╲
                       Yes            No
                       ╱               ╲
              ┌───────▼────┐    ┌──────▼──────┐
              │ Clusters   │    │ DBSCAN      │
              │ spherical? │    │ HDBSCAN     │
              └──────┬─────┘    │ (auto-find  │
                   ╱   ╲        │  clusters)  │
                 Yes    No      └─────────────┘
                 ╱        ╲
         ┌──────▼──┐  ┌───▼────────┐
         │ K-Means │  │ GMM        │
         │         │  │ Spectral   │
         └─────────┘  └────────────┘
```

---

## K-Means Clustering

### What It Is

K-Means is the "Hello World" of clustering. It partitions data into exactly $K$ clusters by minimizing the within-cluster sum of squared distances (inertia).

### How It Works — Step by Step

```
Step 1: Randomly place K centroids (★) in the space

    ○ ○   ○                    ○ ○   ○
  ○  ★₁ ○    ○    →         ○  ★₁ ○    ○
    ○     ○                    ○     ○
              ○   ○                      ○   ○
         ○  ★₂  ○    →            ○  ★₂  ○
              ○                         ○

Step 2: Assign each point to nearest centroid

    ● ●   ●                    
  ●  ★₁ ●    ◆         ● = cluster 1
    ●     ◆             ◆ = cluster 2
              ◆   ◆                      
         ◆  ★₂  ◆    
              ◆                         

Step 3: Move centroids to mean of assigned points

    ● ●   ●                    
  ●  ★₁→ ●    ◆        ★ moves to center of its points
    ●     ◆             
              ◆   ◆                      
         ◆  ←★₂  ◆    
              ◆                         

Step 4: Repeat Steps 2-3 until convergence
         (centroids stop moving)
```

### The Math

**Objective Function (Inertia / WCSS):**

$$J = \sum_{k=1}^{K} \sum_{x_i \in C_k} \|x_i - \mu_k\|^2$$

**Assignment Step:**
$$C_k = \{x_i : \|x_i - \mu_k\|^2 \leq \|x_i - \mu_j\|^2 \text{ for all } j\}$$

**Update Step:**
$$\mu_k = \frac{1}{|C_k|} \sum_{x_i \in C_k} x_i$$

**Time Complexity:** $O(n \cdot K \cdot d \cdot I)$ where $n$ = samples, $K$ = clusters, $d$ = dimensions, $I$ = iterations.

### Code Example — K-Means from Scratch + Scikit-learn

```python
import numpy as np
from sklearn.cluster import KMeans
from sklearn.datasets import make_blobs
from sklearn.preprocessing import StandardScaler
import matplotlib.pyplot as plt

# ============= K-MEANS FROM SCRATCH =============
class KMeansFromScratch:
    def __init__(self, n_clusters=3, max_iters=100, tol=1e-4, random_state=42):
        self.n_clusters = n_clusters
        self.max_iters = max_iters
        self.tol = tol
        self.random_state = random_state
    
    def fit(self, X):
        np.random.seed(self.random_state)
        n_samples = X.shape[0]
        
        # Initialize centroids randomly from data points
        random_indices = np.random.choice(n_samples, self.n_clusters, replace=False)
        self.centroids = X[random_indices].copy()
        
        for iteration in range(self.max_iters):
            # Step 1: Assign points to nearest centroid
            distances = self._compute_distances(X)
            self.labels = np.argmin(distances, axis=1)
            
            # Step 2: Update centroids
            new_centroids = np.zeros_like(self.centroids)
            for k in range(self.n_clusters):
                cluster_points = X[self.labels == k]
                if len(cluster_points) > 0:
                    new_centroids[k] = cluster_points.mean(axis=0)
                else:
                    new_centroids[k] = self.centroids[k]  # Keep if empty
            
            # Check convergence
            shift = np.linalg.norm(new_centroids - self.centroids)
            self.centroids = new_centroids
            
            if shift < self.tol:
                print(f"Converged at iteration {iteration + 1}")
                break
        
        self.inertia_ = self._compute_inertia(X)
        return self
    
    def predict(self, X):
        distances = self._compute_distances(X)
        return np.argmin(distances, axis=1)
    
    def _compute_distances(self, X):
        """Compute distance from each point to each centroid."""
        distances = np.zeros((X.shape[0], self.n_clusters))
        for k in range(self.n_clusters):
            distances[:, k] = np.linalg.norm(X - self.centroids[k], axis=1)
        return distances
    
    def _compute_inertia(self, X):
        """Sum of squared distances to nearest centroid."""
        distances = self._compute_distances(X)
        min_distances = np.min(distances, axis=1)
        return np.sum(min_distances ** 2)

# Generate sample data
X, y_true = make_blobs(
    n_samples=1000, centers=4, cluster_std=1.0, random_state=42
)

# Scale features (CRITICAL for distance-based algorithms!)
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# Our implementation
km_scratch = KMeansFromScratch(n_clusters=4)
km_scratch.fit(X_scaled)
print(f"Custom K-Means Inertia: {km_scratch.inertia_:.2f}")

# ============= SCIKIT-LEARN K-MEANS =============
km = KMeans(
    n_clusters=4,
    init='k-means++',       # Smart initialization (much better than random!)
    n_init=10,              # Run 10 times with different seeds, pick best
    max_iter=300,           # Max iterations per run
    tol=1e-4,              # Convergence tolerance
    random_state=42,
    algorithm='lloyd'       # Options: 'lloyd', 'elkan' (faster for low-d)
)

km.fit(X_scaled)
labels = km.labels_
centroids = km.cluster_centers_

print(f"Sklearn K-Means Inertia: {km.inertia_:.2f}")
print(f"Number of iterations: {km.n_iter_}")
print(f"Cluster sizes: {np.bincount(labels)}")
```

### Finding Optimal K — The Elbow Method + Silhouette Analysis

```python
from sklearn.cluster import KMeans
from sklearn.metrics import silhouette_score, silhouette_samples
import numpy as np

# ============= ELBOW METHOD =============
inertias = []
silhouette_scores = []
K_range = range(2, 11)

for k in K_range:
    km = KMeans(n_clusters=k, init='k-means++', n_init=10, random_state=42)
    km.fit(X_scaled)
    inertias.append(km.inertia_)
    silhouette_scores.append(silhouette_score(X_scaled, km.labels_))
    print(f"K={k}: Inertia={km.inertia_:.1f}, Silhouette={silhouette_scores[-1]:.4f}")

# Output:
# K=2: Inertia=1658.3, Silhouette=0.5821
# K=3: Inertia=1038.5, Silhouette=0.6534
# K=4: Inertia=638.2,  Silhouette=0.7123  ← Best!
# K=5: Inertia=562.1,  Silhouette=0.6245
# ...

# The "elbow" is where adding more clusters doesn't reduce inertia much
# Best silhouette score confirms K=4

# ============= AUTOMATED ELBOW DETECTION =============
def find_elbow(inertias, K_range):
    """Find elbow using the kneedle algorithm (simplified)."""
    # Normalize
    x = np.array(list(K_range))
    y = np.array(inertias)
    x_norm = (x - x.min()) / (x.max() - x.min())
    y_norm = (y - y.min()) / (y.max() - y.min())
    
    # Find point with maximum distance from line connecting endpoints
    line_start = np.array([x_norm[0], y_norm[0]])
    line_end = np.array([x_norm[-1], y_norm[-1]])
    line_vec = line_end - line_start
    line_vec_norm = line_vec / np.linalg.norm(line_vec)
    
    distances = []
    for i in range(len(x_norm)):
        point = np.array([x_norm[i], y_norm[i]])
        vec_to_point = point - line_start
        proj_length = np.dot(vec_to_point, line_vec_norm)
        proj_point = line_start + proj_length * line_vec_norm
        dist = np.linalg.norm(point - proj_point)
        distances.append(dist)
    
    best_k_idx = np.argmax(distances)
    return list(K_range)[best_k_idx]

optimal_k = find_elbow(inertias, K_range)
print(f"\nOptimal K (elbow method): {optimal_k}")
```

> **Pro Tip:** Always use BOTH the Elbow method AND Silhouette score. They don't always agree — use domain knowledge to make the final call.

---

## K-Means++ Initialization {#k-means-initialization}

### Why Regular Random Initialization is Bad

```
Random init might place centroids like this:    K-Means++ spreads them out:

    ★₁★₂★₃                                     ★₁          ★₃
    (all in one corner!)                              ★₂
    
    → Poor convergence                          → Good convergence
    → Local minimum                             → Near-global optimum
```

### How K-Means++ Works

1. Choose first centroid randomly from data points
2. For each data point, compute distance $D(x)$ to nearest existing centroid
3. Choose next centroid with probability proportional to $D(x)^2$
4. Repeat steps 2-3 until K centroids chosen
5. Proceed with standard K-Means

**Result:** Guaranteed $O(\log K)$ competitive ratio with optimal clustering.

```python
def kmeans_plus_plus_init(X, n_clusters, random_state=42):
    """K-Means++ initialization."""
    rng = np.random.RandomState(random_state)
    n_samples = X.shape[0]
    
    # Step 1: Choose first centroid randomly
    centroids = [X[rng.randint(n_samples)]]
    
    for _ in range(1, n_clusters):
        # Step 2: Compute distances to nearest centroid
        distances = np.min([
            np.sum((X - c) ** 2, axis=1) for c in centroids
        ], axis=0)
        
        # Step 3: Choose next centroid with probability ∝ distance²
        probs = distances / distances.sum()
        next_centroid_idx = rng.choice(n_samples, p=probs)
        centroids.append(X[next_centroid_idx])
    
    return np.array(centroids)

# This is what sklearn does internally when you set init='k-means++'
centroids = kmeans_plus_plus_init(X_scaled, n_clusters=4)
print(f"K-Means++ initial centroids shape: {centroids.shape}")
```

---

## Hierarchical Clustering

### What It Is

Hierarchical clustering builds a tree (dendrogram) of clusters. It can be:
- **Agglomerative** (bottom-up): Start with each point as its own cluster, merge the closest pairs
- **Divisive** (top-down): Start with one cluster, recursively split

### How Agglomerative Clustering Works

```
Start: Each point is its own cluster
       A    B    C    D    E    F

Step 1: Merge closest pair (B,C)
       A   [B,C]  D    E    F

Step 2: Merge next closest (E,F)  
       A   [B,C]  D   [E,F]

Step 3: Merge (A,[B,C])
      [A,B,C]    D   [E,F]

Step 4: Merge (D,[E,F])
      [A,B,C]  [D,E,F]

Step 5: Merge all
      [A,B,C,D,E,F]


DENDROGRAM:
Height ▲
  5    │         ┌──────────────────────┐
       │         │                      │
  3    │    ┌────┴────┐            ┌────┴────┐
       │    │         │            │         │
  1.5  │  ┌─┴─┐      │          ┌─┴─┐      │
       │  │   │      │          │   │      │
  0    │  B   C      A          E   F      D
       └──────────────────────────────────────→

Cut at height 3 → 2 clusters: {A,B,C} and {D,E,F}
```

### Linkage Methods

| Linkage | Formula | Behavior |
|---------|---------|----------|
| **Single** (MIN) | $d(C_i, C_j) = \min_{a \in C_i, b \in C_j} d(a,b)$ | Chain-like clusters, sensitive to noise |
| **Complete** (MAX) | $d(C_i, C_j) = \max_{a \in C_i, b \in C_j} d(a,b)$ | Compact, spherical clusters |
| **Average** (UPGMA) | $d(C_i, C_j) = \frac{1}{|C_i||C_j|} \sum_{a \in C_i} \sum_{b \in C_j} d(a,b)$ | Balanced approach |
| **Ward** | Minimizes increase in total within-cluster variance | Most similar to K-Means, usually best |

### Code Example — Hierarchical Clustering

```python
from sklearn.cluster import AgglomerativeClustering
from scipy.cluster.hierarchy import dendrogram, linkage, fcluster
from sklearn.datasets import make_moons, make_blobs
from sklearn.metrics import silhouette_score
import numpy as np

# Generate data
X, y_true = make_blobs(n_samples=500, centers=4, cluster_std=0.8, random_state=42)
X_scaled = StandardScaler().fit_transform(X)

# ============= SKLEARN AGGLOMERATIVE =============
agg = AgglomerativeClustering(
    n_clusters=4,           # Can also use distance_threshold instead
    metric='euclidean',     # Distance metric
    linkage='ward',         # Linkage criterion (ward requires euclidean)
    # distance_threshold=5, # Alternative: cut at specific distance (set n_clusters=None)
)

agg_labels = agg.fit_predict(X_scaled)
print(f"Agglomerative Silhouette: {silhouette_score(X_scaled, agg_labels):.4f}")
print(f"Cluster sizes: {np.bincount(agg_labels)}")

# ============= SCIPY FOR DENDROGRAM =============
# Compute linkage matrix (needed for dendrogram)
Z = linkage(X_scaled, method='ward', metric='euclidean')
# Z[i] = [cluster_1, cluster_2, distance, n_points_in_new_cluster]

print(f"\nLinkage matrix shape: {Z.shape}")  # (n_samples-1, 4)
print(f"Last 5 merges (highest level):")
print(Z[-5:])

# Cut dendrogram at specific number of clusters
labels_from_dendro = fcluster(Z, t=4, criterion='maxclust')
print(f"\nDendrogram cut (4 clusters) Silhouette: "
      f"{silhouette_score(X_scaled, labels_from_dendro):.4f}")

# Cut at specific distance threshold
labels_distance = fcluster(Z, t=5.0, criterion='distance')
print(f"Dendrogram cut (dist=5.0) → {len(np.unique(labels_distance))} clusters")

# ============= CHOOSING LINKAGE METHOD =============
linkage_methods = ['single', 'complete', 'average', 'ward']
for method in linkage_methods:
    if method == 'ward':
        agg_test = AgglomerativeClustering(n_clusters=4, linkage=method)
    else:
        agg_test = AgglomerativeClustering(
            n_clusters=4, linkage=method, metric='euclidean'
        )
    labels_test = agg_test.fit_predict(X_scaled)
    score = silhouette_score(X_scaled, labels_test)
    print(f"Linkage={method:<10} Silhouette={score:.4f}")
```

### When to Use Hierarchical Clustering

- When you need a **hierarchy** of clusters (taxonomy, biology)
- When you don't know K in advance (inspect dendrogram to decide)
- When clusters have **nested structure**
- Small to medium datasets (scales as $O(n^2)$ memory, $O(n^3)$ time)

> **Warning:** Hierarchical clustering does NOT scale well. For >10,000 points, use K-Means or DBSCAN instead.

---

## DBSCAN

### What It Is

**D**ensity-**B**ased **S**patial **C**lustering of **A**pplications with **N**oise. It finds clusters as dense regions separated by sparse regions. Unlike K-Means, it can find clusters of **any shape** and automatically detects **outliers**.

### How It Works — Intuition

Imagine you're at a party. A "cluster" is a group of people standing close together (within arm's reach of at least 4 other people). Someone standing alone in the corner? That's noise/outlier.

### Key Concepts

```
Parameters:
  eps (ε)        = Maximum distance between two neighbors
  min_samples    = Minimum points to form a dense region

Point Types:
  ● Core Point    = Has ≥ min_samples points within eps radius
  ◐ Border Point  = Within eps of a core point, but < min_samples neighbors
  ○ Noise Point   = Neither core nor border (outlier!)

Example (eps=1, min_samples=3):

    ●───●───●       ◐           ○  (noise/outlier)
    │   │   │       │
    ●───●───●       ●───●───●
                    │   │
                    ●───●
    
    Cluster 1       Cluster 2    Noise
```

### The Algorithm

1. For each point, find all neighbors within $\epsilon$ distance
2. If a point has $\geq$ `min_samples` neighbors → **Core point**
3. Start from an unvisited core point, expand cluster by adding all density-reachable points
4. Points not reachable from any core point → **Noise**

### Code Example — DBSCAN

```python
from sklearn.cluster import DBSCAN
from sklearn.datasets import make_moons, make_blobs, make_circles
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import silhouette_score
from sklearn.neighbors import NearestNeighbors
import numpy as np

# Generate non-spherical data (K-Means will fail here!)
X_moons, y_moons = make_moons(n_samples=500, noise=0.08, random_state=42)
X_circles, y_circles = make_circles(n_samples=500, noise=0.05, factor=0.5, random_state=42)

# Scale data
X_moons_scaled = StandardScaler().fit_transform(X_moons)

# ============= DBSCAN =============
dbscan = DBSCAN(
    eps=0.3,                # Neighborhood radius
    min_samples=5,          # Min points to form a core point
    metric='euclidean',     # Distance metric
    algorithm='auto',       # 'ball_tree', 'kd_tree', 'brute', 'auto'
    n_jobs=-1
)

db_labels = dbscan.fit_predict(X_moons_scaled)

# Analyze results
n_clusters = len(set(db_labels)) - (1 if -1 in db_labels else 0)
n_noise = list(db_labels).count(-1)
print(f"Number of clusters: {n_clusters}")
print(f"Number of noise points: {n_noise} ({100*n_noise/len(db_labels):.1f}%)")
print(f"Cluster sizes: {np.bincount(db_labels[db_labels >= 0])}")

# Silhouette score (excluding noise points)
if n_clusters > 1:
    mask = db_labels >= 0  # Exclude noise
    score = silhouette_score(X_moons_scaled[mask], db_labels[mask])
    print(f"Silhouette (excl. noise): {score:.4f}")

# ============= COMPARE: K-Means FAILS on moons =============
km_moons = KMeans(n_clusters=2, random_state=42)
km_labels = km_moons.fit_predict(X_moons_scaled)
print(f"\nK-Means on moons - Silhouette: {silhouette_score(X_moons_scaled, km_labels):.4f}")
print(f"DBSCAN on moons - Silhouette: {score:.4f}")
# DBSCAN will be significantly better on non-spherical data!
```

### Finding Optimal eps — The K-Distance Graph

```python
from sklearn.neighbors import NearestNeighbors
import numpy as np

def find_optimal_eps(X, min_samples=5):
    """
    Use the k-distance graph to find optimal eps.
    Look for the "elbow" in the sorted k-distance plot.
    """
    # Compute distance to k-th nearest neighbor for each point
    nn = NearestNeighbors(n_neighbors=min_samples)
    nn.fit(X)
    distances, _ = nn.kneighbors(X)
    
    # Sort k-th nearest neighbor distances
    k_distances = np.sort(distances[:, min_samples - 1])
    
    # The "elbow" in this sorted plot = good eps value
    # Automated elbow finding:
    n = len(k_distances)
    x = np.arange(n)
    
    # Normalize
    x_norm = x / x.max()
    y_norm = (k_distances - k_distances.min()) / (k_distances.max() - k_distances.min())
    
    # Find maximum curvature point
    dx = np.gradient(y_norm)
    ddx = np.gradient(dx)
    curvature = np.abs(ddx) / (1 + dx**2)**1.5
    
    # Ignore the extremes
    elbow_idx = np.argmax(curvature[n//10:9*n//10]) + n//10
    optimal_eps = k_distances[elbow_idx]
    
    print(f"Suggested eps: {optimal_eps:.4f}")
    print(f"k-distance range: [{k_distances[0]:.4f}, {k_distances[-1]:.4f}]")
    
    return optimal_eps, k_distances

optimal_eps, k_dists = find_optimal_eps(X_moons_scaled, min_samples=5)

# Test multiple eps values
print("\nEps sensitivity analysis:")
for eps in [0.1, 0.2, 0.3, 0.4, 0.5]:
    db = DBSCAN(eps=eps, min_samples=5).fit(X_moons_scaled)
    n_cl = len(set(db.labels_)) - (1 if -1 in db.labels_ else 0)
    n_noise = (db.labels_ == -1).sum()
    print(f"  eps={eps:.1f}: {n_cl} clusters, {n_noise} noise points")
```

### HDBSCAN — The Improved DBSCAN

```python
# pip install hdbscan
import hdbscan

# HDBSCAN doesn't need eps! It varies density automatically
hdb = hdbscan.HDBSCAN(
    min_cluster_size=15,     # Minimum points in a cluster
    min_samples=5,           # Core point threshold (more conservative = more noise)
    cluster_selection_epsilon=0.0,
    metric='euclidean',
    cluster_selection_method='eom'  # 'eom' (Excess of Mass) or 'leaf'
)

hdb_labels = hdb.fit_predict(X_moons_scaled)
print(f"HDBSCAN clusters: {len(set(hdb_labels)) - (1 if -1 in hdb_labels else 0)}")
print(f"HDBSCAN noise: {(hdb_labels == -1).sum()}")

# HDBSCAN provides cluster probabilities!
print(f"Cluster probabilities (first 10): {hdb.probabilities_[:10]}")
# Points with low probability are "soft" members — might be on the boundary
```

> **Pro Tip:** In production, prefer HDBSCAN over DBSCAN. It eliminates the need to tune `eps`, handles varying densities, and provides soft cluster assignments.

---

## Gaussian Mixture Models (GMM) {#gaussian-mixture-models}

### What It Is

GMM assumes data is generated from a mixture of several Gaussian (normal) distributions. Unlike K-Means (hard assignment), GMM provides **soft/probabilistic** cluster membership.

### Intuition

Think of it this way: K-Means says "this point belongs to cluster A." GMM says "this point has a 70% chance of belonging to cluster A and 30% chance of belonging to cluster B."

```
K-Means:                          GMM:
(Hard boundaries)                 (Soft/probabilistic boundaries)

    ●●●●│○○○○                     ●●●●≈≈○○○○
    ●●●●│○○○○                     ●●●●≈≈○○○○
         │                              (overlap region)
    Sharp cut                     Gradual transition
```

### How It Works — Expectation-Maximization (EM)

GMM uses the **EM algorithm** to estimate parameters:

**Model:** Each cluster $k$ is a Gaussian with:
- Mean $\mu_k$ (center)
- Covariance $\Sigma_k$ (shape/orientation)
- Weight $\pi_k$ (proportion of data from this Gaussian)

$$P(x) = \sum_{k=1}^{K} \pi_k \cdot \mathcal{N}(x | \mu_k, \Sigma_k)$$

**EM Algorithm:**

1. **E-Step (Expectation):** Compute responsibility — probability that each point belongs to each cluster:
$$r_{ik} = \frac{\pi_k \cdot \mathcal{N}(x_i | \mu_k, \Sigma_k)}{\sum_{j=1}^{K} \pi_j \cdot \mathcal{N}(x_i | \mu_j, \Sigma_j)}$$

2. **M-Step (Maximization):** Update parameters using responsibilities:
$$\mu_k = \frac{\sum_i r_{ik} x_i}{\sum_i r_{ik}}, \quad \Sigma_k = \frac{\sum_i r_{ik}(x_i - \mu_k)(x_i - \mu_k)^T}{\sum_i r_{ik}}, \quad \pi_k = \frac{\sum_i r_{ik}}{N}$$

3. Repeat until convergence (log-likelihood stops improving).

### Code Example — GMM

```python
from sklearn.mixture import GaussianMixture
from sklearn.datasets import make_blobs
from sklearn.preprocessing import StandardScaler
import numpy as np

# Generate data with different cluster shapes
X, y_true = make_blobs(
    n_samples=1000, centers=4, 
    cluster_std=[1.0, 1.5, 0.5, 2.0],  # Different densities!
    random_state=42
)
X_scaled = StandardScaler().fit_transform(X)

# ============= GMM =============
gmm = GaussianMixture(
    n_components=4,           # Number of Gaussians (clusters)
    covariance_type='full',   # Each cluster has its own full covariance matrix
    max_iter=200,
    n_init=5,                 # Run 5 times, pick best
    init_params='k-means++',  # Initialization method
    random_state=42
)

gmm.fit(X_scaled)
gmm_labels = gmm.predict(X_scaled)

# Soft assignments (probabilities!)
proba = gmm.predict_proba(X_scaled)
print(f"Sample probabilities (first point): {proba[0].round(4)}")
# Output: [0.9987, 0.0001, 0.0012, 0.0000] → mostly cluster 0

# Model parameters
print(f"\nCluster means:\n{gmm.means_}")
print(f"\nCluster weights: {gmm.weights_.round(4)}")
print(f"Converged: {gmm.converged_}")
print(f"Iterations: {gmm.n_iter_}")

# ============= COVARIANCE TYPES =============
# 'full'       → Each cluster has its own arbitrary covariance (most flexible)
# 'tied'       → All clusters share the same covariance
# 'diag'       → Diagonal covariance (axis-aligned ellipses)
# 'spherical'  → Single variance value per cluster (K-Means equivalent!)

cov_types = ['spherical', 'diag', 'tied', 'full']
for cov_type in cov_types:
    gmm_test = GaussianMixture(n_components=4, covariance_type=cov_type, 
                                random_state=42)
    gmm_test.fit(X_scaled)
    bic = gmm_test.bic(X_scaled)    # Bayesian Information Criterion
    aic = gmm_test.aic(X_scaled)    # Akaike Information Criterion
    print(f"Cov={cov_type:<12} BIC={bic:>10.1f}  AIC={aic:>10.1f}")
# Lower BIC/AIC = better model
```

### Selecting Number of Components with BIC/AIC

```python
# Use BIC to find optimal number of clusters
bic_scores = []
aic_scores = []
K_range = range(1, 11)

for k in K_range:
    gmm = GaussianMixture(n_components=k, covariance_type='full', 
                           n_init=5, random_state=42)
    gmm.fit(X_scaled)
    bic_scores.append(gmm.bic(X_scaled))
    aic_scores.append(gmm.aic(X_scaled))

optimal_k_bic = list(K_range)[np.argmin(bic_scores)]
optimal_k_aic = list(K_range)[np.argmin(aic_scores)]
print(f"Optimal K (BIC): {optimal_k_bic}")
print(f"Optimal K (AIC): {optimal_k_aic}")

# BIC penalizes complexity more → tends to select simpler models
# AIC is more liberal → may select more components
# In practice, prefer BIC for model selection
```

### GMM for Anomaly Detection

```python
# Points with low probability under the model = anomalies!
gmm = GaussianMixture(n_components=4, covariance_type='full', random_state=42)
gmm.fit(X_scaled)

# Score each sample (log-likelihood)
log_likelihoods = gmm.score_samples(X_scaled)

# Set threshold at 5th percentile
threshold = np.percentile(log_likelihoods, 5)
anomalies = X_scaled[log_likelihoods < threshold]
print(f"Number of anomalies detected: {len(anomalies)} ({100*len(anomalies)/len(X_scaled):.1f}%)")
print(f"Log-likelihood threshold: {threshold:.4f}")
```

---

## Other Clustering Methods

### Mini-Batch K-Means (For Large Data)

```python
from sklearn.cluster import MiniBatchKMeans
import time

# Generate large dataset
X_large, _ = make_blobs(n_samples=100000, centers=10, random_state=42)
X_large_scaled = StandardScaler().fit_transform(X_large)

# Standard K-Means
start = time.time()
km_std = KMeans(n_clusters=10, random_state=42, n_init=3)
km_std.fit(X_large_scaled)
std_time = time.time() - start

# Mini-Batch K-Means (much faster!)
start = time.time()
mbkm = MiniBatchKMeans(
    n_clusters=10,
    batch_size=1024,        # Process 1024 samples at a time
    max_iter=100,
    random_state=42,
    n_init=3
)
mbkm.fit(X_large_scaled)
mb_time = time.time() - start

print(f"Standard K-Means: {std_time:.2f}s, Inertia: {km_std.inertia_:.1f}")
print(f"Mini-Batch K-Means: {mb_time:.2f}s, Inertia: {mbkm.inertia_:.1f}")
print(f"Speedup: {std_time/mb_time:.1f}x")
# Typically 5-10x faster with very similar results!
```

### Spectral Clustering (For Complex Shapes)

```python
from sklearn.cluster import SpectralClustering

# Works great on non-convex clusters
X_circles, y_circles = make_circles(n_samples=500, noise=0.05, factor=0.5, random_state=42)

spectral = SpectralClustering(
    n_clusters=2,
    affinity='nearest_neighbors',  # or 'rbf'
    n_neighbors=10,
    random_state=42
)

spec_labels = spectral.fit_predict(X_circles)
print(f"Spectral Clustering on circles: Silhouette={silhouette_score(X_circles, spec_labels):.4f}")

# Compare with K-Means (fails on circles)
km_circles = KMeans(n_clusters=2, random_state=42)
km_labels_circles = km_circles.fit_predict(X_circles)
print(f"K-Means on circles: Silhouette={silhouette_score(X_circles, km_labels_circles):.4f}")
```

### Mean Shift Clustering

```python
from sklearn.cluster import MeanShift, estimate_bandwidth

# Automatically determines number of clusters
bandwidth = estimate_bandwidth(X_scaled, quantile=0.2)
ms = MeanShift(bandwidth=bandwidth, bin_seeding=True)
ms_labels = ms.fit_predict(X_scaled)

print(f"Mean Shift: {len(np.unique(ms_labels))} clusters found")
print(f"Cluster centers:\n{ms.cluster_centers_}")
```

---

## Cluster Evaluation Metrics

### With Ground Truth (External Metrics)

```python
from sklearn.metrics import (
    adjusted_rand_score,      # ARI: -1 to 1, higher = better
    normalized_mutual_info_score,  # NMI: 0 to 1
    homogeneity_score,        # All cluster members belong to single class
    completeness_score,       # All class members assigned to single cluster
    v_measure_score,          # Harmonic mean of homogeneity + completeness
    fowlkes_mallows_score     # FMI: geometric mean of precision + recall
)

# Assuming y_true (ground truth) and labels (predicted clusters)
km = KMeans(n_clusters=4, random_state=42)
labels = km.fit_predict(X_scaled)

print("External Metrics (require ground truth):")
print(f"  Adjusted Rand Index: {adjusted_rand_score(y_true, labels):.4f}")
print(f"  Normalized MI:       {normalized_mutual_info_score(y_true, labels):.4f}")
print(f"  Homogeneity:         {homogeneity_score(y_true, labels):.4f}")
print(f"  Completeness:        {completeness_score(y_true, labels):.4f}")
print(f"  V-Measure:           {v_measure_score(y_true, labels):.4f}")
print(f"  Fowlkes-Mallows:     {fowlkes_mallows_score(y_true, labels):.4f}")
```

### Without Ground Truth (Internal Metrics)

```python
from sklearn.metrics import (
    silhouette_score,
    calinski_harabasz_score,   # Variance Ratio Criterion
    davies_bouldin_score       # Lower = better
)

print("\nInternal Metrics (no ground truth needed):")
print(f"  Silhouette Score:    {silhouette_score(X_scaled, labels):.4f}")
# Range: [-1, 1]. Higher = better. >0.5 is good, >0.7 is excellent
print(f"  Calinski-Harabasz:   {calinski_harabasz_score(X_scaled, labels):.1f}")
# Higher = better (ratio of between-cluster to within-cluster variance)
print(f"  Davies-Bouldin:      {davies_bouldin_score(X_scaled, labels):.4f}")
# Lower = better (avg similarity between each cluster and its most similar cluster)
```

### Silhouette Score — Deep Dive

```python
from sklearn.metrics import silhouette_samples
import numpy as np

# Per-sample silhouette scores
sample_silhouettes = silhouette_samples(X_scaled, labels)

# Analyze per cluster
print("\nPer-cluster silhouette analysis:")
for k in range(4):
    cluster_silhouettes = sample_silhouettes[labels == k]
    print(f"  Cluster {k}: mean={cluster_silhouettes.mean():.4f}, "
          f"min={cluster_silhouettes.min():.4f}, "
          f"size={len(cluster_silhouettes)}")

# Silhouette formula for a single point i:
# s(i) = (b(i) - a(i)) / max(a(i), b(i))
# where:
#   a(i) = mean distance to all other points in same cluster (cohesion)
#   b(i) = mean distance to points in nearest other cluster (separation)
# s(i) ∈ [-1, 1]
#   +1 = perfectly clustered
#    0 = on boundary between clusters
#   -1 = probably in wrong cluster
```

---

## Common Mistakes

### 1. Not Scaling Features Before Clustering

```python
# ❌ BAD: Features with different scales dominate distance
from sklearn.datasets import load_iris
iris = load_iris()
km_bad = KMeans(n_clusters=3, random_state=42)
km_bad.fit(iris.data)  # Feature ranges: [0-8] vs [0-2.5]

# ✅ GOOD: Always scale for distance-based methods
from sklearn.preprocessing import StandardScaler
X_iris_scaled = StandardScaler().fit_transform(iris.data)
km_good = KMeans(n_clusters=3, random_state=42)
km_good.fit(X_iris_scaled)
```

### 2. Using K-Means for Non-Spherical Clusters

```python
# ❌ BAD: K-Means assumes spherical clusters
X_moons, _ = make_moons(n_samples=500, noise=0.05, random_state=42)
km_moons = KMeans(n_clusters=2).fit(X_moons)  # Will fail!

# ✅ GOOD: Use DBSCAN or Spectral Clustering
dbscan_moons = DBSCAN(eps=0.2, min_samples=5).fit(X_moons)
```

### 3. Not Trying Multiple K Values

```python
# ❌ BAD: Just guessing K=3
km = KMeans(n_clusters=3).fit(X_scaled)

# ✅ GOOD: Systematic evaluation
for k in range(2, 10):
    km = KMeans(n_clusters=k, random_state=42).fit(X_scaled)
    print(f"K={k}: Inertia={km.inertia_:.1f}, Silhouette={silhouette_score(X_scaled, km.labels_):.4f}")
```

### 4. Ignoring Outliers

```python
# ❌ BAD: K-Means is sensitive to outliers (they pull centroids)
# ✅ GOOD: Use DBSCAN (labels outliers as noise) or remove outliers first
# Or use K-Medoids (more robust than K-Means)
```

### 5. Using Silhouette Score as the Only Metric

```python
# Silhouette score favors convex clusters
# It might give low scores for valid non-spherical clusters
# ✅ Use multiple metrics + visual inspection + domain knowledge
```

---

## Interview Questions

### Conceptual Questions

**Q1: How does K-Means differ from K-Medoids?**
> K-Means uses the mean (centroid) as cluster center — can be any point in space. K-Medoids uses an actual data point (medoid) as center — more robust to outliers but slower ($O(n^2)$).

**Q2: Why does K-Means converge to a local minimum? How do you handle it?**
> The objective (WCSS) is non-convex with many local minima depending on initialization. Handle it by: (1) using K-Means++ initialization, (2) running multiple times with different seeds (`n_init=10`), (3) using the best result (lowest inertia).

**Q3: When would you choose DBSCAN over K-Means?**
> When clusters are non-spherical, when you don't know K, when there are outliers/noise in the data, and when clusters have varying densities (though HDBSCAN is better for varying densities).

**Q4: Explain the difference between hard and soft clustering.**
> Hard clustering (K-Means, DBSCAN): each point belongs to exactly one cluster. Soft clustering (GMM): each point has a probability of belonging to each cluster. Soft clustering is better when cluster boundaries are fuzzy.

**Q5: What is the time complexity of K-Means vs. Hierarchical vs. DBSCAN?**
> - K-Means: $O(n \cdot K \cdot d \cdot I)$ — linear in n
> - Hierarchical: $O(n^2 \log n)$ time, $O(n^2)$ space — doesn't scale
> - DBSCAN: $O(n \log n)$ with spatial index, $O(n^2)$ worst case

**Q6: How do you cluster high-dimensional data?**
> Apply dimensionality reduction first (PCA, UMAP), then cluster. High-dimensional spaces make distances meaningless ("curse of dimensionality"). Alternatively, use subspace clustering or spectral methods.

**Q7: What is the silhouette score and what does a negative value mean?**
> Silhouette measures how similar a point is to its own cluster vs. the nearest other cluster. Range [-1, 1]. Negative means the point is likely assigned to the wrong cluster (closer to another cluster's center).

**Q8: How would you handle clustering with mixed data types (numerical + categorical)?**
> Options: (1) K-Prototypes algorithm (combines K-Means + K-Modes), (2) Gower distance with hierarchical clustering, (3) Encode categoricals and use standard algorithms, (4) Use CatBoost-style target encoding if you have a proxy target.

---

## Quick Reference

### Algorithm Comparison

| Algorithm | Cluster Shape | # Clusters | Outliers | Scalability | Key Parameter |
|-----------|--------------|------------|----------|-------------|---------------|
| K-Means | Spherical | Must specify K | No handling | $O(n)$ | `n_clusters` |
| Hierarchical | Any | Flexible (cut tree) | No handling | $O(n^2)$ | `linkage`, `n_clusters` |
| DBSCAN | Any shape | Auto-detected | Labels as noise | $O(n \log n)$ | `eps`, `min_samples` |
| HDBSCAN | Any shape | Auto-detected | Labels as noise | $O(n \log n)$ | `min_cluster_size` |
| GMM | Ellipsoidal | Must specify K | Via low probability | $O(n)$ | `n_components`, `covariance_type` |
| Spectral | Any shape | Must specify K | No handling | $O(n^3)$ | `n_clusters`, `affinity` |
| Mean Shift | Any shape | Auto-detected | No handling | $O(n^2)$ | `bandwidth` |

### When to Use What

| Situation | Use This |
|-----------|----------|
| Quick baseline, spherical clusters | K-Means |
| Non-spherical clusters, unknown K | DBSCAN / HDBSCAN |
| Need probabilities, soft assignment | GMM |
| Need hierarchy / dendrogram | Hierarchical (Ward) |
| Very large dataset (>100K) | Mini-Batch K-Means or LightGBM |
| Complex shapes, small data | Spectral Clustering |
| High-dimensional data | PCA → K-Means or UMAP → HDBSCAN |

### Key Takeaways

- **Always scale** your data before clustering (except tree-based)
- **K-Means** is fast but assumes spherical clusters of equal size
- **DBSCAN** finds arbitrary shapes and handles noise, but struggles with varying densities
- **GMM** gives probabilistic assignments and is the Bayesian version of K-Means
- **Silhouette Score** is the go-to internal metric, but always validate visually
- **There is no "best" clustering algorithm** — it depends on your data structure
- In practice, try 2-3 algorithms and compare results

---

*Previous: [07-Ensemble-Methods.md](07-Ensemble-Methods.md) | Next: [09-Dimensionality-Reduction.md](09-Dimensionality-Reduction.md)*
