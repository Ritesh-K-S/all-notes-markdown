# Chapter 01: Introduction to Machine Learning

## Table of Contents
- [What is Machine Learning?](#what-is-machine-learning)
- [Types of Machine Learning](#types-of-machine-learning)
- [The ML Workflow](#the-ml-workflow)
- [Bias-Variance Tradeoff](#bias-variance-tradeoff)
- [Overfitting and Underfitting](#overfitting-and-underfitting)
- [Train-Test Split and Validation](#train-test-split-and-validation)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Machine Learning?

### What It Is
Machine Learning is teaching computers to learn patterns from data — without being explicitly programmed for every scenario.

**Analogy for a 15-year-old:** Imagine you're learning to identify dogs vs. cats. Nobody gives you a rulebook ("if ears are pointy AND nose is small, then cat"). Instead, you look at thousands of pictures, and your brain figures out the patterns. That's exactly what ML does — but with math.

### Formal Definition
> Machine Learning is the field of study that gives computers the ability to learn without being explicitly programmed. — Arthur Samuel (1959)

More precisely: A computer program is said to **learn** from experience $E$ with respect to some task $T$ and performance measure $P$, if its performance on $T$, as measured by $P$, improves with experience $E$. — Tom Mitchell (1997)

### Why It Matters
- **Scale:** Humans can't manually write rules for millions of scenarios (spam detection, recommendations)
- **Adaptability:** ML models improve as they see more data — rules-based systems don't
- **Discovery:** ML finds patterns humans can't see (cancer detection, fraud patterns)
- **Automation:** Automates decision-making at scale (credit scoring, autonomous driving)

### Traditional Programming vs. ML

```
Traditional Programming:
┌─────────────┐     ┌───────────┐     ┌──────────┐
│    Data      │ ──→ │  Program  │ ──→ │  Output  │
│  (Input)     │     │  (Rules)  │     │          │
└─────────────┘     └───────────┘     └──────────┘

Machine Learning:
┌─────────────┐     ┌───────────┐     ┌──────────┐
│    Data      │ ──→ │ Learning  │ ──→ │  Model   │
│  + Output    │     │ Algorithm │     │ (Rules)  │
└─────────────┘     └───────────┘     └──────────┘
```

---

## Types of Machine Learning

### Overview Diagram

```
                    Machine Learning
                         │
          ┌──────────────┼──────────────┐
          │              │              │
    Supervised     Unsupervised    Reinforcement
    Learning       Learning        Learning
          │              │              │
    ┌─────┴─────┐  ┌────┴────┐    Agent learns
    │           │  │         │    by trial &
Classification  Regression  Clustering  Dimensionality  error
                            Reduction
```

---

### 1. Supervised Learning

#### What It Is
You teach the model using **labeled data** — data where you already know the correct answer.

**Analogy:** A teacher gives you questions WITH answer keys. You practice, check your answers, and improve.

#### How It Works
- Input: Features ($X$) + Labels ($y$)
- Goal: Learn a function $f$ such that $f(X) \approx y$
- After training: Use $f$ to predict $y$ for new, unseen $X$

#### Two Main Types

| Type | Output | Example | Algorithms |
|------|--------|---------|------------|
| **Classification** | Discrete categories | Spam/Not Spam, Cat/Dog | Logistic Regression, SVM, Random Forest, Neural Networks |
| **Regression** | Continuous numbers | House price, Temperature | Linear Regression, Polynomial Regression, SVR |

#### Code Example: Supervised Learning (Classification)

```python
# Complete supervised learning pipeline
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, classification_report

# Step 1: Load labeled data (features + labels)
iris = load_iris()
X = iris.data       # Features: sepal length, sepal width, petal length, petal width
y = iris.target     # Labels: 0=setosa, 1=versicolor, 2=virginica

# Step 2: Split into training and testing sets (80/20)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Step 3: Choose and train a model
model = RandomForestClassifier(n_estimators=100, random_state=42)
model.fit(X_train, y_train)  # Model learns patterns from labeled training data

# Step 4: Make predictions on unseen data
y_pred = model.predict(X_test)

# Step 5: Evaluate performance
print(f"Accuracy: {accuracy_score(y_test, y_pred):.4f}")
print(classification_report(y_test, y_pred, target_names=iris.target_names))
```

**Output:**
```
Accuracy: 1.0000
              precision    recall  f1-score   support
      setosa       1.00      1.00      1.00        10
  versicolor       1.00      1.00      1.00         9
   virginica       1.00      1.00      1.00        11
    accuracy                           1.00        30
```

---

### 2. Unsupervised Learning

#### What It Is
The model finds hidden patterns in **unlabeled data** — no correct answers provided.

**Analogy:** You're given a bag of mixed coins from different countries. Nobody tells you which country each coin belongs to. You group them by shape, size, color, and weight on your own.

#### How It Works
- Input: Features ($X$) only — NO labels
- Goal: Discover structure, patterns, or groupings in data
- Output: Cluster assignments, reduced dimensions, associations

#### Main Types

| Type | Goal | Example | Algorithms |
|------|------|---------|------------|
| **Clustering** | Group similar data | Customer segments | K-Means, DBSCAN, Hierarchical |
| **Dimensionality Reduction** | Compress features | Visualization, noise removal | PCA, t-SNE, UMAP |
| **Association** | Find rules/patterns | Market basket analysis | Apriori, FP-Growth |
| **Anomaly Detection** | Find outliers | Fraud detection | Isolation Forest, One-Class SVM |

#### Code Example: Unsupervised Learning (Clustering)

```python
# K-Means clustering on unlabeled data
from sklearn.datasets import make_blobs
from sklearn.cluster import KMeans
import numpy as np

# Step 1: Generate unlabeled data (3 natural clusters)
X, _ = make_blobs(n_samples=300, centers=3, cluster_std=0.60, random_state=42)
# Note: We discard labels (_) because unsupervised learning doesn't use them

# Step 2: Apply K-Means clustering
kmeans = KMeans(n_clusters=3, random_state=42, n_init=10)
kmeans.fit(X)  # Model discovers clusters on its own

# Step 3: Get cluster assignments
labels = kmeans.labels_          # Which cluster each point belongs to
centers = kmeans.cluster_centers_ # Center of each cluster

print(f"Cluster sizes: {np.bincount(labels)}")
print(f"Cluster centers:\n{centers}")
print(f"Inertia (lower = tighter clusters): {kmeans.inertia_:.2f}")
```

**Output:**
```
Cluster sizes: [100 100 100]
Cluster centers:
[[-1.56  4.06]
 [ 1.98  0.87]
 [-7.94 -3.75]]
Inertia: 211.35
```

---

### 3. Reinforcement Learning

#### What It Is
An **agent** learns by interacting with an **environment**, taking **actions**, and receiving **rewards** or **penalties**.

**Analogy:** Training a puppy. You don't show it exactly what to do — it tries things, and you reward good behavior and ignore/punish bad behavior. Over time, it learns the best strategy.

#### How It Works

```
┌─────────────────────────────────────────────┐
│                                             │
│    ┌─────────┐   Action    ┌────────────┐  │
│    │         │ ──────────→ │            │  │
│    │  Agent  │             │Environment │  │
│    │         │ ←────────── │            │  │
│    └─────────┘  State +    └────────────┘  │
│                  Reward                     │
└─────────────────────────────────────────────┘
```

#### Key Concepts

| Concept | Definition | Example (Chess) |
|---------|-----------|-----------------|
| **Agent** | The learner/decision-maker | The chess AI |
| **Environment** | What the agent interacts with | The chess board |
| **State** | Current situation | Board position |
| **Action** | What agent can do | Move a piece |
| **Reward** | Feedback signal | +1 for winning, -1 for losing |
| **Policy** | Strategy (state → action mapping) | Which move to make given a position |

#### Applications
- Game AI (AlphaGo, OpenAI Five)
- Robotics (walking, grasping)
- Autonomous driving
- Stock trading
- Resource management

---

### 4. Semi-Supervised Learning

A hybrid approach: small amount of **labeled data** + large amount of **unlabeled data**.

**Why it exists:** Labeling data is expensive (requires human experts), but unlabeled data is cheap.

**Example:** You have 100 labeled medical images + 10,000 unlabeled ones. The model uses the labeled data to learn initial patterns, then applies them to the unlabeled data to improve.

---

### 5. Self-Supervised Learning

The model creates its own labels from the data structure itself.

**Examples:**
- BERT: Masks words and predicts them ("The cat sat on the [MASK]" → "mat")
- GPT: Predicts the next word
- Contrastive Learning: Different augmentations of same image = same class

---

### Comparison Table

| Aspect | Supervised | Unsupervised | Reinforcement |
|--------|-----------|--------------|---------------|
| **Data** | Labeled | Unlabeled | Agent + Environment |
| **Goal** | Predict output | Find patterns | Maximize reward |
| **Feedback** | Correct answer | None | Reward signal |
| **Example** | Email → Spam? | Group customers | Play chess |
| **Challenge** | Need labeled data | Hard to evaluate | Slow to train |

---

## The ML Workflow

### Complete Pipeline

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Problem  │──→│  Data    │──→│ Feature  │──→│  Model   │──→│  Model   │
│Definition│   │Collection│   │Engineering│  │ Training │   │Evaluation│
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
                                                                   │
┌──────────┐   ┌──────────┐                                        │
│Production│←──│  Model   │←───────────────────────────────────────┘
│Monitoring│   │Deployment│
└──────────┘   └──────────┘
```

### Step-by-Step Breakdown

#### Step 1: Problem Definition
- What are you trying to predict/discover?
- Is this a classification, regression, or clustering problem?
- What metric defines "success"?
- What data do you have access to?

#### Step 2: Data Collection & Exploration

```python
import pandas as pd
import numpy as np

# Load data
df = pd.read_csv('data.csv')

# Explore structure
print(df.shape)              # (rows, columns)
print(df.dtypes)             # Data types
print(df.describe())         # Statistical summary
print(df.isnull().sum())     # Missing values per column
print(df.duplicated().sum()) # Duplicate rows
```

#### Step 3: Data Preprocessing & Feature Engineering

```python
from sklearn.preprocessing import StandardScaler, LabelEncoder
from sklearn.impute import SimpleImputer

# Handle missing values
imputer = SimpleImputer(strategy='median')  # Fill missing with median
X_imputed = imputer.fit_transform(X)

# Encode categorical variables
le = LabelEncoder()
df['category_encoded'] = le.fit_transform(df['category'])

# Scale features (important for distance-based algorithms)
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X_imputed)
```

#### Step 4: Model Selection & Training

```python
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.svm import SVC

# Try multiple models
models = {
    'Logistic Regression': LogisticRegression(max_iter=1000),
    'Random Forest': RandomForestClassifier(n_estimators=100),
    'Gradient Boosting': GradientBoostingClassifier(),
    'SVM': SVC(kernel='rbf')
}

for name, model in models.items():
    model.fit(X_train, y_train)
    score = model.score(X_test, y_test)
    print(f"{name}: {score:.4f}")
```

#### Step 5: Model Evaluation

```python
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score,
    f1_score, confusion_matrix, roc_auc_score
)
from sklearn.model_selection import cross_val_score

# Cross-validation (more reliable than single train-test split)
cv_scores = cross_val_score(model, X, y, cv=5, scoring='accuracy')
print(f"CV Accuracy: {cv_scores.mean():.4f} ± {cv_scores.std():.4f}")

# Detailed metrics
y_pred = model.predict(X_test)
print(f"Precision: {precision_score(y_test, y_pred, average='weighted'):.4f}")
print(f"Recall: {recall_score(y_test, y_pred, average='weighted'):.4f}")
print(f"F1-Score: {f1_score(y_test, y_pred, average='weighted'):.4f}")
```

#### Step 6: Hyperparameter Tuning

```python
from sklearn.model_selection import GridSearchCV

# Define parameter grid
param_grid = {
    'n_estimators': [50, 100, 200],
    'max_depth': [3, 5, 10, None],
    'min_samples_split': [2, 5, 10]
}

# Grid search with cross-validation
grid_search = GridSearchCV(
    RandomForestClassifier(random_state=42),
    param_grid,
    cv=5,
    scoring='accuracy',
    n_jobs=-1  # Use all CPU cores
)
grid_search.fit(X_train, y_train)

print(f"Best Parameters: {grid_search.best_params_}")
print(f"Best CV Score: {grid_search.best_score_:.4f}")
```

---

## Bias-Variance Tradeoff

### What It Is
The fundamental tension in ML between two types of error that pull in opposite directions.

**Analogy:** Imagine you're an archer:
- **High Bias** = Your arrows consistently land far from the bullseye (you aim wrong)
- **High Variance** = Your arrows scatter all over (you're inconsistent)
- **Ideal** = Arrows clustered tightly on the bullseye

```
    High Bias          High Variance        High Bias +        Low Bias +
    Low Variance       Low Bias             High Variance      Low Variance
                                                               (IDEAL)
    ┌─────────┐       ┌─────────┐          ┌─────────┐       ┌─────────┐
    │  · ·    │       │·       ·│          │ ·    ·  │       │         │
    │   ··    │       │  ◎      │          │    ◎  · │       │   ·◎·   │
    │    ◎    │       │      ·  │          │  ·     ·│       │    ··   │
    │         │       │ ·    ·  │          │      ·  │       │         │
    └─────────┘       └─────────┘          └─────────┘       └─────────┘
    Underfitting       Overfitting          Worst case         Best case
```

### Mathematical Formulation

For any model, the expected prediction error can be decomposed:

$$\text{Total Error} = \text{Bias}^2 + \text{Variance} + \text{Irreducible Noise}$$

Where:
- $\text{Bias} = E[\hat{f}(x)] - f(x)$ — How far off the average prediction is from the truth
- $\text{Variance} = E[(\hat{f}(x) - E[\hat{f}(x)])^2]$ — How much predictions vary across different training sets
- $\text{Irreducible Noise} = \sigma^2$ — Random noise in the data that no model can capture

### The Tradeoff

| Model Complexity | Bias | Variance | Training Error | Test Error |
|-----------------|------|----------|----------------|------------|
| Too simple (underfitting) | HIGH | LOW | HIGH | HIGH |
| Just right | MEDIUM | MEDIUM | MEDIUM | LOW |
| Too complex (overfitting) | LOW | HIGH | VERY LOW | HIGH |

```
Error
  │
  │╲                    ╱
  │ ╲   Total Error   ╱
  │  ╲    ╱─────╲    ╱
  │   ╲  ╱       ╲  ╱
  │    ╲╱    ★     ╲╱    ← Optimal complexity (sweet spot)
  │    ╱╲         ╱
  │   ╱  ╲───────╱  Variance
  │  ╱    Bias²
  │ ╱
  │╱
  └────────────────────── Model Complexity
  Simple                         Complex
```

### Code Example: Visualizing Bias-Variance

```python
import numpy as np
from sklearn.preprocessing import PolynomialFeatures
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_squared_error

# Generate noisy data from a true quadratic function
np.random.seed(42)
X = np.sort(np.random.uniform(0, 1, 30)).reshape(-1, 1)
y_true = np.sin(2 * np.pi * X).ravel()  # True function
y = y_true + np.random.normal(0, 0.3, 30)  # Add noise

# Fit models with different complexity (polynomial degree)
degrees = [1, 3, 15]  # Underfitting, Good fit, Overfitting

for degree in degrees:
    # Create polynomial features
    poly = PolynomialFeatures(degree=degree)
    X_poly = poly.fit_transform(X)
    
    # Fit model
    model = LinearRegression()
    model.fit(X_poly, y)
    y_pred = model.predict(X_poly)
    
    # Calculate error
    mse = mean_squared_error(y, y_pred)
    print(f"Degree {degree:2d}: MSE = {mse:.4f} (coefficients: {model.coef_.shape[0]})")

# Output:
# Degree  1: MSE = 0.2156 (coefficients: 2)   ← Underfitting (high bias)
# Degree  3: MSE = 0.0842 (coefficients: 4)   ← Good balance
# Degree 15: MSE = 0.0091 (coefficients: 16)  ← Overfitting (high variance)
```

> **Pro Tip:** Low training error with degree 15 doesn't mean it's the best model! Test it on new data and it will perform poorly. Always evaluate on a held-out test set.

---

## Overfitting and Underfitting

### What They Are

| Condition | Description | Symptom | Model Example |
|-----------|-------------|---------|---------------|
| **Underfitting** | Model too simple to capture patterns | High train error, High test error | Linear model on non-linear data |
| **Good Fit** | Model captures true patterns | Low train error, Low test error | Appropriate complexity |
| **Overfitting** | Model memorizes noise | Very low train error, High test error | Deep tree on small data |

### How to Detect

```python
from sklearn.model_selection import learning_curve

def check_fit(model, X, y):
    """Check if model is underfitting, overfitting, or good fit."""
    train_sizes, train_scores, val_scores = learning_curve(
        model, X, y, cv=5, 
        train_sizes=np.linspace(0.1, 1.0, 10),
        scoring='neg_mean_squared_error'
    )
    
    train_mean = -train_scores.mean(axis=1)
    val_mean = -val_scores.mean(axis=1)
    
    # Diagnosis
    final_train_error = train_mean[-1]
    final_val_error = val_mean[-1]
    gap = final_val_error - final_train_error
    
    print(f"Train Error: {final_train_error:.4f}")
    print(f"Validation Error: {final_val_error:.4f}")
    print(f"Gap: {gap:.4f}")
    
    if final_train_error > 0.5 and final_val_error > 0.5:
        print("→ UNDERFITTING: Both errors high. Increase model complexity.")
    elif final_train_error < 0.1 and gap > 0.3:
        print("→ OVERFITTING: Low train error but large gap. Regularize or get more data.")
    else:
        print("→ GOOD FIT: Reasonable balance.")
```

### Solutions

#### For Underfitting (High Bias):
1. **Increase model complexity** (more features, deeper trees, more layers)
2. **Add polynomial/interaction features**
3. **Reduce regularization**
4. **Train longer** (for iterative algorithms)
5. **Try a more powerful algorithm**

#### For Overfitting (High Variance):
1. **Get more training data** (best solution if possible)
2. **Reduce model complexity** (fewer features, shallower trees)
3. **Add regularization** (L1, L2, dropout)
4. **Use cross-validation**
5. **Early stopping** (stop training before overfitting)
6. **Ensemble methods** (bagging reduces variance)
7. **Data augmentation** (artificially increase training data)

```python
from sklearn.linear_model import Ridge, Lasso

# Without regularization (prone to overfitting)
model_overfit = LinearRegression()

# With L2 regularization (Ridge) — penalizes large coefficients
model_ridge = Ridge(alpha=1.0)  # alpha controls regularization strength

# With L1 regularization (Lasso) — can zero out coefficients (feature selection)
model_lasso = Lasso(alpha=0.1)

# Higher alpha = more regularization = simpler model = less overfitting
for alpha in [0.01, 0.1, 1.0, 10.0]:
    model = Ridge(alpha=alpha)
    model.fit(X_train, y_train)
    train_score = model.score(X_train, y_train)
    test_score = model.score(X_test, y_test)
    print(f"Alpha={alpha:5.2f}: Train R²={train_score:.4f}, Test R²={test_score:.4f}")
```

---

## Train-Test Split and Validation

### Why Split Data?

If you evaluate your model on the same data it trained on, you're checking if it can **memorize** — not if it can **generalize**. That's like studying for an exam and then taking the exact same exam.

### Splitting Strategies

#### 1. Simple Train-Test Split

```python
from sklearn.model_selection import train_test_split

# Standard 80/20 split
X_train, X_test, y_train, y_test = train_test_split(
    X, y, 
    test_size=0.2,      # 20% for testing
    random_state=42,    # For reproducibility
    stratify=y          # Maintain class proportions (for classification)
)
```

> **Warning:** A single split can be misleading if you happen to get an "easy" or "hard" test set. Use cross-validation for reliable estimates.

#### 2. Train-Validation-Test Split

```
┌──────────────────────────────────────────────────────────────┐
│                    Full Dataset                               │
├────────────────────────────┬──────────────┬─────────────────┤
│     Training (60%)         │  Val (20%)   │   Test (20%)    │
│  Used to train model       │ Tune params  │ Final eval ONLY │
└────────────────────────────┴──────────────┴─────────────────┘
```

```python
# Two-step split for train/val/test
X_temp, X_test, y_temp, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)
X_train, X_val, y_train, y_val = train_test_split(
    X_temp, y_temp, test_size=0.25, random_state=42  # 0.25 × 0.8 = 0.2
)

print(f"Train: {len(X_train)}, Val: {len(X_val)}, Test: {len(X_test)}")
```

#### 3. K-Fold Cross-Validation

```
Fold 1: [VAL ][TRAIN][TRAIN][TRAIN][TRAIN]
Fold 2: [TRAIN][VAL ][TRAIN][TRAIN][TRAIN]
Fold 3: [TRAIN][TRAIN][VAL ][TRAIN][TRAIN]
Fold 4: [TRAIN][TRAIN][TRAIN][VAL ][TRAIN]
Fold 5: [TRAIN][TRAIN][TRAIN][TRAIN][VAL ]
```

```python
from sklearn.model_selection import cross_val_score, KFold, StratifiedKFold

# Standard K-Fold (k=5)
kfold = KFold(n_splits=5, shuffle=True, random_state=42)

# Stratified K-Fold (preserves class distribution — use for classification)
skfold = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)

# Quick cross-validation
scores = cross_val_score(model, X, y, cv=skfold, scoring='accuracy')
print(f"Accuracy: {scores.mean():.4f} ± {scores.std():.4f}")
```

> **Pro Tip:** Use `StratifiedKFold` for imbalanced datasets. Regular `KFold` might create folds with no samples from minority class.

#### 4. Time Series Split (for temporal data)

```python
from sklearn.model_selection import TimeSeriesSplit

# Never use random split for time series! Future data would leak into training.
tscv = TimeSeriesSplit(n_splits=5)

# Split 1: Train=[0,1], Test=[2]
# Split 2: Train=[0,1,2], Test=[3]
# Split 3: Train=[0,1,2,3], Test=[4]
# ...
```

---

## Key ML Terminology

| Term | Definition |
|------|-----------|
| **Features** ($X$) | Input variables used to make predictions |
| **Target/Label** ($y$) | The variable we want to predict |
| **Training** | Process of learning patterns from data |
| **Inference** | Using a trained model to make predictions |
| **Hyperparameters** | Settings YOU choose (learning rate, #trees) |
| **Parameters** | Values the MODEL learns (weights, biases) |
| **Epoch** | One pass through the entire training dataset |
| **Batch** | Subset of data used in one training step |
| **Generalization** | Ability to perform well on unseen data |
| **Data Leakage** | When test information leaks into training |

---

## Common Mistakes

### 1. Data Leakage
**Mistake:** Fitting scaler on the entire dataset before splitting.
```python
# WRONG ❌ — Data leakage!
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)  # Scaler sees test data statistics
X_train, X_test = train_test_split(X_scaled, ...)

# CORRECT ✅ — Fit only on training data
X_train, X_test = train_test_split(X, ...)
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)  # Fit on train only
X_test_scaled = scaler.transform(X_test)         # Transform test with train's stats
```

### 2. Not Stratifying for Imbalanced Data
**Mistake:** Random split on imbalanced data may put all minority samples in one set.
```python
# If 95% class A, 5% class B — always stratify
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y  # ← This preserves 95/5 ratio in both sets
)
```

### 3. Using Accuracy for Imbalanced Data
**Mistake:** 95% accuracy on imbalanced dataset (95% class A) means nothing.
```python
# Always check per-class metrics for imbalanced data
from sklearn.metrics import classification_report
print(classification_report(y_test, y_pred))
# Use F1-score, Precision, Recall, or AUC-ROC instead
```

### 4. Ignoring Data Preprocessing
**Mistake:** Feeding raw data with different scales to distance-based algorithms.
```python
# SVM, KNN, Neural Networks need scaled features
# Decision Trees, Random Forests DON'T need scaling
```

### 5. Choosing Algorithm Before Understanding Data
**Mistake:** Jumping to "I'll use a neural network!" without exploring the data first.
- Small dataset (< 1000 rows) → Simpler models often win
- Tabular data → Tree-based models (XGBoost, LightGBM) usually best
- Images → CNNs
- Text → Transformers/RNNs
- Time series → ARIMA, LSTM, or Prophet

### 6. Not Setting random_state
**Mistake:** Getting different results every time you run the code.
```python
# Always set random_state for reproducibility during development
model = RandomForestClassifier(n_estimators=100, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=42)
```

---

## Interview Questions

### Conceptual Questions

**Q1: What is the difference between supervised and unsupervised learning?**
> Supervised learning uses labeled data to learn a mapping from inputs to outputs (classification/regression). Unsupervised learning finds hidden patterns in unlabeled data (clustering/dimensionality reduction). The key difference is whether the training data includes target labels.

**Q2: Explain the bias-variance tradeoff.**
> Bias is error from wrong assumptions (model too simple). Variance is error from sensitivity to training data (model too complex). Total error = Bias² + Variance + Noise. You can't minimize both simultaneously — reducing one increases the other. The goal is finding the sweet spot.

**Q3: How do you detect overfitting?**
> Overfitting shows as a large gap between training performance (high) and validation/test performance (low). Use learning curves, cross-validation, or simply compare train vs. test metrics. If train accuracy is 99% but test accuracy is 60%, the model is overfitting.

**Q4: What is cross-validation and why use it?**
> K-Fold CV splits data into K parts, trains on K-1 folds and tests on the remaining fold, rotating K times. It gives a more reliable performance estimate than a single train-test split because every data point gets used for both training and validation. It helps detect overfitting and makes better use of limited data.

**Q5: When would you use unsupervised learning in a real project?**
> Customer segmentation (group users by behavior), anomaly detection (find fraudulent transactions), dimensionality reduction (reduce 1000 features to 50 for visualization), topic modeling (find themes in documents), and data exploration (understand structure before building supervised models).

**Q6: What is data leakage and how do you prevent it?**
> Data leakage occurs when information from the test set influences training. Examples: fitting a scaler on full data before splitting, using future data to predict past (time series), including target-derived features. Prevention: always split first, then preprocess; use pipelines; be careful with time-ordered data.

### Scenario-Based Questions

**Q7: You have a dataset with 10 million rows and 500 features. How do you approach this?**
> 1) Sample a subset for EDA and prototyping. 2) Use scalable algorithms (SGD-based, tree-based). 3) Consider dimensionality reduction (PCA) or feature selection. 4) Use distributed computing if needed (Spark, Dask). 5) Start simple (linear model) and increase complexity only if needed.

**Q8: Your model has 95% accuracy but the business team says it's not useful. What happened?**
> Likely imbalanced classes. If 95% of data is class A, predicting "always A" gives 95% accuracy but catches 0% of the important class B (e.g., fraud, disease). Solution: Use precision/recall/F1 for the minority class, use AUC-ROC, resample the data, or adjust the decision threshold.

---

## Quick Reference

### Algorithm Selection Guide

| Scenario | Recommended Algorithms |
|----------|----------------------|
| Small data, interpretable | Logistic Regression, Decision Tree |
| Tabular data, best accuracy | XGBoost, LightGBM, CatBoost |
| Image classification | CNN (ResNet, EfficientNet) |
| Text classification | Transformers (BERT), TF-IDF + LR |
| Anomaly detection | Isolation Forest, Autoencoder |
| Clustering | K-Means, DBSCAN, Gaussian Mixture |
| Time series | ARIMA, Prophet, LSTM |
| Reinforcement | DQN, PPO, A3C |

### When to Use What Split Strategy

| Situation | Strategy |
|-----------|----------|
| Large dataset (>100K) | Simple train/val/test split |
| Small dataset (<10K) | K-Fold Cross-Validation (K=5 or 10) |
| Imbalanced classes | Stratified K-Fold |
| Time series data | TimeSeriesSplit |
| Very small dataset (<500) | Leave-One-Out CV |

### Key Formulas

| Formula | Purpose |
|---------|---------|
| $\text{Error} = \text{Bias}^2 + \text{Variance} + \sigma^2$ | Error decomposition |
| $\text{Accuracy} = \frac{TP + TN}{TP + TN + FP + FN}$ | Overall correctness |
| $\text{Precision} = \frac{TP}{TP + FP}$ | Of predicted positive, how many correct? |
| $\text{Recall} = \frac{TP}{TP + FN}$ | Of actual positive, how many caught? |
| $F_1 = 2 \cdot \frac{\text{Precision} \cdot \text{Recall}}{\text{Precision} + \text{Recall}}$ | Harmonic mean of precision and recall |

### Checklist Before Training Any Model

- [ ] Explored data (shape, types, distributions, missing values)
- [ ] Handled missing values appropriately
- [ ] Encoded categorical variables
- [ ] Scaled features (if needed by algorithm)
- [ ] Split data BEFORE any preprocessing that uses statistics
- [ ] Chosen appropriate evaluation metric for the problem
- [ ] Used cross-validation (not just one train-test split)
- [ ] Checked for data leakage
- [ ] Established a baseline (simple model or random guess)
- [ ] Compared multiple algorithms before choosing one

---

*Next Chapter: [02-Linear-Regression.md](02-Linear-Regression.md) — Simple/Multiple Linear Regression, Cost Function, Gradient Descent*
