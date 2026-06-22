# Chapter 03: Training and Callbacks

## Table of Contents
- [1. The Training Process — Big Picture](#1-the-training-process--big-picture)
- [2. model.compile() — Configuring Training](#2-modelcompile--configuring-training)
- [3. Loss Functions — Deep Dive](#3-loss-functions--deep-dive)
- [4. Optimizers — Deep Dive](#4-optimizers--deep-dive)
- [5. Metrics — Tracking Performance](#5-metrics--tracking-performance)
- [6. model.fit() — The Training Loop](#6-modelfit--the-training-loop)
- [7. Training History — Analyzing Results](#7-training-history--analyzing-results)
- [8. Callbacks — Controlling Training](#8-callbacks--controlling-training)
- [9. EarlyStopping — Prevent Overfitting](#9-earlystopping--prevent-overfitting)
- [10. ModelCheckpoint — Save Best Model](#10-modelcheckpoint--save-best-model)
- [11. TensorBoard — Visual Monitoring](#11-tensorboard--visual-monitoring)
- [12. Learning Rate Scheduling](#12-learning-rate-scheduling)
- [13. Custom Callbacks](#13-custom-callbacks)
- [14. model.evaluate() and model.predict()](#14-modelevaluate-and-modelpredict)
- [15. Saving and Loading Models](#15-saving-and-loading-models)
- [16. Complete Training Pipeline Example](#16-complete-training-pipeline-example)
- [17. Common Mistakes](#17-common-mistakes)
- [18. Interview Questions](#18-interview-questions)
- [19. Quick Reference](#19-quick-reference)

---

## 1. The Training Process — Big Picture

### Simple Explanation
Training a neural network is like teaching a student:
1. **Show examples** (forward pass) — student tries to answer
2. **Check answers** (loss computation) — how wrong was the student?
3. **Give feedback** (backward pass/gradients) — which concepts to improve?
4. **Student studies** (parameter update) — adjust understanding
5. **Repeat** until the student masters the subject

### How It Works

```
Training Loop (what model.fit() does internally):

for epoch in range(num_epochs):                    # Repeat full dataset N times
    for batch in dataset:                          # Process data in mini-batches
        ┌──────────────────────────────────────┐
        │ 1. FORWARD PASS                       │
        │    predictions = model(batch_x)        │
        │                                        │
        │ 2. COMPUTE LOSS                        │
        │    loss = loss_fn(batch_y, predictions) │
        │                                        │
        │ 3. BACKWARD PASS (Backpropagation)     │
        │    gradients = tape.gradient(loss, w)   │
        │                                        │
        │ 4. UPDATE WEIGHTS                      │
        │    w = w - learning_rate * gradients    │
        └──────────────────────────────────────┘
    
    # End of epoch: evaluate on validation data
    val_loss, val_metrics = model.evaluate(val_data)
    
    # Callbacks execute here (save model, adjust LR, etc.)
```

### Key Terminology

| Term | Meaning | Example |
|------|---------|---------|
| **Epoch** | One complete pass through the training data | 10 epochs = see all data 10 times |
| **Batch** | Subset of data processed at once | batch_size=32 → 32 samples at a time |
| **Step/Iteration** | One batch forward+backward pass | 60000 samples / 32 batch = 1875 steps/epoch |
| **Training Loss** | Error on training data | Measures how well model fits training data |
| **Validation Loss** | Error on unseen data | Measures generalization ability |
| **Learning Rate** | Step size for weight updates | Too big → diverge, too small → slow |
| **Gradient** | Direction to adjust weights | Points toward increasing loss (we go opposite) |

---

## 2. model.compile() — Configuring Training

### What It Is
`compile()` configures the training process. You specify three things:
1. **Optimizer**: *How* to update weights (algorithm + learning rate)
2. **Loss function**: *What* to minimize (measure of error)
3. **Metrics**: *What* to track (human-readable performance)

### How It Works

```python
import tensorflow as tf
from tensorflow.keras import layers, models

# Build a model first
model = models.Sequential([
    layers.Dense(128, activation='relu', input_shape=(784,)),
    layers.Dense(10, activation='softmax')
])

# ============================================
# BASIC COMPILE
# ============================================

model.compile(
    optimizer='adam',                           # String shortcut
    loss='sparse_categorical_crossentropy',    # String shortcut
    metrics=['accuracy']                       # List of metrics
)

# ============================================
# ADVANCED COMPILE (with full control)
# ============================================

model.compile(
    optimizer=tf.keras.optimizers.Adam(
        learning_rate=0.001,    # Initial learning rate
        beta_1=0.9,            # Exponential decay rate for 1st moment
        beta_2=0.999,          # Exponential decay rate for 2nd moment
        epsilon=1e-07,         # Small constant for numerical stability
        clipnorm=1.0           # Gradient clipping by norm
    ),
    loss=tf.keras.losses.SparseCategoricalCrossentropy(
        from_logits=False      # True if output is raw (no softmax)
    ),
    metrics=[
        tf.keras.metrics.SparseCategoricalAccuracy(name='accuracy'),
        tf.keras.metrics.SparseTopKCategoricalAccuracy(k=5, name='top5_accuracy')
    ]
)

# ============================================
# COMPILE FOR DIFFERENT TASKS
# ============================================

# Binary Classification
model.compile(
    optimizer='adam',
    loss='binary_crossentropy',
    metrics=['accuracy', tf.keras.metrics.AUC(name='auc')]
)

# Multi-class Classification (integer labels)
model.compile(
    optimizer='adam',
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

# Multi-class Classification (one-hot labels)
model.compile(
    optimizer='adam',
    loss='categorical_crossentropy',
    metrics=['accuracy']
)

# Regression
model.compile(
    optimizer='adam',
    loss='mse',              # Mean Squared Error
    metrics=['mae', 'mse']   # Mean Absolute Error + MSE
)

# Multi-label Classification
model.compile(
    optimizer='adam',
    loss='binary_crossentropy',  # Each label is independent binary
    metrics=['accuracy']
)
```

> **Important**: `from_logits=True` vs `from_logits=False`:
> - If your last layer has `activation='softmax'` or `activation='sigmoid'` → use `from_logits=False` (default)
> - If your last layer has no activation (raw output) → use `from_logits=True` (more numerically stable)

```python
# ============================================
# NUMERICALLY STABLE VERSION (recommended for production)
# ============================================

# Instead of:
model = models.Sequential([
    layers.Dense(128, activation='relu', input_shape=(784,)),
    layers.Dense(10, activation='softmax')  # softmax in layer
])
model.compile(loss='sparse_categorical_crossentropy')  # from_logits=False

# Prefer:
model = models.Sequential([
    layers.Dense(128, activation='relu', input_shape=(784,)),
    layers.Dense(10)  # NO activation — raw logits
])
model.compile(
    loss=tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True)
)  # Combines softmax + cross-entropy for better numerical stability
```

---

## 3. Loss Functions — Deep Dive

### What It Is
A loss function measures **how wrong** the model's predictions are. Training = minimizing this number.

**Analogy**: A loss function is like a teacher's grading rubric. It tells the student (model) exactly how far off their answer is from the correct one.

### Mathematical Formulas

**Binary Cross-Entropy** (binary classification):

$$L = -\frac{1}{N}\sum_{i=1}^{N} [y_i \log(\hat{y}_i) + (1-y_i)\log(1-\hat{y}_i)]$$

**Categorical Cross-Entropy** (multi-class):

$$L = -\frac{1}{N}\sum_{i=1}^{N}\sum_{c=1}^{C} y_{i,c} \log(\hat{y}_{i,c})$$

**Mean Squared Error** (regression):

$$L = \frac{1}{N}\sum_{i=1}^{N}(y_i - \hat{y}_i)^2$$

**Mean Absolute Error** (regression):

$$L = \frac{1}{N}\sum_{i=1}^{N}|y_i - \hat{y}_i|$$

### Loss Function Guide

| Loss | Task | Labels Format | Output Activation |
|------|------|---------------|-------------------|
| `binary_crossentropy` | Binary classification | 0 or 1 | `sigmoid` |
| `categorical_crossentropy` | Multi-class | One-hot `[0,0,1,0]` | `softmax` |
| `sparse_categorical_crossentropy` | Multi-class | Integer `2` | `softmax` |
| `mse` (Mean Squared Error) | Regression | Continuous | `None`/`linear` |
| `mae` (Mean Absolute Error) | Regression (robust) | Continuous | `None`/`linear` |
| `huber` | Regression (robust) | Continuous | `None`/`linear` |
| `cosine_similarity` | Similarity | Vectors | `None` |

```python
import tensorflow as tf
import numpy as np

# ============================================
# MANUALLY COMPUTING LOSSES
# ============================================

# Binary Cross-Entropy
y_true = tf.constant([1.0, 0.0, 1.0, 1.0])
y_pred = tf.constant([0.9, 0.1, 0.8, 0.6])

bce = tf.keras.losses.BinaryCrossentropy()
print(f"Binary CE: {bce(y_true, y_pred).numpy():.4f}")  # ~0.1871

# Sparse Categorical Cross-Entropy
y_true_sparse = tf.constant([0, 1, 2])  # Integer labels
y_pred_probs = tf.constant([
    [0.9, 0.05, 0.05],  # Correct: class 0
    [0.1, 0.8, 0.1],    # Correct: class 1
    [0.2, 0.3, 0.5]     # Correct: class 2
])

scce = tf.keras.losses.SparseCategoricalCrossentropy()
print(f"Sparse Cat CE: {scce(y_true_sparse, y_pred_probs).numpy():.4f}")

# MSE
y_true_reg = tf.constant([1.0, 2.0, 3.0])
y_pred_reg = tf.constant([1.1, 2.2, 2.8])

mse = tf.keras.losses.MeanSquaredError()
print(f"MSE: {mse(y_true_reg, y_pred_reg).numpy():.4f}")  # 0.03

# ============================================
# CUSTOM LOSS FUNCTION
# ============================================

# Method 1: Simple function
def custom_mse(y_true, y_pred):
    return tf.reduce_mean(tf.square(y_true - y_pred))

# Method 2: Class-based (for losses with state/config)
class WeightedBCE(tf.keras.losses.Loss):
    """Binary cross-entropy with class weights for imbalanced data."""
    
    def __init__(self, pos_weight=1.0, **kwargs):
        super().__init__(**kwargs)
        self.pos_weight = pos_weight
    
    def call(self, y_true, y_pred):
        y_pred = tf.clip_by_value(y_pred, 1e-7, 1 - 1e-7)  # Avoid log(0)
        loss = -(self.pos_weight * y_true * tf.math.log(y_pred) + 
                 (1 - y_true) * tf.math.log(1 - y_pred))
        return tf.reduce_mean(loss)

# Use custom loss
model.compile(optimizer='adam', loss=WeightedBCE(pos_weight=2.0))

# ============================================
# FOCAL LOSS (for heavily imbalanced datasets)
# ============================================

def focal_loss(gamma=2.0, alpha=0.25):
    """Focal loss for handling class imbalance."""
    def loss_fn(y_true, y_pred):
        y_pred = tf.clip_by_value(y_pred, 1e-7, 1 - 1e-7)
        cross_entropy = -y_true * tf.math.log(y_pred)
        weight = alpha * y_true * tf.pow(1 - y_pred, gamma)
        focal = weight * cross_entropy
        return tf.reduce_mean(tf.reduce_sum(focal, axis=-1))
    return loss_fn

# model.compile(optimizer='adam', loss=focal_loss(gamma=2.0))
```

### Choosing the Right Loss

```
What are you predicting?
├── Category (classification)
│   ├── 2 classes → binary_crossentropy + sigmoid
│   ├── N classes (exclusive) → sparse_categorical_crossentropy + softmax
│   ├── N classes (one-hot labels) → categorical_crossentropy + softmax
│   └── N classes (multiple can be true) → binary_crossentropy + sigmoid (per class)
│
└── Number (regression)
    ├── Normal errors → mse (penalizes large errors more)
    ├── Outliers present → mae or huber (robust to outliers)
    └── Percentage errors → mape
```

---

## 4. Optimizers — Deep Dive

### What It Is
An optimizer decides **how to update weights** given the gradients. Different optimizers use different strategies for step size and direction.

**Analogy**: You're lost in the mountains (high loss) and want to reach the valley (low loss). The optimizer is your navigation strategy:
- **SGD**: Walk downhill, fixed step size
- **Momentum**: Roll downhill like a ball (builds up speed)
- **Adam**: Smart GPS that adapts step size per direction + momentum

### How It Works

```
All optimizers follow this pattern:
    weights_new = weights_old - learning_rate × gradient × (corrections)

The "corrections" part is what makes each optimizer different.
```

### Optimizer Comparison

| Optimizer | Key Idea | Learning Rate | Best For |
|-----------|----------|---------------|----------|
| **SGD** | Simple gradient descent | Needs tuning | Convex problems, with scheduler |
| **SGD + Momentum** | Accumulates past gradients | Needs tuning | Computer vision (with schedule) |
| **RMSprop** | Adapts LR per parameter | Less tuning | RNNs |
| **Adam** | Momentum + adaptive LR | Works well at 0.001 | **Default choice** |
| **AdamW** | Adam + weight decay | 0.001 + decay | **Transformers, modern models** |
| **Nadam** | Adam + Nesterov momentum | Works well at 0.001 | Sometimes better than Adam |

### Mathematical Details

**SGD (Stochastic Gradient Descent)**:
$$w_{t+1} = w_t - \eta \cdot \nabla L(w_t)$$

**SGD with Momentum**:
$$v_{t+1} = \beta \cdot v_t + \nabla L(w_t)$$
$$w_{t+1} = w_t - \eta \cdot v_{t+1}$$

**Adam** (Adaptive Moment Estimation):
$$m_t = \beta_1 \cdot m_{t-1} + (1-\beta_1) \cdot g_t \quad \text{(1st moment / mean)}$$
$$v_t = \beta_2 \cdot v_{t-1} + (1-\beta_2) \cdot g_t^2 \quad \text{(2nd moment / variance)}$$
$$\hat{m}_t = \frac{m_t}{1-\beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1-\beta_2^t} \quad \text{(bias correction)}$$
$$w_{t+1} = w_t - \eta \cdot \frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon}$$

```python
import tensorflow as tf

# ============================================
# OPTIMIZER CONFIGURATIONS
# ============================================

# SGD — Simple but needs tuning
sgd = tf.keras.optimizers.SGD(
    learning_rate=0.01,
    momentum=0.9,            # Use momentum for faster convergence
    nesterov=True             # Nesterov momentum (look-ahead)
)

# Adam — Default choice, works well out of the box
adam = tf.keras.optimizers.Adam(
    learning_rate=0.001,      # Default and usually works
    beta_1=0.9,              # Momentum decay (rarely changed)
    beta_2=0.999,            # Variance decay (rarely changed)
    epsilon=1e-7              # Numerical stability (rarely changed)
)

# AdamW — Adam with decoupled weight decay
adamw = tf.keras.optimizers.AdamW(
    learning_rate=0.001,
    weight_decay=0.01         # L2 regularization, decoupled from LR
)

# RMSprop — Good for RNNs
rmsprop = tf.keras.optimizers.RMSprop(
    learning_rate=0.001,
    rho=0.9                   # Decay rate
)

# ============================================
# GRADIENT CLIPPING (prevent exploding gradients)
# ============================================

# Clip gradients by global norm (most common)
optimizer = tf.keras.optimizers.Adam(
    learning_rate=0.001,
    clipnorm=1.0        # If gradient norm > 1.0, scale it down
)

# Clip gradients by value
optimizer = tf.keras.optimizers.Adam(
    learning_rate=0.001,
    clipvalue=0.5       # Clip each gradient element to [-0.5, 0.5]
)

# ============================================
# PRACTICAL: Comparing Optimizers
# ============================================

import numpy as np

# Load data
(x_train, y_train), (x_test, y_test) = tf.keras.datasets.mnist.load_data()
x_train = x_train.reshape(-1, 784).astype('float32') / 255.0
x_test = x_test.reshape(-1, 784).astype('float32') / 255.0

def build_model():
    return tf.keras.Sequential([
        tf.keras.layers.Dense(128, activation='relu', input_shape=(784,)),
        tf.keras.layers.Dense(10, activation='softmax')
    ])

optimizers_to_test = {
    'SGD': tf.keras.optimizers.SGD(0.01),
    'SGD+Momentum': tf.keras.optimizers.SGD(0.01, momentum=0.9),
    'Adam': tf.keras.optimizers.Adam(0.001),
    'RMSprop': tf.keras.optimizers.RMSprop(0.001),
}

results = {}
for name, opt in optimizers_to_test.items():
    model = build_model()
    model.compile(optimizer=opt, loss='sparse_categorical_crossentropy', metrics=['accuracy'])
    history = model.fit(x_train, y_train, epochs=5, batch_size=128, 
                       validation_split=0.2, verbose=0)
    results[name] = history.history['val_accuracy'][-1]
    print(f"{name:20s} → val_accuracy: {results[name]:.4f}")
```

### Choosing the Right Optimizer

```
Task Type:
├── General / First try          → Adam (lr=0.001)
├── Computer Vision (CNN)        → SGD (lr=0.01, momentum=0.9) + scheduler
├── NLP / Transformers           → AdamW (lr=0.0001, weight_decay=0.01) + warmup
├── RNN / LSTM                   → Adam or RMSprop
├── GANs                         → Adam (lr=0.0002, beta_1=0.5)
└── Fine-tuning pretrained       → Adam (lr=1e-5 to 5e-5, very small!)

If training is unstable:
├── Loss is NaN              → Lower learning rate, add gradient clipping
├── Loss oscillates wildly   → Lower learning rate
├── Loss plateaus early      → Increase learning rate or use scheduler
└── Loss decreases very slow → Increase learning rate or use momentum
```

---

## 5. Metrics — Tracking Performance

### What It Is
Metrics measure model performance in human-understandable terms. Unlike loss (optimized by the model), metrics are for **you** to monitor.

```python
import tensorflow as tf

# ============================================
# CLASSIFICATION METRICS
# ============================================

# Accuracy — most basic
# "What % of predictions are correct?"
model.compile(metrics=['accuracy'])

# AUC (Area Under ROC Curve) — for binary classification
# Better than accuracy for imbalanced datasets
model.compile(metrics=[
    tf.keras.metrics.AUC(name='auc'),
    tf.keras.metrics.AUC(curve='PR', name='pr_auc')  # Precision-Recall AUC
])

# Precision and Recall
model.compile(metrics=[
    tf.keras.metrics.Precision(name='precision'),
    tf.keras.metrics.Recall(name='recall')
])

# F1 Score (via F-beta with beta=1)
model.compile(metrics=[
    tf.keras.metrics.F1Score(name='f1', average='macro')
])

# Top-K Accuracy
model.compile(metrics=[
    tf.keras.metrics.SparseTopKCategoricalAccuracy(k=5, name='top5_acc')
])

# ============================================
# REGRESSION METRICS
# ============================================

model.compile(metrics=[
    'mae',                                              # Mean Absolute Error
    'mse',                                              # Mean Squared Error
    tf.keras.metrics.RootMeanSquaredError(name='rmse'), # RMSE
    tf.keras.metrics.MeanAbsolutePercentageError(name='mape')
])

# ============================================
# CUSTOM METRIC
# ============================================

# Simple function-based metric
def r_squared(y_true, y_pred):
    """R² score — how much variance is explained by the model."""
    ss_res = tf.reduce_sum(tf.square(y_true - y_pred))
    ss_tot = tf.reduce_sum(tf.square(y_true - tf.reduce_mean(y_true)))
    return 1 - (ss_res / ss_tot)

model.compile(optimizer='adam', loss='mse', metrics=[r_squared])

# Class-based metric (for stateful metrics that accumulate over batches)
class F1Score(tf.keras.metrics.Metric):
    def __init__(self, name='f1_score', **kwargs):
        super().__init__(name=name, **kwargs)
        self.precision = tf.keras.metrics.Precision()
        self.recall = tf.keras.metrics.Recall()
    
    def update_state(self, y_true, y_pred, sample_weight=None):
        self.precision.update_state(y_true, y_pred, sample_weight)
        self.recall.update_state(y_true, y_pred, sample_weight)
    
    def result(self):
        p = self.precision.result()
        r = self.recall.result()
        return 2 * (p * r) / (p + r + tf.keras.backend.epsilon())
    
    def reset_state(self):
        self.precision.reset_state()
        self.recall.reset_state()
```

---

## 6. model.fit() — The Training Loop

### What It Is
`model.fit()` is the main training function. It runs the training loop: forward pass → loss → backward pass → weight update, for the specified number of epochs.

### How It Works

```python
import tensorflow as tf
import numpy as np

# Load and prepare data
(x_train, y_train), (x_test, y_test) = tf.keras.datasets.mnist.load_data()
x_train = x_train.reshape(-1, 784).astype('float32') / 255.0
x_test = x_test.reshape(-1, 784).astype('float32') / 255.0

# Build model
model = tf.keras.Sequential([
    tf.keras.layers.Dense(256, activation='relu', input_shape=(784,)),
    tf.keras.layers.Dropout(0.3),
    tf.keras.layers.Dense(128, activation='relu'),
    tf.keras.layers.Dense(10, activation='softmax')
])

model.compile(optimizer='adam',
              loss='sparse_categorical_crossentropy',
              metrics=['accuracy'])

# ============================================
# BASIC model.fit()
# ============================================

history = model.fit(
    x_train,                    # Training data
    y_train,                    # Training labels
    epochs=20,                  # Number of epochs
    batch_size=128,             # Samples per gradient update
    validation_split=0.2        # Use 20% of training data for validation
)

# ============================================
# ADVANCED model.fit() — All Parameters
# ============================================

history = model.fit(
    x_train, y_train,
    
    # Core parameters
    epochs=50,                  # Max number of epochs
    batch_size=64,              # Mini-batch size (32-512 typical)
    
    # Validation
    validation_split=0.2,       # OR use validation_data
    # validation_data=(x_val, y_val),  # Explicit validation set
    
    # Callbacks (covered in detail later)
    callbacks=[
        tf.keras.callbacks.EarlyStopping(patience=5),
        tf.keras.callbacks.ModelCheckpoint('best_model.keras')
    ],
    
    # Data handling
    shuffle=True,               # Shuffle data each epoch (default: True)
    
    # Class weights (for imbalanced data)
    class_weight={0: 1.0, 1: 10.0},  # Class 1 is 10x more important
    
    # Sample weights (per-sample importance)
    # sample_weight=np.ones(len(x_train)),
    
    # Training control
    initial_epoch=0,            # Start from this epoch (for resuming)
    steps_per_epoch=None,       # Batches per epoch (None = use all data)
    validation_steps=None,      # Validation batches
    
    # Output control
    verbose=1                   # 0=silent, 1=progress bar, 2=one line/epoch
)

# ============================================
# USING tf.data.Dataset WITH model.fit()
# ============================================

# Create a tf.data pipeline (more efficient for large datasets)
train_dataset = tf.data.Dataset.from_tensor_slices((x_train, y_train))
train_dataset = train_dataset.shuffle(10000).batch(64).prefetch(tf.data.AUTOTUNE)

val_dataset = tf.data.Dataset.from_tensor_slices((x_test, y_test))
val_dataset = val_dataset.batch(64).prefetch(tf.data.AUTOTUNE)

history = model.fit(
    train_dataset,
    epochs=20,
    validation_data=val_dataset
)
```

### Understanding batch_size

```
Total training samples: 60,000
batch_size = 32:
    Steps per epoch = 60,000 / 32 = 1,875
    Each step processes 32 samples
    More updates per epoch → smoother but slower

batch_size = 512:
    Steps per epoch = 60,000 / 512 ≈ 117
    Each step processes 512 samples
    Fewer updates per epoch → faster but noisier

Trade-offs:
┌──────────────┬─────────────────────────┬──────────────────────────┐
│              │ Small Batch (16-64)     │ Large Batch (256-4096)   │
├──────────────┼─────────────────────────┼──────────────────────────┤
│ Speed        │ Slower (more steps)     │ Faster (fewer steps)     │
│ Memory       │ Less GPU memory         │ More GPU memory          │
│ Noise        │ More noise (regularize) │ Less noise               │
│ Generalize   │ Often better            │ May need LR scaling      │
│ Convergence  │ More updates/epoch      │ Fewer updates/epoch      │
└──────────────┴─────────────────────────┴──────────────────────────┘

Rule of thumb: Start with 32 or 64, increase if GPU has spare memory.
If using large batch, scale learning rate proportionally.
```

---

## 7. Training History — Analyzing Results

### What It Is
`model.fit()` returns a `History` object containing loss and metrics for every epoch. Plotting these curves is **essential** for diagnosing training issues.

```python
import tensorflow as tf
import matplotlib.pyplot as plt
import numpy as np

# Assume model is already trained:
# history = model.fit(...)

# ============================================
# ACCESSING HISTORY
# ============================================

# history.history is a dict of lists
print(history.history.keys())
# dict_keys(['loss', 'accuracy', 'val_loss', 'val_accuracy'])

print(f"Final training loss: {history.history['loss'][-1]:.4f}")
print(f"Final validation loss: {history.history['val_loss'][-1]:.4f}")
print(f"Final training accuracy: {history.history['accuracy'][-1]:.4f}")
print(f"Final validation accuracy: {history.history['val_accuracy'][-1]:.4f}")

# ============================================
# PLOTTING TRAINING CURVES
# ============================================

def plot_training_history(history):
    """Plot training and validation loss/accuracy curves."""
    fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 5))
    
    # Loss plot
    ax1.plot(history.history['loss'], label='Training Loss', linewidth=2)
    ax1.plot(history.history['val_loss'], label='Validation Loss', linewidth=2)
    ax1.set_title('Model Loss', fontsize=14)
    ax1.set_xlabel('Epoch')
    ax1.set_ylabel('Loss')
    ax1.legend()
    ax1.grid(True, alpha=0.3)
    
    # Accuracy plot
    ax2.plot(history.history['accuracy'], label='Training Accuracy', linewidth=2)
    ax2.plot(history.history['val_accuracy'], label='Validation Accuracy', linewidth=2)
    ax2.set_title('Model Accuracy', fontsize=14)
    ax2.set_xlabel('Epoch')
    ax2.set_ylabel('Accuracy')
    ax2.legend()
    ax2.grid(True, alpha=0.3)
    
    plt.tight_layout()
    plt.show()

# plot_training_history(history)

# ============================================
# DIAGNOSING TRAINING ISSUES FROM CURVES
# ============================================

# Pattern 1: UNDERFITTING
# - Both train and val loss are HIGH
# - Gap between them is SMALL
# Fix: More capacity (bigger model), more epochs, less regularization

# Pattern 2: OVERFITTING
# - Train loss keeps decreasing
# - Val loss starts INCREASING after some point
# - Big GAP between train and val
# Fix: More data, dropout, regularization, early stopping, data augmentation

# Pattern 3: GOOD FIT
# - Both curves decrease and flatten
# - Small gap between train and val
# - Val loss slightly higher than train (normal)

# Pattern 4: LEARNING RATE TOO HIGH
# - Loss oscillates wildly or goes to NaN
# Fix: Lower learning rate

# Pattern 5: LEARNING RATE TOO LOW
# - Loss decreases very slowly, still going down at last epoch
# Fix: Higher learning rate or more epochs

# ASCII Visualization:
#
# UNDERFITTING:          OVERFITTING:           GOOD FIT:
# Loss                   Loss                   Loss
# │                      │                      │
# │ ──── train           │    ╱── val            │ ──── train
# │ ──── val             │   ╱                   │ ──── val
# │  ╲                   │  ╱   ╲── train        │  ╲
# │   ╲                  │ ╱     ╲               │   ╲╲
# │    ──────            │╱       ╲──            │    ╲╲────
# │         ─────        │                       │      ─────
# └──────────── Epoch    └──────────── Epoch     └──────────── Epoch
#   Both stay high        Gap widens              Both converge, small gap
```

---

## 8. Callbacks — Controlling Training

### What It Is
Callbacks are functions that execute at specific points during training (epoch start/end, batch start/end). They let you monitor, modify, or stop training automatically.

**Analogy**: Callbacks are like a pit crew in racing. While the car (model) is racing (training), the pit crew monitors tire pressure (metrics), refuels (adjusts learning rate), and can signal the driver to stop if something goes wrong (early stopping).

### How It Works

```
Callback Execution Points:

on_train_begin()           ← Training starts
│
├── on_epoch_begin()       ← Each epoch starts
│   │
│   ├── on_batch_begin()   ← Each batch starts
│   │   (forward + backward pass happens here)
│   ├── on_batch_end()     ← Each batch ends
│   │
│   ├── on_batch_begin()   ← Next batch
│   │   ...
│   ├── on_batch_end()
│   │
│   (validation happens here)
│   
├── on_epoch_end()         ← Each epoch ends (most callbacks act here)
│
├── on_epoch_begin()       ← Next epoch
│   ...
├── on_epoch_end()
│
on_train_end()             ← Training ends
```

### Using Multiple Callbacks

```python
import tensorflow as tf

# ============================================
# COMBINING CALLBACKS
# ============================================

callbacks = [
    # Stop if no improvement for 10 epochs
    tf.keras.callbacks.EarlyStopping(
        monitor='val_loss', patience=10, restore_best_weights=True
    ),
    
    # Save best model
    tf.keras.callbacks.ModelCheckpoint(
        'best_model.keras', monitor='val_loss', save_best_only=True
    ),
    
    # Log to TensorBoard
    tf.keras.callbacks.TensorBoard(log_dir='./logs'),
    
    # Reduce LR when plateauing
    tf.keras.callbacks.ReduceLROnPlateau(
        monitor='val_loss', factor=0.5, patience=5
    ),
    
    # CSV logging
    tf.keras.callbacks.CSVLogger('training_log.csv'),
    
    # Terminate if NaN loss
    tf.keras.callbacks.TerminateOnNaN()
]

# history = model.fit(x_train, y_train, epochs=100, callbacks=callbacks,
#                     validation_split=0.2)
```

---

## 9. EarlyStopping — Prevent Overfitting

### What It Is
EarlyStopping monitors a metric and **stops training** when it stops improving. This prevents overfitting (training too long).

**Analogy**: It's like a student studying for an exam. After a point, more studying doesn't improve their score and might cause burnout (overfitting). EarlyStopping says "you've peaked, stop here."

### How It Works

```
                 val_loss
                 │
                 │    ╲
                 │     ╲
                 │      ╲
                 │       ╲── best (epoch 15)
                 │        ╱╲
                 │       ╱  ╲
                 │      ╱    ── no improvement
    patience=5 →│     ╱       for 5 epochs
                 │    ╱        ← STOP HERE (epoch 20)
                 │   ╱
                 └──────────────── epoch
                      10  15  20
    
    With restore_best_weights=True:
    → Model weights are rolled back to epoch 15 (the best)
```

```python
import tensorflow as tf

# ============================================
# BASIC EARLY STOPPING
# ============================================

early_stop = tf.keras.callbacks.EarlyStopping(
    monitor='val_loss',           # Metric to watch
    patience=10,                  # Epochs to wait before stopping
    restore_best_weights=True     # Roll back to best epoch ← ALWAYS USE THIS
)

# model.fit(..., callbacks=[early_stop])

# ============================================
# ADVANCED EARLY STOPPING
# ============================================

early_stop = tf.keras.callbacks.EarlyStopping(
    monitor='val_loss',           # What to monitor
    min_delta=0.001,              # Minimum change to count as improvement
    patience=10,                  # Epochs to wait after last improvement
    mode='min',                   # 'min' for loss, 'max' for accuracy
    restore_best_weights=True,    # Restore to best epoch weights
    start_from_epoch=5,           # Don't check until epoch 5
    verbose=1                     # Print when stopping
)

# ============================================
# UNDERSTANDING PARAMETERS
# ============================================

# monitor: What metric to watch
#   'val_loss'      → Stop when validation loss stops decreasing
#   'val_accuracy'  → Stop when validation accuracy stops increasing
#   'val_auc'       → Stop when AUC stops increasing

# patience: How many epochs to wait
#   Too small (2-3)  → May stop too early (metric might recover)
#   Too large (50+)  → Wastes time, might overfit
#   Recommended: 5-15 for most cases

# min_delta: Minimum improvement to count
#   0.001 → Improvement < 0.001 doesn't count
#   Prevents stopping due to tiny, meaningless improvements

# mode: 'auto' usually works, but be explicit:
#   'min' → monitoring loss (lower is better)
#   'max' → monitoring accuracy/AUC (higher is better)
```

> **Pro Tip**: Always set `restore_best_weights=True`. Without it, your model keeps the weights from the *last* epoch (which may be worse than the best epoch).

---

## 10. ModelCheckpoint — Save Best Model

### What It Is
ModelCheckpoint saves your model to disk during training — either after every epoch, or only when performance improves.

### How It Works

```python
import tensorflow as tf

# ============================================
# SAVE BEST MODEL ONLY
# ============================================

checkpoint = tf.keras.callbacks.ModelCheckpoint(
    filepath='best_model.keras',    # Save path (.keras recommended)
    monitor='val_loss',             # Watch validation loss
    save_best_only=True,            # Only save when val_loss improves
    mode='min',                     # Lower val_loss is better
    verbose=1                       # Print when saving
)

# ============================================
# SAVE CHECKPOINTS WITH EPOCH INFO
# ============================================

checkpoint = tf.keras.callbacks.ModelCheckpoint(
    filepath='checkpoints/model_epoch{epoch:02d}_val{val_loss:.4f}.keras',
    monitor='val_loss',
    save_best_only=False,     # Save every epoch
    save_freq='epoch'         # Save frequency
)

# ============================================
# SAVE ONLY WEIGHTS (smaller file, need model code to load)
# ============================================

checkpoint = tf.keras.callbacks.ModelCheckpoint(
    filepath='weights/weights_best.weights.h5',
    monitor='val_accuracy',
    save_best_only=True,
    save_weights_only=True,   # Only save weights, not architecture
    mode='max'
)

# ============================================
# PRACTICAL USAGE
# ============================================

callbacks = [
    tf.keras.callbacks.ModelCheckpoint(
        'best_model.keras',
        monitor='val_loss',
        save_best_only=True,
        verbose=1
    ),
    tf.keras.callbacks.EarlyStopping(
        monitor='val_loss',
        patience=10,
        restore_best_weights=True
    )
]

# history = model.fit(x_train, y_train, epochs=100,
#                     validation_split=0.2, callbacks=callbacks)

# Load saved model later
# best_model = tf.keras.models.load_model('best_model.keras')
```

### Save Format Comparison

| Format | Extension | Includes | Size | Load With |
|--------|-----------|----------|------|-----------|
| Keras format | `.keras` | Everything (architecture + weights + optimizer) | Larger | `load_model()` |
| Weights only | `.weights.h5` | Only weights | Smaller | `model.load_weights()` |
| SavedModel | directory | Everything + graph (for serving) | Largest | `load_model()` |

---

## 11. TensorBoard — Visual Monitoring

### What It Is
TensorBoard is a web-based dashboard for monitoring training in real-time. It shows loss/metric curves, model graphs, weight distributions, and more.

### How It Works

```python
import tensorflow as tf
import datetime

# ============================================
# BASIC TENSORBOARD SETUP
# ============================================

# Create a unique log directory for each run
log_dir = "logs/fit/" + datetime.datetime.now().strftime("%Y%m%d-%H%M%S")

tensorboard_callback = tf.keras.callbacks.TensorBoard(
    log_dir=log_dir,
    histogram_freq=1,          # Log weight histograms every epoch
    write_graph=True,          # Log model graph
    write_images=False,        # Log weight images (slow)
    update_freq='epoch',       # Log metrics every epoch
    profile_batch=0            # Profile batch (0=disabled, '2,5'=profile batches 2-5)
)

# history = model.fit(x_train, y_train, epochs=20,
#                     validation_split=0.2,
#                     callbacks=[tensorboard_callback])

# ============================================
# LAUNCH TENSORBOARD
# ============================================

# In terminal:
# tensorboard --logdir=logs/fit
# Then open http://localhost:6006 in browser

# In Jupyter notebook:
# %load_ext tensorboard
# %tensorboard --logdir logs/fit

# ============================================
# CUSTOM TENSORBOARD LOGGING
# ============================================

# Log custom scalars, images, or text
log_dir = "logs/custom/" + datetime.datetime.now().strftime("%Y%m%d-%H%M%S")
file_writer = tf.summary.create_file_writer(log_dir)

# In a custom training loop:
for epoch in range(10):
    # ... training code ...
    train_loss = 0.5 - epoch * 0.04  # Example
    
    with file_writer.as_default():
        tf.summary.scalar('custom_loss', train_loss, step=epoch)
        tf.summary.scalar('learning_rate', 0.001, step=epoch)
        
        # Log images
        # tf.summary.image('sample_image', images, step=epoch)
        
        # Log text
        tf.summary.text('notes', f'Epoch {epoch} completed', step=epoch)

# ============================================
# COMPARING MULTIPLE RUNS
# ============================================

# Run 1: logs/fit/run1_adam_lr001/
# Run 2: logs/fit/run2_sgd_lr01/
# Run 3: logs/fit/run3_adam_lr0001/

# TensorBoard automatically overlays all runs for comparison
# tensorboard --logdir=logs/fit
```

### TensorBoard Panels

| Panel | What It Shows | Use For |
|-------|--------------|---------|
| **Scalars** | Loss/metrics over time | Training progress |
| **Graphs** | Model computation graph | Architecture debugging |
| **Distributions** | Weight/bias distributions | Detecting dead neurons |
| **Histograms** | Weight value histograms | Training health |
| **Images** | Logged images | Visualizing predictions |
| **Projector** | Embedding visualization | Clustering, similarity |
| **Profile** | Performance profiling | Finding bottlenecks |

---

## 12. Learning Rate Scheduling

### What It Is
Learning rate scheduling **changes the learning rate during training**. Typically, you want to start with a higher LR (explore broadly) and gradually decrease it (fine-tune).

**Analogy**: When looking for your keys in a house, you first search room by room (big steps), then once you find the right room, you search carefully in every corner (small steps).

### Why It Matters

```
Learning Rate too HIGH:
Loss │  ╱╲  ╱╲  ╱╲
     │ ╱  ╲╱  ╲╱  ╲   → Oscillates, never converges
     └────────────── epoch

Learning Rate too LOW:
Loss │
     │ ╲
     │  ╲
     │   ──────────    → Very slow, may get stuck
     └────────────── epoch

Scheduled Learning Rate:
Loss │
     │ ╲   (high LR: explore)
     │  ╲╲
     │    ╲─── (low LR: fine-tune)
     │      ────────   → Fast convergence + good final result
     └────────────── epoch
```

### Built-in Schedulers

```python
import tensorflow as tf
import numpy as np

# ============================================
# 1. ReduceLROnPlateau (most practical)
# ============================================
# Reduces LR when metric stops improving

reduce_lr = tf.keras.callbacks.ReduceLROnPlateau(
    monitor='val_loss',     # Metric to watch
    factor=0.5,             # Multiply LR by this when triggered (new_lr = lr * 0.5)
    patience=5,             # Epochs to wait before reducing
    min_lr=1e-7,            # Don't go below this LR
    min_delta=0.001,        # Minimum improvement to count
    verbose=1               # Print when reducing
)

# ============================================
# 2. LearningRateScheduler (custom schedule)
# ============================================

# Step decay: halve LR every 10 epochs
def step_decay(epoch, lr):
    if epoch > 0 and epoch % 10 == 0:
        return lr * 0.5
    return lr

lr_scheduler = tf.keras.callbacks.LearningRateScheduler(
    step_decay, verbose=1
)

# Exponential decay
def exp_decay(epoch, lr):
    return lr * tf.math.exp(-0.1)

# Warmup + decay (common for transformers)
def warmup_decay(epoch, lr):
    warmup_epochs = 5
    if epoch < warmup_epochs:
        return epoch / warmup_epochs * 0.001  # Linear warmup to 0.001
    else:
        return 0.001 * tf.math.exp(-0.1 * (epoch - warmup_epochs))

lr_scheduler = tf.keras.callbacks.LearningRateScheduler(warmup_decay, verbose=1)

# ============================================
# 3. tf.keras.optimizers.schedules (built-in schedules)
# ============================================

# Exponential decay schedule
lr_schedule = tf.keras.optimizers.schedules.ExponentialDecay(
    initial_learning_rate=0.001,
    decay_steps=1000,           # Decay every 1000 steps
    decay_rate=0.96,            # Multiply by 0.96
    staircase=True              # Step function (not smooth)
)

# Cosine decay (popular for vision models)
lr_schedule = tf.keras.optimizers.schedules.CosineDecay(
    initial_learning_rate=0.001,
    decay_steps=10000,          # Total training steps
    alpha=0.0001                # Minimum LR (fraction of initial)
)

# Cosine decay with warmup
lr_schedule = tf.keras.optimizers.schedules.CosineDecay(
    initial_learning_rate=0.001,
    decay_steps=10000,
    warmup_target=0.001,        # LR to reach after warmup
    warmup_steps=1000           # Steps for linear warmup
)

# Use schedule directly with optimizer
optimizer = tf.keras.optimizers.Adam(learning_rate=lr_schedule)
# model.compile(optimizer=optimizer, ...)

# ============================================
# 4. POLYNOMIAL DECAY
# ============================================

lr_schedule = tf.keras.optimizers.schedules.PolynomialDecay(
    initial_learning_rate=0.001,
    decay_steps=10000,
    end_learning_rate=0.0001,
    power=1.0                   # 1.0 = linear decay
)

# ============================================
# VISUALIZE SCHEDULES
# ============================================

import matplotlib.pyplot as plt

steps = np.arange(0, 10000)

schedules = {
    'Exponential': tf.keras.optimizers.schedules.ExponentialDecay(
        0.001, 1000, 0.9),
    'Cosine': tf.keras.optimizers.schedules.CosineDecay(
        0.001, 10000),
    'Polynomial': tf.keras.optimizers.schedules.PolynomialDecay(
        0.001, 10000, 0.0001),
}

plt.figure(figsize=(10, 5))
for name, schedule in schedules.items():
    lrs = [schedule(step).numpy() for step in steps]
    plt.plot(steps, lrs, label=name, linewidth=2)

plt.xlabel('Training Step')
plt.ylabel('Learning Rate')
plt.title('Learning Rate Schedules Comparison')
plt.legend()
plt.grid(True, alpha=0.3)
plt.show()
```

### Choosing a Schedule

```
Task / Architecture:
├── CNN (image classification)     → Cosine decay or step decay
├── Transformers / NLP             → Warmup + linear/cosine decay
├── Transfer learning (fine-tune)  → Very low constant LR (1e-5)
├── Don't know what to use         → ReduceLROnPlateau (automatic!)
└── Research / experimentation     → Cosine decay with warm restarts
```

---

## 13. Custom Callbacks

### What It Is
When built-in callbacks aren't enough, you can create custom ones by subclassing `tf.keras.callbacks.Callback`.

```python
import tensorflow as tf
import numpy as np
import time

# ============================================
# CUSTOM CALLBACK: Training Timer
# ============================================

class TrainingTimer(tf.keras.callbacks.Callback):
    """Track and report training time per epoch."""
    
    def on_train_begin(self, logs=None):
        self.train_start = time.time()
        print("Training started...")
    
    def on_epoch_begin(self, epoch, logs=None):
        self.epoch_start = time.time()
    
    def on_epoch_end(self, epoch, logs=None):
        epoch_time = time.time() - self.epoch_start
        total_time = time.time() - self.train_start
        print(f"  Epoch {epoch+1} took {epoch_time:.1f}s | Total: {total_time:.1f}s")
    
    def on_train_end(self, logs=None):
        total = time.time() - self.train_start
        print(f"\nTotal training time: {total:.1f}s ({total/60:.1f} min)")

# ============================================
# CUSTOM CALLBACK: Gradient Monitor
# ============================================

class GradientMonitor(tf.keras.callbacks.Callback):
    """Monitor gradient norms to detect vanishing/exploding gradients."""
    
    def on_epoch_end(self, epoch, logs=None):
        for layer in self.model.layers:
            if layer.trainable_weights:
                for weight in layer.trainable_weights:
                    grad_norm = tf.norm(weight).numpy()
                    if grad_norm < 1e-7:
                        print(f"⚠️ Near-zero weights in {layer.name}: {weight.name}")
                    elif grad_norm > 1000:
                        print(f"🔥 Exploding weights in {layer.name}: {weight.name}")

# ============================================
# CUSTOM CALLBACK: Learning Rate Finder
# ============================================

class LearningRateFinder(tf.keras.callbacks.Callback):
    """Find optimal learning rate by gradually increasing it."""
    
    def __init__(self, min_lr=1e-7, max_lr=1.0, steps=100):
        super().__init__()
        self.min_lr = min_lr
        self.max_lr = max_lr
        self.steps = steps
        self.lrs = []
        self.losses = []
        self.step = 0
    
    def on_batch_end(self, batch, logs=None):
        # Exponentially increase LR
        lr = self.min_lr * (self.max_lr / self.min_lr) ** (self.step / self.steps)
        tf.keras.backend.set_value(self.model.optimizer.learning_rate, lr)
        
        self.lrs.append(lr)
        self.losses.append(logs['loss'])
        self.step += 1
        
        # Stop if loss explodes
        if self.step > self.steps or logs['loss'] > self.losses[0] * 4:
            self.model.stop_training = True
    
    def plot(self):
        import matplotlib.pyplot as plt
        plt.figure(figsize=(10, 5))
        plt.semilogx(self.lrs, self.losses)
        plt.xlabel('Learning Rate (log scale)')
        plt.ylabel('Loss')
        plt.title('Learning Rate Finder')
        plt.grid(True, alpha=0.3)
        plt.show()
        
        # Find best LR (where loss decreases fastest)
        min_loss_idx = np.argmin(self.losses)
        best_lr = self.lrs[min_loss_idx] / 10  # Use 10x lower than minimum
        print(f"Suggested learning rate: {best_lr:.6f}")

# Usage:
# lr_finder = LearningRateFinder()
# model.fit(x_train, y_train, epochs=1, batch_size=64, callbacks=[lr_finder])
# lr_finder.plot()

# ============================================
# CUSTOM CALLBACK: Slack/Email Notification
# ============================================

class TrainingNotifier(tf.keras.callbacks.Callback):
    """Send notification when training completes or fails."""
    
    def on_train_end(self, logs=None):
        val_loss = logs.get('val_loss', 'N/A')
        val_acc = logs.get('val_accuracy', 'N/A')
        message = f"Training complete! Val Loss: {val_loss:.4f}, Val Acc: {val_acc:.4f}"
        print(f"📧 Notification: {message}")
        # In production: send to Slack, email, or SMS API

# ============================================
# USING CUSTOM CALLBACKS
# ============================================

callbacks = [
    TrainingTimer(),
    tf.keras.callbacks.EarlyStopping(patience=5, restore_best_weights=True),
    tf.keras.callbacks.ModelCheckpoint('best.keras', save_best_only=True),
    TrainingNotifier()
]

# model.fit(x_train, y_train, epochs=50, callbacks=callbacks, validation_split=0.2)
```

---

## 14. model.evaluate() and model.predict()

### How It Works

```python
import tensorflow as tf
import numpy as np

# Assume model is trained

# ============================================
# model.evaluate() — Get loss and metrics on a dataset
# ============================================

# Basic evaluation
test_loss, test_accuracy = model.evaluate(x_test, y_test)
print(f"Test Loss: {test_loss:.4f}")
print(f"Test Accuracy: {test_accuracy:.4f}")

# Verbose control
model.evaluate(x_test, y_test, verbose=0)  # Silent
model.evaluate(x_test, y_test, verbose=1)  # Progress bar
model.evaluate(x_test, y_test, verbose=2)  # One line

# With batches
model.evaluate(x_test, y_test, batch_size=256)

# With tf.data
# test_dataset = tf.data.Dataset.from_tensor_slices((x_test, y_test)).batch(64)
# model.evaluate(test_dataset)

# ============================================
# model.predict() — Get predictions
# ============================================

# Get predictions for all test data
predictions = model.predict(x_test)
print(f"Predictions shape: {predictions.shape}")  # (10000, 10) for 10 classes

# Each row is probability distribution over classes
print(f"First prediction: {predictions[0]}")       # [0.01, 0.02, ..., 0.8, ...]
print(f"Sum of probs: {predictions[0].sum():.4f}")  # 1.0000

# Get predicted classes
predicted_classes = np.argmax(predictions, axis=1)
print(f"First 10 predicted: {predicted_classes[:10]}")
print(f"First 10 actual:    {y_test[:10]}")

# Predict single sample (must add batch dimension!)
single = x_test[0:1]  # Shape: (1, 784), NOT (784,)
single_pred = model.predict(single, verbose=0)
print(f"Single prediction: class {np.argmax(single_pred)}")

# For faster inference on small inputs:
single_pred = model(tf.constant(single), training=False)  # Direct call, no overhead

# ============================================
# CLASSIFICATION REPORT
# ============================================

from sklearn.metrics import classification_report, confusion_matrix

y_pred = np.argmax(model.predict(x_test), axis=1)

print("\nClassification Report:")
print(classification_report(y_test, y_pred, digits=4))

# Confusion Matrix
cm = confusion_matrix(y_test, y_pred)
print(f"\nConfusion Matrix:\n{cm}")
```

---

## 15. Saving and Loading Models

### How It Works

```python
import tensorflow as tf

# ============================================
# 1. KERAS FORMAT (.keras) — RECOMMENDED
# ============================================

# Save entire model (architecture + weights + optimizer + config)
model.save('my_model.keras')

# Load model
loaded_model = tf.keras.models.load_model('my_model.keras')

# Verify
loaded_model.evaluate(x_test, y_test)

# ============================================
# 2. WEIGHTS ONLY (.weights.h5)
# ============================================

# Save weights only (need model code to reconstruct)
model.save_weights('model_weights.weights.h5')

# Load into same architecture
new_model = build_model()  # Must define same architecture first!
new_model.load_weights('model_weights.weights.h5')

# ============================================
# 3. SAVEDMODEL FORMAT (for deployment)
# ============================================

# Save as SavedModel (TF Serving, TF Lite, TF.js compatible)
model.export('saved_model_dir')

# Load SavedModel
loaded = tf.keras.models.load_model('saved_model_dir')

# ============================================
# 4. SAVE/LOAD WITH CUSTOM OBJECTS
# ============================================

# If model has custom layers, losses, or metrics:

# Method 1: Register custom objects
@tf.keras.utils.register_keras_serializable()
class CustomDense(tf.keras.layers.Layer):
    def __init__(self, units, **kwargs):
        super().__init__(**kwargs)
        self.units = units
        self.dense = tf.keras.layers.Dense(units)
    
    def call(self, inputs):
        return self.dense(inputs)
    
    def get_config(self):
        config = super().get_config()
        config.update({'units': self.units})
        return config

# Method 2: Pass custom_objects when loading
# loaded = tf.keras.models.load_model('model.keras',
#     custom_objects={'CustomDense': CustomDense})

# ============================================
# 5. SAVE MODEL ARCHITECTURE (JSON)
# ============================================

# Save architecture only (no weights)
json_config = model.to_json()

# Recreate from JSON
reconstructed = tf.keras.models.model_from_json(json_config)
# reconstructed has random weights — need to train or load weights

# ============================================
# 6. CHECKPOINTING WITH TF (low-level)
# ============================================

# tf.train.Checkpoint — more flexible, used in custom training loops
checkpoint = tf.train.Checkpoint(model=model, optimizer=model.optimizer)
checkpoint.save('tf_ckpts/ckpt')

# Restore
checkpoint.restore(tf.train.latest_checkpoint('tf_ckpts'))
```

### Save Format Decision Guide

| Need | Format | Code |
|------|--------|------|
| Resume training later | `.keras` | `model.save('m.keras')` |
| Deploy to server | SavedModel | `model.export('dir')` |
| Transfer weights to different code | `.weights.h5` | `model.save_weights(...)` |
| Version control architecture | JSON | `model.to_json()` |
| Custom training loops | Checkpoint | `tf.train.Checkpoint` |

---

## 16. Complete Training Pipeline Example

```python
import tensorflow as tf
from tensorflow.keras import layers, models, callbacks
import numpy as np
import datetime

# ============================================
# 1. DATA PREPARATION
# ============================================

# Load dataset
(x_train, y_train), (x_test, y_test) = tf.keras.datasets.fashion_mnist.load_data()

# Class names for Fashion MNIST
class_names = ['T-shirt/top', 'Trouser', 'Pullover', 'Dress', 'Coat',
               'Sandal', 'Shirt', 'Sneaker', 'Bag', 'Ankle boot']

# Preprocess
x_train = x_train.astype('float32') / 255.0
x_test = x_test.astype('float32') / 255.0

# Add channel dimension for CNN
x_train = x_train[..., np.newaxis]  # (60000, 28, 28, 1)
x_test = x_test[..., np.newaxis]    # (10000, 28, 28, 1)

# Split training into train + validation
val_size = 5000
x_val, y_val = x_train[:val_size], y_train[:val_size]
x_train, y_train = x_train[val_size:], y_train[val_size:]

print(f"Training:   {x_train.shape}")   # (55000, 28, 28, 1)
print(f"Validation: {x_val.shape}")     # (5000, 28, 28, 1)
print(f"Test:       {x_test.shape}")    # (10000, 28, 28, 1)

# ============================================
# 2. BUILD MODEL
# ============================================

model = models.Sequential([
    # Input
    layers.Input(shape=(28, 28, 1)),
    
    # Convolutional block 1
    layers.Conv2D(32, (3, 3), activation='relu', padding='same'),
    layers.BatchNormalization(),
    layers.Conv2D(32, (3, 3), activation='relu', padding='same'),
    layers.MaxPooling2D((2, 2)),
    layers.Dropout(0.25),
    
    # Convolutional block 2
    layers.Conv2D(64, (3, 3), activation='relu', padding='same'),
    layers.BatchNormalization(),
    layers.Conv2D(64, (3, 3), activation='relu', padding='same'),
    layers.MaxPooling2D((2, 2)),
    layers.Dropout(0.25),
    
    # Dense layers
    layers.Flatten(),
    layers.Dense(256, activation='relu'),
    layers.BatchNormalization(),
    layers.Dropout(0.5),
    layers.Dense(10)  # No activation — using from_logits=True
], name='fashion_mnist_cnn')

model.summary()

# ============================================
# 3. COMPILE
# ============================================

optimizer = tf.keras.optimizers.Adam(learning_rate=0.001)

model.compile(
    optimizer=optimizer,
    loss=tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True),
    metrics=['accuracy']
)

# ============================================
# 4. SETUP CALLBACKS
# ============================================

log_dir = "logs/fashion_mnist/" + datetime.datetime.now().strftime("%Y%m%d-%H%M%S")

my_callbacks = [
    # Early stopping
    callbacks.EarlyStopping(
        monitor='val_loss',
        patience=10,
        restore_best_weights=True,
        verbose=1
    ),
    
    # Save best model
    callbacks.ModelCheckpoint(
        'best_fashion_model.keras',
        monitor='val_loss',
        save_best_only=True,
        verbose=1
    ),
    
    # Reduce learning rate on plateau
    callbacks.ReduceLROnPlateau(
        monitor='val_loss',
        factor=0.5,
        patience=5,
        min_lr=1e-7,
        verbose=1
    ),
    
    # TensorBoard logging
    callbacks.TensorBoard(
        log_dir=log_dir,
        histogram_freq=1
    ),
    
    # Safety: stop if loss becomes NaN
    callbacks.TerminateOnNaN()
]

# ============================================
# 5. TRAIN
# ============================================

history = model.fit(
    x_train, y_train,
    epochs=50,
    batch_size=64,
    validation_data=(x_val, y_val),
    callbacks=my_callbacks
)

# ============================================
# 6. EVALUATE
# ============================================

# Evaluate on test set
test_loss, test_accuracy = model.evaluate(x_test, y_test, verbose=0)
print(f"\nTest Loss: {test_loss:.4f}")
print(f"Test Accuracy: {test_accuracy:.4f}")

# Detailed predictions
predictions = model.predict(x_test)
predicted_classes = np.argmax(predictions, axis=1)

# Per-class accuracy
from sklearn.metrics import classification_report
print("\n" + classification_report(y_test, predicted_classes,
                                    target_names=class_names, digits=4))

# ============================================
# 7. PLOT TRAINING HISTORY
# ============================================

import matplotlib.pyplot as plt

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 5))

ax1.plot(history.history['loss'], label='Train Loss')
ax1.plot(history.history['val_loss'], label='Val Loss')
ax1.set_title('Loss')
ax1.set_xlabel('Epoch')
ax1.legend()
ax1.grid(True, alpha=0.3)

ax2.plot(history.history['accuracy'], label='Train Acc')
ax2.plot(history.history['val_accuracy'], label='Val Acc')
ax2.set_title('Accuracy')
ax2.set_xlabel('Epoch')
ax2.legend()
ax2.grid(True, alpha=0.3)

plt.tight_layout()
plt.savefig('training_curves.png', dpi=150)
plt.show()

# ============================================
# 8. LOAD BEST MODEL FOR DEPLOYMENT
# ============================================

best_model = tf.keras.models.load_model('best_fashion_model.keras')
final_loss, final_acc = best_model.evaluate(x_test, y_test, verbose=0)
print(f"\nBest model — Test Accuracy: {final_acc:.4f}")
```

---

## 17. Common Mistakes

### Mistake 1: Not using validation data

```python
# ❌ WRONG: No validation — can't detect overfitting
model.fit(x_train, y_train, epochs=100)

# ✅ FIX: Always use validation
model.fit(x_train, y_train, epochs=100, validation_split=0.2)
# OR
model.fit(x_train, y_train, epochs=100, validation_data=(x_val, y_val))
```

### Mistake 2: Training too many epochs without EarlyStopping

```python
# ❌ WRONG: Overfits after epoch 20
model.fit(x_train, y_train, epochs=200)

# ✅ FIX: Use EarlyStopping
model.fit(x_train, y_train, epochs=200, callbacks=[
    tf.keras.callbacks.EarlyStopping(patience=10, restore_best_weights=True)
], validation_split=0.2)
```

### Mistake 3: Wrong learning rate

```python
# ❌ WRONG: LR too high → loss explodes
model.compile(optimizer=tf.keras.optimizers.Adam(learning_rate=0.1))

# ❌ WRONG: LR too low → training takes forever
model.compile(optimizer=tf.keras.optimizers.Adam(learning_rate=1e-8))

# ✅ FIX: Start with defaults
model.compile(optimizer='adam')  # Default LR = 0.001, usually works

# Or use learning rate finder, or ReduceLROnPlateau
```

### Mistake 4: Not normalizing input data

```python
# ❌ WRONG: Raw pixel values [0, 255]
model.fit(x_train_raw, y_train, epochs=10)  # Slow convergence, poor results

# ✅ FIX: Normalize to [0, 1] or standardize to mean=0, std=1
x_train = x_train_raw / 255.0  # Simple normalization

# OR
mean = x_train_raw.mean()
std = x_train_raw.std()
x_train = (x_train_raw - mean) / std  # Standardization
```

### Mistake 5: Forgetting restore_best_weights in EarlyStopping

```python
# ❌ WRONG: EarlyStopping without restore
early_stop = tf.keras.callbacks.EarlyStopping(patience=10)
# Model keeps weights from LAST epoch, which may be worse!

# ✅ FIX: Always restore best weights
early_stop = tf.keras.callbacks.EarlyStopping(
    patience=10, restore_best_weights=True  # ← ALWAYS include this
)
```

### Mistake 6: Using validation_split with shuffled data

```python
# ⚠️ CAUTION: validation_split takes the LAST x% of data
# If data is sorted by class, validation will only have last classes!

# ✅ FIX: Shuffle data first, or use explicit validation_data
indices = np.random.permutation(len(x_train))
x_train = x_train[indices]
y_train = y_train[indices]
model.fit(x_train, y_train, validation_split=0.2)
```

---

## 18. Interview Questions

### Q1: What does model.compile() do? What are its main arguments?
**Answer**: `model.compile()` configures the training process. Three main arguments:
- **optimizer**: Algorithm for updating weights (e.g., Adam, SGD) — controls *how* to learn
- **loss**: Function to minimize (e.g., cross-entropy, MSE) — defines *what* to optimize
- **metrics**: Performance measures to track (e.g., accuracy) — for human monitoring, not optimized directly

### Q2: Explain the difference between training loss and validation loss. What does it mean if they diverge?
**Answer**: 
- **Training loss**: Error on data the model trains on. Always decreasing is expected.
- **Validation loss**: Error on held-out data. Measures generalization.
- **Divergence** (val_loss increases while train_loss decreases) = **overfitting** — model memorizes training data but fails on new data. Fix with: more data, dropout, regularization, early stopping, data augmentation.

### Q3: What is the purpose of callbacks? Name five common ones.
**Answer**: Callbacks execute custom logic at specific training points (epoch start/end, batch start/end). Five common callbacks:
1. **EarlyStopping**: Stop when metric plateaus
2. **ModelCheckpoint**: Save model/weights periodically
3. **ReduceLROnPlateau**: Reduce learning rate when metric stops improving
4. **TensorBoard**: Visual logging of metrics, graphs, histograms
5. **LearningRateScheduler**: Custom learning rate schedule

### Q4: What is learning rate scheduling and why is it important?
**Answer**: LR scheduling changes the learning rate during training. Important because:
- **High LR initially**: Explore loss landscape broadly, converge quickly
- **Low LR later**: Fine-tune near the optimum, reach lower loss
- Without scheduling: fixed LR is either too high (oscillates) or too low (slow)
- Common strategies: cosine decay, step decay, warmup + decay, ReduceLROnPlateau

### Q5: Compare Adam and SGD optimizers. When would you use each?
**Answer**:
- **Adam**: Adaptive learning rates per parameter, includes momentum. Converges fast with minimal tuning. Default choice for most tasks.
- **SGD + Momentum**: Simpler, often generalizes better than Adam for CNNs when paired with proper LR scheduling. Requires more tuning.
- Use Adam for: prototyping, NLP, GANs, when you want fast results
- Use SGD for: computer vision (with cosine schedule), when you want best possible accuracy

### Q6: What is `from_logits=True` and why does it matter?
**Answer**: `from_logits=True` tells the loss function that model output is raw (no softmax/sigmoid). The loss function internally applies softmax + cross-entropy in a single operation using the log-sum-exp trick, which is more **numerically stable** than computing softmax first then cross-entropy separately. In production, prefer `from_logits=True` with no activation on the last layer.

### Q7: How do you handle class imbalance during training?
**Answer**: Multiple approaches:
1. **class_weight** in `model.fit()`: Give higher weight to minority class loss
2. **Oversampling**: Duplicate minority samples (SMOTE)
3. **Undersampling**: Reduce majority samples
4. **Focal loss**: Down-weight easy examples, focus on hard ones
5. **Data augmentation**: Create synthetic minority samples
6. **Metrics**: Use AUC, F1, precision-recall instead of accuracy

---

## 19. Quick Reference

### model.compile() Cheat Sheet

```python
# Binary Classification
model.compile(optimizer='adam', loss='binary_crossentropy', metrics=['accuracy'])

# Multi-class (integer labels)
model.compile(optimizer='adam', loss='sparse_categorical_crossentropy', metrics=['accuracy'])

# Multi-class (one-hot labels)
model.compile(optimizer='adam', loss='categorical_crossentropy', metrics=['accuracy'])

# Regression
model.compile(optimizer='adam', loss='mse', metrics=['mae'])
```

### Callback Cheat Sheet

| Callback | Purpose | Key Parameters |
|----------|---------|----------------|
| `EarlyStopping` | Stop when metric plateaus | `patience`, `restore_best_weights` |
| `ModelCheckpoint` | Save best model | `save_best_only`, `monitor` |
| `ReduceLROnPlateau` | Auto-reduce learning rate | `factor`, `patience` |
| `TensorBoard` | Visual logging | `log_dir`, `histogram_freq` |
| `LearningRateScheduler` | Custom LR schedule | `schedule` function |
| `CSVLogger` | Log to CSV file | `filename` |
| `TerminateOnNaN` | Stop if loss is NaN | (none) |

### Optimizer Cheat Sheet

| Optimizer | Default LR | When to Use |
|-----------|-----------|-------------|
| `Adam` | 0.001 | Default choice, most tasks |
| `AdamW` | 0.001 | Transformers, weight decay |
| `SGD(momentum=0.9)` | 0.01 | CNNs with LR schedule |
| `RMSprop` | 0.001 | RNNs |

### Training Diagnostics

| Symptom | Diagnosis | Fix |
|---------|-----------|-----|
| Loss = NaN | Exploding gradients | Lower LR, gradient clipping |
| Loss doesn't decrease | LR too low or model too simple | Increase LR, bigger model |
| Loss oscillates | LR too high | Lower LR |
| Val loss >> Train loss | Overfitting | Dropout, regularization, more data |
| Val loss ≈ Train loss (both high) | Underfitting | Bigger model, more epochs |
| Val loss decreases then increases | Overfitting after epoch X | EarlyStopping at epoch X |

### Essential Training Recipe

```python
model.compile(
    optimizer=tf.keras.optimizers.Adam(0.001),
    loss=tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True),
    metrics=['accuracy']
)

history = model.fit(
    x_train, y_train,
    epochs=100,
    batch_size=64,
    validation_data=(x_val, y_val),
    callbacks=[
        tf.keras.callbacks.EarlyStopping(patience=10, restore_best_weights=True),
        tf.keras.callbacks.ModelCheckpoint('best.keras', save_best_only=True),
        tf.keras.callbacks.ReduceLROnPlateau(factor=0.5, patience=5)
    ]
)
```

---

*Previous: [02-Keras-Sequential-and-Functional-API](02-Keras-Sequential-and-Functional-API.md) | Next: [04-Data-Pipeline-tf-data](04-Data-Pipeline-tf-data.md)*
