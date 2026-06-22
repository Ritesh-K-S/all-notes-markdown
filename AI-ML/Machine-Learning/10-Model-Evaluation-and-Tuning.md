# Chapter 10: Model Evaluation and Tuning

## Table of Contents
- [10.1 Introduction to Model Evaluation](#101-introduction-to-model-evaluation)
- [10.2 Classification Metrics](#102-classification-metrics)
- [10.3 Regression Metrics](#103-regression-metrics)
- [10.4 Cross-Validation Techniques](#104-cross-validation-techniques)
- [10.5 Hyperparameter Tuning](#105-hyperparameter-tuning)
- [10.6 Learning Curves and Validation Curves](#106-learning-curves-and-validation-curves)
- [10.7 Model Selection Strategies](#107-model-selection-strategies)
- [10.8 Common Mistakes](#108-common-mistakes)
- [10.9 Interview Questions](#109-interview-questions)
- [10.10 Quick Reference](#1010-quick-reference)

---

## 10.1 Introduction to Model Evaluation

### What It Is
Model evaluation is the process of measuring how well your machine learning model performs on data it hasn't seen before. Think of it like a school exam — you study (train) on practice questions, but the real test (evaluation) uses different questions to check if you actually *understand* the material, not just memorized answers.

### Why It Matters
- **Prevents deploying bad models** — A model that's 99% accurate on training data but 60% on new data is useless
- **Guides model selection** — Helps you choose between Random Forest vs. XGBoost vs. Neural Networks
- **Identifies overfitting/underfitting** — Tells you if your model is too complex or too simple
- **Business decisions depend on it** — A fraud detection model with wrong metrics could cost millions

### The Fundamental Problem

```
┌──────────────────────────────────────────────────────────┐
│                    ALL YOUR DATA                          │
│                                                          │
│  ┌─────────────────────┐  ┌────────────────────────┐    │
│  │   Training Set      │  │    Test Set             │    │
│  │   (Learn from this) │  │    (Evaluate on this)   │    │
│  │                     │  │                         │    │
│  │   Model sees this   │  │   Model NEVER sees     │    │
│  │   during training   │  │   this during training  │    │
│  └─────────────────────┘  └────────────────────────┘    │
│                                                          │
│  WHY? → To simulate real-world performance               │
└──────────────────────────────────────────────────────────┘
```

> **Key Insight**: The metric you optimize during training is NOT always the metric you care about in production. A spam filter optimized for accuracy might let 5% of spam through — which is millions of spam emails at scale.

---

## 10.2 Classification Metrics

### 10.2.1 Confusion Matrix

#### What It Is
A table that shows how many predictions your model got right and wrong, broken down by category. It's the foundation of ALL classification metrics.

```
                    PREDICTED
                 Positive  Negative
              ┌──────────┬──────────┐
   ACTUAL     │    TP    │    FN    │   ← Actual Positive
  Positive    │  (Hit)   │  (Miss)  │
              ├──────────┼──────────┤
   ACTUAL     │    FP    │    TN    │   ← Actual Negative
  Negative    │ (False   │(Correct  │
              │  Alarm)  │ Reject)  │
              └──────────┴──────────┘
```

- **TP (True Positive)**: Model said "Yes" and it was actually "Yes" ✅
- **TN (True Negative)**: Model said "No" and it was actually "No" ✅
- **FP (False Positive)**: Model said "Yes" but it was actually "No" ❌ (Type I Error)
- **FN (False Negative)**: Model said "No" but it was actually "Yes" ❌ (Type II Error)

#### Real-World Analogy
- **FP (False Positive)** = Fire alarm goes off but there's no fire → Annoying but safe
- **FN (False Negative)** = There's a fire but alarm doesn't go off → Dangerous!

#### Code Example

```python
import numpy as np
from sklearn.metrics import confusion_matrix, ConfusionMatrixDisplay
import matplotlib.pyplot as plt

# Simulating cancer detection results
y_true = [1, 0, 1, 1, 0, 1, 0, 0, 1, 0, 1, 0, 0, 1, 1]  # Actual labels
y_pred = [1, 0, 1, 0, 0, 1, 1, 0, 1, 0, 0, 0, 0, 1, 1]  # Model predictions

# Generate confusion matrix
cm = confusion_matrix(y_true, y_pred)
print("Confusion Matrix:")
print(cm)
# Output:
# [[5, 1],    ← 5 True Negatives, 1 False Positive
#  [2, 7]]   ← 2 False Negatives, 7 True Positives

# Visualize it
disp = ConfusionMatrixDisplay(confusion_matrix=cm, display_labels=['Healthy', 'Cancer'])
disp.plot(cmap='Blues')
plt.title("Cancer Detection - Confusion Matrix")
plt.show()
```

---

### 10.2.2 Accuracy

#### Formula

$$\text{Accuracy} = \frac{TP + TN}{TP + TN + FP + FN}$$

#### When to Use / NOT Use

| Scenario | Use Accuracy? | Why |
|----------|:---:|-----|
| Balanced dataset (50/50 split) | ✅ | Fair representation |
| Imbalanced dataset (95/5 split) | ❌ | Predicting all as majority class gives 95% accuracy |
| Spam detection | ❌ | Missing spam is more costly than false alarm |
| Simple binary with equal costs | ✅ | Both errors equally bad |

```python
from sklearn.metrics import accuracy_score

y_true = [0]*950 + [1]*50  # 95% class 0, 5% class 1 (imbalanced!)
y_pred = [0]*1000           # Dumb model: always predicts 0

print(f"Accuracy: {accuracy_score(y_true, y_pred):.2%}")
# Output: Accuracy: 95.00%  ← Looks great but model is USELESS for class 1!
```

> **Warning**: Never use accuracy alone on imbalanced datasets. A model that always predicts the majority class will have high accuracy but zero usefulness.

---

### 10.2.3 Precision, Recall, and F1-Score

#### Precision — "When the model says YES, how often is it correct?"

$$\text{Precision} = \frac{TP}{TP + FP}$$

**Analogy**: A cautious doctor who only diagnoses cancer when absolutely sure. High precision = fewer false alarms.

**When to prioritize**: When false positives are expensive (spam filter marking important emails as spam, innocent person convicted)

#### Recall (Sensitivity) — "Of all actual YES cases, how many did the model catch?"

$$\text{Recall} = \frac{TP}{TP + FN}$$

**Analogy**: An aggressive doctor who flags every suspicious case. High recall = fewer missed cases.

**When to prioritize**: When false negatives are dangerous (cancer detection, fraud detection, security threats)

#### F1-Score — Harmonic mean of Precision and Recall

$$F_1 = 2 \cdot \frac{\text{Precision} \times \text{Recall}}{\text{Precision} + \text{Recall}}$$

**Why harmonic mean?** It penalizes extreme imbalance. If Precision=1.0 and Recall=0.01, F1=0.02 (not 0.505 like arithmetic mean would give).

#### The Precision-Recall Tradeoff

```
High Threshold (Strict)           Low Threshold (Lenient)
─────────────────────────────────────────────────────────►

Precision: HIGH ████████████░░    Precision: LOW  ████░░░░░░░░░░
Recall:    LOW  ████░░░░░░░░░░    Recall:    HIGH ████████████░░

"Only flag when VERY sure"        "Flag everything suspicious"
Fewer false alarms                Catches more actual positives
But misses many real cases        But many false alarms
```

#### Code Example

```python
from sklearn.metrics import precision_score, recall_score, f1_score, classification_report
from sklearn.linear_model import LogisticRegression
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split

# Create imbalanced dataset (10% positive class)
X, y = make_classification(n_samples=1000, n_features=20, 
                           weights=[0.9, 0.1],  # 90% class 0, 10% class 1
                           random_state=42)

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3, random_state=42)

# Train model
model = LogisticRegression(random_state=42)
model.fit(X_train, y_train)
y_pred = model.predict(X_test)

# Individual metrics
print(f"Precision: {precision_score(y_test, y_pred):.4f}")
print(f"Recall:    {recall_score(y_test, y_pred):.4f}")
print(f"F1-Score:  {f1_score(y_test, y_pred):.4f}")

# Full classification report (best for quick overview)
print("\n", classification_report(y_test, y_pred, target_names=['Negative', 'Positive']))
```

Output:
```
Precision: 0.7857
Recall:    0.6875
F1-Score:  0.7333

              precision    recall  f1-score   support
    Negative       0.97      0.98      0.97       268
    Positive       0.79      0.69      0.73        32
    accuracy                           0.95       300
   macro avg       0.88      0.83      0.85       300
weighted avg       0.95      0.95      0.95       300
```

---

### 10.2.4 F-beta Score

When you want to weight precision vs. recall differently:

$$F_\beta = (1 + \beta^2) \cdot \frac{\text{Precision} \times \text{Recall}}{(\beta^2 \cdot \text{Precision}) + \text{Recall}}$$

| β value | Emphasis | Use Case |
|---------|----------|----------|
| β = 0.5 | Precision 2x more important | Spam detection |
| β = 1.0 | Equal weight (standard F1) | General purpose |
| β = 2.0 | Recall 2x more important | Cancer detection, fraud |

```python
from sklearn.metrics import fbeta_score

# β=2: Recall is 2x more important (medical diagnosis)
f2 = fbeta_score(y_test, y_pred, beta=2)
print(f"F2-Score (recall-focused): {f2:.4f}")

# β=0.5: Precision is 2x more important (spam filter)
f05 = fbeta_score(y_test, y_pred, beta=0.5)
print(f"F0.5-Score (precision-focused): {f05:.4f}")
```

---

### 10.2.5 ROC Curve and AUC

#### What It Is
The ROC (Receiver Operating Characteristic) curve plots **True Positive Rate** vs **False Positive Rate** at every possible classification threshold. AUC (Area Under the Curve) summarizes this into a single number.

$$TPR = \frac{TP}{TP + FN} = \text{Recall}$$

$$FPR = \frac{FP}{FP + TN}$$

```
TPR (Recall)
1.0 ┤        ╭──────────────────
    │       ╱
    │      ╱   ← Good Model (AUC ≈ 0.9)
    │     ╱
0.5 ┤    ╱
    │   ╱  ╱ ← Random (AUC = 0.5)
    │  ╱  ╱
    │ ╱  ╱
0.0 ┤╱──╱───────────────────────
    0.0        0.5           1.0
              FPR (False Positive Rate)
```

**AUC Interpretation**:
| AUC | Meaning |
|-----|---------|
| 1.0 | Perfect model |
| 0.9 - 1.0 | Excellent |
| 0.8 - 0.9 | Good |
| 0.7 - 0.8 | Fair |
| 0.5 | Random guessing (useless) |
| < 0.5 | Worse than random (predictions inverted) |

#### Code Example

```python
from sklearn.metrics import roc_curve, roc_auc_score, auc
import matplotlib.pyplot as plt

# Get probability predictions (not hard labels)
y_prob = model.predict_proba(X_test)[:, 1]  # Probability of positive class

# Calculate ROC curve points
fpr, tpr, thresholds = roc_curve(y_test, y_prob)
roc_auc = auc(fpr, tpr)

# Plot ROC curve
plt.figure(figsize=(8, 6))
plt.plot(fpr, tpr, color='blue', linewidth=2, label=f'ROC Curve (AUC = {roc_auc:.4f})')
plt.plot([0, 1], [0, 1], color='red', linestyle='--', label='Random Classifier')
plt.xlabel('False Positive Rate')
plt.ylabel('True Positive Rate')
plt.title('ROC Curve')
plt.legend(loc='lower right')
plt.grid(True, alpha=0.3)
plt.show()

# Quick AUC score
print(f"AUC-ROC Score: {roc_auc_score(y_test, y_prob):.4f}")
```

> **Pro Tip**: ROC-AUC can be misleading on highly imbalanced datasets. Use **Precision-Recall AUC** instead when the positive class is rare (< 5%).

---

### 10.2.6 Precision-Recall Curve

Better than ROC for imbalanced datasets:

```python
from sklearn.metrics import precision_recall_curve, average_precision_score

# Calculate PR curve
precision_vals, recall_vals, thresholds_pr = precision_recall_curve(y_test, y_prob)
avg_precision = average_precision_score(y_test, y_prob)

plt.figure(figsize=(8, 6))
plt.plot(recall_vals, precision_vals, linewidth=2, 
         label=f'PR Curve (AP = {avg_precision:.4f})')
plt.xlabel('Recall')
plt.ylabel('Precision')
plt.title('Precision-Recall Curve')
plt.legend()
plt.grid(True, alpha=0.3)
plt.show()
```

---

### 10.2.7 Log Loss (Cross-Entropy Loss)

Measures how well probability predictions match reality. Unlike accuracy, it **penalizes confident wrong predictions heavily**.

$$\text{Log Loss} = -\frac{1}{N} \sum_{i=1}^{N} [y_i \log(\hat{p}_i) + (1-y_i) \log(1-\hat{p}_i)]$$

```python
from sklearn.metrics import log_loss

# Compare two models with same accuracy but different confidence
y_true = [1, 1, 0, 0]

# Model A: Confident and correct
y_prob_A = [0.95, 0.90, 0.10, 0.05]

# Model B: Correct but uncertain
y_prob_B = [0.60, 0.55, 0.45, 0.40]

print(f"Log Loss Model A: {log_loss(y_true, y_prob_A):.4f}")  # Lower = better
print(f"Log Loss Model B: {log_loss(y_true, y_prob_B):.4f}")  # Higher = worse
# Output:
# Log Loss Model A: 0.0769
# Log Loss Model B: 0.5765
```

---

### 10.2.8 Multi-Class Metrics

For problems with more than 2 classes:

```python
from sklearn.metrics import classification_report
from sklearn.datasets import load_iris
from sklearn.ensemble import RandomForestClassifier

# Load multi-class dataset
iris = load_iris()
X_train, X_test, y_train, y_test = train_test_split(
    iris.data, iris.target, test_size=0.3, random_state=42)

model = RandomForestClassifier(n_estimators=100, random_state=42)
model.fit(X_train, y_train)
y_pred = model.predict(X_test)

# Full report with per-class metrics
print(classification_report(y_test, y_pred, target_names=iris.target_names))
```

**Averaging strategies for multi-class:**

| Method | Description | When to Use |
|--------|-------------|-------------|
| `macro` | Average across classes (equal weight) | All classes equally important |
| `weighted` | Average weighted by class frequency | Imbalanced classes |
| `micro` | Global TP/FP/FN (same as accuracy for multi-class) | Overall performance |

```python
from sklearn.metrics import f1_score

# Different averaging methods
print(f"F1 (macro):    {f1_score(y_test, y_pred, average='macro'):.4f}")
print(f"F1 (weighted): {f1_score(y_test, y_pred, average='weighted'):.4f}")
print(f"F1 (micro):    {f1_score(y_test, y_pred, average='micro'):.4f}")
```

---

### 10.2.9 Cohen's Kappa

Measures agreement between predictions and actual labels, corrected for chance agreement:

$$\kappa = \frac{p_o - p_e}{1 - p_e}$$

Where $p_o$ = observed accuracy, $p_e$ = expected accuracy by chance.

| Kappa | Interpretation |
|-------|---------------|
| < 0 | Less than chance |
| 0.0 - 0.20 | Slight |
| 0.21 - 0.40 | Fair |
| 0.41 - 0.60 | Moderate |
| 0.61 - 0.80 | Substantial |
| 0.81 - 1.00 | Almost perfect |

```python
from sklearn.metrics import cohen_kappa_score

kappa = cohen_kappa_score(y_test, y_pred)
print(f"Cohen's Kappa: {kappa:.4f}")
```

---

## 10.3 Regression Metrics

### 10.3.1 Mean Absolute Error (MAE)

$$MAE = \frac{1}{n} \sum_{i=1}^{n} |y_i - \hat{y}_i|$$

**Analogy**: Average "how far off" your predictions are. If you guess house prices, MAE tells you "on average, you're off by $X."

**Properties**:
- Easy to interpret (same unit as target)
- Robust to outliers (treats all errors equally)
- Does NOT penalize large errors extra

### 10.3.2 Mean Squared Error (MSE)

$$MSE = \frac{1}{n} \sum_{i=1}^{n} (y_i - \hat{y}_i)^2$$

**Properties**:
- Penalizes large errors MORE (squared)
- Units are squared (hard to interpret directly)
- Sensitive to outliers
- Differentiable (good for optimization)

### 10.3.3 Root Mean Squared Error (RMSE)

$$RMSE = \sqrt{\frac{1}{n} \sum_{i=1}^{n} (y_i - \hat{y}_i)^2}$$

**Properties**:
- Same unit as target (interpretable)
- Penalizes large errors more than MAE
- Most commonly used regression metric

### 10.3.4 R² Score (Coefficient of Determination)

$$R^2 = 1 - \frac{\sum_{i=1}^{n}(y_i - \hat{y}_i)^2}{\sum_{i=1}^{n}(y_i - \bar{y})^2} = 1 - \frac{SS_{res}}{SS_{tot}}$$

**Analogy**: "What percentage of the variation in the data does my model explain?" R²=0.85 means your model explains 85% of why the target variable changes.

| R² Value | Interpretation |
|----------|---------------|
| 1.0 | Perfect prediction |
| 0.7 - 1.0 | Good model |
| 0.4 - 0.7 | Moderate |
| 0.0 | Same as predicting the mean |
| < 0.0 | Worse than predicting the mean |

### 10.3.5 Adjusted R²

Penalizes adding useless features:

$$R^2_{adj} = 1 - \frac{(1-R^2)(n-1)}{n-p-1}$$

Where $n$ = samples, $p$ = features.

### 10.3.6 MAPE (Mean Absolute Percentage Error)

$$MAPE = \frac{100\%}{n} \sum_{i=1}^{n} \left|\frac{y_i - \hat{y}_i}{y_i}\right|$$

**Useful for**: Business reporting ("our predictions are off by X%")
**Problem**: Undefined when $y_i = 0$

### Code Example — All Regression Metrics

```python
from sklearn.metrics import (mean_absolute_error, mean_squared_error, 
                              r2_score, mean_absolute_percentage_error)
import numpy as np

# Simulated house price predictions
y_true = np.array([200000, 350000, 150000, 500000, 280000, 420000, 175000, 310000])
y_pred = np.array([210000, 330000, 160000, 480000, 295000, 400000, 180000, 320000])

mae = mean_absolute_error(y_true, y_pred)
mse = mean_squared_error(y_true, y_pred)
rmse = np.sqrt(mse)  # or mean_squared_error(y_true, y_pred, squared=False)
r2 = r2_score(y_true, y_pred)
mape = mean_absolute_percentage_error(y_true, y_pred)

print(f"MAE:  ${mae:,.0f}")       # Average absolute error
print(f"MSE:  {mse:,.0f}")        # Mean squared error
print(f"RMSE: ${rmse:,.0f}")      # Root mean squared error
print(f"R²:   {r2:.4f}")          # Proportion of variance explained
print(f"MAPE: {mape:.2%}")        # Mean absolute percentage error

# Output:
# MAE:  $15,625
# MSE:  315,625,000
# RMSE: $17,766
# R²:   0.9954
# MAPE: 5.10%
```

### When to Use Which Metric

| Metric | Use When | Avoid When |
|--------|----------|------------|
| MAE | Outliers exist, all errors equal | Need to penalize big errors |
| RMSE | Large errors are much worse | Many outliers in data |
| R² | Comparing models on same data | Different datasets/scales |
| MAPE | Business needs percentage | Target has zeros |

---

## 10.4 Cross-Validation Techniques

### What It Is
Cross-validation is a technique where you repeatedly split your data into train/test sets and average the results. Instead of one lucky/unlucky split, you get a reliable estimate of model performance.

**Analogy**: Instead of taking one practice test, you take 5 different practice tests and average your scores. This gives a much better estimate of how you'll do on the real exam.

### Why It Matters
- Single train/test split is **unreliable** (depends on which data ends up where)
- Maximizes use of limited data (every sample gets to be in test set)
- Gives confidence intervals on performance estimates
- Required for proper hyperparameter tuning

---

### 10.4.1 K-Fold Cross-Validation

```
Data: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]     (5-Fold CV)

Fold 1: Test=[1,2]   Train=[3,4,5,6,7,8,9,10]  → Score: 0.82
Fold 2: Test=[3,4]   Train=[1,2,5,6,7,8,9,10]  → Score: 0.85
Fold 3: Test=[5,6]   Train=[1,2,3,4,7,8,9,10]  → Score: 0.79
Fold 4: Test=[7,8]   Train=[1,2,3,4,5,6,9,10]  → Score: 0.88
Fold 5: Test=[9,10]  Train=[1,2,3,4,5,6,7,8]   → Score: 0.84

Final Score: Mean = 0.836 ± 0.032 (std)
```

```python
from sklearn.model_selection import cross_val_score, KFold
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import make_classification

# Generate data
X, y = make_classification(n_samples=1000, n_features=20, random_state=42)

# Define model
model = RandomForestClassifier(n_estimators=100, random_state=42)

# 5-Fold Cross-Validation
scores = cross_val_score(model, X, y, cv=5, scoring='accuracy')
print(f"CV Scores: {scores}")
print(f"Mean: {scores.mean():.4f} ± {scores.std():.4f}")

# With explicit KFold object (more control)
kfold = KFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(model, X, y, cv=kfold, scoring='f1')
print(f"F1 Scores: {scores}")
print(f"Mean F1: {scores.mean():.4f} ± {scores.std():.4f}")
```

**How to choose K:**
| K value | Pros | Cons |
|---------|------|------|
| K=5 | Good balance, fast | Slightly higher bias |
| K=10 | Lower bias, standard choice | Slower |
| K=N (LOOCV) | Lowest bias | Very slow, high variance |

---

### 10.4.2 Stratified K-Fold

Maintains class proportions in each fold. **Essential for imbalanced datasets**.

```python
from sklearn.model_selection import StratifiedKFold

# Imbalanced data: 90% class 0, 10% class 1
X, y = make_classification(n_samples=1000, weights=[0.9, 0.1], random_state=42)

# Regular KFold might put all minority samples in one fold!
# Stratified ensures each fold has ~10% class 1
skf = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(model, X, y, cv=skf, scoring='f1')
print(f"Stratified CV F1: {scores.mean():.4f} ± {scores.std():.4f}")
```

---

### 10.4.3 Time Series Cross-Validation

**Never randomly shuffle time series data!** Future data must not leak into training.

```
Time →  [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

Fold 1: Train=[1,2,3]       Test=[4]
Fold 2: Train=[1,2,3,4]     Test=[5]
Fold 3: Train=[1,2,3,4,5]   Test=[6]
Fold 4: Train=[1,2,3,4,5,6] Test=[7]

Training window GROWS, test is always NEXT period
```

```python
from sklearn.model_selection import TimeSeriesSplit

tscv = TimeSeriesSplit(n_splits=5)

# Visualize splits
for fold, (train_idx, test_idx) in enumerate(tscv.split(X)):
    print(f"Fold {fold+1}: Train size={len(train_idx)}, Test size={len(test_idx)}")
    print(f"         Train indices: {train_idx[0]}-{train_idx[-1]}, "
          f"Test indices: {test_idx[0]}-{test_idx[-1]}")

scores = cross_val_score(model, X, y, cv=tscv, scoring='accuracy')
print(f"\nTime Series CV: {scores.mean():.4f} ± {scores.std():.4f}")
```

---

### 10.4.4 Repeated Cross-Validation

Runs K-Fold multiple times with different random splits for more stable estimates:

```python
from sklearn.model_selection import RepeatedStratifiedKFold

# 5-fold repeated 3 times = 15 total evaluations
rskf = RepeatedStratifiedKFold(n_splits=5, n_repeats=3, random_state=42)
scores = cross_val_score(model, X, y, cv=rskf, scoring='accuracy')
print(f"Repeated CV (15 folds): {scores.mean():.4f} ± {scores.std():.4f}")
```

---

### 10.4.5 Leave-One-Out Cross-Validation (LOOCV)

Each sample is used as test set once. K = N.

```python
from sklearn.model_selection import LeaveOneOut

loo = LeaveOneOut()
# WARNING: Very slow for large datasets!
# Only use for small datasets (< 200 samples)
# scores = cross_val_score(model, X_small, y_small, cv=loo)
```

---

### 10.4.6 cross_validate (Advanced)

Returns multiple metrics and train scores:

```python
from sklearn.model_selection import cross_validate

# Get multiple metrics at once + training time
scoring = ['accuracy', 'f1', 'roc_auc']
results = cross_validate(model, X, y, cv=5, scoring=scoring, 
                         return_train_score=True)

print(f"Test Accuracy:  {results['test_accuracy'].mean():.4f}")
print(f"Train Accuracy: {results['train_accuracy'].mean():.4f}")
print(f"Test F1:        {results['test_f1'].mean():.4f}")
print(f"Test AUC:       {results['test_roc_auc'].mean():.4f}")
print(f"Fit Time:       {results['fit_time'].mean():.2f}s")

# If train >> test score → Overfitting!
```

---

## 10.5 Hyperparameter Tuning

### What It Is
Hyperparameters are settings you configure BEFORE training (learning rate, tree depth, number of trees). Tuning means finding the best combination of these settings.

**Analogy**: Hyperparameters are like oven settings (temperature, time). The model learns the "recipe" (parameters), but YOU set the oven. Tuning = finding the perfect temperature and time for the best cake.

---

### 10.5.1 Grid Search

Tries EVERY combination in a predefined grid. Exhaustive but slow.

```python
from sklearn.model_selection import GridSearchCV
from sklearn.ensemble import RandomForestClassifier

# Define parameter grid
param_grid = {
    'n_estimators': [50, 100, 200],      # 3 values
    'max_depth': [5, 10, 20, None],       # 4 values
    'min_samples_split': [2, 5, 10],      # 3 values
    'min_samples_leaf': [1, 2, 4]         # 3 values
}
# Total combinations: 3 × 4 × 3 × 3 = 108
# With 5-fold CV: 108 × 5 = 540 model fits!

model = RandomForestClassifier(random_state=42)

grid_search = GridSearchCV(
    estimator=model,
    param_grid=param_grid,
    cv=5,                    # 5-fold cross-validation
    scoring='f1',            # Metric to optimize
    n_jobs=-1,               # Use all CPU cores
    verbose=1,               # Show progress
    return_train_score=True  # Also track training scores
)

grid_search.fit(X_train, y_train)

# Results
print(f"Best Parameters: {grid_search.best_params_}")
print(f"Best F1 Score:   {grid_search.best_score_:.4f}")
print(f"Best Estimator:  {grid_search.best_estimator_}")

# Use best model for predictions
best_model = grid_search.best_estimator_
y_pred = best_model.predict(X_test)
```

---

### 10.5.2 Randomized Search

Samples random combinations. Much faster for large search spaces.

```python
from sklearn.model_selection import RandomizedSearchCV
from scipy.stats import randint, uniform

# Define distributions (continuous ranges, not fixed values)
param_distributions = {
    'n_estimators': randint(50, 500),          # Random int between 50-500
    'max_depth': randint(3, 50),               # Random int between 3-50
    'min_samples_split': randint(2, 20),       # Random int between 2-20
    'min_samples_leaf': randint(1, 10),        # Random int between 1-10
    'max_features': uniform(0.1, 0.9),         # Random float between 0.1-1.0
}

random_search = RandomizedSearchCV(
    estimator=RandomForestClassifier(random_state=42),
    param_distributions=param_distributions,
    n_iter=100,              # Only try 100 random combinations
    cv=5,
    scoring='f1',
    n_jobs=-1,
    random_state=42,
    verbose=1
)

random_search.fit(X_train, y_train)

print(f"Best Parameters: {random_search.best_params_}")
print(f"Best F1 Score:   {random_search.best_score_:.4f}")
```

> **Pro Tip**: RandomizedSearch with 60 iterations finds a solution in the top 5% of the search space with 95% probability. Often better than GridSearch for the same compute budget.

---

### 10.5.3 Bayesian Optimization (Optuna)

Uses past results to intelligently pick the next hyperparameters to try. Much more efficient than random/grid search.

```python
# pip install optuna
import optuna
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.model_selection import cross_val_score

def objective(trial):
    """Optuna objective function - defines the search space"""
    
    # Suggest hyperparameters
    params = {
        'n_estimators': trial.suggest_int('n_estimators', 50, 500),
        'max_depth': trial.suggest_int('max_depth', 3, 20),
        'learning_rate': trial.suggest_float('learning_rate', 0.01, 0.3, log=True),
        'min_samples_split': trial.suggest_int('min_samples_split', 2, 20),
        'subsample': trial.suggest_float('subsample', 0.5, 1.0),
    }
    
    # Train and evaluate
    model = GradientBoostingClassifier(**params, random_state=42)
    scores = cross_val_score(model, X_train, y_train, cv=5, scoring='f1')
    
    return scores.mean()

# Run optimization
study = optuna.create_study(direction='maximize')  # Maximize F1
study.optimize(objective, n_trials=50, show_progress_bar=True)

# Results
print(f"Best F1: {study.best_value:.4f}")
print(f"Best Params: {study.best_params}")

# Train final model with best params
best_model = GradientBoostingClassifier(**study.best_params, random_state=42)
best_model.fit(X_train, y_train)
```

---

### 10.5.4 Halving Search (Successive Halving)

Starts with many candidates on small data subsets, progressively eliminates poor performers:

```python
from sklearn.experimental import enable_hal_search_cv  # noqa
from sklearn.model_selection import HalvingRandomSearchCV

halving_search = HalvingRandomSearchCV(
    estimator=RandomForestClassifier(random_state=42),
    param_distributions=param_distributions,
    n_candidates=100,       # Start with 100 candidates
    factor=3,               # Eliminate 2/3 each round
    cv=5,
    scoring='f1',
    random_state=42,
    n_jobs=-1
)

halving_search.fit(X_train, y_train)
print(f"Best Score: {halving_search.best_score_:.4f}")
```

---

### 10.5.5 Comparison Table — Tuning Methods

| Method | Speed | Quality | Best For |
|--------|-------|---------|----------|
| Grid Search | Slow (exponential) | Guaranteed optimal in grid | Small search spaces (< 100 combos) |
| Random Search | Fast | Very good | Large spaces, quick experiments |
| Bayesian (Optuna) | Moderate | Best | Expensive models, production tuning |
| Halving Search | Fast | Good | Very large candidate pools |

---

## 10.6 Learning Curves and Validation Curves

### 10.6.1 Learning Curves

Shows how model performance changes as training set size grows. Diagnoses overfitting/underfitting.

```
Score
1.0 ┤
    │  xxxxxxxxxxxxxxxxxxxxxxxxx   ← Training Score
    │  x
0.8 ┤  x       ○○○○○○○○○○○○○○○   ← Validation Score  [GOOD FIT]
    │  x      ○○
    │  x    ○○
0.6 ┤  x  ○○
    │  x○○
    │  ○
0.4 ┤
    └──────────────────────────
       Training Set Size →


Score                               Score
1.0 ┤ xxxxxxxxxxxxxxxxxxxxx        1.0 ┤
    │                                   │  xxxxxxxxxxxxxxxxxx
    │                              0.8 ┤
    │                                   │
0.5 ┤                              0.5 ┤  ○○○○○○○○○○○○○○○○○
    │  ○○○○○○○○○○○○○○○○○○              │  ○○
    │                              0.3 ┤
    └──────────────────────        0.2 ┤
    [OVERFITTING: Big gap]              └──────────────────────
                                        [UNDERFITTING: Both low]
```

```python
from sklearn.model_selection import learning_curve
import matplotlib.pyplot as plt
import numpy as np

# Generate learning curve data
train_sizes, train_scores, val_scores = learning_curve(
    estimator=RandomForestClassifier(n_estimators=100, random_state=42),
    X=X, y=y,
    train_sizes=np.linspace(0.1, 1.0, 10),  # 10% to 100% of data
    cv=5,
    scoring='accuracy',
    n_jobs=-1
)

# Calculate mean and std
train_mean = train_scores.mean(axis=1)
train_std = train_scores.std(axis=1)
val_mean = val_scores.mean(axis=1)
val_std = val_scores.std(axis=1)

# Plot
plt.figure(figsize=(10, 6))
plt.plot(train_sizes, train_mean, 'o-', color='blue', label='Training Score')
plt.plot(train_sizes, val_mean, 'o-', color='red', label='Validation Score')
plt.fill_between(train_sizes, train_mean - train_std, train_mean + train_std, alpha=0.1, color='blue')
plt.fill_between(train_sizes, val_mean - val_std, val_mean + val_std, alpha=0.1, color='red')
plt.xlabel('Training Set Size')
plt.ylabel('Accuracy')
plt.title('Learning Curve')
plt.legend(loc='lower right')
plt.grid(True, alpha=0.3)
plt.show()
```

**How to interpret:**
| Pattern | Diagnosis | Solution |
|---------|-----------|----------|
| Both scores high, small gap | Good fit ✅ | You're done! |
| Train high, Val low (big gap) | Overfitting | More data, regularization, simpler model |
| Both scores low | Underfitting | More features, complex model, less regularization |
| Curves converging | More data will help | Collect more data |
| Curves plateaued (parallel) | More data won't help | Need better features/model |

---

### 10.6.2 Validation Curves

Shows how a single hyperparameter affects both train and validation scores:

```python
from sklearn.model_selection import validation_curve

# How does max_depth affect performance?
param_range = [1, 2, 3, 5, 7, 10, 15, 20, 30, 50]

train_scores, val_scores = validation_curve(
    estimator=RandomForestClassifier(n_estimators=100, random_state=42),
    X=X, y=y,
    param_name='max_depth',
    param_range=param_range,
    cv=5,
    scoring='accuracy',
    n_jobs=-1
)

train_mean = train_scores.mean(axis=1)
val_mean = val_scores.mean(axis=1)

plt.figure(figsize=(10, 6))
plt.plot(param_range, train_mean, 'o-', label='Training Score')
plt.plot(param_range, val_mean, 'o-', label='Validation Score')
plt.xlabel('max_depth')
plt.ylabel('Accuracy')
plt.title('Validation Curve (max_depth)')
plt.legend()
plt.grid(True, alpha=0.3)
plt.show()

# Find optimal value
best_idx = np.argmax(val_mean)
print(f"Best max_depth: {param_range[best_idx]} (Val Score: {val_mean[best_idx]:.4f})")
```

---

## 10.7 Model Selection Strategies

### 10.7.1 Nested Cross-Validation

The **gold standard** for unbiased model comparison. Uses two loops:
- **Outer loop**: Estimates true generalization performance
- **Inner loop**: Tunes hyperparameters

```
┌─ Outer Fold 1 ──────────────────────────────────────────────┐
│  Train (80%)                           │  Test (20%)         │
│  ┌─ Inner Fold 1 ─────────────────┐   │                     │
│  │ Train(64%) │ Val(16%)           │   │  Final evaluation   │
│  └─────────────────────────────────┘   │  on this fold       │
│  ┌─ Inner Fold 2 ─────────────────┐   │                     │
│  │ Train(64%) │ Val(16%)           │   │                     │
│  └─────────────────────────────────┘   │                     │
│  → Best params selected                │                     │
│  → Retrain on full 80% with best params│                     │
│  → Evaluate on Test 20%               ←┘                     │
└──────────────────────────────────────────────────────────────┘
```

```python
from sklearn.model_selection import cross_val_score, GridSearchCV

# Inner CV for tuning
inner_cv = StratifiedKFold(n_splits=3, shuffle=True, random_state=42)

# Outer CV for evaluation
outer_cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)

# Model with tuning
param_grid = {'n_estimators': [50, 100, 200], 'max_depth': [5, 10, 20]}
grid_search = GridSearchCV(
    RandomForestClassifier(random_state=42),
    param_grid=param_grid,
    cv=inner_cv,
    scoring='f1',
    n_jobs=-1
)

# Nested CV scores (unbiased estimate)
nested_scores = cross_val_score(grid_search, X, y, cv=outer_cv, scoring='f1')
print(f"Nested CV F1: {nested_scores.mean():.4f} ± {nested_scores.std():.4f}")
```

---

### 10.7.2 Comparing Multiple Models

```python
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.svm import SVC
from sklearn.neighbors import KNeighborsClassifier
import pandas as pd

# Define models to compare
models = {
    'Logistic Regression': LogisticRegression(max_iter=1000, random_state=42),
    'Random Forest': RandomForestClassifier(n_estimators=100, random_state=42),
    'Gradient Boosting': GradientBoostingClassifier(n_estimators=100, random_state=42),
    'SVM (RBF)': SVC(kernel='rbf', random_state=42),
    'KNN': KNeighborsClassifier(n_neighbors=5),
}

# Evaluate all models
results = []
for name, model in models.items():
    scores = cross_val_score(model, X, y, cv=5, scoring='f1')
    results.append({
        'Model': name,
        'Mean F1': scores.mean(),
        'Std F1': scores.std(),
        'Min F1': scores.min(),
        'Max F1': scores.max()
    })

# Compare in a table
results_df = pd.DataFrame(results).sort_values('Mean F1', ascending=False)
print(results_df.to_string(index=False))
```

---

### 10.7.3 Statistical Significance Tests

Is Model A *really* better than Model B, or is the difference just luck?

```python
from scipy import stats

# Get per-fold scores for two models
scores_rf = cross_val_score(RandomForestClassifier(n_estimators=100, random_state=42), 
                            X, y, cv=10, scoring='f1')
scores_gb = cross_val_score(GradientBoostingClassifier(n_estimators=100, random_state=42), 
                            X, y, cv=10, scoring='f1')

# Paired t-test (same folds)
t_stat, p_value = stats.ttest_rel(scores_rf, scores_gb)
print(f"RF Mean: {scores_rf.mean():.4f}, GB Mean: {scores_gb.mean():.4f}")
print(f"t-statistic: {t_stat:.4f}, p-value: {p_value:.4f}")

if p_value < 0.05:
    print("Difference is statistically significant!")
else:
    print("No significant difference between models.")
```

---

## 10.8 Common Mistakes

### Mistake 1: Data Leakage During Cross-Validation
```python
# ❌ WRONG - Scaling BEFORE cross-validation (leaks test statistics)
from sklearn.preprocessing import StandardScaler
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)  # Sees ALL data including test!
scores = cross_val_score(model, X_scaled, y, cv=5)

# ✅ CORRECT - Use Pipeline (scales within each fold)
from sklearn.pipeline import Pipeline
pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('model', RandomForestClassifier(random_state=42))
])
scores = cross_val_score(pipe, X, y, cv=5, scoring='f1')
```

### Mistake 2: Using Accuracy on Imbalanced Data
```python
# ❌ WRONG - 95% accuracy on 95/5 imbalanced data means nothing
scores = cross_val_score(model, X, y, cv=5, scoring='accuracy')

# ✅ CORRECT - Use F1, AUC, or balanced accuracy
scores = cross_val_score(model, X, y, cv=5, scoring='f1')
# OR
scores = cross_val_score(model, X, y, cv=5, scoring='balanced_accuracy')
```

### Mistake 3: Tuning on Test Set
```python
# ❌ WRONG - Using test set to select hyperparameters
for params in all_combinations:
    model.set_params(**params)
    model.fit(X_train, y_train)
    score = model.score(X_test, y_test)  # This IS your final evaluation!

# ✅ CORRECT - Use validation set or cross-validation for tuning
grid_search = GridSearchCV(model, param_grid, cv=5)
grid_search.fit(X_train, y_train)  # Tunes on train splits only
final_score = grid_search.score(X_test, y_test)  # Untouched test set
```

### Mistake 4: Not Using Stratified Folds for Classification
```python
# ❌ WRONG - Regular KFold on imbalanced classification
cv = KFold(n_splits=5)

# ✅ CORRECT - Stratified to maintain class distribution
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
```

### Mistake 5: Random Shuffling Time Series Data
```python
# ❌ WRONG - Future data leaks into training
cv = KFold(n_splits=5, shuffle=True)

# ✅ CORRECT - Temporal ordering preserved
cv = TimeSeriesSplit(n_splits=5)
```

---

## 10.9 Interview Questions

### Q1: What's the difference between precision and recall? When would you prioritize one over the other?
**Answer**: Precision = "Of all predicted positives, how many are correct?" Recall = "Of all actual positives, how many did we catch?" Prioritize recall for cancer detection (missing cancer is worse than false alarm). Prioritize precision for spam filters (marking legitimate email as spam is worse than letting some spam through).

### Q2: Why is accuracy a poor metric for imbalanced datasets?
**Answer**: With 99% negative and 1% positive class, a model that always predicts "negative" gets 99% accuracy but catches zero positive cases. Use F1-score, AUC-PR, or balanced accuracy instead.

### Q3: Explain the bias-variance tradeoff in the context of cross-validation.
**Answer**: K-fold CV with small K (e.g., 2) has high bias (small training sets) but low variance. Large K (e.g., N=LOOCV) has low bias (almost full training set) but high variance (test set = 1 sample). K=5 or K=10 provides a good balance.

### Q4: What is data leakage and how does it affect model evaluation?
**Answer**: Data leakage occurs when information from the test set influences training. Example: fitting a scaler on all data before splitting. This makes validation scores artificially high but the model fails in production. Solution: use Pipelines that fit transformers only on training folds.

### Q5: When would you use Bayesian optimization over Grid Search?
**Answer**: When the search space is large, evaluations are expensive (deep learning), or hyperparameters are continuous. Bayesian optimization learns from previous trials and focuses on promising regions, requiring far fewer evaluations (50-100 vs. thousands).

### Q6: What is nested cross-validation and why is it needed?
**Answer**: Nested CV uses an outer loop for unbiased performance estimation and an inner loop for hyperparameter tuning. Without it, the same data used for tuning inflates the performance estimate (optimistic bias). Nested CV gives the truest estimate of how a tuned model will perform on unseen data.

### Q7: How do you choose between MAE and RMSE?
**Answer**: MAE treats all errors equally and is robust to outliers. RMSE penalizes large errors more heavily (due to squaring). Use MAE when outliers are noise/errors. Use RMSE when large errors are genuinely worse (e.g., predicting arrival times — being 30 min late is much worse than 3x being 10 min late).

### Q8: What does a high AUC but low F1-score mean?
**Answer**: The model has good ranking ability (separates classes well) but the default threshold (0.5) isn't optimal. Adjusting the classification threshold can improve F1. This often happens with imbalanced datasets.

---

## 10.10 Quick Reference

### Classification Metrics Cheat Sheet

| Metric | Formula | Range | Best For |
|--------|---------|-------|----------|
| Accuracy | (TP+TN) / Total | 0-1 | Balanced datasets |
| Precision | TP / (TP+FP) | 0-1 | When FP is costly |
| Recall | TP / (TP+FN) | 0-1 | When FN is costly |
| F1-Score | 2·P·R / (P+R) | 0-1 | Imbalanced, equal P/R importance |
| AUC-ROC | Area under ROC | 0-1 | Ranking/threshold-independent |
| Log Loss | -mean(y·log(p)) | 0-∞ | Probability calibration |
| Cohen's Kappa | Chance-corrected agreement | -1 to 1 | Comparing to random |

### Regression Metrics Cheat Sheet

| Metric | Sensitive to Outliers | Unit | Best For |
|--------|:---------------------:|------|----------|
| MAE | No | Same as target | Robust evaluation |
| MSE | Yes | Squared | Optimization |
| RMSE | Yes | Same as target | Interpretable, penalize large errors |
| R² | Somewhat | Unitless (0-1) | Model comparison |
| MAPE | No | Percentage | Business reporting |

### Cross-Validation Quick Guide

| Method | Use When |
|--------|----------|
| K-Fold (K=5 or 10) | Default choice for most cases |
| Stratified K-Fold | Classification with imbalanced classes |
| TimeSeriesSplit | Temporal/sequential data |
| Repeated K-Fold | Small datasets, need stable estimates |
| LOOCV | Very small datasets (< 100 samples) |
| Nested CV | Comparing tuned models fairly |

### Hyperparameter Tuning Quick Guide

| Method | Evaluations | Best For |
|--------|-------------|----------|
| GridSearchCV | All combinations | Small spaces (< 100) |
| RandomizedSearchCV | N random samples | Large spaces, quick results |
| Optuna (Bayesian) | Smart sampling | Expensive models, production |
| HalvingSearchCV | Progressive elimination | Very large candidate pools |

### sklearn Scoring Parameters

```python
# Common scoring strings for cross_val_score / GridSearchCV
scoring = 'accuracy'           # Classification
scoring = 'f1'                 # Binary classification (imbalanced)
scoring = 'f1_macro'           # Multi-class (equal class weight)
scoring = 'f1_weighted'        # Multi-class (weighted by support)
scoring = 'roc_auc'            # Binary classification (probability)
scoring = 'roc_auc_ovr'       # Multi-class AUC
scoring = 'precision'          # When FP is expensive
scoring = 'recall'             # When FN is expensive
scoring = 'neg_mean_squared_error'  # Regression (negated because sklearn maximizes)
scoring = 'neg_mean_absolute_error' # Regression
scoring = 'r2'                 # Regression
```

---

*Next Chapter: [11 - Feature Engineering for ML](11-Feature-Engineering-for-ML.md)*
