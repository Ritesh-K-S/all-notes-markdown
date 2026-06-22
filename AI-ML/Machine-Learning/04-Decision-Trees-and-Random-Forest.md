# Chapter 04: Decision Trees and Random Forest

## Table of Contents
- [1. Decision Trees — Introduction](#1-decision-trees--introduction)
- [2. How Decision Trees Work](#2-how-decision-trees-work)
- [3. Splitting Criteria — Entropy, Gini, Information Gain](#3-splitting-criteria--entropy-gini-information-gain)
- [4. Building a Decision Tree Step-by-Step](#4-building-a-decision-tree-step-by-step)
- [5. Pruning — Preventing Overfitting](#5-pruning--preventing-overfitting)
- [6. Decision Trees for Regression](#6-decision-trees-for-regression)
- [7. Ensemble Methods — Bagging](#7-ensemble-methods--bagging)
- [8. Random Forest](#8-random-forest)
- [9. Feature Importance](#9-feature-importance)
- [10. Hyperparameter Tuning](#10-hyperparameter-tuning)
- [11. Common Mistakes](#11-common-mistakes)
- [12. Interview Questions](#12-interview-questions)
- [13. Quick Reference](#13-quick-reference)

---

## 1. Decision Trees — Introduction

### What It Is
A Decision Tree is like a flowchart that asks yes/no questions about your data to make a prediction. Imagine you're deciding whether to play outside:
- Is it raining? → No → Play outside!
- Is it raining? → Yes → Is there an umbrella? → Yes → Play outside!
- Is it raining? → Yes → Is there an umbrella? → No → Stay inside!

That's literally a decision tree — a series of if-then rules learned from data.

### Why It Matters
- **Interpretable**: You can explain predictions to non-technical stakeholders ("The loan was rejected because income < $30K AND debt > $50K")
- **No feature scaling needed**: Unlike SVM/KNN, trees don't care about feature magnitudes
- **Handles mixed data**: Works with both numerical and categorical features
- **Foundation for powerful ensembles**: Random Forest, XGBoost, LightGBM — all built on trees
- **Used everywhere**: Healthcare (diagnosis), Finance (credit scoring), Manufacturing (fault detection)

### When to Use Decision Trees
| Use Decision Trees When... | Don't Use When... |
|---|---|
| Interpretability is critical | You need highest accuracy |
| Data has non-linear relationships | Data is very high-dimensional & sparse |
| You have mixed feature types | You need smooth decision boundaries |
| Quick baseline model needed | Small dataset (prone to overfit) |

---

## 2. How Decision Trees Work

### The Anatomy of a Tree

```
                    [Root Node]
                   Is Age > 30?
                  /            \
               Yes              No
              /                   \
      [Internal Node]         [Internal Node]
     Income > 50K?            Student?
      /        \               /       \
    Yes        No            Yes       No
    /            \            /          \
[Leaf]        [Leaf]      [Leaf]      [Leaf]
Buy=Yes      Buy=No      Buy=Yes     Buy=No
```

### Key Terminology

| Term | Meaning |
|------|---------|
| **Root Node** | Top-most decision node (first split) |
| **Internal Node** | A node that splits into further nodes |
| **Leaf Node** | Terminal node — gives the prediction |
| **Branch** | A connection between nodes (decision path) |
| **Depth** | Length of the longest path from root to leaf |
| **Parent/Child** | Node above/below in the hierarchy |
| **Splitting** | Dividing a node into sub-nodes |

### The Core Algorithm (ID3/C4.5/CART)

```
Algorithm: Build_Decision_Tree(data, features)
─────────────────────────────────────────────
1. If all samples belong to same class → return Leaf(class)
2. If no features left → return Leaf(majority_class)
3. Select BEST feature to split on (using impurity measure)
4. Create a node for that feature
5. For each value/threshold of that feature:
   a. Create a branch
   b. Filter data matching that branch
   c. Recursively call Build_Decision_Tree(filtered_data, remaining_features)
6. Return the node
```

### How Does the Tree "Learn"?
The tree learns by finding the **best question to ask at each step** — the question that most cleanly separates the data into pure groups.

**Analogy**: Think of the "20 Questions" game. A good player asks questions that eliminate the most possibilities. A decision tree does the same — each split maximizes information gained.

---

## 3. Splitting Criteria — Entropy, Gini, Information Gain

### 3.1 Entropy (Used in ID3, C4.5)

**What It Is**: Entropy measures the "disorder" or "uncertainty" in a dataset. 
- Pure dataset (all same class) → Entropy = 0
- Maximum mixed (50/50 split) → Entropy = 1 (for binary)

**Formula**:

$$H(S) = -\sum_{i=1}^{c} p_i \log_2(p_i)$$

Where $p_i$ is the proportion of class $i$ in the dataset $S$, and $c$ is the number of classes.

**Intuition**: If you have a bag with 10 red balls, entropy = 0 (no surprise). If you have 5 red + 5 blue balls, entropy = 1 (maximum surprise when you pick one).

**Examples**:
```
Dataset A: [10 Yes, 0 No]  → H = -1·log₂(1) - 0 = 0         (Pure!)
Dataset B: [5 Yes, 5 No]   → H = -0.5·log₂(0.5) - 0.5·log₂(0.5) = 1  (Maximum impurity)
Dataset C: [8 Yes, 2 No]   → H = -0.8·log₂(0.8) - 0.2·log₂(0.2) = 0.72
```

### 3.2 Information Gain

**What It Is**: How much entropy we **reduce** by splitting on a feature.

$$IG(S, A) = H(S) - \sum_{v \in Values(A)} \frac{|S_v|}{|S|} H(S_v)$$

Where:
- $H(S)$ = entropy of parent node
- $S_v$ = subset of $S$ for which feature $A$ has value $v$
- $\frac{|S_v|}{|S|}$ = weight (proportion of samples going to that branch)

**We pick the feature with HIGHEST Information Gain** (biggest entropy reduction).

### 3.3 Gini Impurity (Used in CART)

**What It Is**: Probability that a randomly chosen sample would be misclassified if it were labeled randomly according to the distribution of labels in the node.

$$Gini(S) = 1 - \sum_{i=1}^{c} p_i^2$$

**Range**: 0 (pure) to $1 - \frac{1}{c}$ (maximally impure)

For binary classification: Gini ranges from 0 to 0.5

**Examples**:
```
Dataset A: [10 Yes, 0 No]  → Gini = 1 - (1² + 0²) = 0         (Pure!)
Dataset B: [5 Yes, 5 No]   → Gini = 1 - (0.5² + 0.5²) = 0.5   (Maximum impurity)
Dataset C: [8 Yes, 2 No]   → Gini = 1 - (0.8² + 0.2²) = 0.32
```

### 3.4 Entropy vs Gini — When to Use Which?

| Aspect | Entropy | Gini |
|--------|---------|------|
| Computation | Slower (log calculation) | Faster (squaring) |
| Range (binary) | 0 to 1 | 0 to 0.5 |
| Behavior | Slightly favors balanced splits | Slightly favors purity |
| Used in | ID3, C4.5 | CART (sklearn default) |
| In practice | Very similar results | Default choice in most libraries |

> **Pro Tip**: In 99% of cases, Gini and Entropy give nearly identical trees. Don't overthink this choice. Sklearn uses Gini by default for speed.

### 3.5 Gain Ratio (C4.5 Improvement)

Information Gain has a bias toward features with many values (e.g., "ID" column). **Gain Ratio** fixes this:

$$GainRatio(S, A) = \frac{IG(S, A)}{SplitInfo(S, A)}$$

Where:

$$SplitInfo(S, A) = -\sum_{v \in Values(A)} \frac{|S_v|}{|S|} \log_2 \frac{|S_v|}{|S|}$$

### Code Example — Computing Entropy & Gini

```python
import numpy as np

def entropy(y):
    """Calculate entropy of a label array."""
    # Count occurrences of each class
    _, counts = np.unique(y, return_counts=True)
    # Calculate probabilities
    probabilities = counts / len(y)
    # Entropy formula: -sum(p * log2(p))
    # We use where to handle log(0) = 0 convention
    return -np.sum(probabilities * np.log2(probabilities + 1e-10))

def gini_impurity(y):
    """Calculate Gini impurity of a label array."""
    _, counts = np.unique(y, return_counts=True)
    probabilities = counts / len(y)
    # Gini = 1 - sum(p^2)
    return 1 - np.sum(probabilities ** 2)

def information_gain(parent, left_child, right_child):
    """Calculate information gain from a split."""
    # Weight by proportion of samples in each child
    weight_left = len(left_child) / len(parent)
    weight_right = len(right_child) / len(parent)
    
    # IG = Parent entropy - weighted average of children entropy
    ig = entropy(parent) - (weight_left * entropy(left_child) + 
                             weight_right * entropy(right_child))
    return ig

# Example: Should we play tennis?
# Parent: 9 Yes, 5 No
parent = np.array([1]*9 + [0]*5)  # 1=Yes, 0=No

# Split on "Outlook = Sunny": Left=[2Y, 3N], Right=[7Y, 2N]
left = np.array([1, 1, 0, 0, 0])   # Sunny days
right = np.array([1]*7 + [0]*2)     # Not sunny days

print(f"Parent Entropy: {entropy(parent):.4f}")       # 0.9403
print(f"Parent Gini: {gini_impurity(parent):.4f}")    # 0.4592
print(f"Left Entropy: {entropy(left):.4f}")           # 0.9710
print(f"Right Entropy: {entropy(right):.4f}")         # 0.7642
print(f"Information Gain: {information_gain(parent, left, right):.4f}")  # 0.0481
```

---

## 4. Building a Decision Tree Step-by-Step

### Worked Example: Play Tennis Dataset

```
| Outlook  | Temp | Humidity | Windy | Play? |
|----------|------|----------|-------|-------|
| Sunny    | Hot  | High     | No    | No    |
| Sunny    | Hot  | High     | Yes   | No    |
| Overcast | Hot  | High     | No    | Yes   |
| Rain     | Mild | High     | No    | Yes   |
| Rain     | Cool | Normal   | No    | Yes   |
| Rain     | Cool | Normal   | Yes   | No    |
| Overcast | Cool | Normal   | Yes   | Yes   |
| Sunny    | Mild | High     | No    | No    |
| Sunny    | Cool | Normal   | No    | Yes   |
| Rain     | Mild | Normal   | No    | Yes   |
| Sunny    | Mild | Normal   | Yes   | Yes   |
| Overcast | Mild | High     | Yes   | Yes   |
| Overcast | Hot  | Normal   | No    | Yes   |
| Rain     | Mild | High     | Yes   | No    |
```

**Step 1**: Calculate IG for each feature at root
- IG(Outlook) = 0.246  ← HIGHEST
- IG(Temperature) = 0.029
- IG(Humidity) = 0.152
- IG(Windy) = 0.048

**Step 2**: Split on Outlook (highest IG)

```
              [Outlook?]
           /      |       \
       Sunny   Overcast   Rain
       /          |          \
   [2Y,3N]    [4Y,0N]    [3Y,2N]
                  |
              Leaf: YES
              (pure node!)
```

**Step 3**: Recursively split remaining impure nodes...

### Full Implementation from Scratch

```python
import numpy as np
from collections import Counter

class DecisionTreeNode:
    """A single node in the decision tree."""
    def __init__(self, feature=None, threshold=None, left=None, 
                 right=None, value=None):
        self.feature = feature      # Index of feature to split on
        self.threshold = threshold  # Threshold value for the split
        self.left = left           # Left subtree (feature <= threshold)
        self.right = right         # Right subtree (feature > threshold)
        self.value = value         # Predicted class (leaf node only)

class DecisionTreeClassifier:
    """Decision Tree built from scratch using Gini impurity."""
    
    def __init__(self, max_depth=10, min_samples_split=2, 
                 min_samples_leaf=1):
        self.max_depth = max_depth
        self.min_samples_split = min_samples_split
        self.min_samples_leaf = min_samples_leaf
        self.root = None
    
    def fit(self, X, y):
        """Build the decision tree recursively."""
        self.n_classes = len(np.unique(y))
        self.root = self._build_tree(X, y, depth=0)
        return self
    
    def _gini(self, y):
        """Calculate Gini impurity."""
        counts = Counter(y)
        impurity = 1.0
        for count in counts.values():
            prob = count / len(y)
            impurity -= prob ** 2
        return impurity
    
    def _best_split(self, X, y):
        """Find the best feature and threshold to split on."""
        best_gain = -1
        best_feature = None
        best_threshold = None
        
        current_gini = self._gini(y)
        n_samples = len(y)
        
        # Try every feature
        for feature_idx in range(X.shape[1]):
            # Get unique thresholds (midpoints between sorted values)
            thresholds = np.unique(X[:, feature_idx])
            
            for threshold in thresholds:
                # Split data
                left_mask = X[:, feature_idx] <= threshold
                right_mask = ~left_mask
                
                if (np.sum(left_mask) < self.min_samples_leaf or 
                    np.sum(right_mask) < self.min_samples_leaf):
                    continue
                
                # Calculate weighted Gini after split
                left_gini = self._gini(y[left_mask])
                right_gini = self._gini(y[right_mask])
                
                n_left = np.sum(left_mask)
                n_right = np.sum(right_mask)
                
                weighted_gini = ((n_left / n_samples) * left_gini + 
                                (n_right / n_samples) * right_gini)
                
                # Information gain = parent impurity - weighted child impurity
                gain = current_gini - weighted_gini
                
                if gain > best_gain:
                    best_gain = gain
                    best_feature = feature_idx
                    best_threshold = threshold
        
        return best_feature, best_threshold, best_gain
    
    def _build_tree(self, X, y, depth):
        """Recursively build the tree."""
        n_samples = len(y)
        
        # Stopping conditions
        if (depth >= self.max_depth or 
            n_samples < self.min_samples_split or 
            len(np.unique(y)) == 1):
            # Return leaf with majority class
            leaf_value = Counter(y).most_common(1)[0][0]
            return DecisionTreeNode(value=leaf_value)
        
        # Find best split
        feature, threshold, gain = self._best_split(X, y)
        
        if feature is None or gain <= 0:
            leaf_value = Counter(y).most_common(1)[0][0]
            return DecisionTreeNode(value=leaf_value)
        
        # Split data and recurse
        left_mask = X[:, feature] <= threshold
        right_mask = ~left_mask
        
        left_subtree = self._build_tree(X[left_mask], y[left_mask], depth + 1)
        right_subtree = self._build_tree(X[right_mask], y[right_mask], depth + 1)
        
        return DecisionTreeNode(feature=feature, threshold=threshold,
                               left=left_subtree, right=right_subtree)
    
    def predict(self, X):
        """Predict class for each sample."""
        return np.array([self._traverse(x, self.root) for x in X])
    
    def _traverse(self, x, node):
        """Traverse tree to find prediction for a single sample."""
        if node.value is not None:  # Leaf node
            return node.value
        if x[node.feature] <= node.threshold:
            return self._traverse(x, node.left)
        else:
            return self._traverse(x, node.right)

# Test our implementation
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split

X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

tree = DecisionTreeClassifier(max_depth=5)
tree.fit(X_train, y_train)
predictions = tree.predict(X_test)
accuracy = np.mean(predictions == y_test)
print(f"Our Decision Tree Accuracy: {accuracy:.4f}")  # ~0.9667
```

### Using Scikit-Learn (Production Code)

```python
from sklearn.tree import DecisionTreeClassifier, export_text, plot_tree
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
import matplotlib.pyplot as plt

# Load data
X, y = load_iris(return_X_y=True)
feature_names = load_iris().feature_names
class_names = load_iris().target_names

# Split data
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Create and train the tree
clf = DecisionTreeClassifier(
    criterion='gini',        # 'gini' or 'entropy'
    max_depth=3,             # Limit tree depth to prevent overfitting
    min_samples_split=5,     # Min samples needed to split a node
    min_samples_leaf=2,      # Min samples in leaf node
    random_state=42
)
clf.fit(X_train, y_train)

# Evaluate
print(f"Training Accuracy: {clf.score(X_train, y_train):.4f}")
print(f"Test Accuracy: {clf.score(X_test, y_test):.4f}")

# Print tree as text (great for understanding!)
tree_rules = export_text(clf, feature_names=feature_names)
print("\nDecision Rules:")
print(tree_rules)

# Output:
# |--- petal width (cm) <= 0.80
# |   |--- class: setosa
# |--- petal width (cm) > 0.80
# |   |--- petal width (cm) <= 1.75
# |   |   |--- class: versicolor
# |   |--- petal width (cm) > 1.75
# |   |   |--- class: virginica

# Visualize the tree
plt.figure(figsize=(20, 10))
plot_tree(clf, feature_names=feature_names, class_names=class_names,
          filled=True, rounded=True, fontsize=10)
plt.title("Decision Tree - Iris Dataset")
plt.tight_layout()
plt.savefig("decision_tree.png", dpi=150, bbox_inches='tight')
plt.show()
```

---

## 5. Pruning — Preventing Overfitting

### The Overfitting Problem

An unpruned decision tree will memorize the training data perfectly (0% training error) but perform terribly on new data. It creates super-specific rules that don't generalize.

```
Overfitting Visual:

Training Data:                     Unpruned Tree:
  ● ● ○ ●                         If x1 > 3.2 AND x2 < 1.7 AND
  ○ ● ○ ○                         x3 > 2.1 AND x1 < 3.8 → Class A
  ● ○ ● ●                         (Too specific! Won't generalize)
  
Ideal (Pruned):
  If x1 > 3 → Class A
  (Simple and generalizes well)
```

### 5.1 Pre-Pruning (Early Stopping)

Stop growing the tree BEFORE it's fully built.

| Parameter | What It Does | Typical Values |
|-----------|-------------|----------------|
| `max_depth` | Maximum tree depth | 3-10 |
| `min_samples_split` | Min samples to allow a split | 5-20 |
| `min_samples_leaf` | Min samples in any leaf | 2-10 |
| `max_leaf_nodes` | Maximum number of leaves | Depends on data |
| `max_features` | Max features to consider per split | sqrt(n), log2(n) |
| `min_impurity_decrease` | Min impurity reduction for split | 0.01-0.05 |

### 5.2 Post-Pruning (Cost Complexity Pruning / CCP)

Grow the full tree first, then cut back branches that don't help.

**Cost-Complexity**: Adds a penalty for tree complexity:

$$R_\alpha(T) = R(T) + \alpha |T|$$

Where:
- $R(T)$ = training error (misclassification rate)
- $|T|$ = number of leaf nodes
- $\alpha$ = complexity parameter (higher = more pruning)

```python
from sklearn.tree import DecisionTreeClassifier
from sklearn.model_selection import cross_val_score
import numpy as np

# Method 1: Pre-pruning with hyperparameters
tree_pruned = DecisionTreeClassifier(
    max_depth=4,
    min_samples_split=10,
    min_samples_leaf=5,
    random_state=42
)

# Method 2: Cost Complexity Pruning (Post-pruning)
# First, grow a full tree
full_tree = DecisionTreeClassifier(random_state=42)
full_tree.fit(X_train, y_train)

# Get the effective alphas and corresponding impurities
path = full_tree.cost_complexity_pruning_path(X_train, y_train)
ccp_alphas = path.ccp_alphas  # Array of alpha values
impurities = path.impurities  # Corresponding impurities

# Train trees for each alpha and find the best one
trees = []
for alpha in ccp_alphas:
    tree = DecisionTreeClassifier(ccp_alpha=alpha, random_state=42)
    tree.fit(X_train, y_train)
    trees.append(tree)

# Evaluate each tree using cross-validation
train_scores = [tree.score(X_train, y_train) for tree in trees]
test_scores = [tree.score(X_test, y_test) for tree in trees]

# Find optimal alpha (best test accuracy)
best_idx = np.argmax(test_scores)
best_alpha = ccp_alphas[best_idx]
print(f"Best alpha: {best_alpha:.4f}")
print(f"Best test accuracy: {test_scores[best_idx]:.4f}")
print(f"Number of leaves: {trees[best_idx].get_n_leaves()}")

# Use the optimal alpha
optimal_tree = DecisionTreeClassifier(ccp_alpha=best_alpha, random_state=42)
optimal_tree.fit(X_train, y_train)
```

> **Important**: Always use cross-validation to find the optimal pruning parameters. Don't use test set for tuning!

---

## 6. Decision Trees for Regression

### How It Works for Regression

Instead of predicting a class, regression trees predict a **continuous value** — the **mean** of target values in each leaf.

**Splitting criterion changes**:
- Classification: Gini / Entropy
- Regression: **MSE (Mean Squared Error)** or **MAE (Mean Absolute Error)**

$$MSE_{node} = \frac{1}{n} \sum_{i=1}^{n} (y_i - \bar{y})^2$$

The tree splits to **minimize the weighted MSE** across child nodes.

```python
from sklearn.tree import DecisionTreeRegressor
from sklearn.datasets import make_regression
import numpy as np
import matplotlib.pyplot as plt

# Create synthetic non-linear data
np.random.seed(42)
X = np.sort(5 * np.random.rand(200, 1), axis=0)
y = np.sin(X).ravel() + np.random.normal(0, 0.1, X.shape[0])

# Fit regression trees with different depths
fig, axes = plt.subplots(1, 3, figsize=(18, 5))
depths = [2, 5, 20]

for ax, depth in zip(axes, depths):
    reg = DecisionTreeRegressor(max_depth=depth, random_state=42)
    reg.fit(X, y)
    
    # Predict on dense grid for smooth visualization
    X_test = np.linspace(0, 5, 500).reshape(-1, 1)
    y_pred = reg.predict(X_test)
    
    ax.scatter(X, y, s=10, alpha=0.5, label='Data')
    ax.plot(X_test, y_pred, color='red', linewidth=2, label='Prediction')
    ax.set_title(f'max_depth = {depth}')
    ax.legend()

plt.suptitle("Regression Trees: Effect of Depth")
plt.tight_layout()
plt.show()

# Notice: depth=2 underfits, depth=20 overfits, depth=5 is just right
```

> **Key Insight**: Regression trees create step-function predictions (piecewise constant). They can't extrapolate beyond training data range!

---

## 7. Ensemble Methods — Bagging

### Why Ensembles?

**Problem**: A single decision tree is:
- High variance (small changes in data → completely different tree)
- Prone to overfitting
- Unstable

**Solution**: Combine many trees! "Wisdom of the crowd"

**Analogy**: Instead of asking one doctor for a diagnosis, ask 100 doctors and take a vote. Even if some are wrong, the majority will likely be right.

### Bootstrap Aggregating (Bagging)

**Steps**:
1. Create $B$ bootstrap samples (random samples WITH replacement)
2. Train one decision tree on EACH bootstrap sample
3. Combine predictions:
   - Classification: Majority vote
   - Regression: Average

```
Original Data: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

Bootstrap Sample 1: [2, 5, 5, 8, 1, 3, 8, 9, 2, 7]  → Tree 1
Bootstrap Sample 2: [4, 1, 6, 6, 3, 9, 10, 2, 5, 1] → Tree 2  
Bootstrap Sample 3: [7, 3, 3, 8, 1, 4, 9, 6, 2, 10] → Tree 3
...

Final Prediction = Vote(Tree1, Tree2, Tree3, ...)
```

**Why Bootstrap?** Each tree sees a different "perspective" of the data. This **reduces variance** without increasing bias.

### Out-of-Bag (OOB) Error

Each bootstrap sample uses ~63.2% of original data (due to sampling with replacement). The remaining ~36.8% are "Out-of-Bag" samples.

$$P(\text{sample NOT picked in } n \text{ draws}) = \left(1 - \frac{1}{n}\right)^n \approx e^{-1} \approx 0.368$$

OOB samples provide a **free validation set** — no need for separate holdout!

```python
from sklearn.ensemble import BaggingClassifier
from sklearn.tree import DecisionTreeClassifier
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split

# Generate data
X, y = make_classification(n_samples=1000, n_features=20, 
                           n_informative=10, random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Bagging with Decision Trees
bagging_clf = BaggingClassifier(
    estimator=DecisionTreeClassifier(),  # Base learner
    n_estimators=100,        # Number of trees
    max_samples=0.8,         # 80% of data per bootstrap sample
    max_features=1.0,        # Use all features
    bootstrap=True,          # Sample with replacement
    oob_score=True,          # Calculate OOB score
    random_state=42,
    n_jobs=-1                # Use all CPU cores
)
bagging_clf.fit(X_train, y_train)

print(f"Training Accuracy: {bagging_clf.score(X_train, y_train):.4f}")
print(f"Test Accuracy: {bagging_clf.score(X_test, y_test):.4f}")
print(f"OOB Score: {bagging_clf.oob_score_:.4f}")  # Free validation!
```

---

## 8. Random Forest

### What It Is

Random Forest = Bagging + **Random Feature Selection** at each split.

The key innovation: at each node split, instead of considering ALL features, only consider a **random subset** of features.

### Why Random Feature Selection?

**Problem with regular Bagging**: If one feature is very strong, ALL trees will split on it first → trees become correlated → less variance reduction.

**Solution**: Force diversity by limiting feature choices.

```
Bagging:       Tree1 splits on Feature A (best)
               Tree2 splits on Feature A (best)   ← All same!
               Tree3 splits on Feature A (best)

Random Forest: Tree1 sees {A, C, F} → splits on A
               Tree2 sees {B, D, E} → splits on D  ← Diverse!
               Tree3 sees {C, F, G} → splits on C
```

### `max_features` — The Key Hyperparameter

| Setting | Value | Use When |
|---------|-------|----------|
| `"sqrt"` | $\sqrt{p}$ features | Classification (default) |
| `"log2"` | $\log_2(p)$ features | Alternative for classification |
| `None` / `1.0` | All $p$ features | Becomes regular Bagging |
| `0.3` | 30% of features | Custom — experiment! |

> **For regression**: Default is `max_features=1.0` (all features), but `max_features="sqrt"` or `0.33` often works better.

### Random Forest — Full Implementation

```python
from sklearn.ensemble import RandomForestClassifier, RandomForestRegressor
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.metrics import classification_report
import numpy as np

# Load breast cancer dataset
data = load_breast_cancer()
X, y = data.data, data.target
feature_names = data.feature_names

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

# Create Random Forest
rf = RandomForestClassifier(
    n_estimators=200,         # Number of trees (more = better, but slower)
    max_depth=None,           # No limit — let trees grow fully
    min_samples_split=5,      # Prevent overfitting
    min_samples_leaf=2,       # At least 2 samples in leaves
    max_features='sqrt',      # Random feature selection
    bootstrap=True,           # Bootstrap sampling
    oob_score=True,           # Out-of-bag score
    class_weight='balanced',  # Handle imbalanced classes
    random_state=42,
    n_jobs=-1                 # Parallelize across CPU cores
)

rf.fit(X_train, y_train)

# Results
print(f"Training Accuracy: {rf.score(X_train, y_train):.4f}")
print(f"Test Accuracy: {rf.score(X_test, y_test):.4f}")
print(f"OOB Score: {rf.oob_score_:.4f}")

# Cross-validation (more robust)
cv_scores = cross_val_score(rf, X, y, cv=5, scoring='accuracy')
print(f"\nCross-Validation: {cv_scores.mean():.4f} ± {cv_scores.std():.4f}")

# Detailed classification report
y_pred = rf.predict(X_test)
print("\nClassification Report:")
print(classification_report(y_test, y_pred, target_names=data.target_names))
```

### How Many Trees Do You Need?

```python
import matplotlib.pyplot as plt
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_val_score

# Test different numbers of trees
n_trees_range = [1, 5, 10, 25, 50, 100, 200, 500, 1000]
scores = []

for n_trees in n_trees_range:
    rf = RandomForestClassifier(n_estimators=n_trees, random_state=42, n_jobs=-1)
    cv_score = cross_val_score(rf, X_train, y_train, cv=5).mean()
    scores.append(cv_score)
    print(f"n_estimators={n_trees:4d} → CV Accuracy: {cv_score:.4f}")

# Typically: performance plateaus around 100-300 trees
# More trees = diminishing returns but never hurts accuracy (just computation)

plt.plot(n_trees_range, scores, 'o-')
plt.xlabel('Number of Trees')
plt.ylabel('Cross-Validation Accuracy')
plt.title('Random Forest: Effect of Number of Trees')
plt.xscale('log')
plt.grid(True)
plt.show()
```

> **Pro Tip**: More trees NEVER hurt accuracy — they only cost more compute time. Start with 100, increase to 500 if accuracy matters more than speed.

---

## 9. Feature Importance

### How It's Calculated

**Method 1: Mean Decrease in Impurity (MDI)** — Default in sklearn
- Sum up how much each feature decreases impurity across all splits in all trees
- Normalized to sum to 1.0

**Method 2: Permutation Importance** — More reliable
- Shuffle one feature's values randomly
- Measure how much accuracy drops
- Bigger drop = more important feature

> **Warning**: MDI is biased toward high-cardinality features (many unique values). Always prefer Permutation Importance for reliable results.

```python
import pandas as pd
import matplotlib.pyplot as plt
from sklearn.inspection import permutation_importance

# Method 1: Built-in feature importance (MDI)
importances = rf.feature_importances_
indices = np.argsort(importances)[::-1]

# Print top 10 features
print("Top 10 Features (MDI):")
for i in range(10):
    print(f"  {i+1}. {feature_names[indices[i]]}: {importances[indices[i]]:.4f}")

# Method 2: Permutation Importance (more reliable!)
perm_importance = permutation_importance(
    rf, X_test, y_test, 
    n_repeats=30,       # Repeat shuffling for stability
    random_state=42,
    n_jobs=-1
)

# Plot both side by side
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(16, 6))

# MDI
top_n = 10
ax1.barh(range(top_n), importances[indices[:top_n]])
ax1.set_yticks(range(top_n))
ax1.set_yticklabels([feature_names[i] for i in indices[:top_n]])
ax1.set_title("Feature Importance (MDI)")
ax1.set_xlabel("Importance")

# Permutation
perm_sorted = perm_importance.importances_mean.argsort()[::-1][:top_n]
ax2.barh(range(top_n), perm_importance.importances_mean[perm_sorted])
ax2.set_yticks(range(top_n))
ax2.set_yticklabels([feature_names[i] for i in perm_sorted])
ax2.set_title("Feature Importance (Permutation)")
ax2.set_xlabel("Decrease in Accuracy")

plt.tight_layout()
plt.show()
```

### Feature Selection Using Random Forest

```python
from sklearn.feature_selection import SelectFromModel

# Use RF importance to select features
selector = SelectFromModel(rf, threshold='median')  # Keep top 50%
selector.fit(X_train, y_train)

# Get selected features
selected_mask = selector.get_support()
selected_features = feature_names[selected_mask]
print(f"Selected {len(selected_features)} out of {len(feature_names)} features")
print(f"Selected: {list(selected_features)}")

# Train with selected features only
X_train_selected = selector.transform(X_train)
X_test_selected = selector.transform(X_test)

rf_small = RandomForestClassifier(n_estimators=200, random_state=42)
rf_small.fit(X_train_selected, y_train)
print(f"\nAccuracy with ALL features: {rf.score(X_test, y_test):.4f}")
print(f"Accuracy with SELECTED features: {rf_small.score(X_test_selected, y_test):.4f}")
```

---

## 10. Hyperparameter Tuning

### Key Hyperparameters and Their Effects

| Parameter | Effect | Typical Range |
|-----------|--------|---------------|
| `n_estimators` | More trees = better, diminishing returns | 100-1000 |
| `max_depth` | Controls overfitting; None = unlimited | 5-30 or None |
| `min_samples_split` | Higher = less overfitting | 2-20 |
| `min_samples_leaf` | Higher = smoother boundaries | 1-10 |
| `max_features` | Lower = more diversity, less correlation | sqrt, log2, 0.3-0.7 |
| `class_weight` | Handle imbalanced datasets | 'balanced' or dict |

### Tuning Strategy

```python
from sklearn.model_selection import RandomizedSearchCV
from scipy.stats import randint, uniform

# Define parameter distributions for random search
param_distributions = {
    'n_estimators': randint(100, 1000),
    'max_depth': [None, 5, 10, 15, 20, 30],
    'min_samples_split': randint(2, 20),
    'min_samples_leaf': randint(1, 10),
    'max_features': ['sqrt', 'log2', 0.3, 0.5, 0.7],
    'bootstrap': [True, False]
}

# RandomizedSearchCV — much faster than GridSearch for many params
random_search = RandomizedSearchCV(
    RandomForestClassifier(random_state=42),
    param_distributions=param_distributions,
    n_iter=100,          # Try 100 random combinations
    cv=5,                # 5-fold cross-validation
    scoring='accuracy',
    random_state=42,
    n_jobs=-1,
    verbose=1
)

random_search.fit(X_train, y_train)

print(f"Best Parameters: {random_search.best_params_}")
print(f"Best CV Score: {random_search.best_score_:.4f}")
print(f"Test Score: {random_search.score(X_test, y_test):.4f}")
```

> **Pro Tip**: Start with RandomizedSearchCV (100 iterations), then do GridSearchCV around the best values found for fine-tuning.

---

## 11. Common Mistakes

### Mistake 1: Not Pruning / Limiting Depth
```python
# BAD: Will overfit!
tree = DecisionTreeClassifier()  # max_depth=None by default

# GOOD: Always limit complexity
tree = DecisionTreeClassifier(max_depth=5, min_samples_leaf=5)
```

### Mistake 2: Using a Single Decision Tree in Production
- Single trees are unstable — always use Random Forest or Gradient Boosting
- Exception: When interpretability is legally required (e.g., credit scoring regulations)

### Mistake 3: Ignoring Feature Importance Bias
- MDI importance is biased toward continuous features with many values
- Always validate with Permutation Importance

### Mistake 4: Not Setting `random_state`
```python
# BAD: Results change every run
rf = RandomForestClassifier(n_estimators=100)

# GOOD: Reproducible results
rf = RandomForestClassifier(n_estimators=100, random_state=42)
```

### Mistake 5: Too Few Trees in Random Forest
- With too few trees, results are noisy
- 100 is minimum; 500+ for important applications

### Mistake 6: Not Using `class_weight='balanced'` for Imbalanced Data
```python
# For imbalanced datasets (e.g., fraud detection: 99% non-fraud, 1% fraud)
rf = RandomForestClassifier(class_weight='balanced', random_state=42)
```

### Mistake 7: Scaling Features Before Tree-Based Models
```python
# UNNECESSARY: Trees don't need scaling!
# Don't waste time with StandardScaler or MinMaxScaler for trees/forests
```

---

## 12. Interview Questions

### Conceptual Questions

**Q1: What is the difference between Gini and Entropy?**
> Both measure impurity. Gini measures probability of misclassification; Entropy measures information disorder. In practice, they give nearly identical trees. Gini is computationally faster (no log). Sklearn uses Gini by default.

**Q2: Why is Random Forest better than a single Decision Tree?**
> Single trees have high variance — small data changes lead to completely different trees. RF averages many uncorrelated trees, reducing variance via the law of large numbers, while keeping bias similar. The random feature selection decorrelates trees, making the averaging more effective.

**Q3: How does Random Forest handle overfitting?**
> Three mechanisms: (1) Bagging averages out noise from individual trees; (2) Random feature selection decorrelates trees; (3) Individual trees can be pruned via hyperparameters. Together, these reduce variance without significantly increasing bias.

**Q4: What is OOB (Out-of-Bag) error and why is it useful?**
> Each bootstrap sample excludes ~36.8% of data. These "out-of-bag" samples serve as a built-in validation set. OOB error is almost identical to leave-one-out cross-validation but is free — no extra computation needed.

**Q5: Can Decision Trees handle missing values?**
> CART (sklearn) cannot natively — you must impute. But XGBoost and LightGBM can handle missing values by learning default split directions. Some implementations use surrogate splits (splitting on a correlated feature when the primary one is missing).

**Q6: Why do Decision Trees tend to overfit?**
> Without constraints, a tree can create a leaf for every training sample, memorizing noise. They have unlimited capacity (VC dimension grows with depth). Unlike linear models, they don't assume a simple underlying relationship.

**Q7: How would you handle a 100:1 class imbalance with Random Forest?**
> Options: (1) `class_weight='balanced'` — adjusts impurity calculation; (2) `class_weight={0:1, 1:100}` — manually set weights; (3) Combine with SMOTE for oversampling minority class; (4) Use `BalancedRandomForestClassifier` from imbalanced-learn; (5) Adjust prediction threshold.

### Coding Questions

**Q8: Implement a function to calculate Information Gain.**
> See Section 3 code example above.

**Q9: How do you extract and interpret feature importances?**
> See Section 9 — use `rf.feature_importances_` for MDI, `permutation_importance()` for more reliable results.

**Q10: How would you decide between Random Forest and Gradient Boosting?**
> RF: Less tuning needed, robust to overfitting, easily parallelizable, good default choice. GB: Usually higher accuracy with proper tuning, but prone to overfitting, sequential (slower training). Use RF for quick results, GB when you need that extra 1-2% accuracy and have time to tune.

---

## 13. Quick Reference

### Decision Tree Cheat Sheet

| Aspect | Detail |
|--------|--------|
| Type | Supervised (Classification & Regression) |
| Splitting (Classification) | Gini (default) or Entropy |
| Splitting (Regression) | MSE (default) or MAE |
| Feature Scaling | NOT required |
| Handles Missing Values | No (sklearn) |
| Handles Categorical | Needs encoding in sklearn |
| Prone to Overfitting | YES — always prune! |
| Interpretable | YES — major advantage |
| Time Complexity (training) | $O(n \cdot m \cdot \log n)$ |
| Time Complexity (prediction) | $O(\log n)$ — very fast! |

### Random Forest Cheat Sheet

| Aspect | Detail |
|--------|--------|
| Base | Many Decision Trees |
| Sampling | Bootstrap (with replacement) |
| Feature Selection | Random subset at each split |
| Default `max_features` | sqrt(n) for classification, n for regression |
| Parallelizable | YES (`n_jobs=-1`) |
| Handles Overfitting | Much better than single tree |
| When to Use | Default "first try" for tabular data |
| Weakness | Can't extrapolate; slower than single tree |

### Quick Decision Guide

```
Need interpretability? → Single Decision Tree (pruned)
Need good accuracy, fast? → Random Forest (200 trees)
Need best accuracy, tabular? → Gradient Boosting (XGBoost/LightGBM)
Very imbalanced data? → RF with class_weight='balanced'
Need feature importance? → RF + Permutation Importance
Real-time prediction? → Any tree method (prediction is O(log n))
```

---

*Next Chapter: [05 - SVM (Support Vector Machines)](05-SVM-Support-Vector-Machines.md)*
