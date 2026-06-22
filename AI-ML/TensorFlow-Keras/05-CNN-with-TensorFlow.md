# Chapter 05: CNN with TensorFlow/Keras

## Table of Contents
- [What are CNNs?](#what-are-cnns)
- [Why CNNs Matter](#why-cnns-matter)
- [How CNNs Work](#how-cnns-work)
- [Building CNNs in Keras](#building-cnns-in-keras)
- [Key Layers Deep Dive](#key-layers-deep-dive)
- [Image Classification — Complete Walkthrough](#image-classification-complete-walkthrough)
- [Data Augmentation](#data-augmentation)
- [Pretrained Models and Transfer Learning](#pretrained-models-and-transfer-learning)
- [Advanced CNN Architectures](#advanced-cnn-architectures)
- [Visualizing What CNNs Learn](#visualizing-what-cnns-learn)
- [Production Best Practices](#production-best-practices)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What are CNNs?

### Simple Explanation

Imagine you're looking at a photo of a cat. Your brain doesn't analyze every pixel individually — it recognizes **patterns**:
- First, it sees **edges** (outlines of ears, whiskers)
- Then, it combines edges into **shapes** (triangular ears, oval eyes)
- Finally, it combines shapes into the **whole object** (a cat!)

A **Convolutional Neural Network (CNN)** works exactly the same way. It uses small sliding windows (called **filters** or **kernels**) to detect patterns in images, starting from simple patterns (edges) and building up to complex ones (faces, objects).

### Technical Definition

A CNN is a specialized neural network designed for grid-structured data (images, audio spectrograms). It uses three key ideas:
1. **Local connectivity** — each neuron connects only to a small region of the input
2. **Weight sharing** — the same filter is applied across the entire image
3. **Translation invariance** — a cat is a cat regardless of where it appears in the image

---

## Why CNNs Matter

### Real-World Applications

| Domain | Application | Example |
|--------|-------------|---------|
| Healthcare | Medical imaging | Detecting tumors in X-rays, MRIs |
| Automotive | Self-driving cars | Lane detection, pedestrian recognition |
| Security | Facial recognition | Phone unlock, surveillance |
| Agriculture | Crop monitoring | Disease detection from drone images |
| Manufacturing | Quality control | Detecting defective products |
| Social Media | Content moderation | Flagging inappropriate images |
| Retail | Visual search | "Find similar products" from photos |

### Why Not Use Dense (Fully Connected) Networks for Images?

```
Image: 224 × 224 × 3 (RGB) = 150,528 input values

Dense layer with 1000 neurons:
→ 150,528 × 1000 = 150 MILLION parameters (just first layer!)
→ Massive overfitting
→ No spatial awareness (pixel at (0,0) treated same as (224,224))

Conv2D(32, 3×3):
→ 3 × 3 × 3 × 32 = 864 parameters
→ Learns spatial patterns
→ Translation invariant
```

---

## How CNNs Work

### The Convolution Operation

```
Input Image (5×5):          Filter/Kernel (3×3):
┌─────────────────┐         ┌─────────┐
│ 1  0  1  0  1   │         │ 1  0  1 │
│ 0  1  0  1  0   │   *     │ 0  1  0 │
│ 1  0  1  0  1   │         │ 1  0  1 │
│ 0  1  0  1  0   │         └─────────┘
│ 1  0  1  0  1   │
└─────────────────┘

Step 1: Place filter on top-left 3×3 region
┌─────────┐
│ 1  0  1 │ 0  1
│ 0  1  0 │ 1  0     Element-wise multiply & sum:
│ 1  0  1 │ 0  1     (1×1)+(0×0)+(1×1)+(0×0)+(1×1)+(0×0)+(1×1)+(0×0)+(1×1) = 5
└─────────┘
  0  1  0  1  0
  1  0  1  0  1

Step 2: Slide filter one step right → compute again
Step 3: Continue until filter has covered entire image

Output (Feature Map): 3×3
┌─────────┐
│ 5  2  5 │
│ 2  5  2 │
│ 5  2  5 │
└─────────┘
```

### The Mathematics

For a 2D convolution:

$$\text{Output}(i, j) = \sum_{m} \sum_{n} \text{Input}(i+m, j+n) \cdot \text{Kernel}(m, n) + \text{bias}$$

**Output size formula**:

$$\text{Output Size} = \frac{W - K + 2P}{S} + 1$$

Where:
- $W$ = Input size
- $K$ = Kernel (filter) size
- $P$ = Padding
- $S$ = Stride

**Example**: Input 28×28, kernel 3×3, no padding, stride 1:
$$\frac{28 - 3 + 0}{1} + 1 = 26$$

### CNN Architecture Overview

```
Input Image → [CONV → ReLU → POOL] × N → FLATTEN → [DENSE → ReLU] × M → OUTPUT

Detailed:
┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐   ┌────────┐
│Input │──▶│Conv2D│──▶│ReLU  │──▶│Pool  │──▶│Conv2D│──▶│Pool  │──▶│Flatten │
│28×28 │   │26×26 │   │26×26 │   │13×13 │   │11×11 │   │5×5   │   │        │
│×1    │   │×32   │   │×32   │   │×32   │   │×64   │   │×64   │   │=1600   │
└──────┘   └──────┘   └──────┘   └──────┘   └──────┘   └──────┘   └────────┘
                                                                       │
                                                                       ▼
                                                               ┌────────────┐
                                                               │Dense(128)  │
                                                               │→Dense(10)  │
                                                               │→Softmax    │
                                                               └────────────┘
```

### Pooling — Downsampling

```
Max Pooling (2×2, stride 2):

Input (4×4):              Output (2×2):
┌─────┬─────┐            ┌─────┐
│ 1 3 │ 2 1 │            │  4  │  2  │
│ 4 2 │ 0 2 │   ──▶      ├─────┤     │
├─────┼─────┤            │  6  │  5  │
│ 6 1 │ 5 3 │            └─────┘─────┘
│ 2 0 │ 4 1 │
└─────┴─────┘
max(1,3,4,2)=4  max(2,1,0,2)=2
max(6,1,2,0)=6  max(5,3,4,1)=5
```

**Why pooling?**
- Reduces spatial dimensions → fewer parameters → less computation
- Provides translation invariance (small shifts don't change output)
- Controls overfitting

### Padding

```
"valid" padding (no padding):        "same" padding (zero-pad):
Input: 5×5, Kernel: 3×3              Input: 5×5, Kernel: 3×3

                                     0 0 0 0 0 0 0
┌─────────────┐                      0 ┌─────────────┐ 0
│ x x x x x   │                      0 │ x x x x x   │ 0
│ x x x x x   │ → Output: 3×3       0 │ x x x x x   │ 0 → Output: 5×5
│ x x x x x   │                      0 │ x x x x x   │ 0   (same as input!)
│ x x x x x   │                      0 │ x x x x x   │ 0
│ x x x x x   │                      0 │ x x x x x   │ 0
└─────────────┘                      0 └─────────────┘ 0
                                     0 0 0 0 0 0 0
```

---

## Building CNNs in Keras

### Your First CNN — MNIST

```python
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers
import numpy as np

# Load MNIST dataset
(X_train, y_train), (X_test, y_test) = keras.datasets.mnist.load_data()

# Preprocess
X_train = X_train.astype('float32') / 255.0  # Normalize to [0, 1]
X_test = X_test.astype('float32') / 255.0
X_train = X_train[..., np.newaxis]  # Add channel dim: (28,28) → (28,28,1)
X_test = X_test[..., np.newaxis]

print(f"Training: {X_train.shape}, Test: {X_test.shape}")
# Training: (60000, 28, 28, 1), Test: (10000, 28, 28, 1)

# Build CNN model
model = keras.Sequential([
    # First conv block
    layers.Conv2D(32, (3, 3), activation='relu', input_shape=(28, 28, 1)),
    # 32 filters of size 3×3
    # Output: (26, 26, 32)
    
    layers.MaxPooling2D((2, 2)),
    # Downsample by 2
    # Output: (13, 13, 32)
    
    # Second conv block
    layers.Conv2D(64, (3, 3), activation='relu'),
    # Output: (11, 11, 64)
    
    layers.MaxPooling2D((2, 2)),
    # Output: (5, 5, 64)
    
    # Third conv block
    layers.Conv2D(64, (3, 3), activation='relu'),
    # Output: (3, 3, 64)
    
    # Classification head
    layers.Flatten(),
    # Output: 3 × 3 × 64 = 576
    
    layers.Dense(64, activation='relu'),
    layers.Dense(10, activation='softmax')
    # 10 classes (digits 0-9)
])

model.summary()
```

```
Model: "sequential"
_________________________________________________________________
Layer (type)                 Output Shape              Param #   
=================================================================
conv2d (Conv2D)              (None, 26, 26, 32)        320       
max_pooling2d (MaxPooling2D) (None, 13, 13, 32)        0         
conv2d_1 (Conv2D)            (None, 11, 11, 64)        18496     
max_pooling2d_1 (MaxPooling2D)(None, 5, 5, 64)         0         
conv2d_2 (Conv2D)            (None, 3, 3, 64)          36928     
flatten (Flatten)            (None, 576)               0         
dense (Dense)                (None, 64)                36928     
dense_1 (Dense)              (None, 10)                650       
=================================================================
Total params: 93,322
Trainable params: 93,322
Non-trainable params: 0
```

```python
# Compile and train
model.compile(
    optimizer='adam',
    loss='sparse_categorical_crossentropy',  # Integer labels
    metrics=['accuracy']
)

history = model.fit(
    X_train, y_train,
    epochs=10,
    batch_size=64,
    validation_split=0.1,
    callbacks=[
        keras.callbacks.EarlyStopping(patience=3, restore_best_weights=True)
    ]
)

# Evaluate
test_loss, test_acc = model.evaluate(X_test, y_test, verbose=0)
print(f"Test accuracy: {test_acc:.4f}")
# Test accuracy: ~0.9920
```

### Parameter Count Explained

```python
# Conv2D(32, (3, 3)) with input channels = 1
# Parameters = (kernel_h × kernel_w × input_channels + 1) × num_filters
#            = (3 × 3 × 1 + 1) × 32
#            = 10 × 32 = 320

# Conv2D(64, (3, 3)) with input channels = 32
# Parameters = (3 × 3 × 32 + 1) × 64
#            = 289 × 64 = 18,496
```

---

## Key Layers Deep Dive

### Conv2D — The Core Layer

```python
layers.Conv2D(
    filters=64,           # Number of output feature maps
    kernel_size=(3, 3),   # Size of the sliding window
    strides=(1, 1),       # How far to slide each step
    padding='valid',      # 'valid' (no pad) or 'same' (keep size)
    activation='relu',    # Activation function
    use_bias=True,        # Add bias term
    kernel_initializer='glorot_uniform',  # Weight initialization
    kernel_regularizer=tf.keras.regularizers.l2(0.01),  # Weight decay
    input_shape=(224, 224, 3)  # Only needed for first layer
)
```

**Choosing filters and kernel_size:**

| Stage | Filters | Kernel | Purpose |
|-------|---------|--------|---------|
| Early layers | 32-64 | 3×3 or 5×5 | Detect edges, textures |
| Middle layers | 128-256 | 3×3 | Detect parts, patterns |
| Deep layers | 256-512 | 3×3 | Detect objects, high-level features |

> **Pro Tip**: Two stacked 3×3 convolutions have the same receptive field as one 5×5 but with fewer parameters and more non-linearity. Modern architectures almost exclusively use 3×3 kernels (VGG onwards).

### Pooling Layers

```python
# Max Pooling — most common
layers.MaxPooling2D(pool_size=(2, 2), strides=None, padding='valid')
# strides defaults to pool_size, so (2,2) pool with (2,2) stride

# Average Pooling — smoother, less information loss
layers.AveragePooling2D(pool_size=(2, 2))

# Global Average Pooling — replaces Flatten + Dense for modern architectures
layers.GlobalAveragePooling2D()
# Input: (batch, H, W, C) → Output: (batch, C)
# Takes the average of each feature map → one value per channel
# Much fewer parameters than Flatten + Dense
```

```
GlobalAveragePooling2D:

Input feature maps (7×7×512):
┌─────────┐ ┌─────────┐       ┌─────────┐
│ map 1   │ │ map 2   │  ...  │ map 512 │
│ (7×7)   │ │ (7×7)   │       │ (7×7)   │
└─────────┘ └─────────┘       └─────────┘
     │           │                  │
   avg=0.7    avg=0.3           avg=0.9

Output: [0.7, 0.3, ..., 0.9]  (512 values)
```

### BatchNormalization

```python
# Normalizes the output of the previous layer
# Dramatically speeds up training and acts as mild regularization

model = keras.Sequential([
    layers.Conv2D(32, (3, 3), use_bias=False),  # No bias needed — BN has its own
    layers.BatchNormalization(),                  # Normalize BEFORE activation
    layers.Activation('relu'),                   # Then activate
    
    layers.Conv2D(64, (3, 3), use_bias=False),
    layers.BatchNormalization(),
    layers.Activation('relu'),
    
    layers.GlobalAveragePooling2D(),
    layers.Dense(10, activation='softmax')
])
```

**How BatchNorm works:**

$$\hat{x} = \frac{x - \mu_{\text{batch}}}{\sqrt{\sigma^2_{\text{batch}} + \epsilon}}$$

$$y = \gamma \hat{x} + \beta$$

Where $\gamma$ (scale) and $\beta$ (shift) are learnable parameters.

> **Debate**: `Conv → BN → ReLU` vs `Conv → ReLU → BN`. Original paper uses the former. In practice, both work. The key is consistency.

### Dropout for CNNs

```python
model = keras.Sequential([
    layers.Conv2D(32, (3, 3), activation='relu'),
    layers.MaxPooling2D((2, 2)),
    
    layers.Conv2D(64, (3, 3), activation='relu'),
    layers.MaxPooling2D((2, 2)),
    
    layers.Flatten(),
    layers.Dropout(0.5),         # Drop 50% of connections
    layers.Dense(128, activation='relu'),
    layers.Dropout(0.5),
    layers.Dense(10, activation='softmax')
])

# For conv layers, use SpatialDropout2D instead
model = keras.Sequential([
    layers.Conv2D(32, (3, 3), activation='relu'),
    layers.SpatialDropout2D(0.2),  # Drops entire feature maps, not individual pixels
    layers.Conv2D(64, (3, 3), activation='relu'),
    layers.SpatialDropout2D(0.2),
])
```

**Dropout vs SpatialDropout2D:**
- `Dropout`: Randomly zeros individual elements → breaks spatial structure
- `SpatialDropout2D`: Zeros entire feature maps → preserves spatial structure, better for conv layers

### Depthwise Separable Convolution

```python
# Standard Conv2D: (3×3×64)×128 = 73,728 params
layers.Conv2D(128, (3, 3))

# Separable Conv2D: (3×3×64) + (1×1×64×128) = 576 + 8,192 = 8,768 params
layers.SeparableConv2D(128, (3, 3))
# ~8× fewer parameters, nearly same accuracy
# Used in MobileNet, EfficientNet, Xception
```

---

## Image Classification — Complete Walkthrough

### CIFAR-10 Classification

```python
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers
import matplotlib.pyplot as plt

# ============================================================
# 1. LOAD AND EXPLORE DATA
# ============================================================
(X_train, y_train), (X_test, y_test) = keras.datasets.cifar10.load_data()

class_names = ['airplane', 'automobile', 'bird', 'cat', 'deer',
               'dog', 'frog', 'horse', 'ship', 'truck']

print(f"Training: {X_train.shape}")    # (50000, 32, 32, 3)
print(f"Test: {X_test.shape}")         # (10000, 32, 32, 3)
print(f"Labels: {y_train.shape}")      # (50000, 1)
print(f"Pixel range: [{X_train.min()}, {X_train.max()}]")  # [0, 255]

# Visualize some samples
fig, axes = plt.subplots(2, 5, figsize=(12, 5))
for i, ax in enumerate(axes.flat):
    ax.imshow(X_train[i])
    ax.set_title(class_names[y_train[i][0]])
    ax.axis('off')
plt.tight_layout()
plt.show()


# ============================================================
# 2. PREPROCESS DATA
# ============================================================
# Normalize to [0, 1]
X_train = X_train.astype('float32') / 255.0
X_test = X_test.astype('float32') / 255.0

# Squeeze labels from (N, 1) to (N,)
y_train = y_train.squeeze()
y_test = y_test.squeeze()


# ============================================================
# 3. BUILD DATA PIPELINE
# ============================================================
BATCH_SIZE = 64
AUTOTUNE = tf.data.AUTOTUNE

# Data augmentation
data_augmentation = keras.Sequential([
    layers.RandomFlip("horizontal"),
    layers.RandomRotation(0.1),
    layers.RandomZoom(0.1),
    layers.RandomTranslation(0.1, 0.1),
])

# Training pipeline
train_dataset = (
    tf.data.Dataset.from_tensor_slices((X_train, y_train))
    .shuffle(50000)
    .batch(BATCH_SIZE)
    .map(lambda x, y: (data_augmentation(x, training=True), y),
         num_parallel_calls=AUTOTUNE)
    .prefetch(AUTOTUNE)
)

# Validation pipeline (no augmentation)
test_dataset = (
    tf.data.Dataset.from_tensor_slices((X_test, y_test))
    .batch(BATCH_SIZE)
    .cache()
    .prefetch(AUTOTUNE)
)


# ============================================================
# 4. BUILD MODEL
# ============================================================
def build_cnn(input_shape=(32, 32, 3), num_classes=10):
    """Build a CNN for CIFAR-10"""
    
    model = keras.Sequential([
        layers.Input(shape=input_shape),
        
        # Block 1
        layers.Conv2D(32, (3, 3), padding='same', use_bias=False),
        layers.BatchNormalization(),
        layers.Activation('relu'),
        layers.Conv2D(32, (3, 3), padding='same', use_bias=False),
        layers.BatchNormalization(),
        layers.Activation('relu'),
        layers.MaxPooling2D((2, 2)),
        layers.Dropout(0.25),
        
        # Block 2
        layers.Conv2D(64, (3, 3), padding='same', use_bias=False),
        layers.BatchNormalization(),
        layers.Activation('relu'),
        layers.Conv2D(64, (3, 3), padding='same', use_bias=False),
        layers.BatchNormalization(),
        layers.Activation('relu'),
        layers.MaxPooling2D((2, 2)),
        layers.Dropout(0.25),
        
        # Block 3
        layers.Conv2D(128, (3, 3), padding='same', use_bias=False),
        layers.BatchNormalization(),
        layers.Activation('relu'),
        layers.Conv2D(128, (3, 3), padding='same', use_bias=False),
        layers.BatchNormalization(),
        layers.Activation('relu'),
        layers.GlobalAveragePooling2D(),
        layers.Dropout(0.5),
        
        # Classification head
        layers.Dense(256, activation='relu'),
        layers.Dropout(0.5),
        layers.Dense(num_classes, activation='softmax')
    ])
    
    return model

model = build_cnn()
model.summary()


# ============================================================
# 5. COMPILE
# ============================================================
model.compile(
    optimizer=keras.optimizers.Adam(learning_rate=1e-3),
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)


# ============================================================
# 6. CALLBACKS
# ============================================================
callbacks = [
    keras.callbacks.EarlyStopping(
        monitor='val_accuracy',
        patience=10,
        restore_best_weights=True
    ),
    keras.callbacks.ReduceLROnPlateau(
        monitor='val_loss',
        factor=0.5,          # Halve learning rate
        patience=5,
        min_lr=1e-6
    ),
    keras.callbacks.ModelCheckpoint(
        'best_cifar10_model.keras',
        monitor='val_accuracy',
        save_best_only=True
    ),
]


# ============================================================
# 7. TRAIN
# ============================================================
history = model.fit(
    train_dataset,
    validation_data=test_dataset,
    epochs=100,          # EarlyStopping will stop earlier
    callbacks=callbacks
)


# ============================================================
# 8. EVALUATE
# ============================================================
test_loss, test_acc = model.evaluate(test_dataset, verbose=0)
print(f"\nTest Accuracy: {test_acc:.4f}")
# Expected: ~0.90-0.92 with this architecture


# ============================================================
# 9. PLOT TRAINING HISTORY
# ============================================================
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 5))

ax1.plot(history.history['accuracy'], label='Train')
ax1.plot(history.history['val_accuracy'], label='Validation')
ax1.set_title('Accuracy')
ax1.set_xlabel('Epoch')
ax1.legend()

ax2.plot(history.history['loss'], label='Train')
ax2.plot(history.history['val_loss'], label='Validation')
ax2.set_title('Loss')
ax2.set_xlabel('Epoch')
ax2.legend()

plt.tight_layout()
plt.show()
```

---

## Data Augmentation

### Why Augment?

```
Without augmentation:
- 5,000 cat images → Model memorizes these specific cats
- Can't generalize to new cats in different poses/lighting

With augmentation:
- 5,000 cat images × (flip + rotate + zoom + ...) → effectively millions
- Model learns "cat-ness" instead of specific images
```

### Keras Preprocessing Layers (Recommended)

```python
# Augmentation pipeline for images
augmentation = keras.Sequential([
    layers.RandomFlip("horizontal"),           # Flip left-right
    layers.RandomRotation(0.15),               # ±15% of full rotation
    layers.RandomZoom(0.15),                   # ±15% zoom
    layers.RandomTranslation(0.1, 0.1),        # ±10% shift
    layers.RandomContrast(0.2),                # ±20% contrast
    layers.RandomBrightness(0.2),              # ±20% brightness (TF 2.11+)
])

# Use as part of the model (recommended)
model = keras.Sequential([
    layers.Input(shape=(224, 224, 3)),
    augmentation,                               # Only active during training
    layers.Rescaling(1./255),
    layers.Conv2D(32, 3, activation='relu'),
    # ... rest of model
])
```

### tf.image Augmentations (More Control)

```python
def custom_augmentation(image, label):
    """Custom augmentation with tf.image ops"""
    # Random flip
    image = tf.image.random_flip_left_right(image)
    
    # Random brightness
    image = tf.image.random_brightness(image, max_delta=0.3)
    
    # Random contrast
    image = tf.image.random_contrast(image, lower=0.7, upper=1.3)
    
    # Random saturation
    image = tf.image.random_saturation(image, lower=0.7, upper=1.3)
    
    # Random JPEG quality (simulates compression artifacts)
    image = tf.cast(image * 255, tf.uint8)
    image = tf.image.random_jpeg_quality(image, min_jpeg_quality=50, max_jpeg_quality=100)
    image = tf.cast(image, tf.float32) / 255.0
    
    # Cutout / Random erasing
    image = random_cutout(image, mask_size=16)
    
    # Clip to valid range
    image = tf.clip_by_value(image, 0.0, 1.0)
    
    return image, label


def random_cutout(image, mask_size=16):
    """Randomly erase a square region of the image"""
    h, w, c = image.shape
    top = tf.random.uniform([], 0, h - mask_size, dtype=tf.int32)
    left = tf.random.uniform([], 0, w - mask_size, dtype=tf.int32)
    
    # Create mask
    mask = tf.ones([h, w, c])
    padding = [[top, h - top - mask_size], [left, w - left - mask_size], [0, 0]]
    cutout = tf.pad(tf.zeros([mask_size, mask_size, c]), padding, constant_values=1)
    
    return image * cutout
```

### MixUp and CutMix (Advanced)

```python
def mixup(images, labels, alpha=0.2):
    """MixUp: blend two images and their labels"""
    batch_size = tf.shape(images)[0]
    
    # Sample lambda from Beta distribution
    lam = tf.random.uniform([], 0, alpha)
    
    # Shuffle indices
    indices = tf.random.shuffle(tf.range(batch_size))
    
    # Mix images and labels
    mixed_images = lam * images + (1 - lam) * tf.gather(images, indices)
    mixed_labels = lam * tf.cast(labels, tf.float32) + \
                   (1 - lam) * tf.cast(tf.gather(labels, indices), tf.float32)
    
    return mixed_images, mixed_labels
```

### When to Use Which Augmentation

| Augmentation | Use When | Don't Use When |
|-------------|----------|----------------|
| Horizontal Flip | Most natural images | Text/numbers, medical scans with laterality |
| Vertical Flip | Satellite imagery, microscopy | Faces, cars, most natural images |
| Rotation | Objects at any angle | Fixed-orientation data |
| Color Jitter | Varying lighting conditions | Color is a key feature |
| Cutout/Erasing | Reducing overfitting | Very small objects |
| MixUp/CutMix | Label smoothing, regularization | Fine-grained classification |

---

## Pretrained Models and Transfer Learning

### What is Transfer Learning?

```
Analogy: Learning to play guitar → learning to play ukulele
- Don't start from scratch
- Transfer your "string instrument" knowledge
- Only learn the differences

Same with CNNs:
- ImageNet model already knows edges, textures, shapes, objects
- Fine-tune for YOUR specific task
- Needs much less data and training time
```

### Available Pretrained Models in Keras

```python
# All pretrained on ImageNet (1000 classes, 1.2M images)
tf.keras.applications.VGG16
tf.keras.applications.VGG19
tf.keras.applications.ResNet50
tf.keras.applications.ResNet101
tf.keras.applications.ResNet152
tf.keras.applications.InceptionV3
tf.keras.applications.InceptionResNetV2
tf.keras.applications.MobileNetV2
tf.keras.applications.MobileNetV3Small
tf.keras.applications.MobileNetV3Large
tf.keras.applications.EfficientNetB0  # through B7
tf.keras.applications.EfficientNetV2B0  # through V2L
tf.keras.applications.DenseNet121
tf.keras.applications.NASNetMobile
tf.keras.applications.ConvNeXtTiny
```

### Model Selection Guide

| Model | Params | Top-1 Acc | Speed | Use Case |
|-------|--------|-----------|-------|----------|
| MobileNetV3Small | 2.5M | 67.5% | ⚡⚡⚡ | Mobile/Edge |
| MobileNetV2 | 3.4M | 71.3% | ⚡⚡⚡ | Mobile/Edge |
| EfficientNetB0 | 5.3M | 77.1% | ⚡⚡ | Best accuracy/params tradeoff |
| ResNet50 | 25.6M | 76.0% | ⚡⚡ | General purpose |
| EfficientNetB4 | 19M | 82.9% | ⚡ | High accuracy needed |
| EfficientNetV2L | 118M | 85.7% | 🐌 | Maximum accuracy |
| ConvNeXt | 88.6M | 84.1% | 🐌 | Modern architecture |

### Strategy 1: Feature Extraction (Freeze Base)

```python
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers

IMG_SIZE = 224
NUM_CLASSES = 5  # e.g., flower classification

# Load pretrained model WITHOUT top classification layers
base_model = keras.applications.EfficientNetB0(
    weights='imagenet',           # Use ImageNet weights
    include_top=False,            # Remove classification head
    input_shape=(IMG_SIZE, IMG_SIZE, 3)
)

# FREEZE the base model — don't update its weights
base_model.trainable = False

# Build new model on top
model = keras.Sequential([
    layers.Input(shape=(IMG_SIZE, IMG_SIZE, 3)),
    
    # Preprocessing (EfficientNet has its own preprocessing)
    layers.Rescaling(1./255),
    
    # Pretrained feature extractor
    base_model,
    
    # New classification head
    layers.GlobalAveragePooling2D(),
    layers.Dropout(0.3),
    layers.Dense(256, activation='relu'),
    layers.Dropout(0.3),
    layers.Dense(NUM_CLASSES, activation='softmax')
])

model.compile(
    optimizer='adam',
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

model.summary()
# Total params: ~4.2M
# Trainable params: ~0.2M  ← Only the new head trains!
# Non-trainable params: ~4.0M
```

### Strategy 2: Fine-Tuning (Unfreeze Some Layers)

```python
# Step 1: Train with frozen base (as above) for a few epochs
history_frozen = model.fit(train_dataset, validation_data=val_dataset, epochs=10)

# Step 2: Unfreeze the top layers of base model
base_model.trainable = True

# Freeze everything except last 20 layers
for layer in base_model.layers[:-20]:
    layer.trainable = False

# Step 3: Re-compile with LOWER learning rate (crucial!)
model.compile(
    optimizer=keras.optimizers.Adam(learning_rate=1e-5),  # 10-100× lower
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

# Step 4: Continue training
history_finetuned = model.fit(
    train_dataset,
    validation_data=val_dataset,
    epochs=20,
    callbacks=[
        keras.callbacks.EarlyStopping(patience=5, restore_best_weights=True)
    ]
)
```

> **Critical**: When fine-tuning, use a **much lower learning rate** (1e-5 or less). High learning rates will destroy the pretrained weights.

### Strategy 3: Functional API for Transfer Learning

```python
# More flexibility with Functional API
base_model = keras.applications.ResNet50(
    weights='imagenet',
    include_top=False,
    input_shape=(224, 224, 3)
)
base_model.trainable = False

# Build model with Functional API
inputs = keras.Input(shape=(224, 224, 3))
x = keras.applications.resnet50.preprocess_input(inputs)  # Model-specific preprocessing
x = base_model(x, training=False)  # training=False keeps BN in inference mode
x = layers.GlobalAveragePooling2D()(x)
x = layers.Dropout(0.3)(x)
x = layers.Dense(256, activation='relu')(x)
x = layers.Dropout(0.3)(x)
outputs = layers.Dense(NUM_CLASSES, activation='softmax')(x)

model = keras.Model(inputs, outputs)
```

> **Important**: Pass `training=False` to the base model during feature extraction so BatchNormalization layers use their stored statistics, not batch statistics.

---

## Advanced CNN Architectures

### Residual Block (ResNet)

```python
def residual_block(x, filters, stride=1):
    """A residual block with skip connection"""
    shortcut = x
    
    # Main path
    x = layers.Conv2D(filters, (3, 3), strides=stride, padding='same', use_bias=False)(x)
    x = layers.BatchNormalization()(x)
    x = layers.Activation('relu')(x)
    
    x = layers.Conv2D(filters, (3, 3), padding='same', use_bias=False)(x)
    x = layers.BatchNormalization()(x)
    
    # Adjust shortcut dimensions if needed
    if stride != 1 or shortcut.shape[-1] != filters:
        shortcut = layers.Conv2D(filters, (1, 1), strides=stride, use_bias=False)(shortcut)
        shortcut = layers.BatchNormalization()(shortcut)
    
    # Skip connection: add shortcut to main path
    x = layers.Add()([x, shortcut])
    x = layers.Activation('relu')(x)
    
    return x

# Build a mini ResNet
inputs = keras.Input(shape=(32, 32, 3))
x = layers.Conv2D(64, (3, 3), padding='same', use_bias=False)(inputs)
x = layers.BatchNormalization()(x)
x = layers.Activation('relu')(x)

x = residual_block(x, 64)
x = residual_block(x, 64)
x = residual_block(x, 128, stride=2)  # Downsample
x = residual_block(x, 128)
x = residual_block(x, 256, stride=2)  # Downsample
x = residual_block(x, 256)

x = layers.GlobalAveragePooling2D()(x)
outputs = layers.Dense(10, activation='softmax')(x)

model = keras.Model(inputs, outputs)
```

```
Why ResNet works:

Without skip connections:           With skip connections:
x ──▶ Conv ──▶ Conv ──▶ output     x ──▶ Conv ──▶ Conv ──┐
                                    │                      │
                                    └──────────────────── + ──▶ output
                                         (identity)

The network only needs to learn the RESIDUAL (difference),
not the entire mapping. If the block isn't helpful,
weights go to zero and data flows through the shortcut.
```

### Inception Block

```python
def inception_block(x, filters):
    """Inception block — multiple filter sizes in parallel"""
    f1, f3_reduce, f3, f5_reduce, f5, pool_proj = filters
    
    # 1×1 convolution branch
    branch1 = layers.Conv2D(f1, (1, 1), padding='same', activation='relu')(x)
    
    # 1×1 → 3×3 branch
    branch3 = layers.Conv2D(f3_reduce, (1, 1), padding='same', activation='relu')(x)
    branch3 = layers.Conv2D(f3, (3, 3), padding='same', activation='relu')(branch3)
    
    # 1×1 → 5×5 branch
    branch5 = layers.Conv2D(f5_reduce, (1, 1), padding='same', activation='relu')(x)
    branch5 = layers.Conv2D(f5, (5, 5), padding='same', activation='relu')(branch5)
    
    # MaxPool → 1×1 branch
    branch_pool = layers.MaxPooling2D((3, 3), strides=1, padding='same')(x)
    branch_pool = layers.Conv2D(pool_proj, (1, 1), padding='same', activation='relu')(branch_pool)
    
    # Concatenate all branches
    return layers.Concatenate()([branch1, branch3, branch5, branch_pool])
```

### Depthwise Separable CNN (MobileNet-style)

```python
def mobilenet_block(x, filters, stride=1):
    """MobileNet-style depthwise separable convolution block"""
    # Depthwise convolution (per-channel)
    x = layers.DepthwiseConv2D((3, 3), strides=stride, padding='same', use_bias=False)(x)
    x = layers.BatchNormalization()(x)
    x = layers.Activation('relu6')(x)  # ReLU6: min(max(0, x), 6)
    
    # Pointwise convolution (1×1, cross-channel)
    x = layers.Conv2D(filters, (1, 1), use_bias=False)(x)
    x = layers.BatchNormalization()(x)
    x = layers.Activation('relu6')(x)
    
    return x

# Build lightweight model
inputs = keras.Input(shape=(224, 224, 3))
x = layers.Conv2D(32, (3, 3), strides=2, padding='same', use_bias=False)(inputs)
x = layers.BatchNormalization()(x)
x = layers.Activation('relu6')(x)

x = mobilenet_block(x, 64)
x = mobilenet_block(x, 128, stride=2)
x = mobilenet_block(x, 128)
x = mobilenet_block(x, 256, stride=2)
x = mobilenet_block(x, 256)

x = layers.GlobalAveragePooling2D()(x)
outputs = layers.Dense(10, activation='softmax')(x)

model = keras.Model(inputs, outputs)
```

---

## Visualizing What CNNs Learn

### Visualize Filters

```python
import matplotlib.pyplot as plt
import numpy as np

# Get the first conv layer's weights
first_conv = model.layers[0]  # Assuming first layer is Conv2D
filters, biases = first_conv.get_weights()
print(f"Filter shape: {filters.shape}")  # e.g., (3, 3, 3, 32)

# Normalize filter values for visualization
f_min, f_max = filters.min(), filters.max()
filters = (filters - f_min) / (f_max - f_min)

# Plot first 16 filters
fig, axes = plt.subplots(4, 4, figsize=(8, 8))
for i, ax in enumerate(axes.flat):
    if i < filters.shape[-1]:
        # Display filter (for RGB input, show as color image)
        ax.imshow(filters[:, :, :, i])
    ax.axis('off')
plt.suptitle('Learned Filters (Layer 1)')
plt.tight_layout()
plt.show()
```

### Visualize Feature Maps (Intermediate Activations)

```python
# Create a model that outputs intermediate activations
layer_outputs = [layer.output for layer in model.layers if 'conv' in layer.name]
activation_model = keras.Model(inputs=model.input, outputs=layer_outputs)

# Get activations for a sample image
sample_image = X_test[0:1]  # (1, 32, 32, 3)
activations = activation_model.predict(sample_image)

# Visualize feature maps from first conv layer
first_layer_activation = activations[0]
print(f"Activation shape: {first_layer_activation.shape}")  # (1, 30, 30, 32)

fig, axes = plt.subplots(4, 8, figsize=(16, 8))
for i, ax in enumerate(axes.flat):
    if i < first_layer_activation.shape[-1]:
        ax.imshow(first_layer_activation[0, :, :, i], cmap='viridis')
    ax.axis('off')
plt.suptitle('Feature Maps — First Conv Layer')
plt.tight_layout()
plt.show()
```

### Grad-CAM (Where the Model Looks)

```python
import numpy as np

def make_gradcam_heatmap(img_array, model, last_conv_layer_name, pred_index=None):
    """Generate Grad-CAM heatmap"""
    # Create model that outputs last conv layer + predictions
    grad_model = keras.Model(
        [model.inputs],
        [model.get_layer(last_conv_layer_name).output, model.output]
    )
    
    # Compute gradients
    with tf.GradientTape() as tape:
        conv_outputs, predictions = grad_model(img_array)
        if pred_index is None:
            pred_index = tf.argmax(predictions[0])
        class_channel = predictions[:, pred_index]
    
    # Gradients of the predicted class w.r.t. conv layer output
    grads = tape.gradient(class_channel, conv_outputs)
    
    # Global average pooling of gradients
    pooled_grads = tf.reduce_mean(grads, axis=(0, 1, 2))
    
    # Weight conv outputs by pooled gradients
    conv_outputs = conv_outputs[0]
    heatmap = conv_outputs @ pooled_grads[..., tf.newaxis]
    heatmap = tf.squeeze(heatmap)
    
    # Normalize
    heatmap = tf.maximum(heatmap, 0) / tf.math.reduce_max(heatmap)
    return heatmap.numpy()


# Usage
img = X_test[0:1]
heatmap = make_gradcam_heatmap(img, model, 'conv2d_5')  # Use last conv layer name

# Overlay heatmap on image
plt.figure(figsize=(10, 4))
plt.subplot(1, 3, 1)
plt.imshow(X_test[0])
plt.title('Original')
plt.axis('off')

plt.subplot(1, 3, 2)
plt.imshow(heatmap, cmap='jet')
plt.title('Grad-CAM Heatmap')
plt.axis('off')

plt.subplot(1, 3, 3)
plt.imshow(X_test[0])
plt.imshow(tf.image.resize(heatmap[..., np.newaxis], (32, 32)).numpy().squeeze(),
           cmap='jet', alpha=0.4)
plt.title('Overlay')
plt.axis('off')

plt.tight_layout()
plt.show()
```

---

## Production Best Practices

### 1. Use `tf.keras.applications` Preprocessing

```python
# Each model has its own preprocessing — DON'T just divide by 255!
from tensorflow.keras.applications.efficientnet import preprocess_input
# EfficientNet: expects [0, 255] range! (handles normalization internally)

from tensorflow.keras.applications.resnet50 import preprocess_input
# ResNet: caffe-style preprocessing (BGR, channel means subtracted)

from tensorflow.keras.applications.mobilenet_v2 import preprocess_input
# MobileNet: scales to [-1, 1]
```

### 2. Learning Rate Schedule

```python
# Cosine decay — popular in modern training
lr_schedule = keras.optimizers.schedules.CosineDecay(
    initial_learning_rate=1e-3,
    decay_steps=1000 * 50,  # steps_per_epoch * epochs
    alpha=1e-6              # Minimum learning rate
)

optimizer = keras.optimizers.Adam(learning_rate=lr_schedule)
```

### 3. Mixed Precision Training

```python
# Use float16 for computation, float32 for accumulation
# ~2× speedup on modern GPUs with minimal accuracy loss
tf.keras.mixed_precision.set_global_policy('mixed_float16')

# Important: output layer MUST use float32
outputs = layers.Dense(10, activation='softmax', dtype='float32')(x)
```

### 4. Model Export for Production

```python
# Save as SavedModel (recommended)
model.save('my_model')

# Save as TF Lite (mobile/edge)
converter = tf.lite.TFLiteConverter.from_saved_model('my_model')
converter.optimizations = [tf.lite.Optimize.DEFAULT]
tflite_model = converter.convert()
with open('model.tflite', 'wb') as f:
    f.write(tflite_model)
```

---

## Common Mistakes

### 1. Not Using Pretrained Models

```python
# ❌ WRONG for most tasks — training from scratch
model = keras.Sequential([
    layers.Conv2D(32, 3, activation='relu', input_shape=(224, 224, 3)),
    # ... many layers from scratch
])

# ✅ CORRECT — use transfer learning
base = keras.applications.EfficientNetB0(weights='imagenet', include_top=False)
```

### 2. Wrong Input Preprocessing for Pretrained Models

```python
# ❌ WRONG — dividing by 255 for ResNet (expects caffe preprocessing)
x = image / 255.0
pred = resnet_model.predict(x)

# ✅ CORRECT — use model-specific preprocessing
from tensorflow.keras.applications.resnet50 import preprocess_input
x = preprocess_input(image)  # Handles BGR conversion and mean subtraction
```

### 3. Not Using padding='same'

```python
# ❌ WRONG — output shrinks rapidly with 'valid' padding
layers.Conv2D(64, (3, 3))  # 224→222→220→218→... shrinks fast!

# ✅ CORRECT — use 'same' to maintain spatial dimensions
layers.Conv2D(64, (3, 3), padding='same')  # 224→224→224→...
# Use pooling for intentional downsampling
```

### 4. Too High Learning Rate When Fine-Tuning

```python
# ❌ WRONG — destroys pretrained weights
base_model.trainable = True
model.compile(optimizer=keras.optimizers.Adam(1e-3), ...)

# ✅ CORRECT — use 10-100× lower learning rate
model.compile(optimizer=keras.optimizers.Adam(1e-5), ...)
```

### 5. Forgetting to Set training=False for Base Model

```python
# ❌ WRONG — BN layers use batch statistics during feature extraction
x = base_model(inputs)

# ✅ CORRECT — BN layers use stored statistics
x = base_model(inputs, training=False)
```

### 6. Not Resizing Images Consistently

```python
# ❌ WRONG — different image sizes cause shape errors
# Image 1: (320, 480, 3)
# Image 2: (640, 480, 3)

# ✅ CORRECT — always resize to fixed dimensions
image = tf.image.resize(image, [224, 224])
```

---

## Interview Questions

### Q1: Explain the convolution operation and why CNNs use it instead of fully connected layers.

**Answer**: Convolution slides a small filter (e.g., 3×3) across the image, computing element-wise multiplication and sum at each position. CNNs prefer convolution because:
1. **Parameter sharing** — same filter weights applied everywhere, dramatically reducing parameters
2. **Local connectivity** — each output depends only on a small region, matching how visual features are local
3. **Translation equivariance** — a feature detected at one position can be detected anywhere
4. **Hierarchy** — stacking convolutions builds up from edges → textures → parts → objects

### Q2: What is the receptive field and why does it matter?

**Answer**: The receptive field is the region of the original input that influences a particular output neuron. It grows with each layer:
- Layer 1 (3×3 conv): receptive field = 3×3
- Layer 2 (3×3 conv): receptive field = 5×5
- After 2×2 pooling + 3×3 conv: receptive field grows faster

It matters because the network can only "see" within its receptive field. If the receptive field is smaller than the object you're trying to detect, the network can't classify it correctly. This is why deeper networks (larger receptive fields) work better for complex tasks.

### Q3: Explain the difference between valid and same padding.

**Answer**:
- **Valid**: No padding. Output is smaller than input by `(kernel_size - 1)`. A 28×28 input with 3×3 kernel → 26×26 output.
- **Same**: Zero-pads the input so the output has the same spatial dimensions as the input. 28×28 → 28×28.

Use **same** to preserve dimensions (controlled downsampling via pooling). Use **valid** when you want the edges to have less influence.

### Q4: What is transfer learning and when should you use it?

**Answer**: Transfer learning uses a model pretrained on one task (e.g., ImageNet) as a starting point for a different task. Two strategies:
1. **Feature extraction** — freeze pretrained weights, train only the new head
2. **Fine-tuning** — unfreeze some layers and train with a very low learning rate

Use it when:
- You have limited data (< 10,000 images)
- Your task is similar to the pretraining task
- You need fast development time
- You want state-of-the-art results without massive compute

### Q5: Why does ResNet use skip connections?

**Answer**: Skip connections solve the **degradation problem** — very deep networks (50+ layers) without skips actually perform *worse* than shallow networks. This happens because gradients vanish through many layers.

Skip connections allow gradients to flow directly through the shortcut path, enabling training of 100+ layer networks. The network learns **residual mappings** (the difference from identity), which is easier to optimize than learning the full transformation.

### Q6: Compare MaxPooling vs AveragePooling vs GlobalAveragePooling.

**Answer**:
| Feature | MaxPooling | AveragePooling | GlobalAveragePooling |
|---------|-----------|----------------|---------------------|
| Operation | Takes maximum value | Takes average value | Average over entire spatial dim |
| Output | Smaller spatial dims | Smaller spatial dims | Single vector (1×C) |
| Preserves | Strongest activations | All information equally | Overall feature presence |
| Use case | Most CNNs | Smooth features needed | Replace Flatten+Dense |
| Translation invariance | Higher | Lower | Highest |

### Q7: What is depthwise separable convolution and why is it efficient?

**Answer**: It splits a standard convolution into two steps:
1. **Depthwise** — apply a separate filter to each input channel (no cross-channel mixing)
2. **Pointwise** — 1×1 convolution to mix channels

For a 3×3 conv with 64 input and 128 output channels:
- Standard: $3 × 3 × 64 × 128 = 73,728$ params
- Separable: $(3 × 3 × 64) + (1 × 1 × 64 × 128) = 576 + 8,192 = 8,768$ params

~8.4× fewer parameters with similar accuracy. Used in MobileNet, EfficientNet, Xception.

---

## Quick Reference

### Standard CNN Architecture Template

```python
model = keras.Sequential([
    # [Conv → BN → ReLU → Pool] × N
    layers.Conv2D(32, 3, padding='same', use_bias=False),
    layers.BatchNormalization(),
    layers.Activation('relu'),
    layers.MaxPooling2D(),
    
    layers.Conv2D(64, 3, padding='same', use_bias=False),
    layers.BatchNormalization(),
    layers.Activation('relu'),
    layers.MaxPooling2D(),
    
    # Classification head
    layers.GlobalAveragePooling2D(),
    layers.Dropout(0.5),
    layers.Dense(num_classes, activation='softmax')
])
```

### Layer Output Size Formulas

| Layer | Output Size |
|-------|-------------|
| Conv2D (valid) | $\frac{W - K}{S} + 1$ |
| Conv2D (same) | $\frac{W}{S}$ (ceiled) |
| MaxPooling2D | $\frac{W}{P}$ |
| GlobalAveragePooling2D | $C$ (channels only) |

### Transfer Learning Decision Tree

```
Do you have a pretrained model for a similar task?
├── YES
│   ├── Small dataset (< 1000 samples)
│   │   └── Feature extraction (freeze all)
│   ├── Medium dataset (1K-100K)
│   │   └── Feature extraction → Fine-tune top layers
│   └── Large dataset (100K+)
│       └── Fine-tune entire model (low LR)
└── NO
    └── Train from scratch (need lots of data + compute)
```

### Key Hyperparameters

| Parameter | Typical Values | Notes |
|-----------|---------------|-------|
| Kernel size | 3×3 (almost always) | Two 3×3 = one 5×5, fewer params |
| Filters | 32→64→128→256 | Double when spatial dims halve |
| Pooling | 2×2, stride 2 | Halves spatial dimensions |
| Dropout (conv) | 0.1-0.3 | Use SpatialDropout2D |
| Dropout (dense) | 0.3-0.5 | Higher for dense layers |
| Learning rate | 1e-3 (fresh), 1e-5 (fine-tune) | Reduce for fine-tuning |
| Batch size | 32-128 | Larger = faster, may need LR adjustment |

### Pretrained Model Quick Picks

```
Need speed:      MobileNetV3Small
Need balance:    EfficientNetB0
Need accuracy:   EfficientNetV2L or ConvNeXt
Need simplicity: ResNet50
Need tiny model: MobileNetV3Small + quantization
```

---

*Next: [06-RNN-NLP-with-TensorFlow.md](./06-RNN-NLP-with-TensorFlow.md) — Building RNNs and NLP models with TensorFlow/Keras*
