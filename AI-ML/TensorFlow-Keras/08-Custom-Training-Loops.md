# Chapter 08: Custom Training Loops in TensorFlow

## Table of Contents
- [What are Custom Training Loops?](#what-are-custom-training-loops)
- [Why Custom Training Loops Matter](#why-custom-training-loops-matter)
- [GradientTape — The Foundation](#gradienttape--the-foundation)
- [Building a Custom Training Loop](#building-a-custom-training-loop)
- [Custom Losses](#custom-losses)
- [Custom Metrics](#custom-metrics)
- [Custom Layers](#custom-layers)
- [Model Subclassing](#model-subclassing)
- [Custom Callbacks](#custom-callbacks)
- [Advanced Patterns](#advanced-patterns)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What are Custom Training Loops?

### Simple Explanation

Think of `model.fit()` as an **automatic car** — you just press the gas and it handles gears, engine timing, and fuel injection. A **custom training loop** is like driving a **manual car** — you control every aspect: when to shift gears, how much throttle, when to brake.

In TensorFlow, `model.fit()` handles:
- Forward pass (predictions)
- Loss computation
- Backward pass (gradients)
- Weight updates
- Metric tracking
- Callbacks

A custom training loop gives you explicit control over **every single step**.

### When Would You Need This?

```
model.fit() is sufficient for:            Custom loops needed for:
┌─────────────────────────────┐           ┌─────────────────────────────┐
│ Standard supervised learning │           │ GANs (2 models, alternating)│
│ Simple loss functions        │           │ Reinforcement learning      │
│ Standard optimizers          │           │ Gradient accumulation       │
│ Common metrics               │           │ Custom gradient clipping    │
│ Basic callbacks              │           │ Multi-loss balancing        │
│                              │           │ Research / custom algorithms│
│                              │           │ Adversarial training        │
│                              │           │ Meta-learning (MAML)        │
└─────────────────────────────┘           └─────────────────────────────┘
```

---

## Why Custom Training Loops Matter

### Real-World Scenarios

1. **GANs**: Generator and Discriminator train alternately with different losses
2. **Gradient Accumulation**: Simulate large batch sizes on limited GPU memory
3. **Curriculum Learning**: Change training data difficulty over time
4. **Multi-Task Learning**: Multiple losses with dynamic weighting
5. **Research**: Implement new optimization algorithms from papers
6. **Debugging**: Inspect gradients, activations, and losses step by step

### The Trade-Off

| Aspect | `model.fit()` | Custom Training Loop |
|--------|--------------|---------------------|
| Code complexity | Low | High |
| Flexibility | Limited | Unlimited |
| Debugging | Harder | Easier (explicit) |
| Distributed training | Automatic | Need manual setup |
| Callbacks | Built-in | Must implement |
| Progress bars | Automatic | Must implement |
| Best for | 90% of use cases | Research, GANs, complex training |

---

## GradientTape — The Foundation

### What is GradientTape?

`tf.GradientTape` is TensorFlow's mechanism for **automatic differentiation**. It "records" operations performed on tensors, then computes gradients by replaying the tape backward (like rewinding a cassette tape).

```
Forward Pass (Recording):
┌─────────────────────────────────────────┐
│  x → [Linear] → [ReLU] → [Linear] → y │
│  ─────────TAPE RECORDS ALL OPS──────── │
└─────────────────────────────────────────┘

Backward Pass (Playing backward):
┌─────────────────────────────────────────┐
│  ∂L/∂w ← [∂/∂w] ← [∂/∂a] ← [∂L/∂y]  │
│  ─────────TAPE COMPUTES GRADIENTS───── │
└─────────────────────────────────────────┘
```

### Basic GradientTape Usage

```python
import tensorflow as tf

# ============================================
# Example 1: Simple gradient computation
# ============================================
x = tf.Variable(3.0)  # Must be tf.Variable (or watched tf.Tensor)

with tf.GradientTape() as tape:
    y = x ** 2  # y = x²

# Compute dy/dx
dy_dx = tape.gradient(y, x)
print(f"dy/dx at x=3: {dy_dx.numpy()}")  # 6.0 (because d(x²)/dx = 2x)

# ============================================
# Example 2: Gradients with respect to multiple variables
# ============================================
w = tf.Variable(2.0)
b = tf.Variable(1.0)
x_input = tf.constant(3.0)

with tf.GradientTape() as tape:
    y = w * x_input + b      # y = 2*3 + 1 = 7
    loss = (y - 5.0) ** 2    # loss = (7-5)² = 4

# Compute gradients of loss w.r.t. w and b
gradients = tape.gradient(loss, [w, b])
print(f"dL/dw: {gradients[0].numpy()}")  # 12.0
print(f"dL/db: {gradients[1].numpy()}")  # 4.0

# ============================================
# Example 3: Watching non-Variable tensors
# ============================================
x = tf.constant(3.0)  # Not a Variable!

with tf.GradientTape() as tape:
    tape.watch(x)      # Explicitly tell tape to track this tensor
    y = x ** 2

dy_dx = tape.gradient(y, x)
print(f"dy/dx: {dy_dx.numpy()}")  # 6.0
```

### Persistent Tapes

```python
# By default, tape is consumed after ONE gradient call
# Use persistent=True for multiple gradient computations

x = tf.Variable(3.0)

with tf.GradientTape(persistent=True) as tape:
    y = x ** 2
    z = x ** 3

dy_dx = tape.gradient(y, x)  # 6.0
dz_dx = tape.gradient(z, x)  # 27.0 (3x² at x=3)

print(f"dy/dx: {dy_dx.numpy()}")
print(f"dz/dx: {dz_dx.numpy()}")

del tape  # IMPORTANT: Delete persistent tape to free resources!
```

### Higher-Order Gradients

```python
# Computing second derivatives (Hessian)
x = tf.Variable(3.0)

with tf.GradientTape() as t2:  # Outer tape
    with tf.GradientTape() as t1:  # Inner tape
        y = x ** 3  # y = x³
    
    # First derivative: dy/dx = 3x²
    dy_dx = t1.gradient(y, x)

# Second derivative: d²y/dx² = 6x
d2y_dx2 = t2.gradient(dy_dx, x)

print(f"dy/dx at x=3: {dy_dx.numpy()}")    # 27.0 (3 * 9)
print(f"d²y/dx² at x=3: {d2y_dx2.numpy()}")  # 18.0 (6 * 3)
```

> **Pro Tip**: Nested tapes are essential for implementing algorithms like MAML (Model-Agnostic Meta-Learning) which require gradients of gradients.

---

## Building a Custom Training Loop

### The Complete Pattern

```python
import tensorflow as tf
import numpy as np

# ============================================
# STEP 1: Create model, optimizer, loss, metrics
# ============================================

# Simple model
model = tf.keras.Sequential([
    tf.keras.layers.Dense(128, activation='relu', input_shape=(784,)),
    tf.keras.layers.Dropout(0.3),
    tf.keras.layers.Dense(64, activation='relu'),
    tf.keras.layers.Dropout(0.3),
    tf.keras.layers.Dense(10, activation='softmax')
])

# Optimizer
optimizer = tf.keras.optimizers.Adam(learning_rate=1e-3)

# Loss function
loss_fn = tf.keras.losses.SparseCategoricalCrossentropy()

# Metrics
train_loss = tf.keras.metrics.Mean(name='train_loss')
train_accuracy = tf.keras.metrics.SparseCategoricalAccuracy(name='train_accuracy')
val_loss = tf.keras.metrics.Mean(name='val_loss')
val_accuracy = tf.keras.metrics.SparseCategoricalAccuracy(name='val_accuracy')

# ============================================
# STEP 2: Define training and validation steps
# ============================================

@tf.function  # Compile to graph for speed!
def train_step(x_batch, y_batch):
    """One training step: forward pass → loss → gradients → update."""
    with tf.GradientTape() as tape:
        # Forward pass
        predictions = model(x_batch, training=True)  # training=True for Dropout/BN
        
        # Compute loss
        loss = loss_fn(y_batch, predictions)
    
    # Compute gradients
    gradients = tape.gradient(loss, model.trainable_variables)
    
    # Update weights
    optimizer.apply_gradients(zip(gradients, model.trainable_variables))
    
    # Update metrics
    train_loss.update_state(loss)
    train_accuracy.update_state(y_batch, predictions)

@tf.function
def val_step(x_batch, y_batch):
    """One validation step: forward pass → loss (no gradient update)."""
    predictions = model(x_batch, training=False)  # training=False!
    loss = loss_fn(y_batch, predictions)
    
    val_loss.update_state(loss)
    val_accuracy.update_state(y_batch, predictions)

# ============================================
# STEP 3: Create datasets
# ============================================
(x_train, y_train), (x_test, y_test) = tf.keras.datasets.mnist.load_data()
x_train = x_train.reshape(-1, 784).astype('float32') / 255.0
x_test = x_test.reshape(-1, 784).astype('float32') / 255.0

BATCH_SIZE = 64
train_dataset = tf.data.Dataset.from_tensor_slices((x_train, y_train))
train_dataset = train_dataset.shuffle(10000).batch(BATCH_SIZE).prefetch(tf.data.AUTOTUNE)

val_dataset = tf.data.Dataset.from_tensor_slices((x_test, y_test))
val_dataset = val_dataset.batch(BATCH_SIZE).prefetch(tf.data.AUTOTUNE)

# ============================================
# STEP 4: The training loop
# ============================================
EPOCHS = 10

for epoch in range(EPOCHS):
    # Reset metrics at the start of each epoch
    train_loss.reset_state()
    train_accuracy.reset_state()
    val_loss.reset_state()
    val_accuracy.reset_state()
    
    # Training
    for x_batch, y_batch in train_dataset:
        train_step(x_batch, y_batch)
    
    # Validation
    for x_batch, y_batch in val_dataset:
        val_step(x_batch, y_batch)
    
    # Print results
    print(
        f"Epoch {epoch + 1}/{EPOCHS} — "
        f"Loss: {train_loss.result():.4f}, "
        f"Accuracy: {train_accuracy.result():.4f}, "
        f"Val Loss: {val_loss.result():.4f}, "
        f"Val Accuracy: {val_accuracy.result():.4f}"
    )
```

### Comparison: model.fit() vs Custom Loop

```python
# ============================================
# WITH model.fit() (3 lines)
# ============================================
model.compile(optimizer='adam', loss='sparse_categorical_crossentropy', metrics=['accuracy'])
model.fit(x_train, y_train, epochs=10, validation_data=(x_test, y_test), batch_size=64)

# ============================================
# WITH custom loop (40+ lines)
# ============================================
# (see above)
# But you get FULL CONTROL over every step!
```

---

## Custom Losses

### Why Custom Losses?

Standard losses (`MSE`, `CrossEntropy`) don't cover every scenario. You might need:
- **Focal Loss**: For highly imbalanced datasets
- **Contrastive Loss**: For similarity learning
- **Dice Loss**: For image segmentation
- **Triplet Loss**: For face recognition
- **Perceptual Loss**: For image super-resolution

### Method 1: Simple Function

```python
import tensorflow as tf

# ============================================
# Custom Huber Loss (function-based)
# ============================================
def custom_huber_loss(y_true, y_pred, delta=1.0):
    """
    Huber Loss: combines MSE (for small errors) and MAE (for large errors).
    
    L = { 0.5 * (y - ŷ)²           if |y - ŷ| ≤ δ
        { δ * |y - ŷ| - 0.5 * δ²   if |y - ŷ| > δ
    """
    error = y_true - y_pred
    is_small_error = tf.abs(error) <= delta
    small_error_loss = 0.5 * tf.square(error)
    large_error_loss = delta * tf.abs(error) - 0.5 * delta ** 2
    return tf.where(is_small_error, small_error_loss, large_error_loss)

# Usage
model.compile(optimizer='adam', loss=custom_huber_loss)
```

### Method 2: Subclassing tf.keras.losses.Loss

```python
class FocalLoss(tf.keras.losses.Loss):
    """
    Focal Loss for handling class imbalance.
    
    FL(p) = -α(1-p)^γ * log(p)
    
    When a sample is misclassified (p is small), (1-p)^γ ≈ 1, loss is normal.
    When a sample is well-classified (p is large), (1-p)^γ ≈ 0, loss is DOWN-weighted.
    
    This makes the model focus on HARD examples.
    
    Analogy: Imagine a teacher giving harder problems to students who've already 
    mastered the easy ones. Easy questions get less weight.
    """
    def __init__(self, gamma=2.0, alpha=0.25, **kwargs):
        super().__init__(**kwargs)
        self.gamma = gamma   # Focusing parameter (higher = more focus on hard examples)
        self.alpha = alpha   # Balancing factor
    
    def call(self, y_true, y_pred):
        # Clip predictions to prevent log(0)
        y_pred = tf.clip_by_value(y_pred, 1e-7, 1 - 1e-7)
        
        # Compute cross-entropy
        cross_entropy = -y_true * tf.math.log(y_pred)
        
        # Compute focal weight: (1 - p)^gamma
        focal_weight = tf.pow(1 - y_pred, self.gamma)
        
        # Apply focal weight and alpha balancing
        focal_loss = self.alpha * focal_weight * cross_entropy
        
        return tf.reduce_sum(focal_loss, axis=-1)
    
    def get_config(self):
        config = super().get_config()
        config.update({'gamma': self.gamma, 'alpha': self.alpha})
        return config

# Usage
model.compile(optimizer='adam', loss=FocalLoss(gamma=2.0, alpha=0.25))
```

### Method 3: Multi-Component Loss

```python
class CombinedLoss(tf.keras.losses.Loss):
    """
    Combines multiple loss functions with weights.
    Useful for multi-task learning or regularized objectives.
    """
    def __init__(self, reconstruction_weight=1.0, kl_weight=0.001, **kwargs):
        super().__init__(**kwargs)
        self.reconstruction_weight = reconstruction_weight
        self.kl_weight = kl_weight
    
    def call(self, y_true, y_pred):
        # Assume y_pred contains [reconstruction, z_mean, z_log_var]
        reconstruction = y_pred[:, :784]
        z_mean = y_pred[:, 784:784+64]
        z_log_var = y_pred[:, 784+64:]
        
        # Reconstruction loss
        reconstruction_loss = tf.reduce_mean(
            tf.keras.losses.binary_crossentropy(y_true, reconstruction)
        )
        
        # KL divergence loss
        kl_loss = -0.5 * tf.reduce_mean(
            1 + z_log_var - tf.square(z_mean) - tf.exp(z_log_var)
        )
        
        return (self.reconstruction_weight * reconstruction_loss + 
                self.kl_weight * kl_loss)
```

### Dice Loss (for Segmentation)

```python
def dice_loss(y_true, y_pred, smooth=1e-6):
    """
    Dice Loss: measures overlap between prediction and ground truth.
    
    Dice = 2 * |A ∩ B| / (|A| + |B|)
    
    Perfect overlap → Dice = 1 → Loss = 0
    No overlap → Dice = 0 → Loss = 1
    
    Great for imbalanced segmentation (e.g., small tumor in large image).
    """
    y_true_flat = tf.reshape(y_true, [-1])
    y_pred_flat = tf.reshape(y_pred, [-1])
    
    intersection = tf.reduce_sum(y_true_flat * y_pred_flat)
    union = tf.reduce_sum(y_true_flat) + tf.reduce_sum(y_pred_flat)
    
    dice = (2.0 * intersection + smooth) / (union + smooth)
    return 1.0 - dice

# Often combined with BCE for stability
def bce_dice_loss(y_true, y_pred):
    bce = tf.keras.losses.binary_crossentropy(y_true, y_pred)
    dice = dice_loss(y_true, y_pred)
    return bce + dice
```

### Using Custom Loss in a Custom Training Loop

```python
@tf.function
def train_step_custom_loss(x_batch, y_batch):
    with tf.GradientTape() as tape:
        predictions = model(x_batch, training=True)
        
        # Custom loss with multiple components
        ce_loss = tf.keras.losses.categorical_crossentropy(y_batch, predictions)
        
        # L2 regularization (weight decay)
        l2_loss = tf.add_n([
            tf.nn.l2_loss(v) for v in model.trainable_variables
            if 'kernel' in v.name  # Only regularize weight matrices, not biases
        ]) * 1e-4
        
        total_loss = tf.reduce_mean(ce_loss) + l2_loss
    
    gradients = tape.gradient(total_loss, model.trainable_variables)
    optimizer.apply_gradients(zip(gradients, model.trainable_variables))
    
    return total_loss
```

---

## Custom Metrics

### Why Custom Metrics?

Built-in metrics (`accuracy`, `AUC`) cover common cases, but you may need:
- **F1 Score** (per-batch or epoch-level)
- **IoU** (Intersection over Union for segmentation)
- **BLEU Score** (for translation)
- **Custom business metrics** (e.g., "revenue-weighted accuracy")

### Building a Custom Metric

```python
class F1Score(tf.keras.metrics.Metric):
    """
    F1 Score = 2 * (Precision * Recall) / (Precision + Recall)
    
    Harmonic mean of precision and recall.
    Unlike accuracy, handles class imbalance well.
    """
    def __init__(self, name='f1_score', threshold=0.5, **kwargs):
        super().__init__(name=name, **kwargs)
        self.threshold = threshold
        # State variables (accumulated across batches)
        self.true_positives = self.add_weight(name='tp', initializer='zeros')
        self.false_positives = self.add_weight(name='fp', initializer='zeros')
        self.false_negatives = self.add_weight(name='fn', initializer='zeros')
    
    def update_state(self, y_true, y_pred, sample_weight=None):
        """Called on each batch to accumulate statistics."""
        y_pred = tf.cast(y_pred >= self.threshold, tf.float32)
        y_true = tf.cast(y_true, tf.float32)
        
        tp = tf.reduce_sum(y_true * y_pred)
        fp = tf.reduce_sum((1 - y_true) * y_pred)
        fn = tf.reduce_sum(y_true * (1 - y_pred))
        
        self.true_positives.assign_add(tp)
        self.false_positives.assign_add(fp)
        self.false_negatives.assign_add(fn)
    
    def result(self):
        """Called at the end of each epoch to compute the final metric."""
        precision = self.true_positives / (self.true_positives + self.false_positives + 1e-7)
        recall = self.true_positives / (self.true_positives + self.false_negatives + 1e-7)
        f1 = 2 * (precision * recall) / (precision + recall + 1e-7)
        return f1
    
    def reset_state(self):
        """Called at the start of each epoch to reset accumulated values."""
        self.true_positives.assign(0)
        self.false_positives.assign(0)
        self.false_negatives.assign(0)

# Usage with model.compile()
model.compile(
    optimizer='adam',
    loss='binary_crossentropy',
    metrics=['accuracy', F1Score()]
)

# Usage in custom training loop
f1_metric = F1Score()

for x_batch, y_batch in train_dataset:
    preds = model(x_batch, training=False)
    f1_metric.update_state(y_batch, preds)

print(f"F1 Score: {f1_metric.result().numpy():.4f}")
f1_metric.reset_state()
```

### IoU Metric (Segmentation)

```python
class MeanIoU(tf.keras.metrics.Metric):
    """
    Mean Intersection over Union for semantic segmentation.
    
    IoU = Area of Overlap / Area of Union
    
    Imagine two overlapping circles:
    ┌─────────────┐
    │   A     ┌───┼──────┐
    │         │///│      │  ← /// is the overlap (intersection)
    │         └───┼──────┘
    └─────────────┘   B
    
    IoU = /// / (A + B - ///)
    """
    def __init__(self, num_classes, name='mean_iou', **kwargs):
        super().__init__(name=name, **kwargs)
        self.num_classes = num_classes
        self.confusion_matrix = self.add_weight(
            name='confusion_matrix',
            shape=(num_classes, num_classes),
            initializer='zeros'
        )
    
    def update_state(self, y_true, y_pred, sample_weight=None):
        y_pred = tf.argmax(y_pred, axis=-1)
        y_true = tf.cast(tf.reshape(y_true, [-1]), tf.int32)
        y_pred = tf.cast(tf.reshape(y_pred, [-1]), tf.int32)
        
        cm = tf.math.confusion_matrix(
            y_true, y_pred, num_classes=self.num_classes, dtype=tf.float32
        )
        self.confusion_matrix.assign_add(cm)
    
    def result(self):
        # IoU for each class: TP / (TP + FP + FN)
        sum_over_row = tf.reduce_sum(self.confusion_matrix, axis=0)   # FP + TP
        sum_over_col = tf.reduce_sum(self.confusion_matrix, axis=1)   # FN + TP
        diag = tf.linalg.diag_part(self.confusion_matrix)            # TP
        
        denominator = sum_over_row + sum_over_col - diag
        iou = diag / (denominator + 1e-7)
        
        return tf.reduce_mean(iou)
    
    def reset_state(self):
        self.confusion_matrix.assign(tf.zeros_like(self.confusion_matrix))
```

---

## Custom Layers

### Why Custom Layers?

When built-in layers don't provide the operation you need:
- **Squeeze-and-Excitation blocks** (channel attention)
- **Custom normalization** schemes
- **Domain-specific operations** (e.g., geometric transformations)

### Building a Custom Layer

```python
class CustomDense(tf.keras.layers.Layer):
    """
    A Dense layer built from scratch to understand Layer internals.
    
    y = activation(x @ W + b)
    """
    def __init__(self, units, activation=None, **kwargs):
        super().__init__(**kwargs)
        self.units = units
        self.activation = tf.keras.activations.get(activation)
    
    def build(self, input_shape):
        """
        Called ONCE when the layer first receives input.
        This is where you create weights.
        
        Why build() and not __init__()?
        Because we don't know input_shape until the first forward pass.
        This enables lazy weight creation.
        """
        self.w = self.add_weight(
            name='kernel',
            shape=(input_shape[-1], self.units),
            initializer='glorot_uniform',  # Xavier initialization
            trainable=True
        )
        self.b = self.add_weight(
            name='bias',
            shape=(self.units,),
            initializer='zeros',
            trainable=True
        )
        super().build(input_shape)  # Mark layer as built
    
    def call(self, inputs, training=None):
        """
        Forward pass. Called every time the layer processes data.
        
        Args:
            inputs: Input tensor
            training: Boolean flag (True during training, False during inference)
        """
        output = tf.matmul(inputs, self.w) + self.b
        if self.activation is not None:
            output = self.activation(output)
        return output
    
    def get_config(self):
        """Required for serialization (model saving/loading)."""
        config = super().get_config()
        config.update({
            'units': self.units,
            'activation': tf.keras.activations.serialize(self.activation)
        })
        return config

# Usage
layer = CustomDense(64, activation='relu')
x = tf.random.normal((32, 128))  # batch of 32, 128 features
output = layer(x)
print(f"Output shape: {output.shape}")  # (32, 64)
```

### Squeeze-and-Excitation Block

```python
class SqueezeExcitation(tf.keras.layers.Layer):
    """
    Squeeze-and-Excitation block: learns channel-wise attention.
    
    Intuition: Not all feature channels are equally important.
    This block learns to AMPLIFY useful channels and SUPPRESS less useful ones.
    
    Architecture:
    Input: (batch, H, W, C)
        │
        ▼
    Global Average Pooling → (batch, C)     ← "Squeeze": summarize each channel
        │
        ▼
    Dense(C/r, ReLU) → (batch, C/r)        ← "Excitation": learn relationships
        │
        ▼
    Dense(C, Sigmoid) → (batch, C)          ← Channel attention weights [0, 1]
        │
        ▼
    Scale original input × attention        ← Re-weight channels
        │
        ▼
    Output: (batch, H, W, C)
    """
    def __init__(self, reduction_ratio=16, **kwargs):
        super().__init__(**kwargs)
        self.reduction_ratio = reduction_ratio
    
    def build(self, input_shape):
        channels = input_shape[-1]
        reduced = max(1, channels // self.reduction_ratio)
        
        self.global_pool = tf.keras.layers.GlobalAveragePooling2D()
        self.dense1 = tf.keras.layers.Dense(reduced, activation='relu')
        self.dense2 = tf.keras.layers.Dense(channels, activation='sigmoid')
        self.reshape = tf.keras.layers.Reshape((1, 1, channels))
    
    def call(self, inputs):
        # Squeeze: (batch, H, W, C) → (batch, C)
        se = self.global_pool(inputs)
        
        # Excitation: (batch, C) → (batch, C/r) → (batch, C)
        se = self.dense1(se)
        se = self.dense2(se)
        
        # Reshape: (batch, C) → (batch, 1, 1, C) for broadcasting
        se = self.reshape(se)
        
        # Scale: element-wise multiplication
        return inputs * se  # Broadcasting: (batch, H, W, C) * (batch, 1, 1, C)

# Usage in a model
model = tf.keras.Sequential([
    tf.keras.layers.Conv2D(64, 3, padding='same', activation='relu', input_shape=(28, 28, 1)),
    SqueezeExcitation(reduction_ratio=16),  # Channel attention after conv!
    tf.keras.layers.Conv2D(128, 3, padding='same', activation='relu'),
    SqueezeExcitation(reduction_ratio=16),
    tf.keras.layers.GlobalAveragePooling2D(),
    tf.keras.layers.Dense(10, activation='softmax')
])
```

### Layer with Different Training/Inference Behavior

```python
class MonteCarloDropout(tf.keras.layers.Layer):
    """
    Dropout that stays active during inference too.
    Used for uncertainty estimation (MC Dropout).
    
    Regular Dropout: OFF during inference (deterministic predictions)
    MC Dropout: ON during inference (stochastic predictions → uncertainty)
    """
    def __init__(self, rate=0.5, **kwargs):
        super().__init__(**kwargs)
        self.rate = rate
    
    def call(self, inputs, training=None):
        # ALWAYS apply dropout, regardless of training flag
        return tf.nn.dropout(inputs, rate=self.rate)

# Uncertainty estimation with MC Dropout
def predict_with_uncertainty(model, x, n_samples=100):
    """
    Make multiple predictions with dropout active.
    The variance of predictions indicates uncertainty.
    """
    predictions = tf.stack([model(x, training=True) for _ in range(n_samples)])
    mean = tf.reduce_mean(predictions, axis=0)
    variance = tf.math.reduce_variance(predictions, axis=0)
    return mean, variance

# mean_pred, uncertainty = predict_with_uncertainty(model, test_image, n_samples=100)
# High variance → model is uncertain → don't trust this prediction!
```

---

## Model Subclassing

### What is Model Subclassing?

The most flexible way to define models in TensorFlow. You define the model as a Python class, giving you full control over the forward pass.

```
Three ways to build models (flexibility ↑, simplicity ↓):

1. Sequential API    → Simple stack of layers
2. Functional API    → DAG (directed acyclic graph) of layers  
3. Model Subclassing → Full Python class with custom forward pass
```

### Basic Subclassed Model

```python
class MyModel(tf.keras.Model):
    """
    A simple feedforward network using model subclassing.
    """
    def __init__(self, num_classes=10):
        super().__init__()
        
        # Define layers in __init__
        self.dense1 = tf.keras.layers.Dense(256, activation='relu')
        self.bn1 = tf.keras.layers.BatchNormalization()
        self.dropout1 = tf.keras.layers.Dropout(0.3)
        
        self.dense2 = tf.keras.layers.Dense(128, activation='relu')
        self.bn2 = tf.keras.layers.BatchNormalization()
        self.dropout2 = tf.keras.layers.Dropout(0.3)
        
        self.classifier = tf.keras.layers.Dense(num_classes, activation='softmax')
    
    def call(self, inputs, training=False):
        """
        Define the forward pass.
        
        Args:
            inputs: Input tensor
            training: Boolean — controls Dropout and BatchNorm behavior
        """
        x = self.dense1(inputs)
        x = self.bn1(x, training=training)     # BN needs training flag
        x = self.dropout1(x, training=training) # Dropout needs training flag
        
        x = self.dense2(x)
        x = self.bn2(x, training=training)
        x = self.dropout2(x, training=training)
        
        return self.classifier(x)

# Usage
model = MyModel(num_classes=10)

# Build the model (creates weights)
model.build(input_shape=(None, 784))
model.summary()

# Can use with model.fit()
model.compile(optimizer='adam', loss='sparse_categorical_crossentropy', metrics=['accuracy'])
# model.fit(x_train, y_train, epochs=10)

# Or with custom training loop
# (same as before)
```

### CNN with Residual Connections

```python
class ResidualBlock(tf.keras.layers.Layer):
    """
    Residual Block: output = F(x) + x
    
    The skip connection allows gradients to flow directly backward,
    solving the vanishing gradient problem in deep networks.
    
    ┌──────────────────────────────────────┐
    │ Input x                              │
    │   │                                  │
    │   ├──────────────────────┐           │
    │   │                      │ (skip)    │
    │   ▼                      │           │
    │ Conv2D → BN → ReLU       │           │
    │   │                      │           │
    │   ▼                      │           │
    │ Conv2D → BN              │           │
    │   │                      │           │
    │   ▼                      ▼           │
    │   ┌──── ADD ◄────────────┘           │
    │   │                                  │
    │   ▼                                  │
    │ ReLU → Output                        │
    └──────────────────────────────────────┘
    """
    def __init__(self, filters, strides=1, **kwargs):
        super().__init__(**kwargs)
        self.filters = filters
        self.strides = strides
    
    def build(self, input_shape):
        self.conv1 = tf.keras.layers.Conv2D(
            self.filters, 3, strides=self.strides, padding='same'
        )
        self.bn1 = tf.keras.layers.BatchNormalization()
        
        self.conv2 = tf.keras.layers.Conv2D(
            self.filters, 3, strides=1, padding='same'
        )
        self.bn2 = tf.keras.layers.BatchNormalization()
        
        # If dimensions don't match, use 1x1 conv to project
        if self.strides != 1 or input_shape[-1] != self.filters:
            self.shortcut = tf.keras.Sequential([
                tf.keras.layers.Conv2D(self.filters, 1, strides=self.strides),
                tf.keras.layers.BatchNormalization()
            ])
        else:
            self.shortcut = lambda x: x  # Identity
    
    def call(self, inputs, training=False):
        # Main path
        x = self.conv1(inputs)
        x = self.bn1(x, training=training)
        x = tf.nn.relu(x)
        
        x = self.conv2(x)
        x = self.bn2(x, training=training)
        
        # Skip connection
        shortcut = self.shortcut(inputs)
        
        # Add and activate
        return tf.nn.relu(x + shortcut)


class CustomResNet(tf.keras.Model):
    """A simple ResNet-like model."""
    def __init__(self, num_classes=10):
        super().__init__()
        
        self.conv_initial = tf.keras.layers.Conv2D(
            64, 7, strides=2, padding='same'
        )
        self.bn_initial = tf.keras.layers.BatchNormalization()
        self.pool = tf.keras.layers.MaxPooling2D(3, strides=2, padding='same')
        
        # Residual blocks
        self.res_block1 = ResidualBlock(64)
        self.res_block2 = ResidualBlock(64)
        self.res_block3 = ResidualBlock(128, strides=2)
        self.res_block4 = ResidualBlock(128)
        self.res_block5 = ResidualBlock(256, strides=2)
        self.res_block6 = ResidualBlock(256)
        
        self.global_pool = tf.keras.layers.GlobalAveragePooling2D()
        self.classifier = tf.keras.layers.Dense(num_classes, activation='softmax')
    
    def call(self, inputs, training=False):
        x = self.conv_initial(inputs)
        x = self.bn_initial(x, training=training)
        x = tf.nn.relu(x)
        x = self.pool(x)
        
        x = self.res_block1(x, training=training)
        x = self.res_block2(x, training=training)
        x = self.res_block3(x, training=training)
        x = self.res_block4(x, training=training)
        x = self.res_block5(x, training=training)
        x = self.res_block6(x, training=training)
        
        x = self.global_pool(x)
        return self.classifier(x)

# model = CustomResNet(num_classes=10)
# model.build((None, 224, 224, 3))
# model.summary()
```

---

## Custom Callbacks

### Building Custom Callbacks

```python
class TrainingMonitor(tf.keras.callbacks.Callback):
    """
    Custom callback that monitors training and implements:
    1. Gradient norm logging
    2. Learning rate scheduling
    3. Early stopping with custom logic
    """
    def __init__(self, patience=5, min_delta=0.001):
        super().__init__()
        self.patience = patience
        self.min_delta = min_delta
        self.best_loss = float('inf')
        self.wait = 0
    
    def on_train_begin(self, logs=None):
        """Called at the beginning of training."""
        print("Training started!")
        self.train_losses = []
        self.val_losses = []
    
    def on_epoch_begin(self, epoch, logs=None):
        """Called at the beginning of each epoch."""
        self.epoch_start_time = tf.timestamp()
    
    def on_epoch_end(self, epoch, logs=None):
        """Called at the end of each epoch."""
        elapsed = tf.timestamp() - self.epoch_start_time
        
        train_loss = logs.get('loss', 0)
        val_loss = logs.get('val_loss', 0)
        
        self.train_losses.append(train_loss)
        self.val_losses.append(val_loss)
        
        # Custom early stopping logic
        if val_loss < self.best_loss - self.min_delta:
            self.best_loss = val_loss
            self.wait = 0
            print(f"  ↓ New best val_loss: {val_loss:.4f}")
        else:
            self.wait += 1
            if self.wait >= self.patience:
                self.model.stop_training = True
                print(f"  ✗ Early stopping at epoch {epoch + 1}")
        
        # Log epoch time
        print(f"  Epoch time: {elapsed:.1f}s")
    
    def on_train_batch_end(self, batch, logs=None):
        """Called at the end of each training batch."""
        # Log every 100 batches
        if batch % 100 == 0:
            loss = logs.get('loss', 0)
            print(f"  Batch {batch}: loss = {loss:.4f}")
    
    def on_train_end(self, logs=None):
        """Called at the end of training."""
        print(f"\nTraining complete! Best val_loss: {self.best_loss:.4f}")


class GradientLogger(tf.keras.callbacks.Callback):
    """Logs gradient statistics for debugging."""
    
    def on_train_batch_end(self, batch, logs=None):
        if batch % 50 != 0:
            return
        
        # Access gradient norms from the optimizer
        for var in self.model.trainable_variables:
            if 'kernel' in var.name:
                grad_norm = tf.norm(var).numpy()
                if grad_norm > 100:
                    print(f"  ⚠ Large weights in {var.name}: norm={grad_norm:.2f}")
                elif grad_norm < 1e-7:
                    print(f"  ⚠ Vanishing weights in {var.name}: norm={grad_norm:.10f}")


# Usage
# model.fit(
#     train_ds, epochs=50, validation_data=val_ds,
#     callbacks=[TrainingMonitor(patience=5), GradientLogger()]
# )
```

### Cosine Annealing with Warm Restarts Callback

```python
class CosineAnnealingWarmRestarts(tf.keras.callbacks.Callback):
    """
    Cosine annealing with warm restarts.
    
    LR follows a cosine curve, then "restarts" to a high value.
    This helps the model escape local minima.
    
    LR
    │\    /\    /\
    │ \  /  \  /  \
    │  \/    \/    \/
    └──────────────── epoch
    """
    def __init__(self, max_lr=1e-3, min_lr=1e-6, cycle_length=10, mult_factor=2):
        super().__init__()
        self.max_lr = max_lr
        self.min_lr = min_lr
        self.cycle_length = cycle_length
        self.mult_factor = mult_factor
        self.current_cycle = 0
        self.cycle_epoch = 0
    
    def on_epoch_begin(self, epoch, logs=None):
        # Compute LR using cosine annealing
        cycle_len = self.cycle_length * (self.mult_factor ** self.current_cycle)
        
        cos_value = np.cos(np.pi * self.cycle_epoch / cycle_len)
        lr = self.min_lr + 0.5 * (self.max_lr - self.min_lr) * (1 + cos_value)
        
        tf.keras.backend.set_value(self.model.optimizer.learning_rate, lr)
        
        self.cycle_epoch += 1
        if self.cycle_epoch >= cycle_len:
            self.current_cycle += 1
            self.cycle_epoch = 0
```

---

## Advanced Patterns

### 1. Gradient Accumulation

```python
"""
Gradient Accumulation: Simulate large batch sizes on limited GPU memory.

Problem: You want batch_size=256 but only have GPU memory for batch_size=32.
Solution: Accumulate gradients over 8 mini-batches, then apply once.

Equivalent batch size = mini_batch_size × accumulation_steps
256 = 32 × 8
"""

@tf.function
def train_step_with_accumulation(dataset_iterator, accumulation_steps=8):
    # Initialize gradient accumulators with zeros
    accumulated_gradients = [
        tf.zeros_like(var) for var in model.trainable_variables
    ]
    
    total_loss = 0.0
    
    for step in tf.range(accumulation_steps):
        x_batch, y_batch = next(dataset_iterator)
        
        with tf.GradientTape() as tape:
            predictions = model(x_batch, training=True)
            loss = loss_fn(y_batch, predictions) / tf.cast(accumulation_steps, tf.float32)
        
        # Compute gradients
        gradients = tape.gradient(loss, model.trainable_variables)
        
        # Accumulate (add to running total)
        accumulated_gradients = [
            accum + grad 
            for accum, grad in zip(accumulated_gradients, gradients)
        ]
        
        total_loss += loss
    
    # Apply accumulated gradients (ONE optimizer step for N mini-batches)
    optimizer.apply_gradients(
        zip(accumulated_gradients, model.trainable_variables)
    )
    
    return total_loss
```

### 2. Gradient Clipping

```python
@tf.function
def train_step_with_clipping(x_batch, y_batch, max_grad_norm=1.0):
    """
    Gradient clipping prevents exploding gradients.
    
    Without clipping: gradient = [1000, -500, 2000] → weights explode!
    With clipping:    gradient = [0.5, -0.25, 1.0]  → stable updates
    """
    with tf.GradientTape() as tape:
        predictions = model(x_batch, training=True)
        loss = loss_fn(y_batch, predictions)
    
    gradients = tape.gradient(loss, model.trainable_variables)
    
    # Method 1: Clip by global norm (RECOMMENDED)
    # Scales ALL gradients proportionally if total norm exceeds max_grad_norm
    clipped_gradients, global_norm = tf.clip_by_global_norm(gradients, max_grad_norm)
    
    # Method 2: Clip by value (clips each element independently)
    # clipped_gradients = [tf.clip_by_value(g, -1.0, 1.0) for g in gradients]
    
    # Method 3: Clip by norm (clips each tensor's norm independently)
    # clipped_gradients = [tf.clip_by_norm(g, max_grad_norm) for g in gradients]
    
    optimizer.apply_gradients(zip(clipped_gradients, model.trainable_variables))
    
    return loss, global_norm
```

### 3. GAN Training Loop

```python
class SimpleGAN:
    """
    GAN Custom Training Loop.
    
    GANs need custom loops because:
    1. Two models train alternately
    2. They have DIFFERENT loss functions
    3. One model's output feeds into the other
    
    Generator: Noise → Fake Image (tries to fool discriminator)
    Discriminator: Image → Real/Fake (tries to catch fakes)
    """
    def __init__(self, latent_dim=100):
        self.latent_dim = latent_dim
        self.generator = self._build_generator()
        self.discriminator = self._build_discriminator()
        
        self.g_optimizer = tf.keras.optimizers.Adam(1e-4)
        self.d_optimizer = tf.keras.optimizers.Adam(1e-4)
        self.loss_fn = tf.keras.losses.BinaryCrossentropy()
    
    def _build_generator(self):
        return tf.keras.Sequential([
            tf.keras.layers.Dense(256, activation='relu', input_shape=(self.latent_dim,)),
            tf.keras.layers.BatchNormalization(),
            tf.keras.layers.Dense(512, activation='relu'),
            tf.keras.layers.BatchNormalization(),
            tf.keras.layers.Dense(784, activation='sigmoid'),
            tf.keras.layers.Reshape((28, 28, 1))
        ])
    
    def _build_discriminator(self):
        return tf.keras.Sequential([
            tf.keras.layers.Flatten(input_shape=(28, 28, 1)),
            tf.keras.layers.Dense(512, activation='relu'),
            tf.keras.layers.Dropout(0.3),
            tf.keras.layers.Dense(256, activation='relu'),
            tf.keras.layers.Dropout(0.3),
            tf.keras.layers.Dense(1, activation='sigmoid')
        ])
    
    @tf.function
    def train_step(self, real_images, batch_size):
        # ========== TRAIN DISCRIMINATOR ==========
        noise = tf.random.normal([batch_size, self.latent_dim])
        
        with tf.GradientTape() as d_tape:
            # Generate fake images
            fake_images = self.generator(noise, training=True)
            
            # Discriminator predictions
            real_output = self.discriminator(real_images, training=True)
            fake_output = self.discriminator(fake_images, training=True)
            
            # Discriminator loss: classify real as 1, fake as 0
            d_loss_real = self.loss_fn(tf.ones_like(real_output), real_output)
            d_loss_fake = self.loss_fn(tf.zeros_like(fake_output), fake_output)
            d_loss = d_loss_real + d_loss_fake
        
        d_gradients = d_tape.gradient(d_loss, self.discriminator.trainable_variables)
        self.d_optimizer.apply_gradients(
            zip(d_gradients, self.discriminator.trainable_variables)
        )
        
        # ========== TRAIN GENERATOR ==========
        noise = tf.random.normal([batch_size, self.latent_dim])
        
        with tf.GradientTape() as g_tape:
            fake_images = self.generator(noise, training=True)
            fake_output = self.discriminator(fake_images, training=True)
            
            # Generator loss: wants discriminator to classify fakes as REAL (1)
            g_loss = self.loss_fn(tf.ones_like(fake_output), fake_output)
        
        g_gradients = g_tape.gradient(g_loss, self.generator.trainable_variables)
        self.g_optimizer.apply_gradients(
            zip(g_gradients, self.generator.trainable_variables)
        )
        
        return d_loss, g_loss
    
    def train(self, dataset, epochs):
        for epoch in range(epochs):
            for real_images in dataset:
                batch_size = tf.shape(real_images)[0]
                d_loss, g_loss = self.train_step(real_images, batch_size)
            
            print(f"Epoch {epoch+1}: D_loss={d_loss:.4f}, G_loss={g_loss:.4f}")
```

### 4. Mixed Precision Training

```python
"""
Mixed Precision: Use float16 for speed, float32 for stability.

float32: Standard precision (32 bits per number)
float16: Half precision (16 bits) → 2x less memory, faster compute
mixed:   Use float16 for forward/backward, float32 for weight updates

Modern GPUs (V100, A100, T4) have Tensor Cores optimized for float16.
Mixed precision can give 2-3x speedup with minimal accuracy loss.
"""

# Enable mixed precision globally
tf.keras.mixed_precision.set_global_policy('mixed_float16')

# Build model normally — layers automatically use float16
model = tf.keras.Sequential([
    tf.keras.layers.Dense(512, activation='relu', input_shape=(784,)),
    tf.keras.layers.Dense(256, activation='relu'),
    tf.keras.layers.Dense(10)  # NO softmax here!
])

# Add float32 softmax for numerical stability
# The output layer MUST be float32 for stable loss computation
outputs = tf.keras.layers.Activation('softmax', dtype='float32')(model.output)
model = tf.keras.Model(model.input, outputs)

# Use a loss scale optimizer to prevent float16 underflow
optimizer = tf.keras.optimizers.Adam(learning_rate=1e-3)
# In TF 2.x, loss scaling is handled automatically

model.compile(
    optimizer=optimizer,
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

# Custom training loop with mixed precision
@tf.function
def train_step_mixed(x_batch, y_batch):
    with tf.GradientTape() as tape:
        predictions = model(x_batch, training=True)  # float16 compute
        loss = loss_fn(y_batch, predictions)
        # Scale loss to prevent float16 gradient underflow
        scaled_loss = optimizer.get_scaled_loss(loss)
    
    # Compute scaled gradients
    scaled_gradients = tape.gradient(scaled_loss, model.trainable_variables)
    # Unscale gradients back to float32
    gradients = optimizer.get_unscaled_gradients(scaled_gradients)
    
    optimizer.apply_gradients(zip(gradients, model.trainable_variables))
    return loss
```

### 5. Custom train_step (Best of Both Worlds)

```python
class CustomModel(tf.keras.Model):
    """
    Override train_step to customize training while still using model.fit().
    
    This gives you:
    ✅ Custom training logic
    ✅ Automatic callbacks
    ✅ Automatic progress bars
    ✅ Automatic distributed training
    ✅ Automatic metric tracking
    """
    
    def train_step(self, data):
        x, y = data
        
        with tf.GradientTape() as tape:
            y_pred = self(x, training=True)
            
            # Custom loss computation
            loss = self.compute_loss(y=y, y_pred=y_pred)
            
            # Add custom regularization
            for layer in self.layers:
                if hasattr(layer, 'kernel') and layer.kernel is not None:
                    loss += 1e-4 * tf.nn.l2_loss(layer.kernel)
        
        # Custom gradient processing
        gradients = tape.gradient(loss, self.trainable_variables)
        
        # Gradient clipping
        gradients, _ = tf.clip_by_global_norm(gradients, 5.0)
        
        self.optimizer.apply_gradients(zip(gradients, self.trainable_variables))
        
        # Update metrics
        for metric in self.metrics:
            if metric.name == "loss":
                metric.update_state(loss)
            else:
                metric.update_state(y, y_pred)
        
        return {m.name: m.result() for m in self.metrics}
    
    def test_step(self, data):
        """Custom validation step."""
        x, y = data
        y_pred = self(x, training=False)
        loss = self.compute_loss(y=y, y_pred=y_pred)
        
        for metric in self.metrics:
            if metric.name == "loss":
                metric.update_state(loss)
            else:
                metric.update_state(y, y_pred)
        
        return {m.name: m.result() for m in self.metrics}

# Usage — looks exactly like normal model.fit()!
# model = CustomModel(...)
# model.compile(optimizer='adam', loss='sparse_categorical_crossentropy', metrics=['accuracy'])
# model.fit(train_ds, epochs=10, validation_data=val_ds, callbacks=[...])
```

---

## Common Mistakes

### Mistake 1: Forgetting `training=True/False`

```python
# ❌ WRONG: Not passing training flag
predictions = model(x_batch)  # Dropout stays ON or OFF randomly!

# ✅ CORRECT: Always pass training flag
predictions = model(x_batch, training=True)   # During training
predictions = model(x_batch, training=False)  # During inference
```

### Mistake 2: Not Using @tf.function

```python
# ❌ SLOW: Pure Python execution
def train_step(x, y):
    with tf.GradientTape() as tape:
        pred = model(x, training=True)
        loss = loss_fn(y, pred)
    grads = tape.gradient(loss, model.trainable_variables)
    optimizer.apply_gradients(zip(grads, model.trainable_variables))

# ✅ FAST: Compiled to TF graph (10-100x faster!)
@tf.function
def train_step(x, y):
    # Same code, but runs as an optimized computation graph
    with tf.GradientTape() as tape:
        pred = model(x, training=True)
        loss = loss_fn(y, pred)
    grads = tape.gradient(loss, model.trainable_variables)
    optimizer.apply_gradients(zip(grads, model.trainable_variables))
```

### Mistake 3: Forgetting to Reset Metrics

```python
# ❌ WRONG: Metrics accumulate across epochs!
for epoch in range(10):
    for x, y in train_ds:
        train_step(x, y)
    print(f"Accuracy: {train_accuracy.result()}")
    # Epoch 1: 0.5, Epoch 2: 0.55 (includes epoch 1 data!), ...

# ✅ CORRECT: Reset at start of each epoch
for epoch in range(10):
    train_accuracy.reset_state()  # Reset!
    for x, y in train_ds:
        train_step(x, y)
    print(f"Accuracy: {train_accuracy.result()}")
```

### Mistake 4: Using Python Conditionals in @tf.function

```python
# ❌ WRONG: Python if doesn't work in @tf.function (traced once!)
@tf.function
def bad_function(x, flag):
    if flag:  # Python bool — traced once, then fixed!
        return x * 2
    else:
        return x * 3

# ✅ CORRECT: Use tf.cond for dynamic branching
@tf.function
def good_function(x, flag):
    return tf.cond(flag, lambda: x * 2, lambda: x * 3)
```

### Mistake 5: Not Deleting Persistent GradientTape

```python
# ❌ WRONG: Memory leak!
with tf.GradientTape(persistent=True) as tape:
    y = model(x)
    loss = loss_fn(y, labels)

grad1 = tape.gradient(loss, model.trainable_variables)
grad2 = tape.gradient(loss, model.trainable_variables)
# tape is never deleted → GPU memory leak!

# ✅ CORRECT: Always delete persistent tapes
with tf.GradientTape(persistent=True) as tape:
    y = model(x)
    loss = loss_fn(y, labels)

grad1 = tape.gradient(loss, model.trainable_variables)
grad2 = tape.gradient(loss, model.trainable_variables)
del tape  # Free memory!
```

### Mistake 6: Wrong Gradient Target

```python
# ❌ WRONG: Computing gradients w.r.t. ALL variables (including frozen ones)
grads = tape.gradient(loss, model.variables)  # Includes non-trainable!

# ✅ CORRECT: Only trainable variables
grads = tape.gradient(loss, model.trainable_variables)
```

---

## Interview Questions

### Q1: What is tf.GradientTape and how does it work?
**A**: `tf.GradientTape` is TensorFlow's automatic differentiation engine. It records all operations on watched tensors during the forward pass (inside the `with` block), then computes gradients by applying the chain rule backward through the recorded operations. By default, it watches `tf.Variable` objects. It's consumed after one `tape.gradient()` call unless `persistent=True` is set.

### Q2: When would you use a custom training loop instead of model.fit()?
**A**: Custom training loops are needed when:
- Training GANs (two models with different losses)
- Implementing gradient accumulation for large effective batch sizes
- Custom gradient processing (clipping, modification)
- Research algorithms not supported by standard APIs
- Multi-task learning with dynamic loss weighting
- Reinforcement learning
- Meta-learning (e.g., MAML)

### Q3: What's the difference between model.variables and model.trainable_variables?
**A**: 
- `model.variables`: ALL variables (weights + BatchNorm stats + optimizer slots)
- `model.trainable_variables`: Only variables that should receive gradient updates (excludes frozen layers, BN running stats)
- Always use `trainable_variables` when computing and applying gradients.

### Q4: Explain @tf.function and its benefits.
**A**: `@tf.function` compiles a Python function into a TensorFlow computation graph. Benefits:
- **Speed**: 10-100x faster than eager mode (optimized graph execution)
- **Portability**: Graphs can be saved and run without Python
- **Optimization**: TF can apply graph optimizations (constant folding, operator fusion)

Caveats: Python side effects only execute during tracing. Use `tf.print` instead of `print`, `tf.cond` instead of `if`.

### Q5: How do you implement gradient accumulation?
**A**: Accumulate gradients over N mini-batches before applying them:
1. Divide loss by N (accumulation steps)
2. Compute gradients for each mini-batch
3. Add gradients to running accumulators
4. After N steps, apply accumulated gradients
5. Reset accumulators
This simulates a larger batch size when GPU memory is limited.

### Q6: What is the purpose of the `build()` method in custom layers?
**A**: `build()` is called once when the layer first receives input. It's where you create weights whose shapes depend on the input. This enables lazy weight creation — you don't need to know the input shape when instantiating the layer. The `__init__` stores hyperparameters; `build` creates weights; `call` defines the forward pass.

### Q7: How does mixed precision training work?
**A**: Mixed precision uses float16 for most computations (faster, less memory) and float32 for numerically sensitive operations (loss computation, weight updates). Key components:
- Forward/backward pass in float16
- Master weights in float32
- Loss scaling to prevent float16 gradient underflow
- Final output layer in float32 for numerical stability
- Modern GPUs have Tensor Cores optimized for float16, giving 2-3x speedup.

### Q8: What is the difference between overriding `train_step()` and writing a full custom loop?
**A**: 
- **Override `train_step()`**: Customize WHAT happens in each training step while keeping model.fit()'s infrastructure (callbacks, progress bars, distributed training, metric tracking). Best when you need custom training logic but standard infrastructure.
- **Full custom loop**: Complete control over everything. You must implement progress tracking, callbacks, metric resetting, etc. yourself. Best for maximum flexibility (GANs, research).

---

## Quick Reference

### GradientTape Patterns

```python
# Basic gradient
with tf.GradientTape() as tape:
    loss = compute_loss(model, x, y)
grads = tape.gradient(loss, model.trainable_variables)
optimizer.apply_gradients(zip(grads, model.trainable_variables))

# Multiple gradient calls
with tf.GradientTape(persistent=True) as tape:
    loss1 = ...; loss2 = ...
g1 = tape.gradient(loss1, vars); g2 = tape.gradient(loss2, vars)
del tape

# Higher-order gradients
with tf.GradientTape() as t2:
    with tf.GradientTape() as t1:
        y = f(x)
    dy = t1.gradient(y, x)
d2y = t2.gradient(dy, x)
```

### Custom Training Loop Template

```python
@tf.function
def train_step(x, y):
    with tf.GradientTape() as tape:
        pred = model(x, training=True)
        loss = loss_fn(y, pred)
    grads = tape.gradient(loss, model.trainable_variables)
    optimizer.apply_gradients(zip(grads, model.trainable_variables))
    metric.update_state(y, pred)
    return loss

for epoch in range(EPOCHS):
    metric.reset_state()
    for x, y in dataset:
        loss = train_step(x, y)
    print(f"Epoch {epoch}: {metric.result():.4f}")
```

### Custom Component Hierarchy

| Component | Base Class | Key Methods |
|-----------|-----------|-------------|
| Custom Loss | `tf.keras.losses.Loss` | `call(y_true, y_pred)` |
| Custom Metric | `tf.keras.metrics.Metric` | `update_state()`, `result()`, `reset_state()` |
| Custom Layer | `tf.keras.layers.Layer` | `build()`, `call()`, `get_config()` |
| Custom Model | `tf.keras.Model` | `call()`, optionally `train_step()` |
| Custom Callback | `tf.keras.callbacks.Callback` | `on_epoch_end()`, `on_batch_end()`, etc. |

### Key @tf.function Rules

| Python Feature | In @tf.function | Use Instead |
|---------------|-----------------|-------------|
| `print()` | Runs only at trace time | `tf.print()` |
| `if/else` (dynamic) | Traced once, fixed | `tf.cond()` |
| `for` (dynamic) | Traced once, fixed | `tf.while_loop()` |
| Python list append | Doesn't work | `tf.TensorArray` |
| NumPy operations | Creates constants | `tf.` equivalents |

---

*Previous: [07-Transfer-Learning-TF](./07-Transfer-Learning-TF.md) | Next: [09-TF-Serving-and-Deployment](./09-TF-Serving-and-Deployment.md)*
