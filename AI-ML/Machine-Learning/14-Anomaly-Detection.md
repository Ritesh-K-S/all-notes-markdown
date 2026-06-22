# Chapter 14: Anomaly Detection

## Table of Contents
- [What it is](#what-it-is)
- [Why it matters](#why-it-matters)
- [Types of Anomalies](#types-of-anomalies)
- [Statistical Methods](#statistical-methods)
- [Distance-Based Methods](#distance-based-methods)
- [Isolation Forest](#isolation-forest)
- [One-Class SVM](#one-class-svm)
- [Local Outlier Factor (LOF)](#local-outlier-factor-lof)
- [DBSCAN for Anomaly Detection](#dbscan-for-anomaly-detection)
- [Autoencoder-Based Detection](#autoencoder-based-detection)
- [Time Series Anomaly Detection](#time-series-anomaly-detection)
- [Evaluation Metrics](#evaluation-metrics)
- [Production Considerations](#production-considerations)
- [Code Examples](#code-examples)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What it is

**Anomaly detection** (also called outlier detection) is the identification of data points, events, or observations that deviate significantly from the expected pattern in a dataset.

**Simple Analogy:** Imagine you're a teacher grading exams. Most students score between 50-90. If one student scores 3 and another scores 100, both are "anomalies" — they deviate from what's normal. Anomaly detection algorithms do this automatically across millions of data points and hundreds of dimensions.

**Key Insight:** Anomalies are NOT always errors or bad data. They're often the most interesting and valuable data points:
- A fraudulent transaction (bad anomaly)
- A viral social media post (interesting anomaly)
- An early disease indicator (critical anomaly)
- A manufacturing defect (actionable anomaly)

---

## Why it matters

### Real-World Applications

| Domain | Application | Impact |
|--------|-------------|--------|
| Finance | Fraud detection | Saves billions in losses annually |
| Cybersecurity | Intrusion detection | Catches novel attacks |
| Healthcare | Disease diagnosis | Early detection saves lives |
| Manufacturing | Defect detection | Reduces waste, improves quality |
| IT Operations | System monitoring | Prevents outages (AIOps) |
| E-commerce | Bot detection | Protects from fake reviews/clicks |
| IoT | Sensor failure detection | Predictive maintenance |
| Insurance | Claim fraud | Reduces false claims |

### When You'd Use Anomaly Detection
- When normal data is abundant but anomalies are rare (<1-5%)
- When you can't easily define what "abnormal" looks like
- When new types of anomalies may emerge (zero-day attacks)
- When labeling data is expensive or impossible
- When the cost of missing an anomaly is very high

### Why Not Just Classification?
| Aspect | Classification | Anomaly Detection |
|--------|---------------|-------------------|
| Labels needed | Both classes | Only normal (or none) |
| Class balance | Can handle imbalance | Designed for extreme imbalance |
| New anomaly types | Misses unknown classes | Catches novel anomalies |
| Training data | Both normal + anomalous | Mostly/only normal data |

---

## Types of Anomalies

```
┌─────────────────────────────────────────────────────────────┐
│                    TYPES OF ANOMALIES                         │
├───────────────────┬─────────────────┬───────────────────────┤
│   Point Anomaly   │ Contextual      │ Collective Anomaly    │
│                   │ Anomaly         │                       │
├───────────────────┼─────────────────┼───────────────────────┤
│ A single data     │ Normal in one   │ Group of data points  │
│ point deviates    │ context, anomaly│ that together form    │
│ from the rest     │ in another      │ an anomaly            │
│                   │                 │                       │
│ Example:          │ Example:        │ Example:              │
│ $10,000 purchase  │ 30°C in summer  │ Repeated failed       │
│ on credit card    │ is normal, but  │ login attempts        │
│ usually spending  │ 30°C in winter  │ (each one normal,     │
│ $50-200           │ is anomalous    │ pattern is anomalous) │
└───────────────────┴─────────────────┴───────────────────────┘
```

### Anomaly Detection Approaches

```
┌─────────────────────────────────────────────────────────┐
│              ANOMALY DETECTION TAXONOMY                   │
├──────────────────┬──────────────────┬───────────────────┤
│   Supervised     │  Semi-supervised │   Unsupervised    │
│                  │                  │                   │
│ Has labels for   │ Only normal data │ No labels at all  │
│ both normal &    │ for training     │                   │
│ anomalous        │                  │                   │
│                  │                  │                   │
│ • Random Forest  │ • One-Class SVM  │ • Isolation Forest│
│ • XGBoost        │ • Autoencoders   │ • LOF             │
│ • Neural Nets    │ • GMMs           │ • DBSCAN          │
│                  │                  │ • Statistical     │
└──────────────────┴──────────────────┴───────────────────┘
```

---

## Statistical Methods

### Z-Score Method

**Intuition:** If a data point is more than 3 standard deviations from the mean, it's likely an anomaly.

$$z = \frac{x - \mu}{\sigma}$$

**Rule of thumb:** $|z| > 3$ → anomaly (99.7% of data lies within 3σ)

```
Normal Distribution:
        ┌──────────────────────┐
        │         █            │
        │        ███           │
        │       █████          │
        │      ███████         │
        │     █████████        │
        │    ███████████       │
   ─────┼───████████████──────┼─────
     -3σ   -2σ  -1σ  μ  1σ  2σ  3σ
        │   ←── 99.7% ──→    │
        │                     │
   ANOMALY                  ANOMALY
```

**Limitation:** Assumes normal distribution. Fails for skewed/multimodal data.

### Modified Z-Score (Robust)

Uses **median** and **MAD** (Median Absolute Deviation) instead of mean/std:

$$\text{Modified Z} = \frac{0.6745 \cdot (x_i - \text{median})}{\text{MAD}}$$

$$\text{MAD} = \text{median}(|x_i - \text{median}(x)|)$$

**Why better?** Mean and std are heavily influenced by outliers themselves. Median is robust.

### IQR (Interquartile Range) Method

$$IQR = Q3 - Q1$$

- Lower fence: $Q1 - 1.5 \times IQR$
- Upper fence: $Q3 + 1.5 \times IQR$

Points outside fences are anomalies. This is what box plots use.

### Grubbs' Test

Tests whether the most extreme value in a **normally distributed** dataset is an outlier:

$$G = \frac{\max|x_i - \bar{x}|}{s}$$

Compare G with critical value from Grubbs' table at significance level α.

### Mahalanobis Distance (Multivariate)

For detecting anomalies in multi-dimensional data, accounting for correlations:

$$D_M(x) = \sqrt{(x - \mu)^T \Sigma^{-1} (x - \mu)}$$

Where $\Sigma$ is the covariance matrix. This measures distance accounting for the "shape" of the data distribution.

**Why not Euclidean?**
```
Euclidean distance:           Mahalanobis distance:
(treats all directions        (accounts for correlation)
 equally)

    ●                              ●
   / \                            /   \
  /   \  ← Circle               /     \ ← Ellipse
 /  ●  \    (same distance     /   ●   \   (adjusted for
 \     /     in all             \       /    correlation)
  \   /      directions)         \     /
   \ /                            \   /
    ●                              ●
```

---

## Distance-Based Methods

### K-Nearest Neighbors (KNN) for Anomaly Detection

**Intuition:** Anomalies are far from their nearest neighbors.

**Approach 1: Distance to k-th nearest neighbor**
- Compute distance to k-th nearest neighbor for each point
- Points with large distances are anomalies

**Approach 2: Average distance to k nearest neighbors**
$$\text{anomaly\_score}(x) = \frac{1}{k} \sum_{i=1}^{k} d(x, x_i^{NN})$$

**Pros:** Simple, no distribution assumptions
**Cons:** Computationally expensive O(n²), sensitive to k, struggles with varying density

---

## Isolation Forest

### What it is
An ensemble method that isolates anomalies by randomly partitioning the data. Based on the principle that **anomalies are few and different — they're easier to isolate**.

### Intuition

**Analogy:** Imagine playing "20 questions" to find a specific person in a crowd. If someone is very different from everyone else (e.g., wearing a clown costume at a business conference), you can identify them in very few questions. Normal people take many questions to isolate.

```
Normal Point (deep in tree):     Anomaly (shallow in tree):

         ┌───┐                          ┌───┐
         │ ? │                          │ ? │
        /     \                        /     \
      ┌───┐  ┌───┐                  ┌───┐  ★ Isolated!
      │ ? │  │ ? │                  │ ? │    (only 1 split)
     /   \   /   \                 /     \
   ┌───┐  • •   ┌───┐           •       •
   │ ? │        │ ? │
  /     \      /     \
 •      ●    •       •        ★ = anomaly (short path)
               ↑               ● = normal point (long path)
        normal point
        (many splits to isolate)
```

### How it Works

1. **Build Isolation Trees:**
   - Randomly select a feature
   - Randomly select a split value between min and max of that feature
   - Recursively partition until each point is isolated (or max depth reached)

2. **Calculate Path Length:**
   - For each point, count how many splits it takes to isolate it
   - Average across all trees in the forest

3. **Compute Anomaly Score:**

$$s(x, n) = 2^{-\frac{E(h(x))}{c(n)}}$$

Where:
- $E(h(x))$ = average path length of point $x$ across all trees
- $c(n)$ = average path length of unsuccessful search in BST (normalization factor)
- $c(n) = 2H(n-1) - \frac{2(n-1)}{n}$, where $H(i) = \ln(i) + 0.5772$ (Euler constant)

**Score interpretation:**
- Score close to 1 → anomaly (short path length)
- Score close to 0.5 → normal (average path length)
- Score close to 0 → not anomalous at all (long path, very "normal")

### Advantages
- **Linear time complexity:** O(n log n) for training, O(n) for scoring
- **No distance computation:** Doesn't need pairwise distances
- **Handles high dimensions:** Random feature selection works well
- **No distribution assumptions:** Works on any data shape
- **Memory efficient:** Subsampling makes it scalable

### Hyperparameters
| Parameter | Typical Value | Effect |
|-----------|--------------|--------|
| `n_estimators` | 100-300 | More trees = more stable (but slower) |
| `max_samples` | 256 or 'auto' | Subsample size per tree (256 works surprisingly well) |
| `contamination` | 0.01-0.1 | Expected proportion of anomalies |
| `max_features` | 1.0 | Fraction of features per tree |

> **Key Insight:** Isolation Forest works well with just 256 samples per tree because anomalies are so distinct that even small samples reveal them. This also makes it very fast.

---

## One-Class SVM

### What it is
Learns a decision boundary around "normal" data in high-dimensional space. New points outside the boundary are classified as anomalies.

### Intuition

**Analogy:** Imagine drawing the tightest possible circle (or shape in higher dimensions) around a cluster of normal points. Anything outside that circle is anomalous.

```
Feature space (with RBF kernel):

    ●  ●  ●             ★ = anomaly (outside boundary)
  ●  ●  ●  ●           ● = normal
 ●  ●  ●  ●  ●
  ●  ●  ●  ●       ★
    ●  ●  ●
                    ★
     ↑
  Decision boundary
  (learned hyperplane in
   kernel-transformed space)
```

### Mathematical Formulation

One-Class SVM maps data to a feature space and finds the hyperplane with maximum margin from the origin:

$$\min_{w, \xi, \rho} \frac{1}{2}\|w\|^2 + \frac{1}{\nu n}\sum_{i=1}^{n}\xi_i - \rho$$

Subject to: $(w \cdot \Phi(x_i)) \geq \rho - \xi_i$, $\xi_i \geq 0$

Where:
- $\nu$ ∈ (0, 1]: upper bound on fraction of outliers AND lower bound on fraction of support vectors
- $\Phi(x)$: kernel mapping to higher-dimensional space
- $\rho$: offset (distance of hyperplane from origin)
- $\xi_i$: slack variables (allow some violations)

### Kernel Choice

| Kernel | Formula | Best For |
|--------|---------|----------|
| RBF (default) | $K(x,y) = \exp(-\gamma\|x-y\|^2)$ | Non-linear boundaries, general use |
| Linear | $K(x,y) = x^T y$ | High-dimensional sparse data |
| Polynomial | $K(x,y) = (\gamma x^T y + r)^d$ | When feature interactions matter |

### Key Parameters
- **`nu` (ν):** Controls sensitivity. Lower ν = fewer anomalies expected. Acts as upper bound on anomaly fraction.
- **`gamma`:** RBF kernel width. High γ = tight boundary (overfitting risk). Low γ = loose boundary (underfitting).
- **`kernel`:** RBF works best in most cases.

### When to Use
- Clean normal training data available
- Low-to-medium dimensional data (scales poorly to very high dims)
- Non-linear decision boundary needed
- Small to medium datasets

---

## Local Outlier Factor (LOF)

### What it is
Measures the **local density** of a point compared to its neighbors. Points in low-density regions compared to their neighbors are anomalies.

### Intuition

**Analogy:** Imagine a city with dense neighborhoods and suburbs. A house far from others in a dense neighborhood is suspicious (anomaly), but a house far from others in a rural area is normal. LOF accounts for **varying density**.

```
Cluster 1 (dense):     Cluster 2 (sparse):

  ●●●●                    ●     ●
  ●●●●●                 ●    ●
  ●●★●●                   ●       ●
  ●●●●●                 ●     ●
  ●●●●

★ is anomalous here    BUT the same distance
(far from dense         would be NORMAL here
 neighbors)             (neighbors are also sparse)
```

### How it Works

**Step 1: Compute k-distance and k-distance neighborhood**

$k\text{-distance}(A)$ = distance from A to its k-th nearest neighbor

**Step 2: Compute Reachability Distance**

$$\text{reach-dist}_k(A, B) = \max(k\text{-distance}(B), d(A, B))$$

This smooths out statistical fluctuations — nearby points within B's k-distance are treated as equally close.

**Step 3: Compute Local Reachability Density (LRD)**

$$LRD_k(A) = \frac{1}{\frac{\sum_{B \in N_k(A)} \text{reach-dist}_k(A, B)}{|N_k(A)|}}$$

This is the inverse of the average reachability distance to k neighbors.

**Step 4: Compute LOF Score**

$$LOF_k(A) = \frac{\sum_{B \in N_k(A)} \frac{LRD_k(B)}{LRD_k(A)}}{|N_k(A)|}$$

**Interpretation:**
- $LOF \approx 1$: Point has similar density to its neighbors (normal)
- $LOF > 1$: Point is in a less dense region than neighbors (potential anomaly)
- $LOF \gg 1$: Strong anomaly (much less dense than surroundings)
- $LOF < 1$: Denser than neighbors (very normal)

### Advantages over Global Methods
- Handles clusters of different densities
- Detects local anomalies that global methods miss
- Provides a continuous anomaly score (not just binary)

### Disadvantage
- O(n²) for computing all pairwise distances (use KD-tree for speedup)
- Sensitive to choice of k
- Not great for high-dimensional data (curse of dimensionality)

---

## DBSCAN for Anomaly Detection

### How it Works for Anomalies

DBSCAN naturally identifies anomalies as **noise points** — points that don't belong to any cluster.

```
  ●●●●●●    ←── Cluster 1 (dense enough)
  ●●●●●●

                    ★  ←── Noise point (anomaly!)

       ●●●●    ←── Cluster 2
       ●●●●●
                           ★  ←── Another anomaly
```

**Parameters:**
- `eps` (ε): Neighborhood radius
- `min_samples`: Minimum points to form a dense region

**Point Types:**
- **Core point:** Has ≥ min_samples within eps radius
- **Border point:** Within eps of a core point but fewer than min_samples neighbors
- **Noise point (ANOMALY):** Neither core nor border

### When DBSCAN Works for Anomaly Detection
- Clusters have roughly similar density
- You want cluster assignments AND anomaly labels simultaneously
- Anomalies are isolated points far from all clusters

---

## Autoencoder-Based Detection

### What it is
Train a neural network to compress and reconstruct normal data. When it encounters anomalous data, reconstruction will be poor (high error) because the model only learned the patterns of normal data.

### Intuition

**Analogy:** Imagine a photocopier trained only on photos of cats. If you feed it a photo of a dog, the copy will look wrong — maybe it produces something cat-like that doesn't match the input. The "mismatch" between input and output tells you the input was anomalous.

```
Normal data:                     Anomalous data:
Input: ●●●●●                    Input: ★★★★★
       ↓                                ↓
Encoder: ●●● (compressed)       Encoder: ●●● (compressed)
       ↓                                ↓
Decoder: ●●●●● (reconstructed)  Decoder: ●●●●● (poor reconstruction)
       ↓                                ↓
Error: LOW (good match)          Error: HIGH (bad match = ANOMALY!)
```

### Architecture

```
         Input Layer (d dimensions)
              │
    ┌─────────┴─────────┐
    │   Encoder          │
    │   d → 128 → 64    │
    │                    │
    │   Latent Space     │    ← Bottleneck (forces compression)
    │   (32 dims)        │
    │                    │
    │   Decoder          │
    │   32 → 64 → 128   │
    └─────────┬─────────┘
              │
         Output Layer (d dimensions)
              │
    Reconstruction Error = ||Input - Output||²
              │
    If error > threshold → ANOMALY
```

### Mathematical Foundation

**Training Objective:**
$$\mathcal{L} = \frac{1}{n}\sum_{i=1}^{n}\|x_i - \hat{x}_i\|^2$$

Where $\hat{x}_i = \text{Decoder}(\text{Encoder}(x_i))$

**Anomaly Score:**
$$\text{score}(x) = \|x - \hat{x}\|^2$$

**Threshold Selection:**
- Use percentile of reconstruction errors on validation (normal) data
- Example: threshold = 95th percentile of training errors

### Variational Autoencoder (VAE) for Anomaly Detection

VAE adds a probabilistic interpretation:

$$\text{score}(x) = -\text{ELBO}(x) = -\mathbb{E}_{q(z|x)}[\log p(x|z)] + D_{KL}(q(z|x) \| p(z))$$

Anomalies have low ELBO (can't be well-explained by the learned distribution).

### When to Use Autoencoders
- High-dimensional data (images, sequences, tabular with many features)
- Complex nonlinear patterns in normal data
- Large training dataset of normal examples
- Other methods fail due to curse of dimensionality

### Variants
| Variant | Modification | Best For |
|---------|-------------|----------|
| Vanilla AE | Standard encoder-decoder | Tabular data |
| Convolutional AE | CNN layers | Images |
| LSTM AE | Recurrent layers | Sequences/Time series |
| VAE | Probabilistic latent space | Generative + detection |
| Denoising AE | Add noise to input, reconstruct clean | Robust to noise |

---

## Time Series Anomaly Detection

### Types of Time Series Anomalies

```
Value:
  │    
  │         ★ Point anomaly (spike)
  │        /\
  │   ────/  \────
  │  /              \
  │ /    ~~~~~~~~~~~~\ ←── Contextual anomaly
  │/     (normal value      (same value, wrong time)
  │       wrong time)
  ├───────────────────────── Time
  
  │
  │   ──────    ──────
  │  /      \  /      \     ┌──────────────┐
  │ /        \/        \    │ Collective    │
  │                     ────│ anomaly       │
  │                         │ (sustained    │
  │                         │  deviation)   │
  ├─────────────────────────┴──────────────── Time
```

### Methods for Time Series

#### 1. Moving Average + Threshold
$$\text{anomaly if } |x_t - \text{MA}(x, w)| > k \cdot \sigma_{\text{rolling}}$$

#### 2. Exponential Moving Average (EMA)
$$\text{EMA}_t = \alpha \cdot x_t + (1 - \alpha) \cdot \text{EMA}_{t-1}$$

#### 3. STL Decomposition + Residual Analysis
Decompose: $x_t = \text{Trend}_t + \text{Seasonal}_t + \text{Residual}_t$

Anomalies live in the residual component.

#### 4. Prophet (Facebook)
- Automatic changepoint and seasonality detection
- Built-in uncertainty intervals
- Anomalies = points outside prediction intervals

#### 5. LSTM Autoencoder
- Train LSTM to predict next N steps
- High prediction error = anomaly

---

## Evaluation Metrics

### The Challenge: Imbalanced Data

In anomaly detection, anomalies are typically <1-5% of data. **Accuracy is meaningless** — a model predicting "everything is normal" gets 95%+ accuracy.

### Key Metrics

#### Precision and Recall
$$\text{Precision} = \frac{TP}{TP + FP} = \text{"Of detected anomalies, how many are real?"}$$

$$\text{Recall} = \frac{TP}{TP + FN} = \text{"Of real anomalies, how many did we catch?"}$$

#### F1 Score
$$F1 = 2 \cdot \frac{\text{Precision} \cdot \text{Recall}}{\text{Precision} + \text{Recall}}$$

#### Precision-Recall AUC (PR-AUC)
Better than ROC-AUC for highly imbalanced data:
- ROC-AUC can be misleadingly high due to many true negatives
- PR-AUC focuses on the positive (anomaly) class

#### ROC-AUC
Still useful when contamination isn't extreme (<5% anomalies).

### Business-Specific Metrics

| Context | Optimize For | Why |
|---------|-------------|-----|
| Fraud detection | High recall | Missing fraud is costly |
| Manufacturing QC | Balance | Both false alarms and missed defects are expensive |
| Spam detection | High precision | False positives annoy users |
| Medical diagnosis | High recall | Missing disease is dangerous |
| Intrusion detection | High recall + manageable FP rate | Must catch attacks but alert fatigue is real |

### Threshold Selection Strategies

1. **Fixed percentile:** Top 1% of anomaly scores (when you know contamination rate)
2. **Business-driven:** Set threshold where cost of FP × #FP = cost of FN × #FN
3. **Knee/elbow method:** Plot sorted scores, find the "bend"
4. **Statistical:** Mean + 3×std of scores on validation normal data

---

## Production Considerations

### Architecture for Real-Time Anomaly Detection

```
┌─────────────────────────────────────────────────────────────┐
│              PRODUCTION ANOMALY DETECTION                     │
│                                                              │
│  ┌──────────┐    ┌──────────────┐    ┌─────────────────┐   │
│  │ Data     │    │  Feature     │    │  Model          │   │
│  │ Ingestion│───>│  Engineering │───>│  Inference      │   │
│  │ (Kafka)  │    │  (Flink/     │    │  (Batch/Online) │   │
│  └──────────┘    │   Spark)     │    └────────┬────────┘   │
│                  └──────────────┘             │             │
│                                     ┌────────┴────────┐    │
│                                     │  Alert Engine   │    │
│                                     │  (threshold +   │    │
│                                     │   rate limiting) │    │
│                                     └────────┬────────┘    │
│                                              │             │
│                                     ┌────────┴────────┐    │
│                                     │  Response       │    │
│                                     │  (alert/block/  │    │
│                                     │   quarantine)   │    │
│                                     └─────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  FEEDBACK LOOP: Analyst reviews → labels → retrain   │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Key Challenges

1. **Concept Drift:** What's "normal" changes over time
   - Solution: Retrain periodically, use sliding windows, adaptive thresholds

2. **Alert Fatigue:** Too many false positives → analysts ignore alerts
   - Solution: Ensemble methods, score calibration, alert aggregation

3. **Explainability:** "WHY is this anomalous?"
   - Solution: Feature attribution (SHAP), rule extraction, human-readable explanations

4. **Latency:** Real-time detection needed (milliseconds)
   - Solution: Pre-compute features, lightweight models, feature stores

5. **Adversarial Anomalies:** Attackers evolve to evade detection
   - Solution: Regular retraining, ensemble diversity, adversarial training

### Model Selection Decision Tree

```
Start
  │
  ├─ Have labeled anomalies? ──Yes──> Supervised (XGBoost, Random Forest)
  │                                    Best accuracy if labels are good
  │
  └─ No labels
      │
      ├─ Have clean normal data? ──Yes──> Semi-supervised
      │                                    (One-Class SVM, Autoencoder)
      │
      └─ No clean data either ──> Unsupervised
          │                        (Isolation Forest, LOF)
          │
          ├─ High dimensional? ──Yes──> Isolation Forest or Autoencoder
          │
          ├─ Varying density? ──Yes──> LOF or DBSCAN
          │
          └─ Need speed? ──Yes──> Isolation Forest
```

---

## Code Examples

### Example 1: Statistical Methods

```python
import numpy as np
import pandas as pd
from scipy import stats

# Generate sample data with anomalies
np.random.seed(42)
normal_data = np.random.normal(loc=50, scale=10, size=1000)
anomalies = np.array([120, 130, -20, -30, 150])  # Injected anomalies
data = np.concatenate([normal_data, anomalies])
np.random.shuffle(data)

# --- Method 1: Z-Score ---
def detect_zscore(data, threshold=3):
    """Detect anomalies using Z-score method."""
    mean = np.mean(data)
    std = np.std(data)
    z_scores = np.abs((data - mean) / std)
    anomalies_mask = z_scores > threshold
    return anomalies_mask, z_scores

z_mask, z_scores = detect_zscore(data)
print(f"Z-Score Method: {z_mask.sum()} anomalies detected")
# Output: Z-Score Method: 5 anomalies detected

# --- Method 2: Modified Z-Score (Robust) ---
def detect_modified_zscore(data, threshold=3.5):
    """Detect anomalies using Modified Z-score (MAD-based)."""
    median = np.median(data)
    mad = np.median(np.abs(data - median))
    
    if mad == 0:  # Avoid division by zero
        mad = np.mean(np.abs(data - median))
    
    modified_z = 0.6745 * (data - median) / mad
    anomalies_mask = np.abs(modified_z) > threshold
    return anomalies_mask, modified_z

mod_z_mask, mod_z_scores = detect_modified_zscore(data)
print(f"Modified Z-Score: {mod_z_mask.sum()} anomalies detected")

# --- Method 3: IQR Method ---
def detect_iqr(data, multiplier=1.5):
    """Detect anomalies using IQR method."""
    Q1 = np.percentile(data, 25)
    Q3 = np.percentile(data, 75)
    IQR = Q3 - Q1
    
    lower_fence = Q1 - multiplier * IQR
    upper_fence = Q3 + multiplier * IQR
    
    anomalies_mask = (data < lower_fence) | (data > upper_fence)
    return anomalies_mask, lower_fence, upper_fence

iqr_mask, lower, upper = detect_iqr(data)
print(f"IQR Method: {iqr_mask.sum()} anomalies detected")
print(f"  Fences: [{lower:.1f}, {upper:.1f}]")

# --- Method 4: Mahalanobis Distance (Multivariate) ---
def detect_mahalanobis(data_2d, threshold_percentile=97.5):
    """Detect multivariate anomalies using Mahalanobis distance."""
    mean = np.mean(data_2d, axis=0)
    cov = np.cov(data_2d, rowvar=False)
    cov_inv = np.linalg.inv(cov)
    
    distances = []
    for point in data_2d:
        diff = point - mean
        dist = np.sqrt(diff @ cov_inv @ diff)
        distances.append(dist)
    
    distances = np.array(distances)
    # Chi-squared distribution for threshold
    threshold = np.percentile(distances, threshold_percentile)
    anomalies_mask = distances > threshold
    
    return anomalies_mask, distances

# Create 2D data for Mahalanobis
data_2d = np.column_stack([
    np.random.normal(0, 1, 200),
    np.random.normal(0, 1, 200)
])
# Add correlated anomalies
data_2d = np.vstack([data_2d, [[5, 5], [-4, -4], [6, -3]]])

maha_mask, maha_dist = detect_mahalanobis(data_2d)
print(f"\nMahalanobis: {maha_mask.sum()} anomalies detected")
```

### Example 2: Isolation Forest

```python
import numpy as np
import pandas as pd
from sklearn.ensemble import IsolationForest
from sklearn.preprocessing import StandardScaler
import matplotlib.pyplot as plt

# Generate 2D data with anomalies for visualization
np.random.seed(42)

# Normal data: two clusters
cluster1 = np.random.normal(loc=[2, 2], scale=[0.5, 0.5], size=(200, 2))
cluster2 = np.random.normal(loc=[-2, -2], scale=[0.5, 0.5], size=(200, 2))
normal = np.vstack([cluster1, cluster2])

# Anomalies: scattered points
anomalies = np.random.uniform(low=-6, high=6, size=(20, 2))

# Combine
X = np.vstack([normal, anomalies])
y_true = np.array([0]*400 + [1]*20)  # 0=normal, 1=anomaly

# --- Train Isolation Forest ---
iso_forest = IsolationForest(
    n_estimators=200,         # Number of trees
    max_samples='auto',       # Subsample size (min(256, n_samples))
    contamination=0.05,       # Expected anomaly proportion
    max_features=1.0,         # Use all features per tree
    random_state=42,
    n_jobs=-1                 # Use all CPU cores
)

# Fit and predict
# Returns: 1 for normal, -1 for anomaly
predictions = iso_forest.fit_predict(X)

# Get anomaly scores (lower = more anomalous)
scores = iso_forest.decision_function(X)
# Negate so higher = more anomalous (more intuitive)
anomaly_scores = -scores

# Convert predictions: 1 → 0 (normal), -1 → 1 (anomaly)
pred_labels = (predictions == -1).astype(int)

# Evaluate
from sklearn.metrics import classification_report, confusion_matrix
print("Isolation Forest Results:")
print(classification_report(y_true, pred_labels, target_names=['Normal', 'Anomaly']))
print(f"Confusion Matrix:\n{confusion_matrix(y_true, pred_labels)}")

# --- Feature Importance (which features contribute most) ---
# Get average path length for each sample across all trees
print(f"\nTop 5 most anomalous points (indices): {np.argsort(anomaly_scores)[-5:]}")
print(f"Their scores: {np.sort(anomaly_scores)[-5:]}")

# --- Tuning Contamination ---
# If you don't know the contamination rate:
print("\n--- Effect of Contamination Parameter ---")
for cont in [0.01, 0.05, 0.1, 0.2]:
    iso = IsolationForest(contamination=cont, random_state=42)
    preds = iso.fit_predict(X)
    n_detected = (preds == -1).sum()
    print(f"  contamination={cont:.2f}: {n_detected} anomalies detected "
          f"({n_detected/len(X)*100:.1f}%)")
```

### Example 3: One-Class SVM

```python
import numpy as np
from sklearn.svm import OneClassSVM
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import classification_report
from sklearn.model_selection import ParameterGrid

# Generate training data (ONLY normal samples)
np.random.seed(42)
X_train_normal = np.random.normal(loc=[0, 0], scale=[1, 1], size=(500, 2))

# Test data: mix of normal and anomalous
X_test_normal = np.random.normal(loc=[0, 0], scale=[1, 1], size=(100, 2))
X_test_anomaly = np.random.uniform(low=-6, high=6, size=(20, 2))
X_test = np.vstack([X_test_normal, X_test_anomaly])
y_test = np.array([1]*100 + [-1]*20)  # 1=normal, -1=anomaly (sklearn convention)

# Scale features (important for SVM!)
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train_normal)
X_test_scaled = scaler.transform(X_test)

# --- Train One-Class SVM ---
oc_svm = OneClassSVM(
    kernel='rbf',        # RBF kernel for non-linear boundary
    gamma='scale',       # 1 / (n_features * X.var()), auto-scales
    nu=0.05,            # Upper bound on fraction of outliers in training
)
oc_svm.fit(X_train_scaled)

# Predict: 1 = normal, -1 = anomaly
predictions = oc_svm.predict(X_test_scaled)

# Evaluate
print("One-Class SVM Results:")
print(classification_report(y_test, predictions, target_names=['Anomaly', 'Normal']))

# Decision function: signed distance to boundary
# Negative = outside boundary (anomaly)
decision_scores = oc_svm.decision_function(X_test_scaled)
print(f"\nMost anomalous score: {decision_scores.min():.4f}")
print(f"Most normal score: {decision_scores.max():.4f}")

# --- Hyperparameter Sensitivity ---
print("\n--- Nu Parameter Sensitivity ---")
for nu in [0.01, 0.05, 0.1, 0.2]:
    model = OneClassSVM(kernel='rbf', gamma='scale', nu=nu)
    model.fit(X_train_scaled)
    preds = model.predict(X_test_scaled)
    n_anomalies = (preds == -1).sum()
    print(f"  nu={nu:.2f}: {n_anomalies} anomalies detected out of {len(X_test)}")

print("\n--- Gamma Parameter Sensitivity ---")
for gamma in [0.01, 0.1, 1.0, 10.0]:
    model = OneClassSVM(kernel='rbf', gamma=gamma, nu=0.05)
    model.fit(X_train_scaled)
    preds = model.predict(X_test_scaled)
    n_anomalies = (preds == -1).sum()
    print(f"  gamma={gamma:.2f}: {n_anomalies} anomalies detected")
```

### Example 4: Local Outlier Factor (LOF)

```python
import numpy as np
from sklearn.neighbors import LocalOutlierFactor
from sklearn.datasets import make_blobs
from sklearn.metrics import classification_report

# Create data with varying density clusters
np.random.seed(42)

# Dense cluster
dense_cluster = np.random.normal(loc=[0, 0], scale=[0.3, 0.3], size=(200, 2))
# Sparse cluster
sparse_cluster = np.random.normal(loc=[5, 5], scale=[1.5, 1.5], size=(100, 2))
# Anomalies
anomalies = np.array([[2.5, 2.5], [-3, -3], [8, 0], [0, 8], [-2, 5]])

X = np.vstack([dense_cluster, sparse_cluster, anomalies])
y_true = np.array([0]*300 + [1]*5)  # 0=normal, 1=anomaly

# --- LOF Detection ---
# novelty=False: used for outlier detection (training data contains outliers)
# novelty=True: used for novelty detection (training data is clean)
lof = LocalOutlierFactor(
    n_neighbors=20,          # Number of neighbors for density estimation
    contamination=0.02,      # Expected outlier fraction
    metric='euclidean',      # Distance metric
    novelty=False            # Outlier detection mode
)

# fit_predict returns 1 (inlier) or -1 (outlier)
predictions = lof.fit_predict(X)

# Get LOF scores (more negative = more anomalous)
lof_scores = lof.negative_outlier_factor_

# Convert for evaluation
pred_labels = (predictions == -1).astype(int)
print("LOF Results:")
print(classification_report(y_true, pred_labels, target_names=['Normal', 'Anomaly']))

# Examine scores
print(f"\nLOF Score Distribution:")
print(f"  Normal points - Mean: {lof_scores[y_true==0].mean():.3f}, "
      f"Min: {lof_scores[y_true==0].min():.3f}")
print(f"  Anomaly points - Mean: {lof_scores[y_true==1].mean():.3f}, "
      f"Min: {lof_scores[y_true==1].min():.3f}")

# --- Novelty Detection Mode ---
# Train only on normal data, detect anomalies in new data
lof_novelty = LocalOutlierFactor(
    n_neighbors=20,
    contamination=0.05,
    novelty=True  # Now we can call predict() on new data
)

# Train on clean data only
X_train_clean = X[y_true == 0]  # Only normal points
lof_novelty.fit(X_train_clean)

# Predict on new data
X_new = np.array([[0, 0], [5, 5], [10, 10], [-5, -5]])
new_predictions = lof_novelty.predict(X_new)
new_scores = lof_novelty.decision_function(X_new)

print(f"\nNovelty Detection on New Points:")
for point, pred, score in zip(X_new, new_predictions, new_scores):
    status = "NORMAL" if pred == 1 else "ANOMALY"
    print(f"  Point {point}: {status} (score: {score:.3f})")

# --- Effect of n_neighbors ---
print("\n--- Effect of n_neighbors ---")
for k in [5, 10, 20, 50, 100]:
    lof_k = LocalOutlierFactor(n_neighbors=k, contamination=0.02)
    preds_k = lof_k.fit_predict(X)
    n_detected = (preds_k == -1).sum()
    print(f"  k={k:3d}: {n_detected} anomalies detected")
```

### Example 5: Autoencoder for Anomaly Detection

```python
import numpy as np
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import classification_report, roc_auc_score

# --- Generate High-Dimensional Data ---
np.random.seed(42)
n_features = 30

# Normal data: multivariate normal
X_train_normal = np.random.normal(0, 1, size=(2000, n_features))
# Add some correlations to make it realistic
X_train_normal[:, 1] = 0.8 * X_train_normal[:, 0] + 0.2 * X_train_normal[:, 1]
X_train_normal[:, 5] = -0.5 * X_train_normal[:, 3] + 0.5 * X_train_normal[:, 5]

# Test data
X_test_normal = np.random.normal(0, 1, size=(400, n_features))
X_test_normal[:, 1] = 0.8 * X_test_normal[:, 0] + 0.2 * X_test_normal[:, 1]
X_test_normal[:, 5] = -0.5 * X_test_normal[:, 3] + 0.5 * X_test_normal[:, 5]

# Anomalous test data (different distribution)
X_test_anomaly = np.random.uniform(-3, 3, size=(50, n_features))

X_test = np.vstack([X_test_normal, X_test_anomaly])
y_test = np.array([0]*400 + [1]*50)  # 0=normal, 1=anomaly

# Scale data
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train_normal)
X_test_scaled = scaler.transform(X_test)

# --- Define Autoencoder ---
class Autoencoder(nn.Module):
    def __init__(self, input_dim, encoding_dim=8):
        super().__init__()
        
        # Encoder: compress to low-dimensional representation
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, 64),
            nn.ReLU(),
            nn.BatchNorm1d(64),
            nn.Linear(64, 32),
            nn.ReLU(),
            nn.BatchNorm1d(32),
            nn.Linear(32, encoding_dim),
            nn.ReLU()
        )
        
        # Decoder: reconstruct from compressed representation
        self.decoder = nn.Sequential(
            nn.Linear(encoding_dim, 32),
            nn.ReLU(),
            nn.BatchNorm1d(32),
            nn.Linear(32, 64),
            nn.ReLU(),
            nn.BatchNorm1d(64),
            nn.Linear(64, input_dim)
            # No activation: output can be any value
        )
    
    def forward(self, x):
        encoded = self.encoder(x)
        decoded = self.decoder(encoded)
        return decoded

# --- Training ---
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

model = Autoencoder(input_dim=n_features, encoding_dim=8).to(device)
optimizer = optim.Adam(model.parameters(), lr=0.001, weight_decay=1e-5)
criterion = nn.MSELoss()

# Create DataLoader
train_tensor = torch.FloatTensor(X_train_scaled)
train_dataset = TensorDataset(train_tensor, train_tensor)  # Input = Target
train_loader = DataLoader(train_dataset, batch_size=64, shuffle=True)

# Training loop
n_epochs = 50
train_losses = []

for epoch in range(n_epochs):
    model.train()
    epoch_loss = 0
    
    for batch_input, batch_target in train_loader:
        batch_input = batch_input.to(device)
        batch_target = batch_target.to(device)
        
        # Forward pass
        output = model(batch_input)
        loss = criterion(output, batch_target)
        
        # Backward pass
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        epoch_loss += loss.item()
    
    avg_loss = epoch_loss / len(train_loader)
    train_losses.append(avg_loss)
    
    if (epoch + 1) % 10 == 0:
        print(f"Epoch {epoch+1}/{n_epochs}, Loss: {avg_loss:.6f}")

# --- Anomaly Detection ---
model.eval()
with torch.no_grad():
    # Get reconstruction errors for test data
    X_test_tensor = torch.FloatTensor(X_test_scaled).to(device)
    reconstructed = model(X_test_tensor)
    
    # Per-sample reconstruction error (MSE)
    reconstruction_errors = torch.mean((X_test_tensor - reconstructed) ** 2, dim=1)
    reconstruction_errors = reconstruction_errors.cpu().numpy()

# --- Determine Threshold ---
# Method 1: Use percentile of training reconstruction errors
with torch.no_grad():
    train_reconstructed = model(train_tensor.to(device))
    train_errors = torch.mean((train_tensor.to(device) - train_reconstructed) ** 2, dim=1)
    train_errors = train_errors.cpu().numpy()

# Threshold: 95th percentile of training errors
threshold = np.percentile(train_errors, 95)
print(f"\nThreshold (95th percentile): {threshold:.6f}")

# Classify
predictions = (reconstruction_errors > threshold).astype(int)

# Evaluate
print("\nAutoencoder Anomaly Detection Results:")
print(classification_report(y_test, predictions, target_names=['Normal', 'Anomaly']))
print(f"ROC-AUC Score: {roc_auc_score(y_test, reconstruction_errors):.4f}")

# --- Per-Feature Contribution ---
# Which features contributed most to the anomaly?
with torch.no_grad():
    per_feature_error = (X_test_tensor - reconstructed) ** 2
    per_feature_error = per_feature_error.cpu().numpy()

# For the most anomalous point:
most_anomalous_idx = np.argmax(reconstruction_errors)
top_features = np.argsort(per_feature_error[most_anomalous_idx])[::-1][:5]
print(f"\nMost anomalous point (idx={most_anomalous_idx}):")
print(f"  Top contributing features: {top_features}")
print(f"  Their errors: {per_feature_error[most_anomalous_idx][top_features]}")
```

### Example 6: Time Series Anomaly Detection

```python
import numpy as np
import pandas as pd
from sklearn.ensemble import IsolationForest

# --- Generate Time Series with Anomalies ---
np.random.seed(42)
n_points = 1000

# Normal time series: trend + seasonality + noise
t = np.arange(n_points)
trend = 0.01 * t
seasonality = 5 * np.sin(2 * np.pi * t / 50)  # Period of 50
noise = np.random.normal(0, 1, n_points)
ts = trend + seasonality + noise

# Inject anomalies
anomaly_indices = [100, 250, 500, 750, 900]
for idx in anomaly_indices:
    ts[idx] += np.random.choice([-1, 1]) * 15  # Large spikes

# Create DataFrame
df = pd.DataFrame({
    'timestamp': pd.date_range('2024-01-01', periods=n_points, freq='h'),
    'value': ts
})
df.set_index('timestamp', inplace=True)

# --- Method 1: Moving Average + Threshold ---
def detect_moving_average(series, window=24, threshold_std=3):
    """Detect anomalies using moving average and rolling std."""
    rolling_mean = series.rolling(window=window, center=True).mean()
    rolling_std = series.rolling(window=window, center=True).std()
    
    # Deviation from expected value
    deviation = np.abs(series - rolling_mean)
    
    # Anomaly if deviation > threshold * rolling_std
    anomalies = deviation > (threshold_std * rolling_std)
    
    return anomalies, rolling_mean, rolling_std

ma_anomalies, ma_mean, ma_std = detect_moving_average(df['value'])
print(f"Moving Average Method: {ma_anomalies.sum()} anomalies detected")
print(f"  At indices: {df.index[ma_anomalies].tolist()[:5]}")

# --- Method 2: STL Decomposition ---
from statsmodels.tsa.seasonal import STL

stl = STL(df['value'], period=50, robust=True)
result = stl.fit()

# Anomalies in residuals
residuals = result.resid
residual_std = residuals.std()
stl_anomalies = np.abs(residuals) > 3 * residual_std

print(f"\nSTL Decomposition: {stl_anomalies.sum()} anomalies detected")

# --- Method 3: Feature Engineering + Isolation Forest ---
def create_ts_features(df, window_sizes=[6, 12, 24]):
    """Create features from time series for anomaly detection."""
    features = pd.DataFrame(index=df.index)
    
    # Current value
    features['value'] = df['value']
    
    for w in window_sizes:
        # Rolling statistics
        features[f'rolling_mean_{w}'] = df['value'].rolling(w).mean()
        features[f'rolling_std_{w}'] = df['value'].rolling(w).std()
        features[f'rolling_min_{w}'] = df['value'].rolling(w).min()
        features[f'rolling_max_{w}'] = df['value'].rolling(w).max()
        
        # Deviation from rolling mean
        features[f'deviation_{w}'] = df['value'] - features[f'rolling_mean_{w}']
    
    # Rate of change
    features['diff_1'] = df['value'].diff(1)
    features['diff_2'] = df['value'].diff(2)
    
    # Hour-of-day, day-of-week (for temporal patterns)
    features['hour'] = df.index.hour
    features['dayofweek'] = df.index.dayofweek
    
    return features.dropna()

# Create features
ts_features = create_ts_features(df)
print(f"\nFeature matrix shape: {ts_features.shape}")

# Apply Isolation Forest on features
iso_ts = IsolationForest(
    n_estimators=200,
    contamination=0.01,
    random_state=42
)
ts_predictions = iso_ts.fit_predict(ts_features)
ts_scores = -iso_ts.decision_function(ts_features)

anomaly_mask = ts_predictions == -1
print(f"Isolation Forest on TS features: {anomaly_mask.sum()} anomalies")
print(f"  Detected at: {ts_features.index[anomaly_mask].tolist()}")

# --- Method 4: LSTM Autoencoder for Sequences ---
# (Using PyTorch for sequence-based detection)
import torch
import torch.nn as nn

class LSTMAutoencoder(nn.Module):
    """LSTM Autoencoder for time series anomaly detection."""
    def __init__(self, input_dim=1, hidden_dim=32, n_layers=2):
        super().__init__()
        
        # Encoder LSTM
        self.encoder = nn.LSTM(
            input_size=input_dim,
            hidden_size=hidden_dim,
            num_layers=n_layers,
            batch_first=True,
            dropout=0.2
        )
        
        # Decoder LSTM
        self.decoder = nn.LSTM(
            input_size=hidden_dim,
            hidden_size=hidden_dim,
            num_layers=n_layers,
            batch_first=True,
            dropout=0.2
        )
        
        # Output layer
        self.output_layer = nn.Linear(hidden_dim, input_dim)
    
    def forward(self, x):
        # x shape: (batch, seq_len, input_dim)
        
        # Encode
        _, (hidden, cell) = self.encoder(x)
        
        # Repeat last hidden state for decoder input
        seq_len = x.size(1)
        decoder_input = hidden[-1].unsqueeze(1).repeat(1, seq_len, 1)
        
        # Decode
        decoder_output, _ = self.decoder(decoder_input)
        
        # Project to output dimension
        output = self.output_layer(decoder_output)
        
        return output

# Prepare sequences
def create_sequences(data, seq_length=24):
    """Convert time series to sequences for LSTM."""
    sequences = []
    for i in range(len(data) - seq_length):
        seq = data[i:i + seq_length]
        sequences.append(seq)
    return np.array(sequences)

# Normalize
values = df['value'].values.reshape(-1, 1)
from sklearn.preprocessing import MinMaxScaler
scaler = MinMaxScaler()
values_scaled = scaler.fit_transform(values)

# Create sequences
seq_length = 24
sequences = create_sequences(values_scaled, seq_length)
sequences_tensor = torch.FloatTensor(sequences)

print(f"\nLSTM Autoencoder - Sequences shape: {sequences_tensor.shape}")
# Output: (976, 24, 1) → 976 sequences of length 24

# Train (abbreviated - same pattern as regular autoencoder)
lstm_ae = LSTMAutoencoder(input_dim=1, hidden_dim=32)
optimizer = torch.optim.Adam(lstm_ae.parameters(), lr=0.001)
criterion = nn.MSELoss()

# Quick training loop
lstm_ae.train()
for epoch in range(20):
    # Process in batches
    for i in range(0, len(sequences_tensor), 32):
        batch = sequences_tensor[i:i+32]
        output = lstm_ae(batch)
        loss = criterion(output, batch)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
    
    if (epoch + 1) % 5 == 0:
        print(f"  Epoch {epoch+1}, Loss: {loss.item():.6f}")

# Detect anomalies
lstm_ae.eval()
with torch.no_grad():
    reconstructed = lstm_ae(sequences_tensor)
    errors = torch.mean((sequences_tensor - reconstructed) ** 2, dim=(1, 2))
    errors = errors.numpy()

# Threshold
threshold_lstm = np.percentile(errors, 97)
lstm_anomalies = errors > threshold_lstm
print(f"\nLSTM AE: {lstm_anomalies.sum()} anomalous sequences detected")
```

### Example 7: Complete Pipeline with Multiple Methods (Ensemble)

```python
import numpy as np
import pandas as pd
from sklearn.ensemble import IsolationForest
from sklearn.neighbors import LocalOutlierFactor
from sklearn.svm import OneClassSVM
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import classification_report, roc_auc_score

# --- Generate Realistic Dataset ---
np.random.seed(42)
n_samples = 2000
n_features = 10

# Normal data: multivariate normal with correlations
mean = np.zeros(n_features)
cov = np.eye(n_features)
# Add correlations
cov[0, 1] = cov[1, 0] = 0.7
cov[2, 3] = cov[3, 2] = -0.5
X_normal = np.random.multivariate_normal(mean, cov, n_samples)

# Anomalies: different types
# Type 1: Global outliers (far from everything)
global_outliers = np.random.uniform(-5, 5, size=(30, n_features))
# Type 2: Local outliers (normal range but wrong correlation)
local_outliers = np.random.normal(0, 1, size=(20, n_features))
local_outliers[:, 0] = -local_outliers[:, 1]  # Reverse the correlation

X = np.vstack([X_normal, global_outliers, local_outliers])
y_true = np.array([0]*n_samples + [1]*50)

# Scale
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# --- Method 1: Isolation Forest ---
iso_forest = IsolationForest(n_estimators=200, contamination=0.025, random_state=42)
iso_scores = -iso_forest.fit(X_scaled).decision_function(X_scaled)

# --- Method 2: LOF ---
lof = LocalOutlierFactor(n_neighbors=20, contamination=0.025)
lof.fit_predict(X_scaled)
lof_scores = -lof.negative_outlier_factor_

# --- Method 3: One-Class SVM ---
ocsvm = OneClassSVM(kernel='rbf', gamma='scale', nu=0.025)
ocsvm.fit(X_scaled)
ocsvm_scores = -ocsvm.decision_function(X_scaled)

# --- Normalize Scores to [0, 1] ---
def normalize_scores(scores):
    """Min-max normalize anomaly scores."""
    return (scores - scores.min()) / (scores.max() - scores.min())

iso_norm = normalize_scores(iso_scores)
lof_norm = normalize_scores(lof_scores)
ocsvm_norm = normalize_scores(ocsvm_scores)

# --- Ensemble: Average Scores ---
ensemble_scores = (iso_norm + lof_norm + ocsvm_norm) / 3

# --- Evaluate Each Method ---
print("=" * 60)
print("ANOMALY DETECTION - METHOD COMPARISON")
print("=" * 60)

methods = {
    'Isolation Forest': iso_norm,
    'LOF': lof_norm,
    'One-Class SVM': ocsvm_norm,
    'Ensemble (Average)': ensemble_scores
}

# Use top-N threshold (detect top 2.5% as anomalies)
threshold_percentile = 97.5

for name, scores in methods.items():
    threshold = np.percentile(scores, threshold_percentile)
    predictions = (scores > threshold).astype(int)
    
    precision = np.sum((predictions == 1) & (y_true == 1)) / max(np.sum(predictions == 1), 1)
    recall = np.sum((predictions == 1) & (y_true == 1)) / max(np.sum(y_true == 1), 1)
    auc = roc_auc_score(y_true, scores)
    
    print(f"\n{name}:")
    print(f"  ROC-AUC: {auc:.4f}")
    print(f"  Precision: {precision:.4f}")
    print(f"  Recall: {recall:.4f}")
    print(f"  Detected: {predictions.sum()} anomalies")

# --- Majority Voting Ensemble ---
def majority_vote(scores_list, percentile=97.5):
    """Classify as anomaly if majority of methods agree."""
    votes = np.zeros(len(scores_list[0]))
    for scores in scores_list:
        threshold = np.percentile(scores, percentile)
        votes += (scores > threshold).astype(int)
    # Majority: more than half the methods flag it
    return (votes > len(scores_list) / 2).astype(int)

vote_predictions = majority_vote([iso_norm, lof_norm, ocsvm_norm])
print(f"\nMajority Vote Ensemble:")
print(f"  Detected: {vote_predictions.sum()} anomalies")
precision_v = np.sum((vote_predictions == 1) & (y_true == 1)) / max(vote_predictions.sum(), 1)
recall_v = np.sum((vote_predictions == 1) & (y_true == 1)) / max(np.sum(y_true == 1), 1)
print(f"  Precision: {precision_v:.4f}, Recall: {recall_v:.4f}")
```

### Pro Tips

> **Pro Tip 1:** Always start with visualization. Plot your data distributions first. Many "anomalies" are just data quality issues (missing values coded as -999, unit conversion errors).

> **Pro Tip 2:** Use ensemble methods in production. No single algorithm catches all types of anomalies. Isolation Forest is great for global outliers, LOF for local ones, autoencoders for subtle distribution shifts.

> **Pro Tip 3:** The `contamination` parameter is NOT a tuning knob — it should reflect your actual expected anomaly rate. If you don't know it, use the decision function scores and set a threshold based on business requirements.

> **Pro Tip 4:** For time series, always remove trend and seasonality BEFORE anomaly detection. Otherwise, values at peaks of seasonal cycles get flagged as anomalous.

> **Pro Tip 5:** In production, track your model's false positive rate over time. If analysts dismiss >50% of alerts, your threshold is too sensitive. Calibrate using historical analyst feedback.

---

## Common Mistakes

### 1. Training on Contaminated Data
**Mistake:** Training unsupervised models on data that already contains anomalies, which shifts the "normal" baseline.
**Fix:** Clean your training data as much as possible. Use robust methods (Isolation Forest is naturally robust to some contamination; One-Class SVM is not).

### 2. Not Scaling Features
**Mistake:** Using distance-based methods (LOF, KNN, One-Class SVM) without scaling features.
**Fix:** Always standardize or normalize features. A feature with range [0, 1000] will dominate one with range [0, 1].

```python
# WRONG
from sklearn.neighbors import LocalOutlierFactor
lof = LocalOutlierFactor()
lof.fit_predict(X_raw)  # Features at different scales!

# RIGHT
from sklearn.preprocessing import StandardScaler
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_raw)
lof.fit_predict(X_scaled)
```

### 3. Evaluating with Accuracy
**Mistake:** "My model has 99% accuracy!" — but 99% of data is normal, so predicting everything as normal gets 99%.
**Fix:** Use Precision, Recall, F1, PR-AUC. Focus on the anomaly class.

### 4. Ignoring Temporal Order
**Mistake:** Randomly splitting time series data into train/test, causing temporal leakage.
**Fix:** Always use temporal splits. Train on past, test on future.

### 5. Static Thresholds in Dynamic Systems
**Mistake:** Setting a fixed threshold and never updating it as the data distribution shifts.
**Fix:** Use adaptive thresholds (rolling percentile), retrain models regularly, monitor threshold performance.

### 6. Treating All Anomalies Equally
**Mistake:** Same response for all detected anomalies regardless of severity or confidence.
**Fix:** Use anomaly scores (not just binary labels) to prioritize. Route high-confidence anomalies to immediate action, low-confidence to review queue.

### 7. Not Considering Feature Interactions
**Mistake:** Checking each feature independently for anomalies (univariate approach) when the anomaly is in feature relationships.
**Fix:** Use multivariate methods. A credit card transaction of $100 is normal. A $100 transaction at 3 AM in a foreign country is not. The anomaly is in the combination.

### 8. Overfitting Autoencoders
**Mistake:** Using a very large autoencoder that perfectly reconstructs even anomalies.
**Fix:** Use a bottleneck (encoding dimension << input dimension). Add regularization. Monitor reconstruction error on a validation set.

---

## Interview Questions

### Conceptual Questions

**Q1: What's the difference between outlier detection and novelty detection?**
> **Outlier detection:** Training data contains outliers — the goal is to identify them. The model must be robust to contamination (e.g., Isolation Forest, LOF with novelty=False).
> **Novelty detection:** Training data is clean (only normal samples) — the goal is to detect new, previously unseen types of data. The model learns "normal" then flags anything different (e.g., One-Class SVM, Autoencoder, LOF with novelty=True).

**Q2: Why does Isolation Forest work? What's the key insight?**
> Anomalies are "few and different." In a random binary partition of feature space, anomalies are isolated with fewer splits because they're in sparse regions. Normal points require many splits because they're surrounded by similar points. The path length to isolate a point is inversely proportional to its "anomalousness." This makes Isolation Forest efficient (O(n log n)) and effective without computing distances.

**Q3: When would you choose LOF over Isolation Forest?**
> LOF when: (1) data has clusters of varying density (LOF detects local anomalies that are normal globally), (2) you need to understand WHY something is anomalous relative to its neighborhood, (3) dataset is small-medium. 
> Isolation Forest when: (1) large dataset (LOF is O(n²)), (2) high-dimensional data, (3) you need speed, (4) anomalies are global outliers.

**Q4: How do autoencoders detect anomalies? What assumptions does this make?**
> Autoencoders learn to compress and reconstruct normal data through a bottleneck. The bottleneck forces the model to learn only the most important patterns of normal data. Anomalous data, having different patterns, cannot be reconstructed well — the reconstruction error is high. Assumption: anomalies are distributionally different from normal data and the bottleneck is tight enough that the model can't memorize everything.

**Q5: How would you handle concept drift in an anomaly detection system?**
> (1) Retrain periodically (daily/weekly) with recent data as the new "normal." (2) Use sliding window approaches where the model only considers recent history. (3) Implement adaptive thresholds that adjust based on recent score distributions. (4) Monitor model performance metrics and trigger retraining when degradation is detected. (5) Use online learning algorithms that update incrementally.

**Q6: What's the difference between supervised fraud detection and anomaly detection for fraud?**
> Supervised: Needs labeled fraud examples. Can model specific fraud patterns. Misses new/unseen fraud types. Works well when fraud patterns are stable and you have good labels.
> Anomaly detection: Doesn't need fraud labels, just models "normal" behavior. Can catch novel fraud types (zero-day fraud). Higher false positive rate. Better for detecting emerging threats. In practice, combine both: supervised for known patterns + anomaly detection for unknown patterns.

### System Design Questions

**Q7: Design a real-time fraud detection system.**
> Architecture: (1) **Feature store** with real-time features (transaction velocity, geographic anomalies, device fingerprint). (2) **Rule engine** for known fraud patterns (instant, no ML). (3) **ML model 1** — supervised model (XGBoost) on labeled data for known fraud types. (4) **ML model 2** — Isolation Forest on user behavioral features for novel fraud. (5) **Ensemble** scores from both models + rules. (6) **Decision engine** — auto-block high confidence, queue medium for review. (7) **Feedback loop** — analyst decisions → relabel → retrain weekly. Latency: <100ms for scoring. Retrain supervised daily, anomaly model weekly.

**Q8: Your anomaly detection model has too many false positives. How do you fix it?**
> (1) Raise the threshold (trade recall for precision). (2) Add more features to better characterize "normal." (3) Use ensemble (anomaly only if multiple models agree). (4) Segment data — different user groups may have different "normals." (5) Add temporal context (is this anomalous FOR THIS TIME OF DAY?). (6) Use analyst feedback to create labeled data → train supervised model on common false positive patterns. (7) Implement alert aggregation (group related alerts).

### Coding Questions

**Q9: Implement a simple Isolation Tree node split.**
```python
def isolation_split(data, feature_idx):
    """Perform one random split on a feature."""
    feature_values = data[:, feature_idx]
    min_val, max_val = feature_values.min(), feature_values.max()
    
    # Random split value
    split_val = np.random.uniform(min_val, max_val)
    
    left_mask = feature_values < split_val
    right_mask = ~left_mask
    
    return split_val, data[left_mask], data[right_mask]
```

**Q10: How would you determine the optimal contamination parameter for Isolation Forest?**
> If you have some labeled data: Use it to find the threshold on decision_function scores that maximizes F1 (or precision at acceptable recall). If no labels: (1) Domain knowledge — ask subject matter experts what % anomaly rate they expect. (2) Elbow method on sorted anomaly scores — look for a natural gap. (3) Start conservative (low contamination = fewer alerts) and iterate based on analyst feedback. (4) Use statistical methods on the score distribution to find natural break points.

---

## Quick Reference

### Algorithm Comparison

| Algorithm | Time Complexity | Handles High-D | Varying Density | Training Data |
|-----------|----------------|-----------------|-----------------|---------------|
| Z-Score | O(n) | No | No | Any |
| Mahalanobis | O(n·d²) | Moderate | No | Normal only |
| Isolation Forest | O(n log n) | Yes | Moderate | Any |
| One-Class SVM | O(n²-n³) | Moderate | No | Normal only |
| LOF | O(n²) | No | Yes | Any |
| DBSCAN | O(n log n) | No | No (fixed eps) | Any |
| Autoencoder | O(n·epochs) | Yes | Yes | Normal only |
| KNN-based | O(n²) | No | No | Any |

### Decision Guide

| Scenario | Recommended Method |
|----------|-------------------|
| Quick baseline, no tuning | Isolation Forest |
| Varying density clusters | LOF |
| Only clean normal data | One-Class SVM or Autoencoder |
| High-dimensional features | Isolation Forest or Autoencoder |
| Need explainability | Statistical methods or LOF |
| Time series data | STL + statistical or LSTM-AE |
| Need real-time scoring | Isolation Forest (pre-trained) |
| Production ensemble | IF + LOF + Autoencoder |
| Very large scale | Isolation Forest (subsampling built-in) |

### Hyperparameter Cheat Sheet

| Algorithm | Key Parameters | Start Values |
|-----------|---------------|--------------|
| Isolation Forest | n_estimators, contamination, max_samples | 200, 0.01-0.05, 256 |
| One-Class SVM | kernel, nu, gamma | rbf, 0.01-0.1, scale |
| LOF | n_neighbors, contamination | 20, 0.01-0.05 |
| DBSCAN | eps, min_samples | auto (k-dist plot), 5-10 |
| Autoencoder | encoding_dim, epochs, threshold | d/4, 50-100, 95th percentile |

### Key Formulas

| Formula | Purpose |
|---------|---------|
| $z = \frac{x - \mu}{\sigma}$ | Z-score (univariate) |
| $D_M = \sqrt{(x-\mu)^T \Sigma^{-1} (x-\mu)}$ | Mahalanobis distance |
| $s(x,n) = 2^{-E(h(x))/c(n)}$ | Isolation Forest score |
| $LOF(A) = \frac{\sum \frac{LRD(B)}{LRD(A)}}{|N(A)|}$ | Local Outlier Factor |
| $\text{score}(x) = \|x - \hat{x}\|^2$ | Autoencoder anomaly score |

### Python Libraries

| Library | Methods Available | Best For |
|---------|------------------|----------|
| `scikit-learn` | IF, LOF, One-Class SVM, DBSCAN | General purpose |
| `PyOD` | 30+ algorithms, unified API | Research & comparison |
| `ADTK` | Time series specific | Time series anomalies |
| `alibi-detect` | Drift + anomaly detection | Production + DL methods |
| `prophet` | Time series forecasting | Seasonal time series |
| `pytorch` / `tensorflow` | Autoencoders, custom models | Deep learning approaches |

### Key Takeaways

- **No single method works everywhere** — use ensembles in production
- **Isolation Forest** is the best starting point (fast, robust, few assumptions)
- **Scaling is critical** for distance-based methods (LOF, SVM, KNN)
- **Evaluation is hard** — use PR-AUC, not accuracy; time-based splits, not random
- **Production systems need feedback loops** — analyst decisions improve the model
- **Threshold > Model** — even a great model fails with a bad threshold
- **Context matters** — the same value can be normal or anomalous depending on context
- **Start simple, add complexity only when needed** — Z-score → IF → Autoencoder
