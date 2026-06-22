# Chapter 07: Transfer Learning with TensorFlow

## Table of Contents
- [What is Transfer Learning?](#what-is-transfer-learning)
- [Why Transfer Learning Matters](#why-transfer-learning-matters)
- [How Transfer Learning Works](#how-transfer-learning-works)
- [TensorFlow Hub](#tensorflow-hub)
- [tf.keras.applications — Pretrained Models](#tfkerasapplications--pretrained-models)
- [Feature Extraction](#feature-extraction)
- [Fine-Tuning](#fine-tuning)
- [Transfer Learning for NLP](#transfer-learning-for-nlp)
- [Advanced Strategies](#advanced-strategies)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Transfer Learning?

### Simple Explanation

Imagine you already know how to ride a bicycle. When you try to ride a motorcycle, you don't start from zero — you already understand balance, steering, and braking. You just need to learn the new parts (engine, gears, throttle). **Transfer Learning** is exactly this: taking knowledge learned from one task and applying it to a different but related task.

In deep learning terms:
- A model trained on **ImageNet** (14 million images, 1000 categories) has learned to recognize edges, textures, shapes, and objects.
- Instead of training a new model from scratch for your cat-vs-dog classifier, you **reuse** that pretrained model's knowledge.

### Formal Definition

> **Transfer Learning** is a machine learning technique where a model developed for one task is reused as the starting point for a model on a second task. It leverages the learned feature representations from a source domain/task to improve generalization in a target domain/task.

---

## Why Transfer Learning Matters

### The Problem Without It

```
Training from scratch:
┌──────────────────────────────────────────────┐
│  Need: Millions of labeled images            │
│  Need: Days/weeks of GPU training            │
│  Need: Expensive hardware (multiple GPUs)    │
│  Risk: Overfitting on small datasets         │
│  Risk: Poor generalization                   │
└──────────────────────────────────────────────┘

With Transfer Learning:
┌──────────────────────────────────────────────┐
│  Need: Hundreds to thousands of images       │
│  Need: Minutes to hours of training          │
│  Need: Single GPU (or even CPU for small)    │
│  Benefit: Better generalization              │
│  Benefit: State-of-the-art performance       │
└──────────────────────────────────────────────┘
```

### When to Use Transfer Learning

| Scenario | Use Transfer Learning? | Why? |
|----------|----------------------|------|
| Small dataset (< 10K images) | ✅ Yes | Prevents overfitting |
| Limited compute budget | ✅ Yes | Saves training time |
| Target domain similar to source | ✅ Yes (fine-tune) | Features transfer well |
| Target domain very different | ✅ Yes (feature extraction) | Low-level features still useful |
| Massive dataset + unlimited compute | ⚠️ Maybe | Training from scratch may match/exceed |
| Entirely unrelated domains | ❌ Rarely | Negative transfer risk |

### Real-World Applications

- **Medical Imaging**: Models pretrained on ImageNet fine-tuned for X-ray diagnosis
- **Satellite Imagery**: ImageNet models adapted for land-use classification
- **NLP**: BERT/GPT pretrained on web text, fine-tuned for sentiment analysis
- **Audio**: Models pretrained on speech, adapted for music genre classification
- **Manufacturing**: Pretrained models adapted for defect detection

---

## How Transfer Learning Works

### The Intuition: What Do Neural Networks Learn?

```
Layer Depth vs. Feature Abstraction:

Input Image → [Pixel Values]
     │
     ▼
┌─────────────────────┐
│   Layer 1-2         │  → Edges, gradients, colors
│   (VERY GENERAL)    │  → Useful for ALMOST ANY vision task
├─────────────────────┤
│   Layer 3-5         │  → Textures, patterns, shapes
│   (GENERAL)         │  → Useful for MOST vision tasks
├─────────────────────┤
│   Layer 6-10        │  → Parts of objects (eyes, wheels)
│   (SOMEWHAT SPECIFIC)│  → Useful for RELATED tasks
├─────────────────────┤
│   Layer 11+         │  → Full objects, scene understanding
│   (TASK SPECIFIC)   │  → Specific to the ORIGINAL task
├─────────────────────┤
│   Final Dense Layer │  → Classification head
│   (COMPLETELY SPECIFIC) │ → MUST be replaced
└─────────────────────┘
```

### Two Main Strategies

```
Strategy 1: FEATURE EXTRACTION              Strategy 2: FINE-TUNING
┌────────────────────────┐                 ┌────────────────────────┐
│  Pretrained Layers     │ ← FROZEN ❄️     │  Pretrained Layers     │ ← FROZEN ❄️
│  (don't update)        │                 │  (initially frozen)     │
│                        │                 │                        │
│  ████████████████████  │                 │  ████████████████████  │
│  ████████████████████  │                 │  ████████████████████  │
│  ████████████████████  │                 ├────────────────────────┤
│  ████████████████████  │                 │  Upper Pretrained      │ ← UNFROZEN 🔥
├────────────────────────┤                 │  (fine-tune these)     │
│  New Classification    │ ← TRAINABLE 🔥  │                        │
│  Head (your layers)    │                 ├────────────────────────┤
└────────────────────────┘                 │  New Classification    │ ← TRAINABLE 🔥
                                           │  Head (your layers)    │
                                           └────────────────────────┘

When to use:                               When to use:
• Very small dataset                       • Moderate-large dataset
• Target domain similar to source          • Need higher accuracy
• Quick results needed                     • Have GPU resources
• Limited compute                          • Target domain slightly different
```

### The Mathematics

In transfer learning, we decompose the model parameters:

$$\theta = \{\theta_{\text{pretrained}}, \theta_{\text{new}}\}$$

**Feature Extraction:**
$$\min_{\theta_{\text{new}}} \mathcal{L}(f(\mathbf{x}; \theta_{\text{pretrained}}, \theta_{\text{new}}), \mathbf{y})$$

Only $\theta_{\text{new}}$ is optimized. $\theta_{\text{pretrained}}$ is frozen.

**Fine-Tuning:**
$$\min_{\theta_{\text{pretrained}}^{(k:)}, \theta_{\text{new}}} \mathcal{L}(f(\mathbf{x}; \theta_{\text{pretrained}}, \theta_{\text{new}}), \mathbf{y})$$

Parameters from layer $k$ onward are optimized with a small learning rate.

> **Key insight**: Fine-tuning uses a much smaller learning rate (typically 10-100x smaller) than training from scratch, to avoid destroying the pretrained features.

---

## TensorFlow Hub

### What is TensorFlow Hub?

TensorFlow Hub is a repository of trained machine learning models ready for fine-tuning and deployment. Think of it as "npm for ML models."

### Installation

```python
# Install TensorFlow Hub
# pip install tensorflow-hub

import tensorflow as tf
import tensorflow_hub as hub
import numpy as np
```

### Using TF Hub for Image Classification (Feature Extraction)

```python
import tensorflow as tf
import tensorflow_hub as hub

# ============================================
# STEP 1: Load a pretrained model from TF Hub
# ============================================
# MobileNetV2 feature vector — outputs 1280-dimensional embeddings
# This model was trained on ImageNet (1000 classes)
feature_extractor_url = "https://tfhub.dev/google/tf2-preview/mobilenet_v2/feature_vector/4"

# Create a Keras layer from the TF Hub model
# trainable=False means we FREEZE the pretrained weights
feature_extractor_layer = hub.KerasLayer(
    feature_extractor_url,
    input_shape=(224, 224, 3),  # MobileNetV2 expects 224x224 images
    trainable=False             # Freeze pretrained weights
)

# ============================================
# STEP 2: Build a new model on top
# ============================================
model = tf.keras.Sequential([
    feature_extractor_layer,            # Pretrained backbone (frozen)
    tf.keras.layers.Dense(128, activation='relu'),  # New hidden layer
    tf.keras.layers.Dropout(0.3),       # Regularization
    tf.keras.layers.Dense(5, activation='softmax')  # 5 classes (your task)
])

model.summary()
# Output:
# Layer (type)                Output Shape              Param #
# ================================================================
# keras_layer (KerasLayer)    (None, 1280)              2,257,984  ← Frozen
# dense (Dense)               (None, 128)               163,968    ← Trainable
# dropout (Dropout)           (None, 128)               0
# dense_1 (Dense)             (None, 5)                 645        ← Trainable
# ================================================================
# Total params: 2,422,597
# Trainable params: 164,613     ← Only ~7% of total!
# Non-trainable params: 2,257,984

# ============================================
# STEP 3: Compile and train
# ============================================
model.compile(
    optimizer=tf.keras.optimizers.Adam(learning_rate=1e-3),
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

# model.fit(train_dataset, epochs=10, validation_data=val_dataset)
```

### Using TF Hub for Text Embedding

```python
import tensorflow as tf
import tensorflow_hub as hub

# Load Universal Sentence Encoder
# Converts sentences → 512-dimensional vectors
embed = hub.load("https://tfhub.dev/google/universal-sentence-encoder/4")

# Encode some sentences
sentences = [
    "TensorFlow is a machine learning framework.",
    "Keras makes deep learning easy.",
    "I love pizza and pasta."
]

embeddings = embed(sentences)
print(f"Shape: {embeddings.shape}")  # (3, 512)

# Check similarity between sentences
from numpy import dot
from numpy.linalg import norm

def cosine_similarity(a, b):
    return dot(a, b) / (norm(a) * norm(b))

# Sentences about ML should be more similar to each other
sim_01 = cosine_similarity(embeddings[0].numpy(), embeddings[1].numpy())
sim_02 = cosine_similarity(embeddings[0].numpy(), embeddings[2].numpy())
print(f"ML sentences similarity: {sim_01:.4f}")   # ~0.65
print(f"ML vs food similarity: {sim_02:.4f}")      # ~0.15

# Build a text classification model
text_model = tf.keras.Sequential([
    hub.KerasLayer(
        "https://tfhub.dev/google/universal-sentence-encoder/4",
        input_shape=[],           # Accepts raw strings
        dtype=tf.string,
        trainable=False
    ),
    tf.keras.layers.Dense(64, activation='relu'),
    tf.keras.layers.Dense(1, activation='sigmoid')  # Binary classification
])

text_model.compile(
    optimizer='adam',
    loss='binary_crossentropy',
    metrics=['accuracy']
)
```

---

## tf.keras.applications — Pretrained Models

### Available Models

TensorFlow comes with many pretrained models built-in. No need for TF Hub.

| Model | Top-1 Accuracy | Parameters | Size (MB) | Speed | Best For |
|-------|---------------|------------|-----------|-------|----------|
| MobileNetV2 | 71.3% | 3.4M | 14 | ⚡⚡⚡ | Mobile/Edge deployment |
| MobileNetV3 | 75.6% | 5.4M | 22 | ⚡⚡⚡ | Mobile/Edge (improved) |
| EfficientNetB0 | 77.1% | 5.3M | 29 | ⚡⚡ | Balance of speed/accuracy |
| EfficientNetB7 | 84.3% | 66M | 256 | ⚡ | Max accuracy (large) |
| ResNet50 | 74.9% | 25.6M | 98 | ⚡⚡ | General purpose |
| ResNet152 | 76.6% | 60.3M | 232 | ⚡ | Higher accuracy |
| InceptionV3 | 77.9% | 23.8M | 92 | ⚡⚡ | Multi-scale features |
| VGG16 | 71.3% | 138.4M | 528 | ⚡ | Simple architecture |
| DenseNet121 | 75.0% | 8.0M | 33 | ⚡⚡ | Feature reuse |
| Xception | 79.0% | 22.9M | 88 | ⚡⚡ | Depthwise separable |
| NASNetLarge | 82.5% | 88.9M | 343 | ⚡ | AutoML designed |

### How to Choose a Pretrained Model

```
Decision Tree:

                    ┌─ Deploying on mobile/edge?
                    │   YES → MobileNetV2/V3
                    │   NO ──┐
                    │        ├─ Need max accuracy?
                    │        │   YES → EfficientNetB4-B7
                    │        │   NO ──┐
                    │        │        ├─ General purpose?
                    │        │        │   YES → ResNet50 or EfficientNetB0
                    │        │        │   NO ──┐
                    │        │        │        ├─ Multi-scale features?
                    │        │        │        │   YES → InceptionV3
                    │        │        │        │   NO → DenseNet121
```

### Loading Pretrained Models

```python
import tensorflow as tf
from tensorflow.keras.applications import (
    ResNet50, VGG16, MobileNetV2, EfficientNetB0,
    InceptionV3, DenseNet121
)

# ============================================
# Method 1: Load with ImageNet weights (for feature extraction)
# ============================================
# include_top=False removes the final classification layer
# This gives us just the feature extractor
base_model = ResNet50(
    weights='imagenet',      # Load ImageNet pretrained weights
    include_top=False,       # Remove the 1000-class Dense layer
    input_shape=(224, 224, 3)  # Specify input shape
)

print(f"Output shape: {base_model.output_shape}")
# Output shape: (None, 7, 7, 2048)  ← Spatial feature maps

# ============================================
# Method 2: Load full model for inference
# ============================================
full_model = ResNet50(
    weights='imagenet',
    include_top=True  # Keep the classification head
)

# Make a prediction
from tensorflow.keras.applications.resnet50 import preprocess_input, decode_predictions
from tensorflow.keras.preprocessing import image

img = image.load_img('elephant.jpg', target_size=(224, 224))
x = image.img_to_array(img)
x = np.expand_dims(x, axis=0)   # Add batch dimension: (1, 224, 224, 3)
x = preprocess_input(x)          # Model-specific preprocessing

preds = full_model.predict(x)
results = decode_predictions(preds, top=3)[0]
for _, label, score in results:
    print(f"{label}: {score:.4f}")
# african_elephant: 0.9234
# tusker: 0.0523
# Indian_elephant: 0.0187

# ============================================
# Method 3: Load different models the same way
# ============================================
mobilenet = MobileNetV2(weights='imagenet', include_top=False, input_shape=(224, 224, 3))
efficientnet = EfficientNetB0(weights='imagenet', include_top=False, input_shape=(224, 224, 3))
inception = InceptionV3(weights='imagenet', include_top=False, input_shape=(299, 299, 3))
# Note: InceptionV3 requires 299x299 input!
```

> **Important**: Different models expect different input sizes and preprocessing:
> - ResNet, VGG, MobileNet → 224×224
> - InceptionV3, Xception → 299×299
> - EfficientNetB0 → 224×224, B1 → 240×240, ..., B7 → 600×600
> - Always use the model's own `preprocess_input` function!

---

## Feature Extraction

### Complete Feature Extraction Pipeline

```python
import tensorflow as tf
import numpy as np

# ============================================
# STEP 1: Prepare the data
# ============================================
# Using tf.keras.utils.image_dataset_from_directory for loading images
IMG_SIZE = (224, 224)
BATCH_SIZE = 32

train_ds = tf.keras.utils.image_dataset_from_directory(
    'data/train',             # Directory with subdirectories per class
    image_size=IMG_SIZE,
    batch_size=BATCH_SIZE,
    label_mode='categorical'  # One-hot encoded labels
)

val_ds = tf.keras.utils.image_dataset_from_directory(
    'data/validation',
    image_size=IMG_SIZE,
    batch_size=BATCH_SIZE,
    label_mode='categorical'
)

# Performance optimization
AUTOTUNE = tf.data.AUTOTUNE
train_ds = train_ds.prefetch(buffer_size=AUTOTUNE)
val_ds = val_ds.prefetch(buffer_size=AUTOTUNE)

# ============================================
# STEP 2: Load and freeze the base model
# ============================================
base_model = tf.keras.applications.MobileNetV2(
    weights='imagenet',
    include_top=False,
    input_shape=(224, 224, 3)
)

# FREEZE all layers in the base model
base_model.trainable = False

# Verify: no trainable weights in base model
print(f"Base model trainable variables: {len(base_model.trainable_variables)}")
# Output: 0

# ============================================
# STEP 3: Add preprocessing and classification head
# ============================================
# MobileNetV2 has its own preprocessing
preprocess_input = tf.keras.applications.mobilenet_v2.preprocess_input

# Data augmentation (only applied during training)
data_augmentation = tf.keras.Sequential([
    tf.keras.layers.RandomFlip("horizontal"),
    tf.keras.layers.RandomRotation(0.2),
    tf.keras.layers.RandomZoom(0.2),
])

# Build the full model using Functional API
inputs = tf.keras.Input(shape=(224, 224, 3))
x = data_augmentation(inputs)            # Augment during training
x = preprocess_input(x)                  # Scale pixels to [-1, 1]
x = base_model(x, training=False)        # Extract features (frozen)
x = tf.keras.layers.GlobalAveragePooling2D()(x)  # (batch, 7, 7, 1280) → (batch, 1280)
x = tf.keras.layers.Dropout(0.3)(x)      # Regularization
outputs = tf.keras.layers.Dense(5, activation='softmax')(x)  # 5 classes

model = tf.keras.Model(inputs, outputs)

# ============================================
# STEP 4: Compile and train
# ============================================
model.compile(
    optimizer=tf.keras.optimizers.Adam(learning_rate=1e-3),  # Higher LR is fine
    loss='categorical_crossentropy',
    metrics=['accuracy']
)

model.summary()
# ================================================================
# Total params: 2,261,829
# Trainable params: 3,845        ← Only the classification head!
# Non-trainable params: 2,257,984
# ================================================================

# Train only the new layers
history = model.fit(
    train_ds,
    epochs=10,
    validation_data=val_ds
)
```

### Why GlobalAveragePooling2D?

```
Without GAP:
base_model output: (batch, 7, 7, 2048)
Flatten → (batch, 100352)  ← HUGE! Causes overfitting + slow training

With GlobalAveragePooling2D:
base_model output: (batch, 7, 7, 2048)
GAP → (batch, 2048)        ← Average each 7×7 feature map to a single number

                    ┌───────────┐
Feature Map k:      │ 2  4  1   │
(7×7 simplified     │ 3  5  2   │  → Average = 3.0
 to 3×3)            │ 1  6  3   │
                    └───────────┘
                    
This produces ONE number per feature map.
2048 feature maps → 2048 numbers → Dense layer input
```

> **Pro Tip**: `GlobalAveragePooling2D` is almost always preferred over `Flatten` in transfer learning. It reduces parameters dramatically, acts as a regularizer, and allows the model to accept inputs of any spatial size.

---

## Fine-Tuning

### The Fine-Tuning Process

Fine-tuning involves unfreezing some of the top layers of the frozen base model and jointly training both the newly-added classifier layers and the top layers of the base model.

```
Fine-Tuning Timeline:

Phase 1: Feature Extraction (epochs 1-10)
┌─────────────────────────────┐
│  Base Model    → FROZEN ❄️   │  Learning Rate: 1e-3
│  New Head      → TRAINING 🔥 │  
└─────────────────────────────┘
          ↓
Phase 2: Fine-Tuning (epochs 11-20)  
┌─────────────────────────────┐
│  Lower Layers  → FROZEN ❄️   │  Learning Rate: 1e-5 (100x smaller!)
│  Upper Layers  → UNFROZEN 🔥 │  
│  New Head      → TRAINING 🔥 │  
└─────────────────────────────┘

WHY two phases?
• If you unfreeze everything from the start, the random weights in the 
  new head produce large gradients that DESTROY the pretrained features.
• First train the head to produce reasonable outputs, THEN fine-tune.
```

### Complete Fine-Tuning Code

```python
import tensorflow as tf

# ============================================
# PHASE 1: Feature Extraction (same as before)
# ============================================
base_model = tf.keras.applications.MobileNetV2(
    weights='imagenet',
    include_top=False,
    input_shape=(224, 224, 3)
)
base_model.trainable = False

preprocess_input = tf.keras.applications.mobilenet_v2.preprocess_input

inputs = tf.keras.Input(shape=(224, 224, 3))
x = preprocess_input(inputs)
x = base_model(x, training=False)
x = tf.keras.layers.GlobalAveragePooling2D()(x)
x = tf.keras.layers.Dropout(0.3)(x)
outputs = tf.keras.layers.Dense(5, activation='softmax')(x)

model = tf.keras.Model(inputs, outputs)

model.compile(
    optimizer=tf.keras.optimizers.Adam(learning_rate=1e-3),
    loss='categorical_crossentropy',
    metrics=['accuracy']
)

# Train Phase 1
# history_phase1 = model.fit(train_ds, epochs=10, validation_data=val_ds)

# ============================================
# PHASE 2: Fine-Tuning
# ============================================
# Unfreeze the base model
base_model.trainable = True

# Let's see how many layers are in the base model
print(f"Number of layers in base model: {len(base_model.layers)}")
# Output: 154

# Fine-tune from layer 100 onwards (freeze layers 0-99)
fine_tune_at = 100

for layer in base_model.layers[:fine_tune_at]:
    layer.trainable = False  # Keep these frozen

# Verify
trainable_count = sum(1 for layer in base_model.layers if layer.trainable)
frozen_count = sum(1 for layer in base_model.layers if not layer.trainable)
print(f"Trainable layers: {trainable_count}")
print(f"Frozen layers: {frozen_count}")

# CRITICAL: Recompile with a MUCH lower learning rate
# If learning rate is too high, you'll destroy pretrained features
model.compile(
    optimizer=tf.keras.optimizers.Adam(learning_rate=1e-5),  # 100x smaller!
    loss='categorical_crossentropy',
    metrics=['accuracy']
)

# Continue training (not from epoch 0!)
# history_phase2 = model.fit(
#     train_ds,
#     epochs=20,           # Total epochs
#     initial_epoch=10,    # Resume from where Phase 1 ended
#     validation_data=val_ds
# )
```

### Choosing Which Layers to Unfreeze

```python
# ============================================
# Strategy 1: Unfreeze top N layers
# ============================================
base_model.trainable = True
for layer in base_model.layers[:-20]:  # Freeze all except last 20
    layer.trainable = False

# ============================================
# Strategy 2: Unfreeze specific blocks (ResNet example)
# ============================================
base_model = tf.keras.applications.ResNet50(
    weights='imagenet', include_top=False, input_shape=(224, 224, 3)
)
base_model.trainable = True

# ResNet has blocks named 'conv5_*', 'conv4_*', etc.
# Only unfreeze conv5 block
for layer in base_model.layers:
    if 'conv5' in layer.name:
        layer.trainable = True
    else:
        layer.trainable = False

# ============================================
# Strategy 3: Unfreeze only BatchNorm layers (keep frozen!)
# ============================================
# IMPORTANT: BatchNorm layers should usually stay frozen during fine-tuning
# because the batch statistics are different from pretraining
base_model.trainable = True
for layer in base_model.layers:
    if isinstance(layer, tf.keras.layers.BatchNormalization):
        layer.trainable = False  # Keep BN frozen!

# ============================================
# Print trainable status of all layers
# ============================================
for layer in base_model.layers[-10:]:  # Last 10 layers
    print(f"{layer.name:30s} trainable={layer.trainable}")
```

> **Critical Warning**: When fine-tuning, always keep `training=False` when calling the base model if it contains BatchNorm layers:
> ```python
> x = base_model(x, training=False)  # Not training=True!
> ```
> This ensures BatchNorm uses its **pretrained** running mean/variance statistics, not the batch statistics from your (small) dataset.

### Learning Rate Schedules for Fine-Tuning

```python
# ============================================
# Discriminative Learning Rates (different LR per layer group)
# ============================================
# Lower layers → smaller learning rate (preserve pretrained features)
# Upper layers → larger learning rate (adapt to new task)

# Unfortunately, Keras doesn't natively support per-layer LR.
# Workaround: Use separate optimizers or learning rate multipliers.

# Method: Using a cosine decay schedule
initial_learning_rate = 1e-5
decay_steps = 1000

lr_schedule = tf.keras.optimizers.schedules.CosineDecay(
    initial_learning_rate=initial_learning_rate,
    decay_steps=decay_steps,
    alpha=1e-7  # Minimum learning rate
)

optimizer = tf.keras.optimizers.Adam(learning_rate=lr_schedule)

model.compile(
    optimizer=optimizer,
    loss='categorical_crossentropy',
    metrics=['accuracy']
)
```

---

## Transfer Learning for NLP

### Using Pretrained Text Models

```python
import tensorflow as tf
import tensorflow_hub as hub
import tensorflow_text  # Required for BERT preprocessing

# ============================================
# BERT-based Transfer Learning
# ============================================
# Load BERT preprocessor and encoder
preprocessor_url = "https://tfhub.dev/tensorflow/bert_en_uncased_preprocess/3"
encoder_url = "https://tfhub.dev/tensorflow/small_bert/bert_en_uncased_L-4_H-512_A-8/2"

# Build the model
text_input = tf.keras.layers.Input(shape=(), dtype=tf.string, name='text')

# Preprocessing: tokenize + create input IDs, attention masks
preprocessing_layer = hub.KerasLayer(preprocessor_url, name='preprocessing')
encoder_inputs = preprocessing_layer(text_input)

# BERT encoder: converts tokens → contextual embeddings
encoder = hub.KerasLayer(encoder_url, trainable=True, name='BERT_encoder')
outputs = encoder(encoder_inputs)

# Use the [CLS] token output for classification
net = outputs['pooled_output']  # Shape: (batch_size, 512)
net = tf.keras.layers.Dropout(0.1)(net)
net = tf.keras.layers.Dense(1, activation='sigmoid', name='classifier')(net)

model = tf.keras.Model(text_input, net)

model.compile(
    optimizer=tf.keras.optimizers.Adam(learning_rate=5e-5),  # BERT-specific LR
    loss='binary_crossentropy',
    metrics=['accuracy']
)

model.summary()

# Train on your text data
# model.fit(train_texts, train_labels, epochs=5, validation_split=0.2)
```

### Transfer Learning with Keras NLP (Modern Approach)

```python
# pip install keras-nlp
import keras_nlp

# Load pretrained BERT backbone
backbone = keras_nlp.models.BertBackbone.from_preset("bert_base_en_uncased")

# Create a classifier directly
classifier = keras_nlp.models.BertClassifier.from_preset(
    "bert_base_en_uncased",
    num_classes=4,
    activation='softmax'
)

# Compile and train
classifier.compile(
    optimizer=tf.keras.optimizers.Adam(learning_rate=5e-5),
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

# classifier.fit(train_ds, epochs=3, validation_data=val_ds)
```

---

## Advanced Strategies

### 1. Progressive Resizing

```python
# Train with small images first, then increase resolution
# This acts as a form of regularization and speeds up training

# Phase 1: 128x128 (fast, acts as regularization)
model_128 = build_model(input_shape=(128, 128, 3))
# model_128.fit(train_ds_128, epochs=5)

# Phase 2: 224x224 (full resolution)
# Transfer weights from 128x128 model
model_224 = build_model(input_shape=(224, 224, 3))
# Copy compatible weights
for l1, l2 in zip(model_128.layers, model_224.layers):
    try:
        l2.set_weights(l1.get_weights())
    except ValueError:
        pass  # Skip layers with different shapes (e.g., input layer)

# model_224.fit(train_ds_224, epochs=10)
```

### 2. Knowledge Distillation

```python
import tensorflow as tf

# Teacher: Large, accurate model
teacher_model = tf.keras.applications.EfficientNetB7(
    weights='imagenet', include_top=False, input_shape=(224, 224, 3)
)

# Student: Small, fast model
student_model = tf.keras.applications.MobileNetV2(
    weights=None, include_top=False, input_shape=(224, 224, 3)
)

class Distiller(tf.keras.Model):
    def __init__(self, student, teacher):
        super().__init__()
        self.student = student
        self.teacher = teacher
        self.temperature = 3.0    # Softens probability distributions
        self.alpha = 0.5          # Balance between hard and soft targets
    
    def compile(self, optimizer, student_loss, distillation_loss, metrics):
        super().compile(optimizer=optimizer, metrics=metrics)
        self.student_loss_fn = student_loss
        self.distillation_loss_fn = distillation_loss
    
    def train_step(self, data):
        x, y = data
        
        # Teacher predictions (no gradient needed)
        teacher_preds = self.teacher(x, training=False)
        
        with tf.GradientTape() as tape:
            # Student predictions
            student_preds = self.student(x, training=True)
            
            # Hard label loss (standard cross-entropy)
            student_loss = self.student_loss_fn(y, student_preds)
            
            # Soft label loss (match teacher's soft predictions)
            distillation_loss = self.distillation_loss_fn(
                tf.nn.softmax(teacher_preds / self.temperature, axis=1),
                tf.nn.softmax(student_preds / self.temperature, axis=1)
            ) * (self.temperature ** 2)
            
            # Combined loss
            loss = self.alpha * student_loss + (1 - self.alpha) * distillation_loss
        
        # Update student only
        gradients = tape.gradient(loss, self.student.trainable_variables)
        self.optimizer.apply_gradients(zip(gradients, self.student.trainable_variables))
        
        self.compiled_metrics.update_state(y, student_preds)
        return {m.name: m.result() for m in self.metrics}
```

### 3. Domain Adaptation with Gradual Unfreezing

```python
import tensorflow as tf

def gradual_unfreeze_training(model, base_model, train_ds, val_ds, num_layers_per_stage=20):
    """
    Gradually unfreeze layers from top to bottom.
    This is gentler than unfreezing all at once.
    """
    total_layers = len(base_model.layers)
    num_stages = total_layers // num_layers_per_stage
    
    for stage in range(num_stages):
        # Calculate which layers to unfreeze
        unfreeze_from = total_layers - (stage + 1) * num_layers_per_stage
        unfreeze_from = max(0, unfreeze_from)
        
        # Unfreeze layers
        for layer in base_model.layers[unfreeze_from:]:
            if not isinstance(layer, tf.keras.layers.BatchNormalization):
                layer.trainable = True
        
        # Decrease learning rate as we unfreeze more
        lr = 1e-4 * (0.5 ** stage)
        
        model.compile(
            optimizer=tf.keras.optimizers.Adam(learning_rate=lr),
            loss='categorical_crossentropy',
            metrics=['accuracy']
        )
        
        print(f"\nStage {stage + 1}: Unfreezing from layer {unfreeze_from}, LR={lr}")
        trainable = sum(1 for l in base_model.layers if l.trainable)
        print(f"Trainable layers: {trainable}/{total_layers}")
        
        model.fit(train_ds, epochs=3, validation_data=val_ds)
    
    return model
```

### 4. Multi-Model Feature Extraction (Ensemble)

```python
import tensorflow as tf

def build_multi_backbone_model(num_classes=10):
    """Combine features from multiple pretrained models."""
    
    input_layer = tf.keras.Input(shape=(224, 224, 3))
    
    # Backbone 1: MobileNetV2
    mobilenet = tf.keras.applications.MobileNetV2(
        weights='imagenet', include_top=False, input_shape=(224, 224, 3)
    )
    mobilenet.trainable = False
    x1 = mobilenet(input_layer)
    x1 = tf.keras.layers.GlobalAveragePooling2D()(x1)  # (batch, 1280)
    
    # Backbone 2: ResNet50
    resnet = tf.keras.applications.ResNet50(
        weights='imagenet', include_top=False, input_shape=(224, 224, 3)
    )
    resnet.trainable = False
    x2 = resnet(input_layer)
    x2 = tf.keras.layers.GlobalAveragePooling2D()(x2)  # (batch, 2048)
    
    # Concatenate features from both models
    combined = tf.keras.layers.Concatenate()([x1, x2])  # (batch, 3328)
    
    x = tf.keras.layers.Dense(256, activation='relu')(combined)
    x = tf.keras.layers.Dropout(0.4)(x)
    outputs = tf.keras.layers.Dense(num_classes, activation='softmax')(x)
    
    model = tf.keras.Model(input_layer, outputs)
    return model

# multi_model = build_multi_backbone_model(num_classes=10)
# multi_model.summary()
```

---

## Common Mistakes

### Mistake 1: Not Freezing the Base Model Before Training the Head

```python
# ❌ WRONG: Training everything from the start
base_model = tf.keras.applications.ResNet50(weights='imagenet', include_top=False)
# base_model.trainable = True  ← Default! Everything is trainable!
# The random head produces large gradients → DESTROYS pretrained weights

# ✅ CORRECT: Freeze first, train head, then optionally fine-tune
base_model = tf.keras.applications.ResNet50(weights='imagenet', include_top=False)
base_model.trainable = False  # Freeze!
# Train head first, then unfreeze upper layers with small LR
```

### Mistake 2: Using Too High a Learning Rate When Fine-Tuning

```python
# ❌ WRONG: Same learning rate as training from scratch
model.compile(optimizer=tf.keras.optimizers.Adam(learning_rate=1e-3), ...)

# ✅ CORRECT: Use 10-100x smaller learning rate for fine-tuning
model.compile(optimizer=tf.keras.optimizers.Adam(learning_rate=1e-5), ...)
```

### Mistake 3: Not Using the Correct Preprocessing

```python
# ❌ WRONG: Generic normalization
x = x / 255.0  # Simple [0, 1] scaling

# ✅ CORRECT: Use model-specific preprocessing
from tensorflow.keras.applications.resnet50 import preprocess_input
x = preprocess_input(x)  # Handles mean subtraction, channel ordering, etc.

# Each model family has its own:
# - ResNet, VGG: caffe-style (BGR, mean subtraction)
# - MobileNet, NASNet: tf-style ([-1, 1] scaling)
# - EfficientNet: expects raw [0, 255] input (no preprocessing!)
```

### Mistake 4: Not Keeping BatchNorm Frozen

```python
# ❌ WRONG: Calling base_model with training=True
x = base_model(inputs, training=True)  # BN uses batch stats → bad!

# ✅ CORRECT: Always use training=False for pretrained BN
x = base_model(inputs, training=False)  # BN uses pretrained running stats
```

### Mistake 5: Wrong Input Size

```python
# ❌ WRONG: Using 224x224 for InceptionV3
inception = InceptionV3(weights='imagenet', include_top=False, input_shape=(224, 224, 3))
# Will raise an error or produce garbage results!

# ✅ CORRECT: Use 299x299 for InceptionV3
inception = InceptionV3(weights='imagenet', include_top=False, input_shape=(299, 299, 3))
```

### Mistake 6: Forgetting to Recompile After Changing Trainability

```python
# ❌ WRONG: Changing trainability without recompiling
base_model.trainable = True
# model.fit(...)  ← Optimizer still has the old graph!

# ✅ CORRECT: Always recompile after changing trainability
base_model.trainable = True
model.compile(
    optimizer=tf.keras.optimizers.Adam(learning_rate=1e-5),
    loss='categorical_crossentropy',
    metrics=['accuracy']
)
# model.fit(...)  ← Now the optimizer knows about new trainable variables
```

---

## Interview Questions

### Q1: What is transfer learning and why is it useful?
**A**: Transfer learning reuses a model trained on a large dataset (source task) as a starting point for a different task (target task). It's useful because:
1. Reduces training time dramatically
2. Works well with small datasets (prevents overfitting)
3. Leverages general features (edges, textures) learned from millions of images
4. Achieves better performance than training from scratch in most practical scenarios

### Q2: What's the difference between feature extraction and fine-tuning?
**A**: 
- **Feature extraction**: Freeze all pretrained layers, only train the new classification head. The pretrained model acts as a fixed feature extractor. Best for small datasets or when domains are similar.
- **Fine-tuning**: Unfreeze some top layers of the pretrained model and train them jointly with the new head using a very small learning rate. Best when you have more data and need to adapt features to your specific domain.

### Q3: Why should you use a smaller learning rate when fine-tuning?
**A**: Pretrained weights are already well-optimized for feature extraction. A large learning rate would cause large gradient updates that destroy these carefully learned representations. A small learning rate (typically 1e-5) makes gentle adjustments to adapt features to the new task without catastrophic forgetting.

### Q4: What is catastrophic forgetting?
**A**: When a neural network trained on task A is subsequently trained on task B, it may completely forget how to perform task A. In transfer learning, this means the pretrained features get overwritten by the new task's gradients. Mitigation strategies:
- Use small learning rates
- Freeze lower layers
- Use gradual unfreezing
- Elastic Weight Consolidation (EWC)

### Q5: Why do we keep BatchNormalization layers frozen during fine-tuning?
**A**: BatchNorm layers maintain running mean and variance statistics from pretraining (computed over millions of images). During fine-tuning on a small dataset, batch statistics would be noisy and unrepresentative, causing the model to produce poor outputs. Keeping BN frozen ensures it uses the stable pretrained statistics.

### Q6: When would transfer learning NOT work well?
**A**:
- When source and target domains are completely unrelated (e.g., ImageNet → audio spectrograms)
- When the target dataset is very large and diverse (training from scratch may match performance)
- When the task requires fundamentally different features than what the source model learned
- This is called **negative transfer**

### Q7: How do you decide how many layers to fine-tune?
**A**: It depends on:
- **Dataset size**: More data → can fine-tune more layers
- **Domain similarity**: Similar domains → fine-tune fewer layers; different domains → fine-tune more
- **Rule of thumb**: Start with feature extraction, then gradually unfreeze top layers
- **Monitor validation loss**: If it increases, you're unfreezing too many layers or using too high a LR

### Q8: Explain knowledge distillation.
**A**: Knowledge distillation trains a small "student" model to mimic a large "teacher" model. The student learns from the teacher's soft probability outputs (softmax with temperature) rather than just hard labels. The soft outputs contain richer information about class relationships (e.g., the teacher might say "this cat image is 0.7 cat, 0.2 tiger, 0.1 lion" — teaching the student about animal similarity). The temperature parameter controls how "soft" the probability distribution is.

$$\text{soft\_targets} = \text{softmax}\left(\frac{z_i}{T}\right) = \frac{e^{z_i/T}}{\sum_j e^{z_j/T}}$$

---

## Quick Reference

### Transfer Learning Cheat Sheet

| Aspect | Feature Extraction | Fine-Tuning |
|--------|-------------------|-------------|
| Base model layers | All frozen | Top layers unfrozen |
| Learning rate | 1e-3 (normal) | 1e-5 (very small) |
| Training time | Fast | Slower |
| Data needed | Less (100s) | More (1000s) |
| Risk of overfitting | Lower | Higher |
| Accuracy potential | Good | Better |
| When to use | Small data, quick results | More data, need max accuracy |

### Model Selection Quick Guide

| Need | Model | Input Size |
|------|-------|------------|
| Mobile deployment | MobileNetV2/V3 | 224×224 |
| Best accuracy | EfficientNetB4-B7 | 380-600 |
| General purpose | ResNet50 / EfficientNetB0 | 224×224 |
| Multi-scale features | InceptionV3 | 299×299 |
| Feature reuse | DenseNet121 | 224×224 |

### Key Code Patterns

```python
# Feature Extraction
base = tf.keras.applications.ResNet50(weights='imagenet', include_top=False)
base.trainable = False
model = tf.keras.Sequential([base, GlobalAveragePooling2D(), Dense(N, 'softmax')])
model.compile(optimizer=Adam(1e-3), loss='categorical_crossentropy')

# Fine-Tuning (after training head)
base.trainable = True
for layer in base.layers[:100]: layer.trainable = False
model.compile(optimizer=Adam(1e-5), loss='categorical_crossentropy')  # Recompile!
```

### Preprocessing Functions

| Model Family | Preprocessing | Pixel Range |
|-------------|---------------|-------------|
| VGG, ResNet | `resnet50.preprocess_input` | BGR, mean subtracted |
| MobileNet | `mobilenet_v2.preprocess_input` | [-1, 1] |
| Inception, Xception | `inception_v3.preprocess_input` | [-1, 1] |
| EfficientNet | None (raw pixels) | [0, 255] |

---

*Next Chapter: [08-Custom-Training-Loops](./08-Custom-Training-Loops.md)*
