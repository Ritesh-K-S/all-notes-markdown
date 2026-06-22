# Chapter 02: Keras — Sequential and Functional API

## Table of Contents
- [1. What is Keras?](#1-what-is-keras)
- [2. Sequential API — Simple Stack of Layers](#2-sequential-api--simple-stack-of-layers)
- [3. Understanding Layers](#3-understanding-layers)
- [4. Dense (Fully Connected) Layer — Deep Dive](#4-dense-fully-connected-layer--deep-dive)
- [5. Activation Functions](#5-activation-functions)
- [6. Model Summary and Visualization](#6-model-summary-and-visualization)
- [7. Functional API — Flexible Model Building](#7-functional-api--flexible-model-building)
- [8. Multi-Input and Multi-Output Models](#8-multi-input-and-multi-output-models)
- [9. Shared Layers and Reuse](#9-shared-layers-and-reuse)
- [10. Model Subclassing — Full Control](#10-model-subclassing--full-control)
- [11. Sequential vs Functional vs Subclassing — When to Use What](#11-sequential-vs-functional-vs-subclassing--when-to-use-what)
- [12. Layer Weights and Initialization](#12-layer-weights-and-initialization)
- [13. Custom Layers](#13-custom-layers)
- [14. Common Mistakes](#14-common-mistakes)
- [15. Interview Questions](#15-interview-questions)
- [16. Quick Reference](#16-quick-reference)

---

## 1. What is Keras?

### Simple Explanation
If TensorFlow is a **car engine** (powerful but complex), Keras is the **steering wheel and dashboard** — it gives you a simple, intuitive interface to control the engine.

Keras is a **high-level API** for building and training deep learning models. Since TensorFlow 2.x, Keras is fully integrated as `tf.keras` — you don't install it separately.

### Why It Matters
- **Simplicity**: Build a neural network in 5 lines of code
- **Flexibility**: Three ways to build models (Sequential, Functional, Subclassing) — from simple to complex
- **Production-ready**: Same API for prototyping and deployment
- **Ecosystem**: Pre-built layers, optimizers, losses, metrics, callbacks — everything you need
- **Industry standard**: Used by Google, Netflix, CERN, NASA

### Keras 3 vs tf.keras

```
┌─────────────────────────────────────────────────────┐
│ Keras History                                        │
├─────────────────────────────────────────────────────┤
│ Keras (standalone)  → Original by François Chollet   │
│ tf.keras (TF 2.x)   → Keras integrated into TF      │
│ Keras 3 (2024+)     → Multi-backend: TF, JAX, PyTorch│
└─────────────────────────────────────────────────────┘

For this guide, we use tf.keras (the TensorFlow-integrated version).
If you're using Keras 3, the API is nearly identical.
```

```python
# Import Keras from TensorFlow
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers

# Verify
print(tf.keras.__version__)
```

---

## 2. Sequential API — Simple Stack of Layers

### What It Is
The Sequential API lets you build models by **stacking layers in order**, one after another. Data flows from the first layer → second layer → ... → last layer.

**Analogy**: Building a sandwich — you stack ingredients one on top of another. Bread → cheese → lettuce → tomato → bread. Data flows through each "ingredient."

### When to Use
- Model has **one input** and **one output**
- Layers are connected **linearly** (no branches, no skip connections)
- Most beginner models and many production models

### How It Works

```
Input → [Layer 1] → [Layer 2] → [Layer 3] → Output

Example: Image Classification
(784,) → [Dense 128] → [Dense 64] → [Dense 10] → predictions
```

```python
import tensorflow as tf
from tensorflow.keras import layers, models

# ============================================
# METHOD 1: Pass list of layers to Sequential
# ============================================

model = models.Sequential([
    layers.Dense(128, activation='relu', input_shape=(784,)),  # Layer 1
    layers.Dense(64, activation='relu'),                        # Layer 2
    layers.Dense(10, activation='softmax')                      # Output layer
])

# ============================================
# METHOD 2: Add layers one by one (more flexible)
# ============================================

model = models.Sequential()
model.add(layers.Dense(128, activation='relu', input_shape=(784,)))
model.add(layers.Dense(64, activation='relu'))
model.add(layers.Dense(10, activation='softmax'))

# ============================================
# METHOD 3: Using Input layer explicitly
# ============================================

model = models.Sequential([
    layers.Input(shape=(784,)),                    # Explicit input shape
    layers.Dense(128, activation='relu'),
    layers.Dense(64, activation='relu'),
    layers.Dense(10, activation='softmax')
])

# View model architecture
model.summary()
```

**Output of `model.summary()`**:

```
Model: "sequential"
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┓
┃ Layer (type)                 ┃ Output Shape            ┃ Param #         ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━┩
│ dense (Dense)                │ (None, 128)             │ 100,480         │
│ dense_1 (Dense)              │ (None, 64)              │ 8,256           │
│ dense_2 (Dense)              │ (None, 10)              │ 650             │
└─────────────────────────────┴─────────────────────────┴─────────────────┘
 Total params: 109,386
 Trainable params: 109,386
 Non-trainable params: 0
```

### Understanding Parameter Count

```
Dense Layer Parameters = (input_features × output_units) + output_units
                       = (input × output) + bias

Layer 1: (784 × 128) + 128 = 100,352 + 128 = 100,480
Layer 2: (128 × 64)  + 64  = 8,192   + 64  = 8,256
Layer 3: (64 × 10)   + 10  = 640     + 10  = 650

Total: 100,480 + 8,256 + 650 = 109,386 parameters

Visual:
  Input (784) ──→ Dense (128) ──→ Dense (64) ──→ Dense (10)
                   │                │               │
                   │ 784×128        │ 128×64        │ 64×10
                   │ weights        │ weights       │ weights
                   │ +128 biases    │ +64 biases    │ +10 biases
```

### Complete Working Example

```python
import tensorflow as tf
from tensorflow.keras import layers, models
import numpy as np

# ============================================
# COMPLETE EXAMPLE: MNIST Digit Classification
# ============================================

# 1. Load data
(x_train, y_train), (x_test, y_test) = tf.keras.datasets.mnist.load_data()

# 2. Preprocess
x_train = x_train.reshape(-1, 784).astype('float32') / 255.0  # Flatten + normalize
x_test = x_test.reshape(-1, 784).astype('float32') / 255.0

print(f"Training data: {x_train.shape}")   # (60000, 784)
print(f"Test data: {x_test.shape}")        # (10000, 784)
print(f"Labels: {y_train.shape}")          # (60000,) — integers 0-9

# 3. Build model (Sequential)
model = models.Sequential([
    layers.Dense(256, activation='relu', input_shape=(784,)),
    layers.Dropout(0.3),        # Regularization: randomly drop 30% of neurons
    layers.Dense(128, activation='relu'),
    layers.Dropout(0.2),
    layers.Dense(10, activation='softmax')  # 10 classes, probabilities
])

# 4. Compile (configure training)
model.compile(
    optimizer='adam',                              # Optimization algorithm
    loss='sparse_categorical_crossentropy',        # Loss for integer labels
    metrics=['accuracy']                           # What to track
)

# 5. Train
history = model.fit(
    x_train, y_train,
    epochs=10,
    batch_size=128,
    validation_split=0.2  # Use 20% of training data for validation
)

# 6. Evaluate
test_loss, test_acc = model.evaluate(x_test, y_test)
print(f"\nTest accuracy: {test_acc:.4f}")  # ~97-98%

# 7. Predict
predictions = model.predict(x_test[:5])
print(f"\nPredictions (probabilities):\n{predictions[0]}")
print(f"Predicted classes: {np.argmax(predictions, axis=1)}")
print(f"Actual classes: {y_test[:5]}")
```

---

## 3. Understanding Layers

### What It Is
Layers are the **building blocks** of neural networks. Each layer:
1. Takes an input tensor
2. Applies a transformation (weights + activation)
3. Outputs a tensor

**Analogy**: Think of layers like filters in photo editing. Each filter transforms the image in a specific way (brightness, contrast, blur). Stacking filters creates complex transformations.

### Common Layer Types

| Layer | What It Does | Use Case |
|-------|-------------|----------|
| `Dense` | Fully connected — every input connects to every output | Classification, regression |
| `Conv2D` | Slides filters over 2D input | Image recognition |
| `MaxPooling2D` | Reduces spatial dimensions by taking max | Downsampling images |
| `Flatten` | Converts multi-dim to 1D | Before Dense after Conv2D |
| `Dropout` | Randomly sets neurons to 0 | Regularization |
| `BatchNormalization` | Normalizes layer outputs | Faster, more stable training |
| `Embedding` | Maps integers to dense vectors | NLP, categorical features |
| `LSTM` / `GRU` | Processes sequences | Time series, text |
| `Input` | Defines input shape | Required for Functional API |

### Layer Details

```python
import tensorflow as tf
from tensorflow.keras import layers

# ============================================
# DENSE LAYER (Fully Connected)
# ============================================
# Formula: output = activation(input @ W + b)

dense = layers.Dense(
    units=128,              # Number of output neurons
    activation='relu',      # Activation function
    use_bias=True,          # Include bias term (default: True)
    kernel_initializer='glorot_uniform',  # Weight initialization (default)
    bias_initializer='zeros',             # Bias initialization (default)
    kernel_regularizer=None,              # Weight regularization (e.g., L2)
    name='my_dense_layer'                 # Layer name
)

# ============================================
# FLATTEN LAYER
# ============================================
# Converts (batch, height, width, channels) → (batch, height*width*channels)
flatten = layers.Flatten()

x = tf.random.normal((32, 28, 28, 1))   # Batch of 32 grayscale images
print(f"Before flatten: {x.shape}")       # (32, 28, 28, 1)
print(f"After flatten: {flatten(x).shape}")  # (32, 784)

# ============================================
# DROPOUT LAYER
# ============================================
# Randomly sets a fraction of inputs to 0 during TRAINING
# Does nothing during INFERENCE (prediction)

dropout = layers.Dropout(rate=0.5)  # Drop 50% of neurons

x = tf.ones((1, 10))
# During training:
train_output = dropout(x, training=True)
print(f"Dropout (training): {train_output}")  # Some values are 0

# During inference:
test_output = dropout(x, training=False)
print(f"Dropout (inference): {test_output}")  # All values preserved

# ============================================
# BATCH NORMALIZATION
# ============================================
# Normalizes outputs to mean≈0, std≈1
# Makes training faster and more stable

bn = layers.BatchNormalization()

x = tf.random.normal((32, 128))       # Batch of 32 samples, 128 features
normalized = bn(x, training=True)
print(f"Before BN - mean: {tf.reduce_mean(x):.4f}, std: {tf.math.reduce_std(x):.4f}")
print(f"After BN  - mean: {tf.reduce_mean(normalized):.4f}, std: {tf.math.reduce_std(normalized):.4f}")

# ============================================
# RESHAPE LAYER
# ============================================

reshape = layers.Reshape(target_shape=(7, 7, 1))  # Reshape to image-like
x = tf.random.normal((32, 49))
print(f"Reshape: {x.shape} → {reshape(x).shape}")  # (32, 49) → (32, 7, 7, 1)
```

---

## 4. Dense (Fully Connected) Layer — Deep Dive

### What It Is
The Dense layer is the **most fundamental** layer in deep learning. Every input neuron is connected to every output neuron — hence "fully connected."

### How It Works

```
Mathematically:
    output = activation(input · W + b)

Where:
    input:  (batch_size, input_features)   e.g., (32, 784)
    W:      (input_features, units)        e.g., (784, 128)
    b:      (units,)                       e.g., (128,)
    output: (batch_size, units)            e.g., (32, 128)

Visually (3 inputs → 4 outputs):

    Input        Weights         Output
    ┌───┐                        ┌───┐
    │ x₁├───w₁₁──────────────→ │ y₁│
    │   ├───w₁₂──┐              │   │
    │   ├───w₁₃──┤──────────→  │ y₂│
    ├───┤   ···   │              ├───┤
    │ x₂├───w₂₁──┤──────────→  │ y₃│
    │   ├───w₂₂──┘              │   │
    │   ├───w₂₃──────────────→ │ y₄│
    ├───┤                        ├───┤
    │ x₃├───(connected to all)   │   │
    └───┘                        └───┘

    Every input is connected to every output via a weight.
    Each output also has a bias term added.
```

```python
import tensorflow as tf
from tensorflow.keras import layers
import numpy as np

# ============================================
# UNDERSTANDING DENSE LAYER INTERNALS
# ============================================

# Create a simple Dense layer
dense = layers.Dense(4, activation='relu', input_shape=(3,))

# Build it (creates weights)
dense.build(input_shape=(None, 3))

# Inspect weights
weights, biases = dense.get_weights()
print(f"Weight matrix shape: {weights.shape}")  # (3, 4) — 3 inputs × 4 outputs
print(f"Bias vector shape: {biases.shape}")      # (4,)   — one per output

print(f"\nWeights:\n{weights}")
print(f"\nBiases: {biases}")  # Initialized to zeros by default

# ============================================
# MANUAL COMPUTATION (proving Dense layer math)
# ============================================

# Set known weights for demonstration
dense.set_weights([
    np.array([[0.1, 0.2, 0.3, 0.4],    # Weights from input 1 to outputs
              [0.5, 0.6, 0.7, 0.8],     # Weights from input 2 to outputs
              [0.9, 1.0, 1.1, 1.2]]),    # Weights from input 3 to outputs
    np.array([0.01, 0.02, 0.03, 0.04])  # Biases
])

# Input
x = tf.constant([[1.0, 2.0, 3.0]])  # shape (1, 3)

# Dense layer computation
output = dense(x)
print(f"\nDense output: {output.numpy()}")

# Manual computation: relu(x @ W + b)
manual = x @ dense.kernel + dense.bias  # Before activation
manual_relu = tf.nn.relu(manual)         # After ReLU
print(f"Manual output: {manual_relu.numpy()}")
# They match!

# Step by step:
# x @ W = [1*0.1+2*0.5+3*0.9, 1*0.2+2*0.6+3*1.0, 1*0.3+2*0.7+3*1.1, 1*0.4+2*0.8+3*1.2]
#        = [3.8, 4.4, 5.0, 5.6]
# + bias = [3.81, 4.42, 5.03, 5.64]
# relu   = [3.81, 4.42, 5.03, 5.64]  (all positive, so no change)
```

---

## 5. Activation Functions

### What It Is
Activation functions add **non-linearity** to neural networks. Without them, stacking multiple layers would be equivalent to a single linear transformation — no matter how many layers you stack.

**Analogy**: A neural network without activation functions is like a recipe that only uses one ingredient. Activation functions are the "spices" that give the network its ability to learn complex patterns.

### Why It Matters

```
Without activation: y = W₃(W₂(W₁·x)) = (W₃·W₂·W₁)·x = W·x
                    → Just a linear transformation, no matter how many layers!

With activation: y = f₃(W₃ · f₂(W₂ · f₁(W₁·x)))
                 → Can approximate ANY function! (Universal Approximation Theorem)
```

### Common Activation Functions

```python
import tensorflow as tf
import numpy as np

# ============================================
# ReLU (Rectified Linear Unit) — MOST POPULAR
# ============================================
# f(x) = max(0, x)
# Output: [0, ∞)
# Use: Hidden layers (default choice)

x = tf.constant([-3.0, -1.0, 0.0, 1.0, 3.0])
print(f"ReLU: {tf.nn.relu(x).numpy()}")  # [0. 0. 0. 1. 3.]

# Or in a layer:
dense = tf.keras.layers.Dense(64, activation='relu')

# ============================================
# SIGMOID
# ============================================
# f(x) = 1 / (1 + e^(-x))
# Output: (0, 1)
# Use: Binary classification OUTPUT layer, gates in LSTM

print(f"Sigmoid: {tf.nn.sigmoid(x).numpy()}")
# [0.047, 0.269, 0.5, 0.731, 0.953]

# ============================================
# SOFTMAX
# ============================================
# f(xᵢ) = e^(xᵢ) / Σ(e^(xⱼ))
# Output: (0, 1) for each element, sum = 1
# Use: Multi-class classification OUTPUT layer

logits = tf.constant([2.0, 1.0, 0.5])
print(f"Softmax: {tf.nn.softmax(logits).numpy()}")
# [0.593, 0.242, 0.165] — probabilities summing to 1.0

# ============================================
# TANH (Hyperbolic Tangent)
# ============================================
# f(x) = (e^x - e^(-x)) / (e^x + e^(-x))
# Output: (-1, 1)
# Use: RNN hidden states, when you need negative outputs

print(f"Tanh: {tf.nn.tanh(x).numpy()}")
# [-0.995, -0.762, 0.0, 0.762, 0.995]

# ============================================
# LEAKY ReLU — Fixes "dying ReLU" problem
# ============================================
# f(x) = x if x > 0, else alpha * x
# Output: (-∞, ∞)
# Use: When ReLU neurons are "dying" (always outputting 0)

print(f"Leaky ReLU (α=0.1): {tf.nn.leaky_relu(x, alpha=0.1).numpy()}")
# [-0.3, -0.1, 0.0, 1.0, 3.0]

# As a layer:
leaky = tf.keras.layers.LeakyReLU(negative_slope=0.1)

# ============================================
# ELU (Exponential Linear Unit)
# ============================================
# f(x) = x if x > 0, else α(e^x - 1)
# Use: Can produce negative outputs, smoother than ReLU

print(f"ELU: {tf.nn.elu(x).numpy()}")

# ============================================
# SWISH / SiLU — Modern, used in EfficientNet
# ============================================
# f(x) = x · sigmoid(x)
# Use: Deep networks, often outperforms ReLU

print(f"Swish: {tf.nn.swish(x).numpy()}")

# ============================================
# GELU — Used in Transformers (BERT, GPT)
# ============================================
# f(x) = x · Φ(x), where Φ is the standard normal CDF
# Use: Transformer architectures

print(f"GELU: {tf.nn.gelu(x).numpy()}")
```

### Activation Function Cheat Sheet

| Activation | Formula | Range | Use Where | Pros | Cons |
|-----------|---------|-------|-----------|------|------|
| **ReLU** | $\max(0, x)$ | $[0, \infty)$ | Hidden layers (default) | Fast, simple, no vanishing gradient | Dead neurons |
| **Sigmoid** | $\frac{1}{1+e^{-x}}$ | $(0, 1)$ | Binary output | Probability interpretation | Vanishing gradient |
| **Softmax** | $\frac{e^{x_i}}{\sum e^{x_j}}$ | $(0, 1)$, sum=1 | Multi-class output | Probability distribution | Expensive |
| **Tanh** | $\frac{e^x - e^{-x}}{e^x + e^{-x}}$ | $(-1, 1)$ | RNNs, hidden layers | Zero-centered | Vanishing gradient |
| **Leaky ReLU** | $\max(\alpha x, x)$ | $(-\infty, \infty)$ | When ReLUs die | No dead neurons | Extra hyperparameter |
| **Swish** | $x \cdot \sigma(x)$ | $(-0.28, \infty)$ | Deep networks | Smooth, often better than ReLU | Slower |
| **GELU** | $x \cdot \Phi(x)$ | $\approx(-0.17, \infty)$ | Transformers | Works great with attention | Slower |

### Choosing the Right Activation

```
OUTPUT LAYER:
├── Binary classification      → sigmoid  (1 neuron)
├── Multi-class classification  → softmax  (N neurons, one per class)
├── Multi-label classification  → sigmoid  (N neurons, independent)
└── Regression                  → linear/None (1 or more neurons)

HIDDEN LAYERS:
├── Default choice              → ReLU
├── ReLU neurons dying          → Leaky ReLU or ELU
├── Transformer architecture    → GELU
├── Very deep network           → Swish
└── RNN/LSTM gates              → sigmoid + tanh (built-in)
```

---

## 6. Model Summary and Visualization

### How It Works

```python
import tensorflow as tf
from tensorflow.keras import layers, models

model = models.Sequential([
    layers.Input(shape=(28, 28, 1)),           # Grayscale images
    layers.Conv2D(32, (3, 3), activation='relu'),
    layers.MaxPooling2D((2, 2)),
    layers.Conv2D(64, (3, 3), activation='relu'),
    layers.MaxPooling2D((2, 2)),
    layers.Flatten(),
    layers.Dense(64, activation='relu'),
    layers.Dropout(0.5),
    layers.Dense(10, activation='softmax')
])

# ============================================
# MODEL SUMMARY
# ============================================
model.summary()

# ============================================
# PROGRAMMATIC ACCESS TO MODEL INFO
# ============================================
print(f"\nModel name: {model.name}")
print(f"Number of layers: {len(model.layers)}")
print(f"Total parameters: {model.count_params()}")

# Iterate through layers
for i, layer in enumerate(model.layers):
    print(f"\nLayer {i}: {layer.name}")
    print(f"  Type: {layer.__class__.__name__}")
    print(f"  Output shape: {layer.output_shape}")
    print(f"  Parameters: {layer.count_params()}")
    if layer.get_weights():
        for w in layer.weights:
            print(f"  Weight: {w.name}, shape: {w.shape}")

# ============================================
# SAVE MODEL ARCHITECTURE TO JSON
# ============================================
json_config = model.to_json()
# Recreate from JSON:
# new_model = models.model_from_json(json_config)

# ============================================
# VISUAL PLOT (saves as image)
# ============================================
# Requires: pip install pydot graphviz
# Also install Graphviz system package

tf.keras.utils.plot_model(
    model,
    to_file='model_architecture.png',
    show_shapes=True,           # Show input/output shapes
    show_layer_names=True,      # Show layer names
    show_layer_activations=True,  # Show activation functions
    show_dtype=False,
    rankdir='TB'                # TB=top-to-bottom, LR=left-to-right
)
```

---

## 7. Functional API — Flexible Model Building

### What It Is
The Functional API lets you build **any model architecture** — including models with multiple inputs, multiple outputs, shared layers, skip connections, and branches. It treats layers as **functions** that you call on tensors.

**Analogy**: Sequential is like a **straight highway** — you can only go forward. Functional API is like a **city road network** — you can have intersections, roundabouts, merging roads, and parallel streets.

### When to Use
- Model has **multiple inputs** or **multiple outputs**
- Model has **skip connections** (ResNet)
- Model has **shared layers** (Siamese networks)
- Any **non-linear topology**

### How It Works

```
Sequential:        Functional:
  Layer 1            Input
    ↓                 ↓
  Layer 2         ┌── Layer A ──┐
    ↓             │             │
  Layer 3         Layer B    Layer C
    ↓             │             │
  Output          └── Merge ────┘
                       ↓
                     Output
```

### Basic Functional API

```python
import tensorflow as tf
from tensorflow.keras import layers, models

# ============================================
# BASIC FUNCTIONAL API MODEL
# ============================================

# Step 1: Define input
inputs = layers.Input(shape=(784,), name='input_layer')

# Step 2: Chain layers as function calls
x = layers.Dense(256, activation='relu', name='hidden_1')(inputs)
x = layers.Dropout(0.3)(x)
x = layers.Dense(128, activation='relu', name='hidden_2')(x)
x = layers.Dropout(0.2)(x)

# Step 3: Define output
outputs = layers.Dense(10, activation='softmax', name='output')(x)

# Step 4: Create model by specifying inputs and outputs
model = models.Model(inputs=inputs, outputs=outputs, name='mnist_classifier')

model.summary()

# Compile and train exactly the same way as Sequential
model.compile(optimizer='adam',
              loss='sparse_categorical_crossentropy',
              metrics=['accuracy'])
```

### Skip Connections (ResNet-style)

```python
# ============================================
# SKIP CONNECTION (Residual Connection)
# ============================================
# The key idea behind ResNet — add input directly to output
# Helps with vanishing gradient in very deep networks

# Formula: output = F(x) + x  (learn the RESIDUAL, not the mapping)

def residual_block(x, units):
    """A basic residual block."""
    shortcut = x  # Save input for skip connection
    
    # Main path
    x = layers.Dense(units, activation='relu')(x)
    x = layers.BatchNormalization()(x)
    x = layers.Dense(units)(x)  # No activation before addition
    x = layers.BatchNormalization()(x)
    
    # Skip connection: add input to output
    # If dimensions don't match, project the shortcut
    if shortcut.shape[-1] != units:
        shortcut = layers.Dense(units)(shortcut)
    
    x = layers.Add()([x, shortcut])  # Element-wise addition
    x = layers.Activation('relu')(x)  # Activation AFTER addition
    return x

# Build model with residual blocks
inputs = layers.Input(shape=(784,))
x = layers.Dense(256, activation='relu')(inputs)

# Stack residual blocks
x = residual_block(x, 256)
x = residual_block(x, 256)
x = residual_block(x, 128)

outputs = layers.Dense(10, activation='softmax')(x)

model = models.Model(inputs=inputs, outputs=outputs, name='resnet_style')
model.summary()

# ============================================
# WHY SKIP CONNECTIONS WORK
# ============================================
# 
# Without skip: the layer must learn y = F(x)
# With skip:    the layer must learn y = F(x) + x
#               which means F(x) = y - x (the RESIDUAL)
#
# Learning the residual is EASIER because:
# 1. If the layer is not needed, F(x) can learn to be 0 → y = x (identity)
# 2. Gradients flow directly through the skip connection (no vanishing)
# 3. Makes it possible to train 100+ layer networks
```

### Branching Architecture

```python
# ============================================
# BRANCHING: Process input through multiple paths
# ============================================

inputs = layers.Input(shape=(784,))

# Branch 1: Wide and shallow
branch1 = layers.Dense(512, activation='relu')(inputs)
branch1 = layers.Dense(256, activation='relu')(branch1)

# Branch 2: Narrow and deep
branch2 = layers.Dense(64, activation='relu')(inputs)
branch2 = layers.Dense(64, activation='relu')(branch2)
branch2 = layers.Dense(64, activation='relu')(branch2)
branch2 = layers.Dense(256, activation='relu')(branch2)

# Merge branches
merged = layers.Concatenate()([branch1, branch2])  # (None, 512)
merged = layers.Dense(128, activation='relu')(merged)
outputs = layers.Dense(10, activation='softmax')(merged)

model = models.Model(inputs=inputs, outputs=outputs, name='branched_model')
model.summary()
```

---

## 8. Multi-Input and Multi-Output Models

### What It Is
Real-world models often need multiple data sources (multi-input) or predict multiple things at once (multi-output).

### Multi-Input Example

```python
# ============================================
# MULTI-INPUT: House Price Prediction
# ============================================
# Input 1: Numerical features (size, bedrooms, age)
# Input 2: Text description (embedded)

# Numerical input
num_input = layers.Input(shape=(5,), name='numerical_features')
x1 = layers.Dense(32, activation='relu')(num_input)
x1 = layers.Dense(16, activation='relu')(x1)

# Text input (already tokenized and padded)
text_input = layers.Input(shape=(100,), name='text_description')
x2 = layers.Embedding(input_dim=10000, output_dim=64)(text_input)
x2 = layers.GlobalAveragePooling1D()(x2)
x2 = layers.Dense(32, activation='relu')(x2)

# Merge both inputs
combined = layers.Concatenate()([x1, x2])
combined = layers.Dense(64, activation='relu')(combined)
combined = layers.Dropout(0.3)(combined)

# Output: single price prediction
price_output = layers.Dense(1, name='price')(combined)  # Regression, no activation

model = models.Model(
    inputs=[num_input, text_input],
    outputs=price_output,
    name='house_price_model'
)

model.summary()

# Compile
model.compile(optimizer='adam', loss='mse', metrics=['mae'])

# Training with multiple inputs — pass as list or dict
import numpy as np

# Dummy data
num_data = np.random.rand(1000, 5)
text_data = np.random.randint(0, 10000, (1000, 100))
prices = np.random.rand(1000, 1) * 500000

# Method 1: Pass as list (order must match Input order)
model.fit([num_data, text_data], prices, epochs=5, batch_size=32)

# Method 2: Pass as dict (use input layer names)
model.fit(
    {'numerical_features': num_data, 'text_description': text_data},
    prices,
    epochs=5, batch_size=32
)
```

### Multi-Output Example

```python
# ============================================
# MULTI-OUTPUT: Movie classifier
# ============================================
# Input: Movie poster image
# Output 1: Genre (multi-class)
# Output 2: Rating (regression)
# Output 3: Is sequel? (binary)

inputs = layers.Input(shape=(64, 64, 3), name='poster_image')

# Shared feature extraction
x = layers.Conv2D(32, 3, activation='relu')(inputs)
x = layers.MaxPooling2D()(x)
x = layers.Conv2D(64, 3, activation='relu')(x)
x = layers.MaxPooling2D()(x)
x = layers.Flatten()(x)
x = layers.Dense(128, activation='relu')(x)

# Output 1: Genre (10 classes)
genre_output = layers.Dense(64, activation='relu')(x)
genre_output = layers.Dense(10, activation='softmax', name='genre')(genre_output)

# Output 2: Rating (0-10)
rating_output = layers.Dense(32, activation='relu')(x)
rating_output = layers.Dense(1, name='rating')(rating_output)

# Output 3: Is sequel? (binary)
sequel_output = layers.Dense(32, activation='relu')(x)
sequel_output = layers.Dense(1, activation='sigmoid', name='sequel')(sequel_output)

model = models.Model(
    inputs=inputs,
    outputs=[genre_output, rating_output, sequel_output],
    name='movie_classifier'
)

# Compile with DIFFERENT loss per output
model.compile(
    optimizer='adam',
    loss={
        'genre': 'categorical_crossentropy',
        'rating': 'mse',
        'sequel': 'binary_crossentropy'
    },
    loss_weights={
        'genre': 1.0,      # Weight for genre loss
        'rating': 0.5,     # Rating loss contributes less
        'sequel': 0.3      # Sequel loss contributes least
    },
    metrics={
        'genre': 'accuracy',
        'rating': 'mae',
        'sequel': 'accuracy'
    }
)

model.summary()
```

---

## 9. Shared Layers and Reuse

### What It Is
Sometimes you want the **same layer** (same weights) to process different inputs. This is called **weight sharing** or **layer reuse**.

**Use cases**: Siamese networks (comparing two inputs), processing multiple similar features.

```python
# ============================================
# SIAMESE NETWORK (Shared Weights)
# ============================================
# Compare two images to determine if they're the same person

# Shared feature extractor (same weights for both inputs)
shared_dense = layers.Dense(128, activation='relu', name='shared_encoder')
shared_dense2 = layers.Dense(64, activation='relu', name='shared_encoder2')

# Input 1
input_a = layers.Input(shape=(784,), name='image_a')
encoded_a = shared_dense(input_a)      # Uses shared weights
encoded_a = shared_dense2(encoded_a)

# Input 2
input_b = layers.Input(shape=(784,), name='image_b')
encoded_b = shared_dense(input_b)      # SAME weights as above!
encoded_b = shared_dense2(encoded_b)

# Compute distance between encodings
distance = layers.Lambda(
    lambda tensors: tf.abs(tensors[0] - tensors[1])
)([encoded_a, encoded_b])

# Output: probability that inputs are same person
output = layers.Dense(1, activation='sigmoid', name='similarity')(distance)

siamese_model = models.Model(
    inputs=[input_a, input_b],
    outputs=output,
    name='siamese_network'
)

siamese_model.summary()
# Notice: shared_encoder appears once in parameters
# but is used in two places
```

---

## 10. Model Subclassing — Full Control

### What It Is
Model subclassing gives you **complete control** over the forward pass by writing a Python class. You define layers in `__init__` and the computation in `call`.

**Analogy**: Sequential is **ordering from a menu**, Functional is **choosing from a buffet**, Subclassing is **cooking from scratch**.

### How It Works

```python
import tensorflow as tf
from tensorflow.keras import layers, models

# ============================================
# BASIC SUBCLASSED MODEL
# ============================================

class MyModel(models.Model):
    def __init__(self, num_classes=10):
        super().__init__()
        # Define layers (but don't connect them yet)
        self.dense1 = layers.Dense(256, activation='relu')
        self.dropout1 = layers.Dropout(0.3)
        self.dense2 = layers.Dense(128, activation='relu')
        self.dropout2 = layers.Dropout(0.2)
        self.classifier = layers.Dense(num_classes, activation='softmax')
    
    def call(self, inputs, training=False):
        """Forward pass — defines how data flows through layers."""
        x = self.dense1(inputs)
        x = self.dropout1(x, training=training)  # Dropout behaves differently
        x = self.dense2(x)                       # during training vs inference
        x = self.dropout2(x, training=training)
        return self.classifier(x)

# Create model
model = MyModel(num_classes=10)

# Build model (needed to see summary before first forward pass)
model.build(input_shape=(None, 784))
model.summary()

# Compile and train normally
model.compile(optimizer='adam',
              loss='sparse_categorical_crossentropy',
              metrics=['accuracy'])

# Generate dummy data for demonstration
import numpy as np
x_train = np.random.rand(1000, 784).astype('float32')
y_train = np.random.randint(0, 10, 1000)

model.fit(x_train, y_train, epochs=5, batch_size=32)

# ============================================
# ADVANCED: Subclassed Model with custom logic
# ============================================

class ResidualModel(models.Model):
    def __init__(self, num_classes=10):
        super().__init__()
        self.dense_input = layers.Dense(128, activation='relu')
        
        # Residual block layers
        self.res_dense1 = layers.Dense(128, activation='relu')
        self.res_bn1 = layers.BatchNormalization()
        self.res_dense2 = layers.Dense(128)
        self.res_bn2 = layers.BatchNormalization()
        
        self.classifier = layers.Dense(num_classes, activation='softmax')
    
    def call(self, inputs, training=False):
        x = self.dense_input(inputs)
        
        # Residual block
        shortcut = x
        x = self.res_dense1(x)
        x = self.res_bn1(x, training=training)
        x = self.res_dense2(x)
        x = self.res_bn2(x, training=training)
        x = x + shortcut  # Skip connection!
        x = tf.nn.relu(x)
        
        return self.classifier(x)

model = ResidualModel(num_classes=10)
model.build(input_shape=(None, 784))
model.summary()
```

---

## 11. Sequential vs Functional vs Subclassing — When to Use What

### Comparison Table

| Feature | Sequential | Functional | Subclassing |
|---------|-----------|------------|-------------|
| **Simplicity** | ⭐⭐⭐ Easiest | ⭐⭐ Medium | ⭐ Hardest |
| **Multiple I/O** | ❌ | ✅ | ✅ |
| **Skip connections** | ❌ | ✅ | ✅ |
| **Shared layers** | ❌ | ✅ | ✅ |
| **Custom logic in forward pass** | ❌ | ❌ | ✅ |
| **Serialization** | ✅ Easy | ✅ Easy | ⚠️ Harder |
| **model.summary()** | ✅ | ✅ | ⚠️ Need build() |
| **Debugging** | Easy | Easy | Flexible |
| **Dynamic architectures** | ❌ | ❌ | ✅ |
| **plot_model()** | ✅ | ✅ | ❌ |

### Decision Flowchart

```
Is your model a simple stack of layers?
├── YES → Use Sequential API ✅
└── NO
    ├── Do you need custom logic in the forward pass?
    │   ├── YES → Use Subclassing 
    │   └── NO
    │       └── Do you need multiple inputs/outputs, skip connections, or shared layers?
    │           ├── YES → Use Functional API ✅
    │           └── NO → Use Sequential API ✅

Rule of thumb:
  90% of models → Sequential or Functional
  10% of models → Subclassing (research, custom architectures)
```

> **Pro Tip**: Start with Sequential. If you need more flexibility, switch to Functional. Only use Subclassing when Functional can't express what you need (e.g., dynamic computation based on input values, recursive architectures).

---

## 12. Layer Weights and Initialization

### Why It Matters
How you initialize weights **dramatically** affects training. Bad initialization can cause:
- **Vanishing gradients**: Weights too small → gradients shrink to 0 → no learning
- **Exploding gradients**: Weights too large → gradients grow to infinity → training diverges
- **Symmetry**: All weights identical → all neurons learn the same thing

### Common Initializers

| Initializer | Formula | Best For |
|-------------|---------|----------|
| `glorot_uniform` (Xavier) | $U[-\sqrt{6/(n_{in}+n_{out})}, \sqrt{6/(n_{in}+n_{out})}]$ | sigmoid, tanh (default) |
| `glorot_normal` | $N(0, \sqrt{2/(n_{in}+n_{out})})$ | sigmoid, tanh |
| `he_uniform` | $U[-\sqrt{6/n_{in}}, \sqrt{6/n_{in}}]$ | ReLU variants |
| `he_normal` | $N(0, \sqrt{2/n_{in}})$ | **ReLU** (recommended) |
| `lecun_normal` | $N(0, \sqrt{1/n_{in}})$ | SELU |
| `zeros` | All 0s | Biases (default) |
| `ones` | All 1s | Special cases |
| `orthogonal` | Orthogonal matrix | RNNs |

```python
import tensorflow as tf
from tensorflow.keras import layers, initializers

# ============================================
# USING INITIALIZERS
# ============================================

# Method 1: String shortcut
dense1 = layers.Dense(128, activation='relu',
                      kernel_initializer='he_normal',     # Weight init
                      bias_initializer='zeros')            # Bias init

# Method 2: Initializer objects (more control)
dense2 = layers.Dense(128, activation='relu',
                      kernel_initializer=initializers.HeNormal(seed=42),
                      bias_initializer=initializers.Constant(0.01))

# Method 3: Custom initializer
def my_init(shape, dtype=None):
    return tf.random.normal(shape, mean=0.0, stddev=0.01, dtype=dtype)

dense3 = layers.Dense(128, kernel_initializer=my_init)

# ============================================
# BEST PRACTICES
# ============================================

# ReLU activation → He initialization
layers.Dense(128, activation='relu', kernel_initializer='he_normal')

# Sigmoid/Tanh → Glorot (Xavier) initialization (this is the default)
layers.Dense(128, activation='tanh', kernel_initializer='glorot_uniform')

# SELU activation → LeCun initialization
layers.Dense(128, activation='selu', kernel_initializer='lecun_normal')

# ============================================
# INSPECTING AND MODIFYING WEIGHTS
# ============================================

model = tf.keras.Sequential([
    layers.Dense(4, activation='relu', input_shape=(3,)),
    layers.Dense(2, activation='softmax')
])

# Get all weights
all_weights = model.get_weights()
print(f"Number of weight arrays: {len(all_weights)}")
# [layer1_kernel, layer1_bias, layer2_kernel, layer2_bias]

for i, w in enumerate(all_weights):
    print(f"  Weight {i}: shape={w.shape}")

# Get weights of specific layer
layer_weights = model.layers[0].get_weights()
kernel, bias = layer_weights
print(f"\nLayer 0 kernel:\n{kernel}")
print(f"Layer 0 bias: {bias}")

# Set custom weights
import numpy as np
model.layers[0].set_weights([
    np.ones((3, 4)) * 0.5,  # kernel
    np.zeros(4)              # bias
])

# Freeze/unfreeze layers
model.layers[0].trainable = False   # Freeze — weights won't update during training
model.layers[1].trainable = True    # Unfreeze
print(f"\nTrainable weights: {len(model.trainable_weights)}")
print(f"Non-trainable weights: {len(model.non_trainable_weights)}")
```

---

## 13. Custom Layers

### What It Is
When built-in layers don't do what you need, you can create custom layers by subclassing `tf.keras.layers.Layer`.

```python
import tensorflow as tf
from tensorflow.keras import layers

# ============================================
# SIMPLE CUSTOM LAYER
# ============================================

class ScaledDense(layers.Layer):
    """Dense layer that scales output by a learnable parameter."""
    
    def __init__(self, units, activation=None, **kwargs):
        super().__init__(**kwargs)
        self.units = units
        self.activation = tf.keras.activations.get(activation)
    
    def build(self, input_shape):
        """Create weights — called once when layer first sees data."""
        self.kernel = self.add_weight(
            name='kernel',
            shape=(input_shape[-1], self.units),
            initializer='glorot_uniform',
            trainable=True  # Will be updated during training
        )
        self.bias = self.add_weight(
            name='bias',
            shape=(self.units,),
            initializer='zeros',
            trainable=True
        )
        self.scale = self.add_weight(
            name='scale',
            shape=(1,),
            initializer='ones',
            trainable=True  # Learnable scaling factor!
        )
        super().build(input_shape)
    
    def call(self, inputs):
        """Forward pass computation."""
        output = tf.matmul(inputs, self.kernel) + self.bias
        output = output * self.scale  # Apply learnable scaling
        if self.activation is not None:
            output = self.activation(output)
        return output
    
    def get_config(self):
        """Required for serialization (saving/loading)."""
        config = super().get_config()
        config.update({
            'units': self.units,
            'activation': tf.keras.activations.serialize(self.activation)
        })
        return config

# Use custom layer in a model
model = tf.keras.Sequential([
    layers.Input(shape=(784,)),
    ScaledDense(128, activation='relu', name='scaled_dense_1'),
    ScaledDense(64, activation='relu', name='scaled_dense_2'),
    layers.Dense(10, activation='softmax')
])

model.summary()

# ============================================
# CUSTOM LAYER: Multi-Head Self-Attention (simplified)
# ============================================

class SimpleSelfAttention(layers.Layer):
    """Simplified single-head self-attention."""
    
    def __init__(self, embed_dim, **kwargs):
        super().__init__(**kwargs)
        self.embed_dim = embed_dim
        self.query_dense = layers.Dense(embed_dim)
        self.key_dense = layers.Dense(embed_dim)
        self.value_dense = layers.Dense(embed_dim)
    
    def call(self, inputs):
        # inputs shape: (batch, seq_len, embed_dim)
        Q = self.query_dense(inputs)
        K = self.key_dense(inputs)
        V = self.value_dense(inputs)
        
        # Attention scores: Q @ K^T / sqrt(d_k)
        scores = tf.matmul(Q, K, transpose_b=True)
        scores = scores / tf.math.sqrt(tf.cast(self.embed_dim, tf.float32))
        
        # Softmax to get attention weights
        attention_weights = tf.nn.softmax(scores, axis=-1)
        
        # Weighted sum of values
        output = tf.matmul(attention_weights, V)
        return output
    
    def get_config(self):
        config = super().get_config()
        config.update({'embed_dim': self.embed_dim})
        return config
```

---

## 14. Common Mistakes

### Mistake 1: Wrong output layer activation

```python
# ❌ WRONG: softmax for binary classification
model.add(layers.Dense(1, activation='softmax'))  # softmax with 1 unit is ALWAYS 1.0!

# ✅ FIX: sigmoid for binary
model.add(layers.Dense(1, activation='sigmoid'))

# ❌ WRONG: sigmoid for multi-class
model.add(layers.Dense(10, activation='sigmoid'))  # Each output independent, won't sum to 1

# ✅ FIX: softmax for multi-class
model.add(layers.Dense(10, activation='softmax'))
```

### Mistake 2: Forgetting to flatten before Dense

```python
# ❌ WRONG: Dense after Conv2D without flattening
model = tf.keras.Sequential([
    layers.Conv2D(32, 3, activation='relu', input_shape=(28, 28, 1)),
    # layers.Dense(10)  # ERROR: Dense gets 3D input, expects 2D!
])

# ✅ FIX: Add Flatten() layer
model = tf.keras.Sequential([
    layers.Conv2D(32, 3, activation='relu', input_shape=(28, 28, 1)),
    layers.Flatten(),       # (batch, 26, 26, 32) → (batch, 21632)
    layers.Dense(10, activation='softmax')
])
```

### Mistake 3: Not specifying input shape

```python
# ❌ WRONG: No input shape anywhere
model = tf.keras.Sequential([
    layers.Dense(128, activation='relu'),  # What's the input size?
    layers.Dense(10)
])
# model.summary()  # ERROR: model not built yet!

# ✅ FIX: Specify input_shape in first layer OR use Input layer
model = tf.keras.Sequential([
    layers.Dense(128, activation='relu', input_shape=(784,)),
    layers.Dense(10)
])
```

### Mistake 4: Mismatched loss and activation

```python
# ❌ WRONG: categorical_crossentropy with integer labels
model.compile(loss='categorical_crossentropy')
model.fit(x, y_integers)  # y = [0, 1, 2, ...]  ← WRONG!

# ✅ FIX: Use sparse_categorical_crossentropy for integer labels
model.compile(loss='sparse_categorical_crossentropy')  # Works with [0, 1, 2, ...]

# OR: One-hot encode labels
y_onehot = tf.one_hot(y_integers, depth=10)
model.compile(loss='categorical_crossentropy')  # Now works with one-hot
```

### Mistake 5: Not passing training flag to Dropout/BN

```python
# ❌ WRONG (in subclassed model): Dropout always active
class BadModel(tf.keras.Model):
    def call(self, inputs):
        x = self.dense(inputs)
        x = self.dropout(x)  # No training flag! Dropout active during inference too!
        return x

# ✅ FIX: Pass training parameter
class GoodModel(tf.keras.Model):
    def call(self, inputs, training=False):
        x = self.dense(inputs)
        x = self.dropout(x, training=training)  # Correct!
        return x
```

---

## 15. Interview Questions

### Q1: What are the three ways to build models in Keras? When would you use each?
**Answer**:
1. **Sequential**: Simple linear stack of layers. Use for straightforward architectures (most common cases)
2. **Functional**: DAG of layers. Use for multi-input/output, skip connections, shared layers
3. **Subclassing**: Custom Python class. Use when you need dynamic computation in the forward pass

### Q2: What is the difference between `categorical_crossentropy` and `sparse_categorical_crossentropy`?
**Answer**: 
- `categorical_crossentropy`: Labels must be **one-hot encoded** `[0, 0, 1, 0, ...]`
- `sparse_categorical_crossentropy`: Labels are **integers** `[0, 1, 2, ...]`
- They compute the same loss — just different label formats
- Use `sparse_` to save memory (no need to one-hot encode)

### Q3: Why do we need activation functions?
**Answer**: Without activation functions, a multi-layer neural network collapses to a single linear transformation ($y = Wx + b$), regardless of depth. Activation functions introduce non-linearity, enabling the network to learn complex, non-linear mappings. The Universal Approximation Theorem proves that a network with at least one hidden layer and a non-linear activation can approximate any continuous function.

### Q4: What is weight initialization and why does it matter?
**Answer**: Weight initialization sets the starting values of model parameters. It matters because:
- **Too small** → vanishing gradients (layers near input don't learn)
- **Too large** → exploding gradients (training diverges, NaN loss)
- **All same** → symmetry problem (all neurons learn identical features)
- Best practice: He initialization for ReLU, Glorot (Xavier) for sigmoid/tanh

### Q5: What is a skip/residual connection and why does it help?
**Answer**: A skip connection adds the input of a block directly to its output: $y = F(x) + x$. Benefits:
- Prevents vanishing gradients (gradient flows through the skip)
- Makes it easier to learn identity mappings (F(x) just needs to learn 0)
- Enables training of very deep networks (100+ layers)
- Key innovation of ResNet (2015)

### Q6: Explain the Functional API with an example use case.
**Answer**: The Functional API treats layers as functions called on tensors, allowing arbitrary graph topologies. Example: a multi-input model that combines user demographics (tabular data) with user reviews (text data) to predict purchase probability. Each input goes through its own processing pipeline, then the features are concatenated and fed through shared dense layers.

### Q7: What is a Siamese Network?
**Answer**: A Siamese network uses **shared weights** to process two inputs through the same encoder, then compares the outputs (e.g., using distance metrics). Used for:
- Face verification (same person?)
- Signature verification
- Duplicate detection
- Few-shot learning

---

## 16. Quick Reference

### Model Building Cheat Sheet

```python
# ============================================
# SEQUENTIAL API
# ============================================
model = tf.keras.Sequential([
    tf.keras.layers.Dense(128, activation='relu', input_shape=(784,)),
    tf.keras.layers.Dense(10, activation='softmax')
])

# ============================================
# FUNCTIONAL API
# ============================================
inputs = tf.keras.layers.Input(shape=(784,))
x = tf.keras.layers.Dense(128, activation='relu')(inputs)
outputs = tf.keras.layers.Dense(10, activation='softmax')(x)
model = tf.keras.Model(inputs=inputs, outputs=outputs)

# ============================================
# SUBCLASSING
# ============================================
class MyModel(tf.keras.Model):
    def __init__(self):
        super().__init__()
        self.dense1 = tf.keras.layers.Dense(128, activation='relu')
        self.dense2 = tf.keras.layers.Dense(10, activation='softmax')
    
    def call(self, inputs, training=False):
        x = self.dense1(inputs)
        return self.dense2(x)
```

### Output Layer Cheat Sheet

| Task | Output Units | Activation | Loss |
|------|-------------|------------|------|
| Binary classification | 1 | `sigmoid` | `binary_crossentropy` |
| Multi-class (exclusive) | N | `softmax` | `sparse_categorical_crossentropy` |
| Multi-class (one-hot labels) | N | `softmax` | `categorical_crossentropy` |
| Multi-label | N | `sigmoid` | `binary_crossentropy` |
| Regression | 1 | `None`/`linear` | `mse` or `mae` |
| Regression (positive only) | 1 | `relu` or `softplus` | `mse` |

### Layer Reference

| Layer | Key Parameters | When to Use |
|-------|---------------|-------------|
| `Dense(units)` | units, activation | Fully connected |
| `Conv2D(filters, kernel)` | filters, kernel_size, strides, padding | Image features |
| `MaxPooling2D(pool_size)` | pool_size | Downsample |
| `Flatten()` | — | Before Dense after Conv |
| `Dropout(rate)` | rate (0-1) | Regularization |
| `BatchNormalization()` | — | Stable training |
| `Embedding(vocab, dim)` | input_dim, output_dim | Text/categorical |
| `LSTM(units)` / `GRU(units)` | units, return_sequences | Sequences |
| `Input(shape)` | shape | Functional API entry |
| `Concatenate()` | axis | Merge branches |
| `Add()` | — | Skip connections |

### Initialization Rules

```
Activation Function    →    Initializer
─────────────────────────────────────────
ReLU / Leaky ReLU      →    He Normal ('he_normal')
Sigmoid / Tanh         →    Glorot Uniform ('glorot_uniform') [default]
SELU                   →    LeCun Normal ('lecun_normal')
Linear / None          →    Glorot Uniform (default)
```

---

*Previous: [01-TensorFlow-Fundamentals](01-TensorFlow-Fundamentals.md) | Next: [03-Training-and-Callbacks](03-Training-and-Callbacks.md)*
