# Chapter 06: KNN and Naive Bayes

## Table of Contents
- [Part A: K-Nearest Neighbors (KNN)](#part-a-k-nearest-neighbors-knn)
  - [1. Introduction to KNN](#1-introduction-to-knn)
  - [2. How KNN Works](#2-how-knn-works)
  - [3. Distance Metrics](#3-distance-metrics)
  - [4. Choosing K](#4-choosing-k)
  - [5. KNN for Regression](#5-knn-for-regression)
  - [6. Weighted KNN](#6-weighted-knn)
  - [7. The Curse of Dimensionality](#7-the-curse-of-dimensionality)
  - [8. Practical Implementation — KNN](#8-practical-implementation--knn)
  - [9. Speeding Up KNN](#9-speeding-up-knn)
- [Part B: Naive Bayes](#part-b-naive-bayes)
  - [10. Bayes' Theorem Refresher](#10-bayes-theorem-refresher)
  - [11. Naive Bayes Classifier](#11-naive-bayes-classifier)
  - [12. Naive Bayes Variants](#12-naive-bayes-variants)
  - [13. Text Classification with Naive Bayes](#13-text-classification-with-naive-bayes)
  - [14. Laplace Smoothing](#14-laplace-smoothing)
  - [15. Practical Implementation — Naive Bayes](#15-practical-implementation--naive-bayes)
- [16. KNN vs Naive Bayes Comparison](#16-knn-vs-naive-bayes-comparison)
- [17. Common Mistakes](#17-common-mistakes)
- [18. Interview Questions](#18-interview-questions)
- [19. Quick Reference](#19-quick-reference)

---

# Part A: K-Nearest Neighbors (KNN)

## 1. Introduction to KNN

### What It Is
KNN is the simplest machine learning algorithm: to classify a new point, look at the K closest training points and take a vote. That's it!

**Analogy for a 15-year-old**: You just moved to a new neighborhood. To figure out if your area is "rich" or "not rich," you look at the 5 houses closest to yours. If 4 out of 5 are mansions → you're probably in a rich neighborhood. KNN works exactly like this — it judges you by your neighbors.

### Why It Matters
- **Zero training time** — KNN memorizes all data (lazy learner)
- **Simple to understand and implement** — great baseline
- **Non-parametric** — makes no assumptions about data distribution
- **Naturally handles multi-class** — no special modifications needed
- **Still used in**: Recommendation systems, imputation, anomaly detection

### Key Characteristics

| Property | KNN |
|----------|-----|
| Learning Type | Instance-based (lazy learning) |
| Training Time | O(1) — just stores data |
| Prediction Time | O(n·d) — computes all distances |
| Parametric? | No |
| Assumptions | Nearby points have similar labels |
| Feature Scaling | **REQUIRED** |

---

## 2. How KNN Works

### Algorithm

```
KNN Classification:
─────────────────────
Input: Training data, new point x, K (number of neighbors)

1. Calculate distance from x to EVERY training point
2. Sort by distance (ascending)
3. Take the K nearest neighbors
4. Count votes:
   - Classification: Majority class wins
   - Regression: Average of K neighbors' values
5. Return prediction
```

### Visual Example

```
K=3: Look at 3 nearest neighbors

    ○         ○
       ○              ●
    ○     ★ ← new point    ●
       ○         ●
    ○              ● ●

3 nearest to ★: [○, ○, ●]
Vote: ○ wins (2 vs 1)
Prediction: ○
```

```
K=7: Look at 7 nearest neighbors

    ○         ○
       ○              ●
    ○     ★ ← new point    ●
       ○         ●
    ○              ● ●

7 nearest to ★: [○, ○, ○, ○, ●, ●, ●]
Vote: ○ wins (4 vs 3)
Prediction: ○
```

### Why "Lazy Learning"?

| Eager Learners (Decision Tree, SVM) | Lazy Learners (KNN) |
|------|------|
| Build a model during training | No model built — just stores data |
| Fast prediction | Slow prediction (compute distances) |
| Can't easily update | Add new data instantly |
| Compact model | Must keep ALL training data |

---

## 3. Distance Metrics

### 3.1 Euclidean Distance (L2) — Default

The straight-line distance between two points:

$$d(x, y) = \sqrt{\sum_{i=1}^{n} (x_i - y_i)^2}$$

**When to use**: Default choice. Works well when features are continuous and on similar scales.

### 3.2 Manhattan Distance (L1)

Distance measured along axes (like walking in a grid city):

$$d(x, y) = \sum_{i=1}^{n} |x_i - y_i|$$

**When to use**: High-dimensional data, when differences in individual features matter more than overall distance. More robust to outliers than Euclidean.

### 3.3 Minkowski Distance (Generalized)

$$d(x, y) = \left(\sum_{i=1}^{n} |x_i - y_i|^p\right)^{1/p}$$

- $p=1$: Manhattan
- $p=2$: Euclidean
- $p=\infty$: Chebyshev (maximum of absolute differences)

### 3.4 Cosine Distance

$$d(x, y) = 1 - \frac{x \cdot y}{||x|| \cdot ||y||}$$

**When to use**: Text data, when direction matters more than magnitude. Two documents about the same topic will have similar angles regardless of length.

### 3.5 Hamming Distance

$$d(x, y) = \frac{1}{n}\sum_{i=1}^{n} \mathbb{1}(x_i \neq y_i)$$

**When to use**: Categorical features, binary strings.

### Distance Metric Comparison

```python
from sklearn.metrics.pairwise import (euclidean_distances, 
                                       manhattan_distances,
                                       cosine_distances)
import numpy as np

# Two points in 3D space
a = np.array([[1, 2, 3]])
b = np.array([[4, 6, 3]])

# Euclidean: sqrt((4-1)² + (6-2)² + (3-3)²) = sqrt(9+16+0) = 5.0
print(f"Euclidean: {euclidean_distances(a, b)[0][0]:.2f}")

# Manhattan: |4-1| + |6-2| + |3-3| = 3 + 4 + 0 = 7.0
print(f"Manhattan: {manhattan_distances(a, b)[0][0]:.2f}")

# Cosine: 1 - (a·b)/(|a|·|b|)
print(f"Cosine: {cosine_distances(a, b)[0][0]:.4f}")
```

### Which Distance Metric to Choose?

| Scenario | Best Metric |
|----------|-------------|
| General purpose, continuous features | Euclidean |
| High-dimensional data | Manhattan (less affected by curse of dimensionality) |
| Text/document similarity | Cosine |
| Categorical/binary features | Hamming |
| Outlier-robust needed | Manhattan |
| GPS/geographic data | Haversine |

---

## 4. Choosing K

### The Effect of K

```
K=1: Very complex boundary (overfits)    K=large: Very smooth boundary (underfits)

  ○○●○○                                   ○○○○○
  ○●●●○   ← captures noise               ○○○○○   ← ignores local patterns
  ○○●○○                                   ○○○○○
  
  High variance, low bias                  Low variance, high bias
```

### K Selection Rules of Thumb

1. **Start with K = √n** (where n = number of training samples)
2. **Use odd K** for binary classification (avoids ties)
3. **K=1**: Maximum complexity, captures noise
4. **K=n**: Predicts majority class always (useless)
5. **Typical range**: 3-15 for most problems

### Finding Optimal K with Cross-Validation

```python
from sklearn.neighbors import KNeighborsClassifier
from sklearn.model_selection import cross_val_score
from sklearn.preprocessing import StandardScaler
from sklearn.datasets import load_breast_cancer
import numpy as np
import matplotlib.pyplot as plt

# Load and scale data
data = load_breast_cancer()
X, y = data.data, data.target

# CRITICAL: Scale features for KNN
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# Test K from 1 to 30
k_range = range(1, 31)
cv_scores = []

for k in k_range:
    knn = KNeighborsClassifier(n_neighbors=k)
    scores = cross_val_score(knn, X_scaled, y, cv=10, scoring='accuracy')
    cv_scores.append(scores.mean())

# Find best K
best_k = k_range[np.argmax(cv_scores)]
print(f"Best K: {best_k}, CV Accuracy: {max(cv_scores):.4f}")

# Plot K vs Accuracy
plt.figure(figsize=(10, 6))
plt.plot(k_range, cv_scores, 'o-', linewidth=2)
plt.axvline(x=best_k, color='red', linestyle='--', label=f'Best K={best_k}')
plt.xlabel('K (Number of Neighbors)')
plt.ylabel('Cross-Validation Accuracy')
plt.title('KNN: Choosing K')
plt.legend()
plt.grid(True)
plt.show()
```

---

## 5. KNN for Regression

### How It Works

Instead of majority vote, take the **average** (or weighted average) of K nearest neighbors' target values.

$$\hat{y} = \frac{1}{K} \sum_{i \in N_K(x)} y_i$$

```python
from sklearn.neighbors import KNeighborsRegressor
import numpy as np
import matplotlib.pyplot as plt

# Generate non-linear data
np.random.seed(42)
X = np.sort(5 * np.random.rand(100, 1), axis=0)
y = np.sin(X).ravel() + np.random.normal(0, 0.1, X.shape[0])

# Fit KNN regressors with different K values
fig, axes = plt.subplots(1, 3, figsize=(18, 5))
k_values = [1, 5, 20]

for ax, k in zip(axes, k_values):
    knn_reg = KNeighborsRegressor(n_neighbors=k, weights='uniform')
    knn_reg.fit(X, y)
    
    X_test = np.linspace(0, 5, 500).reshape(-1, 1)
    y_pred = knn_reg.predict(X_test)
    
    ax.scatter(X, y, s=20, alpha=0.5, label='Data')
    ax.plot(X_test, y_pred, 'r-', linewidth=2, label=f'KNN (K={k})')
    ax.set_title(f'K = {k}')
    ax.legend()

plt.suptitle('KNN Regression: Effect of K')
plt.tight_layout()
plt.show()
# K=1: Jagged (overfits), K=5: Good fit, K=20: Smooth (underfits)
```

---

## 6. Weighted KNN

### The Problem with Equal Votes

With uniform voting, a point that's barely within the K-neighborhood has the same influence as the nearest neighbor. This can cause issues:

```
K=5: All 5 vote equally

  ● (far away, barely in K)
  
  ○ (very close)
  ○ (very close)        ★ ← new point
  ○ (very close)
  
  ● (far away, barely in K)

Uniform: ○=3, ●=2 → Predict ○ ✓ (correct, but barely)
What if K=5 had 3● far away and 2○ close? Uniform might be wrong!
```

### Distance-Weighted KNN

Closer neighbors get more "vote weight":

$$w_i = \frac{1}{d(x, x_i)^2}$$

```python
from sklearn.neighbors import KNeighborsClassifier

# Uniform weights (default)
knn_uniform = KNeighborsClassifier(n_neighbors=7, weights='uniform')

# Distance-weighted (closer neighbors count more)
knn_distance = KNeighborsClassifier(n_neighbors=7, weights='distance')

# Custom weights (e.g., Gaussian kernel)
def gaussian_weight(distances):
    """Gaussian kernel weighting — exponential decay with distance."""
    sigma = 1.0
    return np.exp(-(distances ** 2) / (2 * sigma ** 2))

knn_custom = KNeighborsClassifier(n_neighbors=7, weights=gaussian_weight)
```

> **Pro Tip**: `weights='distance'` often improves results and makes the model less sensitive to K choice. Always try it!

---

## 7. The Curse of Dimensionality

### The Problem

As dimensions increase, ALL points become roughly equidistant. KNN breaks because "nearest neighbor" becomes meaningless.

```
Dimensions:  2D        10D        100D       1000D
Distance     
spread:    [0.3-2.1]  [2.5-3.8]  [9.5-10.2]  [31.0-31.5]
           ↑                                    ↑
           Big range                            All same!
           (neighbors                           (can't distinguish
            are meaningful)                      near vs far)
```

### Mathematical Insight

For uniformly distributed points in a $d$-dimensional unit hypercube, the expected distance to the nearest neighbor:

$$E[d_{nearest}] \approx \left(\frac{1}{n}\right)^{1/d}$$

As $d$ grows, this approaches 1 (the diagonal of the hypercube), meaning the nearest neighbor is almost as far as the farthest!

### Practical Rule of Thumb

- **KNN works well**: up to ~20-30 meaningful features
- **Above 30 features**: Apply dimensionality reduction first (PCA, feature selection)
- **Hundreds of features**: KNN is NOT recommended without preprocessing

### Solutions

```python
from sklearn.decomposition import PCA
from sklearn.pipeline import Pipeline
from sklearn.neighbors import KNeighborsClassifier
from sklearn.preprocessing import StandardScaler

# Solution 1: PCA before KNN
knn_with_pca = Pipeline([
    ('scaler', StandardScaler()),
    ('pca', PCA(n_components=10)),  # Reduce to 10 dimensions
    ('knn', KNeighborsClassifier(n_neighbors=5, weights='distance'))
])

# Solution 2: Feature selection before KNN
from sklearn.feature_selection import SelectKBest, f_classif
knn_with_selection = Pipeline([
    ('scaler', StandardScaler()),
    ('selector', SelectKBest(f_classif, k=15)),  # Keep top 15 features
    ('knn', KNeighborsClassifier(n_neighbors=5))
])
```

---

## 8. Practical Implementation — KNN

### Complete Classification Pipeline

```python
from sklearn.neighbors import KNeighborsClassifier
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.model_selection import train_test_split, GridSearchCV
from sklearn.datasets import load_wine
from sklearn.metrics import classification_report
import numpy as np

# Load data
data = load_wine()
X, y = data.data, data.target
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# KNN Pipeline (MUST include scaling!)
knn_pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('knn', KNeighborsClassifier())
])

# Hyperparameter tuning
param_grid = {
    'knn__n_neighbors': [3, 5, 7, 9, 11, 15, 21],
    'knn__weights': ['uniform', 'distance'],
    'knn__metric': ['euclidean', 'manhattan', 'minkowski'],
    'knn__p': [1, 2, 3]  # Minkowski parameter
}

grid_search = GridSearchCV(
    knn_pipe, param_grid, cv=5, 
    scoring='accuracy', n_jobs=-1
)
grid_search.fit(X_train, y_train)

print(f"Best Parameters: {grid_search.best_params_}")
print(f"Best CV Accuracy: {grid_search.best_score_:.4f}")
print(f"Test Accuracy: {grid_search.score(X_test, y_test):.4f}")

# Detailed results
y_pred = grid_search.predict(X_test)
print("\nClassification Report:")
print(classification_report(y_test, y_pred, target_names=data.target_names))
```

### KNN for Missing Value Imputation

```python
from sklearn.impute import KNNImputer
import numpy as np
import pandas as pd

# Create data with missing values
data = pd.DataFrame({
    'age': [25, 30, np.nan, 45, 50, np.nan, 35],
    'income': [50000, 60000, 70000, np.nan, 90000, 55000, np.nan],
    'score': [80, np.nan, 90, 85, 95, 70, 75]
})

print("Before imputation:")
print(data)

# KNN Imputation — fills missing values using K nearest neighbors
imputer = KNNImputer(
    n_neighbors=3,         # Use 3 nearest neighbors
    weights='distance',    # Closer neighbors contribute more
    metric='nan_euclidean' # Handles missing values in distance calc
)

data_imputed = pd.DataFrame(
    imputer.fit_transform(data),
    columns=data.columns
)

print("\nAfter KNN imputation:")
print(data_imputed)
```

---

## 9. Speeding Up KNN

### The Problem

Brute-force KNN: O(n·d) per prediction. For 1M training points, every prediction requires 1M distance calculations!

### Data Structures for Fast Neighbor Search

| Algorithm | How It Works | Best For |
|-----------|-------------|----------|
| `brute` | Computes all distances | Small datasets (<30 features) |
| `kd_tree` | Binary tree partitioning space | Low dimensions (<20) |
| `ball_tree` | Hierarchical ball decomposition | Higher dimensions (20-50) |

```python
from sklearn.neighbors import KNeighborsClassifier
import time

# Brute force (exact, slow for large data)
knn_brute = KNeighborsClassifier(n_neighbors=5, algorithm='brute')

# KD-Tree (fast for low-dimensional data)
knn_kdtree = KNeighborsClassifier(n_neighbors=5, algorithm='kd_tree', leaf_size=30)

# Ball Tree (better for higher dimensions)
knn_ball = KNeighborsClassifier(n_neighbors=5, algorithm='ball_tree', leaf_size=30)

# Auto (sklearn picks the best)
knn_auto = KNeighborsClassifier(n_neighbors=5, algorithm='auto')

# Benchmark
from sklearn.datasets import make_classification
X_large, y_large = make_classification(n_samples=50000, n_features=20, random_state=42)

for name, model in [('brute', knn_brute), ('kd_tree', knn_kdtree), 
                     ('ball_tree', knn_ball)]:
    start = time.time()
    model.fit(X_large, y_large)
    model.predict(X_large[:100])
    elapsed = time.time() - start
    print(f"{name:10s}: {elapsed:.3f}s")
```

### Approximate Nearest Neighbors (ANN)

For very large datasets (millions of points), use approximate methods:

```python
# pip install annoy faiss-cpu
# Facebook's FAISS — blazing fast approximate NN
# Spotify's Annoy — memory-efficient, good for recommendations

# Example with FAISS (if installed)
# import faiss
# index = faiss.IndexFlatL2(d)  # L2 distance
# index.add(X_train)
# distances, indices = index.search(X_test, k=5)
```

---

# Part B: Naive Bayes

## 10. Bayes' Theorem Refresher

### The Formula

$$P(A|B) = \frac{P(B|A) \cdot P(A)}{P(B)}$$

| Term | Name | Meaning |
|------|------|---------|
| $P(A\|B)$ | Posterior | Probability of A given we observed B |
| $P(B\|A)$ | Likelihood | Probability of observing B if A is true |
| $P(A)$ | Prior | Initial belief about A (before seeing evidence) |
| $P(B)$ | Evidence | Total probability of observing B |

### Intuition with a Real Example

**Scenario**: A medical test for a disease that affects 1% of people. The test is 90% accurate.
- You test positive. What's the probability you actually have the disease?

```
P(Disease) = 0.01 (1% of people have it)
P(Positive | Disease) = 0.90 (90% detection rate)
P(Positive | No Disease) = 0.10 (10% false positive rate)

P(Disease | Positive) = P(Positive|Disease) × P(Disease)
                        ─────────────────────────────────
                                  P(Positive)

P(Positive) = P(Pos|Disease)×P(Disease) + P(Pos|No Disease)×P(No Disease)
            = 0.90 × 0.01 + 0.10 × 0.99 = 0.009 + 0.099 = 0.108

P(Disease | Positive) = (0.90 × 0.01) / 0.108 = 0.083 = 8.3%
```

> **Surprising!** Even with a positive test, there's only 8.3% chance of disease. This is because the disease is rare (low prior), so false positives dominate.

---

## 11. Naive Bayes Classifier

### What It Is

Naive Bayes applies Bayes' theorem to classification, with the "naive" assumption that **all features are independent** given the class.

**For classification**:

$$P(y|x_1, x_2, ..., x_n) = \frac{P(x_1, x_2, ..., x_n|y) \cdot P(y)}{P(x_1, x_2, ..., x_n)}$$

### The "Naive" Assumption

$$P(x_1, x_2, ..., x_n|y) = P(x_1|y) \cdot P(x_2|y) \cdot ... \cdot P(x_n|y) = \prod_{i=1}^{n} P(x_i|y)$$

**This means**: Features are assumed independent given the class. In reality, this is almost NEVER true — but NB still works surprisingly well!

### Why "Naive" Works

1. Classification only needs the RANKING (which class has highest probability), not exact probabilities
2. Even if individual probabilities are wrong, the product's ordering can still be correct
3. Errors in one feature's probability can cancel out errors in another

### Decision Rule

$$\hat{y} = \arg\max_y \left[ P(y) \cdot \prod_{i=1}^{n} P(x_i|y) \right]$$

(We drop the denominator $P(X)$ since it's the same for all classes)

### Worked Example: Spam Classification

```
Training Data:
Email 1: "Free money now" → Spam
Email 2: "Hi how are you" → Not Spam
Email 3: "Win free prize" → Spam
Email 4: "Meeting tomorrow" → Not Spam

New email: "Free prize money"

Step 1: Calculate priors
P(Spam) = 2/4 = 0.5
P(Not Spam) = 2/4 = 0.5

Step 2: Calculate likelihoods (word frequencies per class)
P("free"|Spam) = 2/6 words in spam = 0.33
P("prize"|Spam) = 1/6 = 0.17
P("money"|Spam) = 1/6 = 0.17

P("free"|Not Spam) = 0/7 ≈ 0.01 (with smoothing)
P("prize"|Not Spam) = 0/7 ≈ 0.01
P("money"|Not Spam) = 0/7 ≈ 0.01

Step 3: Calculate posteriors
P(Spam|"free prize money") ∝ 0.5 × 0.33 × 0.17 × 0.17 = 0.00477
P(Not Spam|"free prize money") ∝ 0.5 × 0.01 × 0.01 × 0.01 = 0.0000005

Step 4: Predict → SPAM (much higher)
```

---

## 12. Naive Bayes Variants

### 12.1 Gaussian Naive Bayes

**Assumes**: Features follow a Gaussian (normal) distribution within each class.

$$P(x_i|y) = \frac{1}{\sqrt{2\pi\sigma_y^2}} \exp\left(-\frac{(x_i - \mu_y)^2}{2\sigma_y^2}\right)$$

Where $\mu_y$ and $\sigma_y$ are the mean and standard deviation of feature $x_i$ for class $y$.

**Use when**: Continuous features, general-purpose classification

```python
from sklearn.naive_bayes import GaussianNB
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Gaussian NB — no hyperparameters to tune!
gnb = GaussianNB()
gnb.fit(X_train, y_train)

print(f"Accuracy: {gnb.score(X_test, y_test):.4f}")
print(f"Class priors: {gnb.class_prior_}")
print(f"Class means shape: {gnb.theta_.shape}")  # (n_classes, n_features)
print(f"Class variances shape: {gnb.var_.shape}")
```

### 12.2 Multinomial Naive Bayes

**Assumes**: Features are counts or frequencies (non-negative integers).

$$P(x_i|y) = \frac{N_{yi} + \alpha}{N_y + \alpha \cdot n}$$

Where $N_{yi}$ = count of feature $i$ in class $y$, $\alpha$ = smoothing parameter.

**Use when**: Text classification (word counts/TF-IDF), document categorization

```python
from sklearn.naive_bayes import MultinomialNB
from sklearn.feature_extraction.text import CountVectorizer, TfidfVectorizer

# Example: Text classification
texts = ["I love this movie", "Terrible film", "Great acting", 
         "Worst movie ever", "Amazing story", "Boring and dull"]
labels = [1, 0, 1, 0, 1, 0]  # 1=positive, 0=negative

# Convert text to word counts
vectorizer = CountVectorizer()
X = vectorizer.fit_transform(texts)

# Multinomial NB — perfect for word counts
mnb = MultinomialNB(alpha=1.0)  # alpha = Laplace smoothing
mnb.fit(X, labels)

# Predict new text
new_texts = ["I love great acting", "Terrible boring movie"]
X_new = vectorizer.transform(new_texts)
predictions = mnb.predict(X_new)
probabilities = mnb.predict_proba(X_new)

for text, pred, prob in zip(new_texts, predictions, probabilities):
    sentiment = "Positive" if pred == 1 else "Negative"
    print(f"'{text}' → {sentiment} (confidence: {max(prob):.2f})")
```

### 12.3 Bernoulli Naive Bayes

**Assumes**: Features are binary (0/1) — presence or absence.

$$P(x_i|y) = P(i|y) \cdot x_i + (1 - P(i|y)) \cdot (1 - x_i)$$

**Key difference from Multinomial**: Bernoulli penalizes the ABSENCE of features, Multinomial doesn't.

**Use when**: Binary features, short text (binary word occurrence), document classification

```python
from sklearn.naive_bayes import BernoulliNB
from sklearn.feature_extraction.text import CountVectorizer

# Bernoulli NB — works with binary features
vectorizer = CountVectorizer(binary=True)  # Binary: word present or not
X_binary = vectorizer.fit_transform(texts)

bnb = BernoulliNB(alpha=1.0, binarize=0.0)  # binarize threshold
bnb.fit(X_binary, labels)
```

### 12.4 Complement Naive Bayes

- Designed specifically for **imbalanced datasets**
- Uses complement of each class to estimate parameters
- Often works better than Multinomial NB on imbalanced text data

```python
from sklearn.naive_bayes import ComplementNB

# Great for imbalanced text classification
cnb = ComplementNB(alpha=1.0)
```

### Variant Comparison

| Variant | Feature Type | Best For | Key Parameter |
|---------|-------------|----------|---------------|
| **Gaussian** | Continuous (real) | General classification, sensor data | var_smoothing |
| **Multinomial** | Counts/frequencies | Text (word counts, TF-IDF) | alpha |
| **Bernoulli** | Binary (0/1) | Short text, binary features | alpha, binarize |
| **Complement** | Counts | Imbalanced text classification | alpha |

---

## 13. Text Classification with Naive Bayes

### Complete Spam Detection Pipeline

```python
from sklearn.naive_bayes import MultinomialNB
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.pipeline import Pipeline
from sklearn.metrics import classification_report, confusion_matrix
import numpy as np

# Simulated spam dataset (in practice, use SMS Spam Collection or similar)
emails = [
    "Win a free iPhone now", "Meeting at 3pm tomorrow",
    "You won $1000 click here", "Hi, can we reschedule?",
    "Free entry to prize draw", "Project update attached",
    "Congratulations you won", "Lunch plans for today?",
    "Claim your reward now", "Team standup in 5 minutes",
    "Limited offer buy now", "Can you review this PR?",
    "Get rich quick scheme", "Happy birthday!",
    "Free trial no credit card", "Quarterly report attached",
    "Act now limited time", "See you at the conference",
    "Exclusive deal for you", "Thanks for your help",
]
labels = [1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0,1,0]  # 1=spam, 0=ham

X_train, X_test, y_train, y_test = train_test_split(
    emails, labels, test_size=0.3, random_state=42
)

# Build pipeline
spam_pipeline = Pipeline([
    ('tfidf', TfidfVectorizer(
        lowercase=True,          # Convert to lowercase
        stop_words='english',    # Remove common words
        ngram_range=(1, 2),      # Unigrams + bigrams
        max_features=5000,       # Vocabulary size limit
        sublinear_tf=True        # Apply log normalization to TF
    )),
    ('nb', MultinomialNB(alpha=0.1))  # Smoothing parameter
])

# Train and evaluate
spam_pipeline.fit(X_train, y_train)
y_pred = spam_pipeline.predict(X_test)

print(f"Accuracy: {spam_pipeline.score(X_test, y_test):.4f}")
print("\nClassification Report:")
print(classification_report(y_test, y_pred, target_names=['Ham', 'Spam']))

# Predict new emails
new_emails = [
    "Free money no strings attached",
    "Can we meet at 2pm?",
    "You've been selected to win"
]
predictions = spam_pipeline.predict(new_emails)
probas = spam_pipeline.predict_proba(new_emails)

for email, pred, proba in zip(new_emails, predictions, probas):
    label = "SPAM" if pred == 1 else "HAM"
    conf = max(proba) * 100
    print(f"  [{label}] ({conf:.0f}%) '{email}'")
```

### Feature Analysis — Most Informative Words

```python
# Which words are most indicative of each class?
def show_top_features(pipeline, n=10):
    """Show most informative features for each class."""
    feature_names = pipeline.named_steps['tfidf'].get_feature_names_out()
    
    # Log probabilities for each class
    log_probs = pipeline.named_steps['nb'].feature_log_prob_
    
    for i, class_name in enumerate(['Ham', 'Spam']):
        top_indices = np.argsort(log_probs[i])[-n:][::-1]
        top_words = [feature_names[j] for j in top_indices]
        print(f"\nTop {n} words for {class_name}:")
        print(f"  {', '.join(top_words)}")

show_top_features(spam_pipeline)
```

---

## 14. Laplace Smoothing

### The Zero-Probability Problem

If a word NEVER appears in a class during training, its probability = 0, which makes the ENTIRE product = 0:

$$P(\text{Spam}|\text{email}) \propto P(y) \times ... \times \underbrace{P(\text{"blockchain"}|\text{Spam})}_{= 0!} \times ... = 0$$

This is catastrophic — one unseen word kills the entire classification!

### The Fix: Laplace (Add-α) Smoothing

$$P(x_i|y) = \frac{\text{count}(x_i, y) + \alpha}{\text{total count in class } y + \alpha \cdot |V|}$$

Where:
- $\alpha$ = smoothing parameter (usually 1.0)
- $|V|$ = vocabulary size (number of features)

**Effect**: No probability is ever exactly zero.

```python
# Alpha controls smoothing strength
from sklearn.naive_bayes import MultinomialNB

# alpha=1.0: Standard Laplace smoothing (default)
nb_standard = MultinomialNB(alpha=1.0)

# alpha=0.1: Less smoothing (trust training data more)
nb_less_smooth = MultinomialNB(alpha=0.1)

# alpha=0.01: Almost no smoothing (careful — zero prob possible)
nb_minimal = MultinomialNB(alpha=0.01)

# Find best alpha with cross-validation
from sklearn.model_selection import GridSearchCV

param_grid = {'nb__alpha': [0.001, 0.01, 0.1, 0.5, 1.0, 2.0, 5.0]}
grid = GridSearchCV(spam_pipeline, param_grid, cv=5, scoring='accuracy')
# grid.fit(X_train, y_train)
# print(f"Best alpha: {grid.best_params_}")
```

### Alpha Selection Guide

| Alpha | Effect | Use When |
|-------|--------|----------|
| 0.001 | Almost no smoothing | Large training data, low vocab |
| 0.1 | Light smoothing | Moderate training data |
| 1.0 | Standard (Laplace) | Default, works well generally |
| 2.0+ | Heavy smoothing | Very small training data |

---

## 15. Practical Implementation — Naive Bayes

### Gaussian NB for Numeric Data

```python
from sklearn.naive_bayes import GaussianNB
from sklearn.datasets import load_wine
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import classification_report
import numpy as np

# Load wine dataset
data = load_wine()
X, y = data.data, data.target

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Gaussian NB — note: scaling is NOT required (it estimates mean/var per class)
# But it won't hurt either
gnb = GaussianNB(var_smoothing=1e-9)  # Small value added to variance
gnb.fit(X_train, y_train)

print(f"Training Accuracy: {gnb.score(X_train, y_train):.4f}")
print(f"Test Accuracy: {gnb.score(X_test, y_test):.4f}")
print(f"\nClassification Report:")
print(classification_report(y_test, gnb.predict(X_test), 
                           target_names=data.target_names))

# Incremental learning — NB supports partial_fit!
# Useful for streaming data / large datasets that don't fit in memory
gnb_incremental = GaussianNB()
# Process data in batches
batch_size = 20
classes = np.unique(y_train)

for i in range(0, len(X_train), batch_size):
    X_batch = X_train[i:i+batch_size]
    y_batch = y_train[i:i+batch_size]
    gnb_incremental.partial_fit(X_batch, y_batch, classes=classes)

print(f"\nIncremental NB Accuracy: {gnb_incremental.score(X_test, y_test):.4f}")
```

### Comparing All NB Variants on Real Data

```python
from sklearn.naive_bayes import (GaussianNB, MultinomialNB, 
                                  BernoulliNB, ComplementNB)
from sklearn.datasets import fetch_20newsgroups
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.model_selection import cross_val_score
from sklearn.pipeline import Pipeline

# Load real text data
categories = ['alt.atheism', 'talk.religion.misc', 
              'comp.graphics', 'sci.space']
newsgroups = fetch_20newsgroups(subset='all', categories=categories)

# Test different NB variants
results = {}

for name, model in [
    ('Multinomial', MultinomialNB(alpha=0.1)),
    ('Complement', ComplementNB(alpha=0.1)),
    ('Bernoulli', BernoulliNB(alpha=0.1)),
]:
    pipe = Pipeline([
        ('tfidf', TfidfVectorizer(max_features=10000, stop_words='english')),
        ('nb', model)
    ])
    scores = cross_val_score(pipe, newsgroups.data, newsgroups.target, cv=5)
    results[name] = scores.mean()
    print(f"{name:15s}: {scores.mean():.4f} ± {scores.std():.4f}")

# Also test Gaussian (need dense matrix)
pipe_gaussian = Pipeline([
    ('tfidf', TfidfVectorizer(max_features=500, stop_words='english')),
    # GaussianNB needs dense input
])
# Note: GaussianNB is usually NOT the best choice for text data
```

---

## 16. KNN vs Naive Bayes Comparison

| Aspect | KNN | Naive Bayes |
|--------|-----|-------------|
| **Type** | Instance-based (lazy) | Probabilistic (eager) |
| **Training Speed** | O(1) — no training | O(n·d) — compute statistics |
| **Prediction Speed** | O(n·d) — slow | O(d) — very fast |
| **Memory** | Stores ALL data | Stores only statistics |
| **Feature Scaling** | Required | Not required |
| **Handles Missing Values** | Needs imputation | Naturally handles (skip) |
| **Feature Independence** | Not assumed | Assumed (naive) |
| **Best For** | Small data, non-linear | Text, high-dim, streaming |
| **Curse of Dimensionality** | Severely affected | Not affected |
| **Incremental Learning** | Add points easily | partial_fit() supported |
| **Probability Calibration** | Poor | Often overconfident |
| **When to Pick** | Low dims, need non-linearity | High dims, need speed |

### Decision Guide

```
Need fast training + fast prediction? → Naive Bayes
Text classification? → Naive Bayes (Multinomial)
Few features, lots of data? → KNN (with distance weighting)
High dimensional data? → Naive Bayes
Need a quick baseline? → Both! Compare them
Streaming/online learning? → Naive Bayes (partial_fit)
Non-linear decision boundary? → KNN
Feature interactions matter? → KNN (NB assumes independence)
```

---

## 17. Common Mistakes

### KNN Mistakes

**Mistake 1: Not Scaling Features**
```python
# BAD — Feature with range [0, 1000000] dominates distance!
knn = KNeighborsClassifier()
knn.fit(X_raw, y)

# GOOD — Always scale for KNN
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
knn_pipe = Pipeline([('scaler', StandardScaler()), ('knn', KNeighborsClassifier())])
```

**Mistake 2: Using KNN on High-Dimensional Data Without Reduction**
```python
# BAD — 500 features with KNN
knn.fit(X_500_features, y)  # All distances become similar

# GOOD — Reduce dimensions first
from sklearn.decomposition import PCA
pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('pca', PCA(n_components=20)),
    ('knn', KNeighborsClassifier())
])
```

**Mistake 3: Even K for Binary Classification**
```python
# BAD — K=4 can lead to ties (2 vs 2)
knn = KNeighborsClassifier(n_neighbors=4)

# GOOD — Use odd K for binary classification
knn = KNeighborsClassifier(n_neighbors=5)
```

**Mistake 4: Not Trying Distance Weighting**
```python
# Often better than uniform — always try both!
knn = KNeighborsClassifier(n_neighbors=7, weights='distance')
```

### Naive Bayes Mistakes

**Mistake 5: Using Gaussian NB for Text Data**
```python
# BAD — Gaussian NB assumes normal distribution (text counts aren't normal)
gnb = GaussianNB()
gnb.fit(tfidf_matrix.toarray(), y)

# GOOD — Use MultinomialNB for text
mnb = MultinomialNB(alpha=0.1)
mnb.fit(tfidf_matrix, y)
```

**Mistake 6: Forgetting Smoothing for Small Datasets**
```python
# BAD — With small data, unseen features get zero probability
mnb = MultinomialNB(alpha=0)  # No smoothing!

# GOOD — Always use some smoothing
mnb = MultinomialNB(alpha=1.0)  # Laplace smoothing
```

**Mistake 7: Trusting NB Probability Estimates**
```python
# BAD — NB probabilities are often poorly calibrated
# Don't use predict_proba() for threshold-based decisions without calibration

# GOOD — Calibrate probabilities if you need them
from sklearn.calibration import CalibratedClassifierCV
calibrated_nb = CalibratedClassifierCV(MultinomialNB(), cv=5)
calibrated_nb.fit(X_train, y_train)
# Now predict_proba() gives better calibrated probabilities
```

**Mistake 8: Using Negative Values with MultinomialNB**
```python
# BAD — MultinomialNB requires non-negative features
from sklearn.preprocessing import StandardScaler
X_scaled = StandardScaler().fit_transform(X)  # Can be negative!
MultinomialNB().fit(X_scaled, y)  # ERROR!

# GOOD — Use MinMaxScaler or leave unscaled for MultinomialNB
from sklearn.preprocessing import MinMaxScaler
X_positive = MinMaxScaler().fit_transform(X)  # [0, 1] range
MultinomialNB().fit(X_positive, y)  # Works!
```

---

## 18. Interview Questions

### KNN Questions

**Q1: Why is KNN called a "lazy learner"?**
> KNN doesn't learn any model parameters during training — it simply memorizes the entire training dataset. All computation happens during prediction time when distances are calculated. This is in contrast to "eager learners" (like Decision Trees, SVM) that build a model during training.

**Q2: How does the choice of K affect bias and variance?**
> K=1: Zero bias, maximum variance (captures noise perfectly). K=n: Maximum bias (always predicts majority class), zero variance. As K increases, bias increases and variance decreases. The optimal K balances this tradeoff. Typically found via cross-validation.

**Q3: What's the time complexity of KNN?**
> Training: O(1) — just stores data. Prediction: O(n·d) per query (compute distance to all n points in d dimensions). This makes KNN slow for large datasets. KD-Trees reduce prediction to O(d·log n) for low-dimensional data.

**Q4: How do you handle categorical features in KNN?**
> Options: (1) One-hot encode (increases dimensionality); (2) Use Hamming distance for categorical features; (3) Use Gower distance (mixed metric for numeric + categorical); (4) Label encode ordinal features. One-hot encoding is most common.

**Q5: When would you prefer KNN over complex models?**
> When: (1) Dataset is small; (2) Decision boundary is highly non-linear; (3) Quick baseline needed; (4) Data distribution changes over time (add/remove points easily); (5) Interpretability of predictions matters (show similar examples).

### Naive Bayes Questions

**Q6: Why is the "naive" assumption useful despite being wrong?**
> Three reasons: (1) For classification, we only need correct ranking of P(y|x), not exact values — independence violations may not change rankings; (2) Estimation of $P(x_i|y)$ is much more data-efficient than $P(x_1,...,x_n|y)$, which requires exponentially more data; (3) In high-dimensional spaces, modeling full joint distributions is intractable — NB makes it tractable.

**Q7: When does Naive Bayes perform poorly?**
> When: (1) Features are strongly correlated (violates independence heavily); (2) When exact probability estimates are needed (NB is poorly calibrated); (3) When feature interactions determine the class (e.g., XOR problem); (4) When you have enough data for more complex models.

**Q8: What's the difference between Multinomial and Bernoulli NB for text?**
> Multinomial: Uses word frequencies/counts — "free" appearing 5 times matters more than appearing once. Bernoulli: Only considers presence/absence of words — penalizes absent words. For long documents, Multinomial is better. For short text (tweets, SMS), Bernoulli can work well.

**Q9: How does Naive Bayes handle the zero-frequency problem?**
> Laplace smoothing (additive smoothing): Add α to every count. This ensures no probability is ever zero. α=1 is standard Laplace; smaller α (like 0.01) gives less smoothing. Without smoothing, a single unseen word would make the entire product zero.

**Q10: Can Naive Bayes handle continuous features?**
> Yes, via Gaussian Naive Bayes, which assumes features follow a normal distribution within each class. It estimates μ and σ for each feature-class pair. Alternatives: discretize continuous features into bins and use Multinomial NB.

**Q11: Why is Naive Bayes so fast?**
> Training: Just compute mean/variance (Gaussian) or count frequencies (Multinomial) — one pass through data, O(n·d). Prediction: Multiply d probabilities and compare classes — O(d·k) where k=number of classes. No distance calculations, no iterations, no optimization.

**Q12: Compare KNN and Naive Bayes for a spam detection system.**
> NB wins for spam detection because: (1) Text is high-dimensional (curse of dim hurts KNN); (2) NB is much faster at prediction (crucial for real-time email filtering); (3) NB naturally handles varying document lengths; (4) NB can be incrementally updated with partial_fit; (5) Independence assumption works reasonably well for word occurrences. KNN would need all emails in memory and compute distances to all of them for each new email.

---

## 19. Quick Reference

### KNN Cheat Sheet

| Aspect | Detail |
|--------|--------|
| Type | Supervised (Classification & Regression) |
| Learning | Lazy (instance-based) |
| Training Time | O(1) |
| Prediction Time | O(n·d) |
| Feature Scaling | **REQUIRED** |
| Key Hyperparameter | K (number of neighbors) |
| Default K Rule | √n or cross-validation |
| Handles Non-linearity | Naturally |
| Curse of Dimensionality | Severely affected (>20-30 dims) |
| Best For | Small data, low dimensions |

### Naive Bayes Cheat Sheet

| Aspect | Detail |
|--------|--------|
| Type | Supervised (Classification) |
| Core Assumption | Feature independence given class |
| Training Time | O(n·d) — single pass |
| Prediction Time | O(d·k) — very fast |
| Feature Scaling | NOT required |
| Key Hyperparameter | alpha (smoothing) |
| Probability Calibration | Poor (overconfident) |
| Curse of Dimensionality | NOT affected |
| Best For | Text classification, high-dim sparse data |
| Incremental Learning | Yes (partial_fit) |

### Algorithm Selection Quick Guide

```
┌─────────────────────────────────────────────┐
│ Text/NLP task?          → MultinomialNB     │
│ Binary features?        → BernoulliNB       │
│ Continuous + Gaussian?  → GaussianNB        │
│ Imbalanced text?        → ComplementNB      │
│ Low-dim, non-linear?    → KNN (distance)    │
│ Need fast prediction?   → Naive Bayes       │
│ Need interpretable?     → KNN (show neighbors)│
│ Streaming data?         → NB (partial_fit)  │
└─────────────────────────────────────────────┘
```

### Essential Imports

```python
# KNN
from sklearn.neighbors import KNeighborsClassifier, KNeighborsRegressor
from sklearn.neighbors import NearestNeighbors  # Just find neighbors
from sklearn.impute import KNNImputer  # Missing value imputation

# Naive Bayes
from sklearn.naive_bayes import GaussianNB       # Continuous features
from sklearn.naive_bayes import MultinomialNB    # Word counts / TF-IDF
from sklearn.naive_bayes import BernoulliNB      # Binary features
from sklearn.naive_bayes import ComplementNB     # Imbalanced data

# Common companions
from sklearn.preprocessing import StandardScaler  # Required for KNN!
from sklearn.feature_extraction.text import TfidfVectorizer  # For NB text
from sklearn.pipeline import Pipeline             # Always use pipelines
from sklearn.calibration import CalibratedClassifierCV  # Fix NB probabilities
```

---

*Previous: [05 - SVM (Support Vector Machines)](05-SVM-Support-Vector-Machines.md)*  
*Next: [07 - Ensemble Methods](07-Ensemble-Methods.md)*
