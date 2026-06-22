# Chapter 01: TensorFlow Fundamentals

## Table of Contents
- [1. What is TensorFlow?](#1-what-is-tensorflow)
- [2. Installation and Setup](#2-installation-and-setup)
- [3. Tensors — The Core Data Structure](#3-tensors--the-core-data-structure)
- [4. Tensor Operations](#4-tensor-operations)
- [5. Indexing and Slicing](#5-indexing-and-slicing)
- [6. Reshaping Tensors](#6-reshaping-tensors)
- [7. Broadcasting](#7-broadcasting)
- [8. Eager Execution vs Graph Mode](#8-eager-execution-vs-graph-mode)
- [9. tf.function — Performance with Graphs](#9-tffunction--performance-with-graphs)
- [10. GradientTape — Automatic Differentiation](#10-gradienttape--automatic-differentiation)
- [11. Variables vs Constants](#11-variables-vs-constants)
- [12. TensorFlow vs NumPy](#12-tensorflow-vs-numpy)
- [13. GPU/TPU Acceleration](#13-gputpu-acceleration)
- [14. Common Mistakes](#14-common-mistakes)
- [15. Interview Questions](#15-interview-questions)
- [16. Quick Reference](#16-quick-reference)

---

## 1. What is TensorFlow?

### Simple Explanation
Imagine you're building with LEGO blocks. NumPy gives you basic blocks to build math stuff. TensorFlow gives you **motorized, smart LEGO blocks** — they can do math incredibly fast (using GPUs), remember how they were assembled (computation graphs), and automatically figure out how to improve themselves (automatic differentiation).

TensorFlow is Google's open-source library for **numerical computation** and **machine learning**. It's named after its core concept: **tensors** (multi-dimensional arrays) that **flow** through a graph of operations.

### Why It Matters
- **Industry Standard**: Used by Google, Uber, Airbnb, Intel, Twitter — powers Google Search, Gmail, Google Photos
- **Production-Ready**: Unlike many research frameworks, TF is built for deployment (mobile, web, embedded, cloud)
- **Ecosystem**: TensorBoard (visualization), TF Lite (mobile), TF.js (browser), TF Serving (production APIs), TFX (ML pipelines)
- **Scale**: Can run on CPUs, GPUs, TPUs — single machine to distributed clusters
- **Jobs**: Most ML engineer job postings list TensorFlow as a required or preferred skill

### TensorFlow Timeline

```
TF 1.x (2015-2018)     → Graph-based, Session.run(), placeholder hell
TF 2.0 (Sep 2019)      → Eager execution default, Keras integrated, tf.function
TF 2.4+ (2020-2021)    → Mixed precision, distributed strategies matured
TF 2.10+ (2022-2023)   → keras.saving, improved custom training
TF 2.16+ (2024-2026)   → Keras 3 multi-backend, JAX/PyTorch interop
```

> **Important**: TF 2.x is a **completely different experience** from TF 1.x. If you see `tf.Session()`, `tf.placeholder()`, or `feed_dict` in tutorials, they're outdated TF 1.x code. Avoid them.

---

## 2. Installation and Setup

### How It Works

```python
# Install TensorFlow (CPU version works everywhere)
# pip install tensorflow

# For GPU support (NVIDIA GPUs with CUDA)
# pip install tensorflow[and-cuda]

# Verify installation
import tensorflow as tf
print(tf.__version__)          # e.g., 2.16.1
print(tf.config.list_physical_devices('GPU'))  # Lists available GPUs
```

### Check Your Setup

```python
import tensorflow as tf
import numpy as np

# Version check
print(f"TensorFlow version: {tf.__version__}")
print(f"NumPy version: {np.__version__}")

# GPU check
gpus = tf.config.list_physical_devices('GPU')
if gpus:
    print(f"✅ GPU available: {len(gpus)} GPU(s) detected")
    for gpu in gpus:
        print(f"   - {gpu.name}")
else:
    print("⚠️ No GPU detected. Running on CPU.")
    print("   CPU is fine for learning, but slow for large models.")

# Quick test — create a tensor
x = tf.constant([1, 2, 3])
print(f"Test tensor: {x}")
print(f"Device: {x.device}")  # Shows which device (CPU/GPU) is being used
```

> **Pro Tip**: For learning, CPU is perfectly fine. You only need GPU for training large models. Google Colab gives free GPU access.

---

## 3. Tensors — The Core Data Structure

### What It Is
A **tensor** is just a multi-dimensional array — a generalization of scalars, vectors, and matrices.

Think of it like Russian nesting dolls:
- **Scalar** (0D tensor): A single number → `5`
- **Vector** (1D tensor): A list of numbers → `[1, 2, 3]`
- **Matrix** (2D tensor): A table of numbers → `[[1,2], [3,4]]`
- **3D Tensor**: A cube of numbers (e.g., a color image: height × width × RGB)
- **4D Tensor**: A batch of images (batch × height × width × channels)
- **5D Tensor**: A batch of videos (batch × frames × height × width × channels)

### Visual Representation

```
Scalar (0D)     Vector (1D)      Matrix (2D)           3D Tensor
rank=0          rank=1           rank=2                rank=3

  5             [1, 2, 3]        [[1, 2, 3],           [[[1, 2],
                                  [4, 5, 6]]             [3, 4]],
                                                         [[5, 6],
                                                          [7, 8]]]

shape=()        shape=(3,)       shape=(2, 3)          shape=(2, 2, 2)
```

### Why It Matters
Everything in deep learning is a tensor:
- **Images**: 4D tensor `(batch_size, height, width, channels)`
- **Text**: 3D tensor `(batch_size, sequence_length, embedding_dim)`
- **Audio**: 3D tensor `(batch_size, time_steps, features)`
- **Model weights**: 2D tensors (matrices)

### Creating Tensors

```python
import tensorflow as tf
import numpy as np

# ============================================
# 1. CREATING TENSORS FROM PYTHON VALUES
# ============================================

# Scalar (0D) — a single value
scalar = tf.constant(7)
print(f"Scalar: {scalar}")
print(f"  ndim: {scalar.ndim}")    # 0
print(f"  shape: {scalar.shape}")  # ()

# Vector (1D) — a list of values
vector = tf.constant([10, 20, 30])
print(f"\nVector: {vector}")
print(f"  ndim: {vector.ndim}")    # 1
print(f"  shape: {vector.shape}")  # (3,)

# Matrix (2D) — a table
matrix = tf.constant([[1, 2, 3],
                      [4, 5, 6]])
print(f"\nMatrix:\n{matrix}")
print(f"  ndim: {matrix.ndim}")    # 2
print(f"  shape: {matrix.shape}")  # (2, 3)

# 3D Tensor — e.g., batch of sequences
tensor_3d = tf.constant([[[1, 2], [3, 4]],
                          [[5, 6], [7, 8]],
                          [[9, 10], [11, 12]]])
print(f"\n3D Tensor shape: {tensor_3d.shape}")  # (3, 2, 2)
print(f"  ndim: {tensor_3d.ndim}")              # 3
```

### Data Types (dtypes)

```python
# ============================================
# 2. TENSOR DATA TYPES
# ============================================

# Default: int32 for integers, float32 for floats
int_tensor = tf.constant([1, 2, 3])
print(f"Default int dtype: {int_tensor.dtype}")    # tf.int32

float_tensor = tf.constant([1.0, 2.0, 3.0])
print(f"Default float dtype: {float_tensor.dtype}")  # tf.float32

# Specify dtype explicitly
float16_tensor = tf.constant([1.0, 2.0], dtype=tf.float16)  # Half precision
float64_tensor = tf.constant([1.0, 2.0], dtype=tf.float64)  # Double precision
bool_tensor = tf.constant([True, False, True])               # Boolean
string_tensor = tf.constant(["hello", "world"])              # String

# Cast between types
x = tf.constant([1, 2, 3])           # int32
x_float = tf.cast(x, dtype=tf.float32)  # Now float32
print(f"After cast: {x_float.dtype}")   # tf.float32

# ⚠️ CRITICAL: Neural networks expect float32 (or float16 for mixed precision)
# Always cast integer data to float32 before feeding to models
```

| dtype | Use Case | Memory | Precision |
|-------|----------|--------|-----------|
| `tf.float32` | **Default for training** — best balance | 4 bytes | Good |
| `tf.float16` | Mixed precision training — faster on modern GPUs | 2 bytes | Lower |
| `tf.float64` | Scientific computing (rarely needed in DL) | 8 bytes | Highest |
| `tf.int32` | Labels, indices | 4 bytes | Exact |
| `tf.int64` | Large indices, sparse tensors | 8 bytes | Exact |
| `tf.bool` | Masks, conditions | 1 byte | Binary |
| `tf.string` | Text data (before tokenization) | Variable | N/A |

### Special Tensor Creation

```python
# ============================================
# 3. SPECIAL TENSORS
# ============================================

# Zeros and Ones
zeros = tf.zeros(shape=(3, 4))       # 3x4 matrix of zeros
ones = tf.ones(shape=(2, 3))         # 2x3 matrix of ones
print(f"Zeros:\n{zeros}\n")
print(f"Ones:\n{ones}\n")

# Fill with a specific value
filled = tf.fill(dims=(2, 3), value=42.0)  # 2x3 matrix filled with 42
print(f"Filled:\n{filled}\n")

# Identity matrix (useful for initializations)
eye = tf.eye(num_rows=4)  # 4x4 identity matrix
print(f"Identity:\n{eye}\n")

# Range (like Python's range but returns a tensor)
r = tf.range(start=0, limit=10, delta=2)  # [0, 2, 4, 6, 8]
print(f"Range: {r}\n")

# Linspace (evenly spaced values)
lin = tf.linspace(start=0.0, stop=1.0, num=5)  # [0, 0.25, 0.5, 0.75, 1.0]
print(f"Linspace: {lin}\n")

# ============================================
# 4. RANDOM TENSORS — Essential for ML
# ============================================

# Set seed for reproducibility
tf.random.set_seed(42)

# Normal distribution (mean=0, std=1) — used for weight initialization
normal = tf.random.normal(shape=(3, 3), mean=0.0, stddev=1.0)
print(f"Normal:\n{normal}\n")

# Uniform distribution [0, 1) — used for some initializations
uniform = tf.random.uniform(shape=(2, 4), minval=0, maxval=1)
print(f"Uniform:\n{uniform}\n")

# Truncated normal — values beyond 2σ are re-drawn
# Better than normal for weight init (avoids extreme values)
trunc = tf.random.truncated_normal(shape=(3, 3), mean=0.0, stddev=0.1)
print(f"Truncated Normal:\n{trunc}\n")

# Shuffle — randomly reorder along first axis
data = tf.constant([[1, 2], [3, 4], [5, 6], [7, 8]])
shuffled = tf.random.shuffle(data)
print(f"Shuffled:\n{shuffled}")
```

> **Pro Tip**: Always set `tf.random.set_seed()` at the start of your scripts for reproducible results. For full reproducibility, also set `np.random.seed()` and Python's `random.seed()`.

### Converting Between TensorFlow and NumPy

```python
# ============================================
# 5. TENSORFLOW ↔ NUMPY CONVERSION
# ============================================

# NumPy → TensorFlow
np_array = np.array([1.0, 2.0, 3.0])
tf_tensor = tf.constant(np_array)          # Method 1: tf.constant
tf_tensor2 = tf.convert_to_tensor(np_array) # Method 2: tf.convert_to_tensor
print(f"NumPy → TF: {tf_tensor}")

# TensorFlow → NumPy
back_to_numpy = tf_tensor.numpy()           # Method 1: .numpy()
back_to_numpy2 = np.array(tf_tensor)        # Method 2: np.array()
print(f"TF → NumPy: {back_to_numpy}")
print(f"Type: {type(back_to_numpy)}")       # <class 'numpy.ndarray'>

# ⚠️ WARNING: Conversion copies data between CPU/GPU
# If tensor is on GPU, .numpy() copies it to CPU first (slow for large tensors)
# Avoid unnecessary conversions in training loops!
```

---

## 4. Tensor Operations

### Why It Matters
Every neural network computation is a series of tensor operations: matrix multiplications, element-wise additions, activations. Understanding these is fundamental.

### Element-wise Operations

```python
import tensorflow as tf

a = tf.constant([1.0, 2.0, 3.0])
b = tf.constant([4.0, 5.0, 6.0])

# ============================================
# ARITHMETIC (element-wise)
# ============================================

# Addition
print(a + b)              # [5. 7. 9.]  — operator overload
print(tf.add(a, b))       # Same thing   — explicit function

# Subtraction
print(a - b)              # [-3. -3. -3.]
print(tf.subtract(a, b))

# Multiplication (element-wise, NOT matrix multiply!)
print(a * b)              # [4. 10. 18.]
print(tf.multiply(a, b))

# Division
print(b / a)              # [4. 2.5 2.]
print(tf.divide(b, a))

# Floor Division
print(tf.math.floordiv(tf.constant([7, 8, 9]), tf.constant([2, 3, 4])))  # [3, 2, 2]

# Modulo
print(tf.math.mod(tf.constant([7, 8, 9]), tf.constant([2, 3, 4])))  # [1, 2, 1]

# Power
print(a ** 2)             # [1. 4. 9.]
print(tf.pow(a, 2))

# Square root
print(tf.sqrt(a))         # [1. 1.414 1.732]

# Absolute value
print(tf.abs(tf.constant([-1, -2, 3])))  # [1, 2, 3]
```

### Matrix Operations (The Heart of Deep Learning)

```python
# ============================================
# MATRIX MULTIPLICATION — Most critical operation in DL
# ============================================

# Matrix multiplication: (m×n) @ (n×p) → (m×p)
# Inner dimensions must match!

A = tf.constant([[1, 2],
                 [3, 4]])    # Shape: (2, 2)

B = tf.constant([[5, 6],
                 [7, 8]])    # Shape: (2, 2)

# Three ways to do matrix multiplication
result1 = A @ B              # Python operator (cleanest)
result2 = tf.matmul(A, B)    # Explicit function
result3 = tf.linalg.matmul(A, B)  # Full path

print(f"A @ B:\n{result1}")
# [[1*5+2*7, 1*6+2*8],     = [[19, 22],
#  [3*5+4*7, 3*6+4*8]]       [43, 50]]

# ============================================
# WHY MATMUL MATTERS IN NEURAL NETWORKS
# ============================================
# A single dense layer is: output = activation(input @ weights + bias)
# 
#   input:   (batch_size, input_features)    e.g., (32, 784)
#   weights: (input_features, output_units)  e.g., (784, 128)
#   output:  (batch_size, output_units)      e.g., (32, 128)

input_data = tf.random.normal((32, 784))   # 32 images, 784 pixels each
weights = tf.random.normal((784, 128))     # 784→128 weight matrix
bias = tf.zeros((128,))                    # 128 bias terms

output = input_data @ weights + bias       # This IS a Dense layer!
print(f"Dense layer output shape: {output.shape}")  # (32, 128)

# ============================================
# DOT PRODUCT (for vectors)
# ============================================
v1 = tf.constant([1.0, 2.0, 3.0])
v2 = tf.constant([4.0, 5.0, 6.0])

dot = tf.tensordot(v1, v2, axes=1)  # 1*4 + 2*5 + 3*6 = 32
print(f"Dot product: {dot}")        # 32.0

# ============================================
# TRANSPOSE
# ============================================
M = tf.constant([[1, 2, 3],
                 [4, 5, 6]])   # Shape: (2, 3)

Mt = tf.transpose(M)           # Shape: (3, 2)
print(f"Original shape: {M.shape}")       # (2, 3)
print(f"Transposed shape: {Mt.shape}")    # (3, 2)
print(f"Transposed:\n{Mt}")
```

### Aggregation Operations

```python
# ============================================
# REDUCTION / AGGREGATION OPERATIONS
# ============================================

x = tf.constant([[1.0, 2.0, 3.0],
                 [4.0, 5.0, 6.0]])

# Reduce across ALL elements (no axis specified)
print(f"Sum (all): {tf.reduce_sum(x)}")        # 21.0
print(f"Mean (all): {tf.reduce_mean(x)}")      # 3.5
print(f"Max (all): {tf.reduce_max(x)}")        # 6.0
print(f"Min (all): {tf.reduce_min(x)}")        # 1.0
print(f"Prod (all): {tf.reduce_prod(x)}")      # 720.0
print(f"Std: {tf.math.reduce_std(x)}")         # 1.707...

# Reduce along a SPECIFIC AXIS
# axis=0 → collapse rows (operate down columns)
# axis=1 → collapse columns (operate across rows)

print(f"\nSum axis=0 (column sums): {tf.reduce_sum(x, axis=0)}")  # [5. 7. 9.]
print(f"Sum axis=1 (row sums): {tf.reduce_sum(x, axis=1)}")      # [6. 15.]
print(f"Mean axis=0: {tf.reduce_mean(x, axis=0)}")                # [2.5 3.5 4.5]
print(f"Mean axis=1: {tf.reduce_mean(x, axis=1)}")                # [2. 5.]

# Argmax / Argmin — index of max/min value
print(f"\nArgmax axis=0: {tf.argmax(x, axis=0)}")  # [1, 1, 1] (row index of max in each col)
print(f"Argmax axis=1: {tf.argmax(x, axis=1)}")    # [2, 2] (col index of max in each row)

# ============================================
# PRACTICAL EXAMPLE: Softmax + Argmax for classification
# ============================================
logits = tf.constant([2.0, 1.0, 0.5, 3.0, 0.1])  # Raw model outputs
probabilities = tf.nn.softmax(logits)               # Convert to probabilities
predicted_class = tf.argmax(probabilities)           # Class with highest probability

print(f"\nLogits: {logits.numpy()}")
print(f"Probabilities: {probabilities.numpy()}")    # Sums to 1.0
print(f"Predicted class: {predicted_class.numpy()}")  # 3 (index of 3.0)
```

### Comparison Operations

```python
# ============================================
# COMPARISON OPERATIONS (return boolean tensors)
# ============================================

a = tf.constant([1, 2, 3, 4, 5])
b = tf.constant([5, 4, 3, 2, 1])

print(f"a == b: {tf.equal(a, b)}")           # [F, F, T, F, F]
print(f"a > b:  {tf.greater(a, b)}")         # [F, F, F, T, T]
print(f"a >= b: {tf.greater_equal(a, b)}")   # [F, F, T, T, T]
print(f"a < b:  {tf.less(a, b)}")            # [T, T, F, F, F]

# Use boolean tensors for filtering
x = tf.constant([10, -5, 3, -8, 7])
mask = x > 0                                 # [T, F, T, F, T]
positive_values = tf.boolean_mask(x, mask)   # [10, 3, 7]
print(f"Positive values: {positive_values}")

# Where (conditional selection — like np.where)
result = tf.where(x > 0, x, tf.zeros_like(x))  # Replace negatives with 0
print(f"ReLU manually: {result}")               # [10, 0, 3, 0, 7]
```

---

## 5. Indexing and Slicing

### How It Works

```python
import tensorflow as tf

# ============================================
# INDEXING — Same syntax as NumPy
# ============================================

t = tf.constant([10, 20, 30, 40, 50])

print(t[0])      # 10 — first element
print(t[-1])     # 50 — last element
print(t[1:4])    # [20, 30, 40] — slice from index 1 to 3
print(t[::2])    # [10, 30, 50] — every other element

# 2D Indexing
matrix = tf.constant([[1, 2, 3],
                      [4, 5, 6],
                      [7, 8, 9]])

print(matrix[0, 0])      # 1 — first row, first col
print(matrix[1, :])      # [4, 5, 6] — entire second row
print(matrix[:, 2])      # [3, 6, 9] — entire third column
print(matrix[0:2, 1:3])  # [[2,3], [5,6]] — submatrix

# ============================================
# tf.gather — Advanced indexing
# ============================================

# Select specific indices along an axis
data = tf.constant([[10, 20, 30],
                    [40, 50, 60],
                    [70, 80, 90]])

# Gather specific rows
rows = tf.gather(data, indices=[0, 2])  # Rows 0 and 2
print(f"Gathered rows:\n{rows}")
# [[10, 20, 30],
#  [70, 80, 90]]

# Gather specific columns
cols = tf.gather(data, indices=[0, 2], axis=1)  # Columns 0 and 2
print(f"Gathered cols:\n{cols}")
# [[10, 30],
#  [40, 60],
#  [70, 90]]

# ============================================
# tf.gather_nd — Gather by N-dimensional indices
# ============================================

# Useful for selecting specific elements by coordinates
elements = tf.gather_nd(data, indices=[[0, 1], [2, 0]])  # data[0,1] and data[2,0]
print(f"Gathered elements: {elements}")  # [20, 70]

# ⚠️ IMPORTANT: Tensors are IMMUTABLE — you can't assign to indices
# t[0] = 999  ← This would FAIL!
# Use tf.tensor_scatter_nd_update() for "updating" tensors (creates a new one)
```

---

## 6. Reshaping Tensors

### Why It Matters
Reshaping is one of the most common operations in deep learning. You constantly need to change tensor shapes to make them compatible for operations (e.g., flattening an image before a dense layer, adding a batch dimension).

### How It Works

```python
import tensorflow as tf

# ============================================
# tf.reshape — Change shape without changing data
# ============================================

x = tf.range(12)  # [0, 1, 2, ..., 11], shape=(12,)

# Reshape to matrix
a = tf.reshape(x, (3, 4))    # 3 rows × 4 cols = 12 elements ✅
b = tf.reshape(x, (4, 3))    # 4 rows × 3 cols = 12 elements ✅
c = tf.reshape(x, (2, 2, 3)) # 2 × 2 × 3 = 12 elements ✅
# d = tf.reshape(x, (3, 5))  # 3 × 5 = 15 ≠ 12 ❌ ERROR!

print(f"Original: {x.shape}")  # (12,)
print(f"Reshaped to (3,4):\n{a}")
print(f"Reshaped to (2,2,3):\n{c}")

# Use -1 to auto-calculate one dimension
auto = tf.reshape(x, (3, -1))  # -1 becomes 4 (12/3=4)
print(f"Auto reshape (3,-1): shape={auto.shape}")  # (3, 4)

# ============================================
# COMMON RESHAPE PATTERNS IN DEEP LEARNING
# ============================================

# 1. Flatten an image: (28, 28) → (784,)
image = tf.random.normal((28, 28))
flat = tf.reshape(image, (-1,))  # or tf.reshape(image, (784,))
print(f"Flatten: {image.shape} → {flat.shape}")  # (28,28) → (784,)

# 2. Add batch dimension: (28, 28) → (1, 28, 28)
batched = tf.expand_dims(image, axis=0)
print(f"Add batch: {image.shape} → {batched.shape}")  # (28,28) → (1,28,28)

# 3. Add channel dimension: (28, 28) → (28, 28, 1)
with_channel = tf.expand_dims(image, axis=-1)
print(f"Add channel: {image.shape} → {with_channel.shape}")  # (28,28) → (28,28,1)

# 4. Remove dimensions of size 1
x = tf.constant([[[1, 2, 3]]])  # shape=(1, 1, 3)
squeezed = tf.squeeze(x)
print(f"Squeeze: {x.shape} → {squeezed.shape}")  # (1,1,3) → (3,)

# Squeeze specific axis only
squeezed_0 = tf.squeeze(x, axis=0)
print(f"Squeeze axis=0: {x.shape} → {squeezed_0.shape}")  # (1,1,3) → (1,3)

# ============================================
# tf.transpose — Reorder dimensions
# ============================================

# Common: Convert channels-first ↔ channels-last
# PyTorch uses (B, C, H, W) — channels first
# TensorFlow uses (B, H, W, C) — channels last

img_channels_first = tf.random.normal((32, 3, 224, 224))  # (B, C, H, W)
img_channels_last = tf.transpose(img_channels_first, perm=[0, 2, 3, 1])  # (B, H, W, C)
print(f"Channels first → last: {img_channels_first.shape} → {img_channels_last.shape}")
# (32, 3, 224, 224) → (32, 224, 224, 3)

# ============================================
# CONCATENATION and STACKING
# ============================================

a = tf.constant([[1, 2], [3, 4]])
b = tf.constant([[5, 6], [7, 8]])

# Concatenate along axis=0 (stack vertically)
cat0 = tf.concat([a, b], axis=0)
print(f"Concat axis=0: {cat0.shape}")  # (4, 2)

# Concatenate along axis=1 (stack horizontally)  
cat1 = tf.concat([a, b], axis=1)
print(f"Concat axis=1: {cat1.shape}")  # (2, 4)

# Stack — creates a NEW dimension
stacked = tf.stack([a, b], axis=0)
print(f"Stack: {stacked.shape}")       # (2, 2, 2)
```

> **Pro Tip**: `tf.reshape` doesn't copy data — it just creates a new view. But `tf.transpose` may copy data depending on the layout. Keep this in mind for performance.

---

## 7. Broadcasting

### What It Is
Broadcasting is TensorFlow's way of handling operations between tensors of different shapes. Instead of requiring identical shapes, TF automatically "stretches" smaller tensors to match larger ones — **without actually copying data**.

### How It Works

```
Broadcasting Rules:
1. Compare shapes from the RIGHT (trailing dimensions)
2. Dimensions are compatible if:
   a. They are equal, OR
   b. One of them is 1
3. If a dimension is 1, it gets "stretched" to match the other

Examples:
  (3, 4) + (4,)     → (3, 4)  ✅  (4 matches 4)
  (3, 4) + (3, 1)   → (3, 4)  ✅  (3=3, 1→4)
  (3, 4) + (1, 4)   → (3, 4)  ✅  (1→3, 4=4)
  (3, 4) + (3,)     → ERROR   ❌  (4 ≠ 3)
  (2, 3, 4) + (3, 4) → (2, 3, 4)  ✅
  (2, 3, 4) + (1,)   → (2, 3, 4)  ✅
```

```python
import tensorflow as tf

# ============================================
# BROADCASTING EXAMPLES
# ============================================

# Scalar + Tensor (scalar broadcasts to all elements)
x = tf.constant([1, 2, 3])
print(x + 10)  # [11, 12, 13]

# Vector + Matrix (vector broadcasts across rows)
matrix = tf.constant([[1, 2, 3],
                      [4, 5, 6]])    # (2, 3)
vector = tf.constant([10, 20, 30])    # (3,)
print(matrix + vector)
# [[11, 22, 33],
#  [14, 25, 36]]

# Column vector + Matrix (broadcasts across columns)
col = tf.constant([[100],
                   [200]])           # (2, 1)
print(matrix + col)
# [[101, 102, 103],
#  [204, 205, 206]]

# ============================================
# PRACTICAL: Normalizing data (zero mean, unit variance)
# ============================================
data = tf.constant([[1.0, 200.0, 0.5],
                    [2.0, 400.0, 0.8],
                    [3.0, 600.0, 0.3]])  # (3, 3)

# Mean and std per column
mean = tf.reduce_mean(data, axis=0)      # (3,)  → [2.0, 400.0, 0.533]
std = tf.math.reduce_std(data, axis=0)   # (3,)

# Broadcasting: (3,3) - (3,) → each column gets its mean subtracted
normalized = (data - mean) / std
print(f"Normalized:\n{normalized}")
```

---

## 8. Eager Execution vs Graph Mode

### What It Is

**Eager Execution** = operations run immediately, like regular Python. You get results right away.  
**Graph Mode** = TensorFlow builds a computation graph first, then optimizes and runs it all at once.

**Analogy**: 
- **Eager** = Cooking each dish one at a time, tasting as you go
- **Graph** = Planning the entire menu, optimizing the kitchen workflow, then cooking everything efficiently

### Why It Matters

```
┌─────────────────────────────────────────────────────────┐
│                 Eager Execution (Default in TF2)         │
├─────────────────────────────────────────────────────────┤
│ ✅ Easy to debug (use print, pdb, breakpoints)          │
│ ✅ Natural Python flow (if/else, for loops work)        │
│ ✅ Instant results (good for prototyping)               │
│ ❌ Slower (no graph optimization, no parallelism)       │
│ ❌ Can't deploy as efficiently                          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              Graph Mode (via tf.function)                │
├─────────────────────────────────────────────────────────┤
│ ✅ Fast (optimized, parallelized, hardware-accelerated) │
│ ✅ Portable (can export to SavedModel, TF Lite, TF.js) │
│ ✅ Distributed training ready                           │
│ ❌ Harder to debug (print doesn't work normally)        │
│ ❌ Python side effects are tricky                       │
└─────────────────────────────────────────────────────────┘
```

```python
import tensorflow as tf

# ============================================
# EAGER EXECUTION (Default)
# ============================================

# Operations execute immediately — just like NumPy
a = tf.constant(5.0)
b = tf.constant(3.0)
c = a + b
print(c)           # tf.Tensor(8.0, shape=(), dtype=float32)
print(c.numpy())   # 8.0 — Can access value immediately!

# Verify eager mode is on
print(tf.executing_eagerly())  # True (default in TF 2.x)
```

---

## 9. tf.function — Performance with Graphs

### What It Is
`tf.function` is a decorator that converts a Python function into a TensorFlow graph. It's the bridge between eager execution's ease-of-use and graph mode's performance.

### How It Works

```
Without @tf.function:
    Python calls TF ops one by one → slow

With @tf.function:
    1. First call: TF "traces" the function → builds a graph
    2. Subsequent calls: Runs the optimized graph → FAST
    
    ┌──────────────┐     ┌───────────────┐     ┌──────────────┐
    │ Python Code  │ ──→ │ Trace → Graph │ ──→ │ Optimized    │
    │ (1st call)   │     │ (once)        │     │ Execution    │
    └──────────────┘     └───────────────┘     └──────────────┘
```

```python
import tensorflow as tf
import time

# ============================================
# BASIC tf.function USAGE
# ============================================

# Regular Python function (runs eagerly)
def regular_add(a, b):
    return a + b

# Graph-compiled function (runs as optimized graph)
@tf.function
def graph_add(a, b):
    return a + b

a = tf.constant(5.0)
b = tf.constant(3.0)

# Both produce the same result
print(regular_add(a, b))  # 8.0
print(graph_add(a, b))    # 8.0

# ============================================
# PERFORMANCE COMPARISON
# ============================================

@tf.function
def graph_matmul(x):
    for _ in range(100):
        x = tf.matmul(x, x)
    return x

def eager_matmul(x):
    for _ in range(100):
        x = tf.matmul(x, x)
    return x

x = tf.random.normal((100, 100))

# Warm up (first call traces the graph)
_ = graph_matmul(x)

# Benchmark
start = time.time()
for _ in range(10):
    eager_matmul(x)
eager_time = time.time() - start

start = time.time()
for _ in range(10):
    graph_matmul(x)
graph_time = time.time() - start

print(f"Eager: {eager_time:.4f}s")
print(f"Graph: {graph_time:.4f}s")
print(f"Speedup: {eager_time/graph_time:.1f}x")

# ============================================
# TRACING BEHAVIOR — Understanding retracing
# ============================================

@tf.function
def my_function(x):
    print("Tracing!")  # This runs ONLY during tracing, not on every call
    tf.print("Executing:", x)  # This runs on every call
    return x * 2

# First call with shape (2,) → traces the graph
my_function(tf.constant([1.0, 2.0]))
# Output: "Tracing!" (from Python print) + "Executing: [1 2]" (from tf.print)

# Second call with SAME shape → reuses graph, no retracing
my_function(tf.constant([3.0, 4.0]))
# Output: "Executing: [3 4]" (only tf.print, no "Tracing!")

# Third call with DIFFERENT shape → retraces!
my_function(tf.constant([5.0, 6.0, 7.0]))
# Output: "Tracing!" + "Executing: [5 6 7]"

# ============================================
# SPECIFYING INPUT SIGNATURES (avoid retracing)
# ============================================

@tf.function(input_signature=[tf.TensorSpec(shape=[None], dtype=tf.float32)])
def flexible_function(x):
    print("Tracing!")  # Only traces ONCE because shape=[None] handles any length
    return x * 2

flexible_function(tf.constant([1.0, 2.0]))       # Traces once
flexible_function(tf.constant([1.0, 2.0, 3.0]))  # Reuses graph! No retrace
```

### Common tf.function Pitfalls

```python
# ============================================
# PITFALL 1: Python side effects in tf.function
# ============================================

my_list = []

@tf.function
def buggy_append(x):
    my_list.append(x)  # ⚠️ This only runs during TRACING
    return x

buggy_append(tf.constant(1.0))
buggy_append(tf.constant(2.0))
buggy_append(tf.constant(3.0))
print(len(my_list))  # 1, NOT 3! Only traced once.

# ============================================
# PITFALL 2: Python if/else vs tf.cond
# ============================================

@tf.function
def bad_branch(x):
    if x > 0:       # ⚠️ Python if: evaluated ONCE during tracing
        return x
    else:
        return -x

# Better: use tf.cond for dynamic branching
@tf.function
def good_branch(x):
    return tf.cond(x > 0, lambda: x, lambda: -x)  # ✅ Traced both branches

# ============================================
# PITFALL 3: Python for loop vs tf.while_loop
# ============================================

@tf.function
def python_loop(n):
    result = 0
    for i in range(n):  # ⚠️ If n is a Python int, loop is UNROLLED during tracing
        result += i     # 1000 iterations → 1000 nodes in graph!
    return result

# Better for dynamic loops:
@tf.function
def tf_loop(n):
    result = tf.constant(0)
    for i in tf.range(n):  # ✅ tf.range creates a graph loop
        result += i
    return result
```

> **Rule of Thumb**: Inside `@tf.function`, use TF ops (`tf.cond`, `tf.while_loop`, `tf.print`) instead of Python equivalents (`if`, `while`, `print`).

---

## 10. GradientTape — Automatic Differentiation

### What It Is
`tf.GradientTape` is TensorFlow's system for **automatic differentiation** — it records operations on tensors, then calculates gradients (derivatives) automatically. This is how neural networks learn.

### Why It Matters
Backpropagation = computing gradients of the loss with respect to every weight. Doing this manually for a network with millions of parameters is impossible. GradientTape does it automatically.

**Analogy**: Imagine you're walking through a forest and leaving breadcrumbs (recording operations). When you want to find your way back (compute gradients), you just follow the breadcrumbs in reverse.

### How It Works

```
Forward Pass (recording):
    x → [op1] → a → [op2] → b → [op3] → loss
    
    GradientTape records: op1, op2, op3
    
Backward Pass (gradient computation):
    ∂loss/∂x = ∂loss/∂b × ∂b/∂a × ∂a/∂x    (chain rule!)
    
    GradientTape computes this automatically
```

```python
import tensorflow as tf

# ============================================
# BASIC GRADIENT COMPUTATION
# ============================================

# Compute derivative of y = x² at x = 3
# dy/dx = 2x = 6

x = tf.Variable(3.0)  # Must be a Variable (not constant) to track gradients

with tf.GradientTape() as tape:
    y = x ** 2  # y = x²

# Compute gradient: dy/dx
dy_dx = tape.gradient(y, x)
print(f"dy/dx at x=3: {dy_dx.numpy()}")  # 6.0 ✅ (2 × 3 = 6)

# ============================================
# MULTIPLE VARIABLES
# ============================================

w = tf.Variable(2.0)
b = tf.Variable(1.0)
x = tf.constant(5.0)

with tf.GradientTape() as tape:
    y = w * x + b  # y = 2*5 + 1 = 11
    
# Compute gradients w.r.t. both w and b
dw, db = tape.gradient(y, [w, b])
print(f"dy/dw = {dw.numpy()}")  # 5.0 (dy/dw = x = 5)
print(f"dy/db = {db.numpy()}")  # 1.0 (dy/db = 1)

# ============================================
# GRADIENT FOR LOSS FUNCTION (real ML scenario)
# ============================================

# Simple linear regression: y_pred = w*x + b
# Loss = MSE = mean((y_true - y_pred)²)

# Data
x_train = tf.constant([1.0, 2.0, 3.0, 4.0])
y_train = tf.constant([3.0, 5.0, 7.0, 9.0])  # y = 2x + 1

# Parameters (learnable)
w = tf.Variable(0.0)  # Initial guess
b = tf.Variable(0.0)  # Initial guess

learning_rate = 0.01

# One training step
with tf.GradientTape() as tape:
    y_pred = w * x_train + b
    loss = tf.reduce_mean((y_train - y_pred) ** 2)  # MSE

gradients = tape.gradient(loss, [w, b])
print(f"Loss: {loss.numpy():.4f}")
print(f"Gradient w.r.t. w: {gradients[0].numpy():.4f}")
print(f"Gradient w.r.t. b: {gradients[1].numpy():.4f}")

# Update parameters (gradient descent)
w.assign_sub(learning_rate * gradients[0])  # w = w - lr * dL/dw
b.assign_sub(learning_rate * gradients[1])  # b = b - lr * dL/db
print(f"Updated w: {w.numpy():.4f}, b: {b.numpy():.4f}")

# ============================================
# FULL TRAINING LOOP (from scratch!)
# ============================================

w = tf.Variable(0.0)
b = tf.Variable(0.0)
learning_rate = 0.05

for epoch in range(100):
    with tf.GradientTape() as tape:
        y_pred = w * x_train + b
        loss = tf.reduce_mean((y_train - y_pred) ** 2)
    
    gradients = tape.gradient(loss, [w, b])
    w.assign_sub(learning_rate * gradients[0])
    b.assign_sub(learning_rate * gradients[1])
    
    if epoch % 20 == 0:
        print(f"Epoch {epoch}: loss={loss.numpy():.4f}, w={w.numpy():.4f}, b={b.numpy():.4f}")

print(f"\nFinal: w={w.numpy():.4f}, b={b.numpy():.4f}")
# Should converge to w≈2.0, b≈1.0 (the true values!)
```

### Advanced GradientTape

```python
# ============================================
# PERSISTENT TAPE (use gradient multiple times)
# ============================================

x = tf.Variable(3.0)

# Normal tape: can only call .gradient() ONCE
# Persistent tape: can call .gradient() MULTIPLE times
with tf.GradientTape(persistent=True) as tape:
    y = x ** 2
    z = x ** 3

dy_dx = tape.gradient(y, x)  # 6.0
dz_dx = tape.gradient(z, x)  # 27.0 (3 × 3² = 27)

# IMPORTANT: Delete persistent tape to free memory
del tape

print(f"dy/dx = {dy_dx.numpy()}")  # 6.0
print(f"dz/dx = {dz_dx.numpy()}")  # 27.0

# ============================================
# HIGHER-ORDER GRADIENTS (gradient of gradient)
# ============================================

x = tf.Variable(3.0)

with tf.GradientTape() as tape2:
    with tf.GradientTape() as tape1:
        y = x ** 3              # y = x³
    dy_dx = tape1.gradient(y, x)  # dy/dx = 3x² = 27
d2y_dx2 = tape2.gradient(dy_dx, x)  # d²y/dx² = 6x = 18

print(f"First derivative: {dy_dx.numpy()}")    # 27.0
print(f"Second derivative: {d2y_dx2.numpy()}")  # 18.0

# ============================================
# WATCHING NON-VARIABLE TENSORS
# ============================================

x = tf.constant(3.0)  # Constants are NOT watched by default

with tf.GradientTape() as tape:
    tape.watch(x)  # Explicitly watch this constant
    y = x ** 2

dy_dx = tape.gradient(y, x)
print(f"Gradient of constant: {dy_dx.numpy()}")  # 6.0
```

---

## 11. Variables vs Constants

### What It Is

| Feature | `tf.constant` | `tf.Variable` |
|---------|---------------|---------------|
| **Mutability** | Immutable (can't change) | Mutable (can change in-place) |
| **Use case** | Input data, hyperparameters | Model weights, biases |
| **GradientTape** | Not tracked by default | Automatically tracked |
| **Memory** | Can be placed on any device | Has special memory management |
| **Creation** | `tf.constant([1, 2])` | `tf.Variable([1, 2])` |

### How It Works

```python
import tensorflow as tf

# ============================================
# tf.constant — Immutable
# ============================================

c = tf.constant([1.0, 2.0, 3.0])
# c.assign([4, 5, 6])    # ❌ ERROR! Constants can't be modified
# c[0] = 99              # ❌ ERROR! Can't assign to elements

# To "change" a constant, create a new one
c_new = c * 2  # Creates a NEW tensor, c is unchanged

# ============================================
# tf.Variable — Mutable (for model parameters)
# ============================================

v = tf.Variable([1.0, 2.0, 3.0])

# Modify in-place
v.assign([4.0, 5.0, 6.0])          # Replace entire value
print(f"After assign: {v.numpy()}")  # [4. 5. 6.]

v.assign_add([1.0, 1.0, 1.0])      # Add in-place
print(f"After assign_add: {v.numpy()}")  # [5. 6. 7.]

v.assign_sub([2.0, 2.0, 2.0])      # Subtract in-place
print(f"After assign_sub: {v.numpy()}")  # [3. 4. 5.]

v[0].assign(99.0)                   # Modify single element
print(f"After element assign: {v.numpy()}")  # [99. 4. 5.]

# ============================================
# WHY THIS MATTERS IN TRAINING
# ============================================
# During training, we update weights using gradients:
#   weight.assign_sub(learning_rate * gradient)
# 
# This MUST be a Variable (not constant) because we modify it in-place.
# Using assign_sub is efficient — no new tensor allocation.
```

---

## 12. TensorFlow vs NumPy

### Comparison

| Feature | NumPy | TensorFlow |
|---------|-------|------------|
| **GPU Support** | ❌ CPU only | ✅ CPU, GPU, TPU |
| **Auto Differentiation** | ❌ No | ✅ GradientTape |
| **Graph Compilation** | ❌ No | ✅ tf.function |
| **Distributed Computing** | ❌ No | ✅ Built-in strategies |
| **Deployment** | ❌ Not designed for it | ✅ TF Serving, TF Lite, TF.js |
| **Syntax** | More Pythonic | Similar but some differences |
| **Speed (CPU)** | Often faster for small ops | Overhead for small ops |
| **Speed (GPU)** | N/A | Much faster for large ops |
| **Mutability** | Arrays are mutable | Tensors are immutable |

```python
import numpy as np
import tensorflow as tf

# Most operations look identical
np_arr = np.array([1, 2, 3])
tf_ten = tf.constant([1, 2, 3])

# Same operations, different frameworks
print(np.sum(np_arr))          # 6
print(tf.reduce_sum(tf_ten))   # 6

print(np.mean(np_arr.astype(float)))  # 2.0
print(tf.reduce_mean(tf.cast(tf_ten, tf.float32)))  # 2.0

# Key naming differences:
# NumPy              TensorFlow
# np.sum()           tf.reduce_sum()
# np.mean()          tf.reduce_mean()
# np.max()           tf.reduce_max()
# np.concatenate()   tf.concat()
# np.stack()         tf.stack()
# arr.reshape()      tf.reshape()
# np.dot()           tf.matmul() or tf.tensordot()
# np.where()         tf.where()
```

---

## 13. GPU/TPU Acceleration

### How It Works

```python
import tensorflow as tf

# ============================================
# CHECK AVAILABLE DEVICES
# ============================================

print("Available devices:")
for device in tf.config.list_physical_devices():
    print(f"  {device}")

# Check GPU specifically
gpus = tf.config.list_physical_devices('GPU')
print(f"\nGPUs available: {len(gpus)}")

# ============================================
# DEVICE PLACEMENT
# ============================================

# TF automatically places ops on GPU if available
# To manually control placement:

with tf.device('/CPU:0'):
    cpu_tensor = tf.constant([1.0, 2.0, 3.0])
    print(f"CPU tensor device: {cpu_tensor.device}")

# If GPU is available:
# with tf.device('/GPU:0'):
#     gpu_tensor = tf.constant([1.0, 2.0, 3.0])
#     print(f"GPU tensor device: {gpu_tensor.device}")

# ============================================
# GPU MEMORY MANAGEMENT
# ============================================

# By default, TF grabs ALL GPU memory
# To allow memory growth (recommended):
gpus = tf.config.list_physical_devices('GPU')
if gpus:
    for gpu in gpus:
        tf.config.experimental.set_memory_growth(gpu, True)
    # Now TF only allocates GPU memory as needed

# Or set a hard memory limit:
# tf.config.set_logical_device_configuration(
#     gpus[0],
#     [tf.config.LogicalDeviceConfiguration(memory_limit=4096)]  # 4GB
# )

# ============================================
# PERFORMANCE COMPARISON: CPU vs GPU
# ============================================

import time

size = 5000

# Matrix multiplication benchmark
a = tf.random.normal((size, size))
b = tf.random.normal((size, size))

# Warm up
_ = tf.matmul(a, b)

start = time.time()
for _ in range(10):
    c = tf.matmul(a, b)
elapsed = time.time() - start
print(f"10x matmul ({size}×{size}): {elapsed:.3f}s on {c.device}")
```

> **Pro Tip**: Set `TF_CPP_MIN_LOG_LEVEL=2` as environment variable to suppress TF info/warning messages:
> ```python
> import os
> os.environ['TF_CPP_MIN_LOG_LEVEL'] = '2'  # 0=all, 1=no INFO, 2=no WARNING, 3=no ERROR
> import tensorflow as tf
> ```

---

## 14. Common Mistakes

### Mistake 1: Mixing dtypes

```python
# ❌ WRONG: Mixing int and float
a = tf.constant([1, 2, 3])       # int32
b = tf.constant([1.0, 2.0, 3.0]) # float32
# c = a + b  # ERROR: cannot compute Add with int32 and float32

# ✅ FIX: Cast to same dtype
c = tf.cast(a, tf.float32) + b
```

### Mistake 2: Forgetting that tensors are immutable

```python
# ❌ WRONG: Trying to modify a tensor
t = tf.constant([1, 2, 3])
# t[0] = 10  # ERROR!

# ✅ FIX: Use tf.tensor_scatter_nd_update or create new tensor
t_new = tf.tensor_scatter_nd_update(t, [[0]], [10])
```

### Mistake 3: Using NumPy ops inside tf.function

```python
# ❌ WRONG: NumPy inside tf.function
@tf.function
def bad(x):
    return np.mean(x.numpy())  # Can't call .numpy() in graph mode!

# ✅ FIX: Use TF ops
@tf.function
def good(x):
    return tf.reduce_mean(x)
```

### Mistake 4: Not using persistent tape

```python
# ❌ WRONG: Using tape.gradient() twice
x = tf.Variable(3.0)
with tf.GradientTape() as tape:
    y = x ** 2
    z = x ** 3
g1 = tape.gradient(y, x)
# g2 = tape.gradient(z, x)  # ERROR: tape already consumed!

# ✅ FIX: Use persistent=True
with tf.GradientTape(persistent=True) as tape:
    y = x ** 2
    z = x ** 3
g1 = tape.gradient(y, x)
g2 = tape.gradient(z, x)  # Works!
del tape  # Always delete persistent tape
```

### Mistake 5: Not setting random seeds

```python
# ❌ WRONG: Non-reproducible results
x = tf.random.normal((3, 3))  # Different every run!

# ✅ FIX: Set seeds
tf.random.set_seed(42)
x = tf.random.normal((3, 3))  # Same every run
```

---

## 15. Interview Questions

### Q1: What is a Tensor? How is it different from a NumPy array?
**Answer**: A tensor is a multi-dimensional array, similar to a NumPy ndarray. Key differences:
1. Tensors can run on **GPUs/TPUs**, NumPy arrays can't
2. Tensors support **automatic differentiation** (GradientTape)
3. Tensors are **immutable** (unless tf.Variable), NumPy arrays are mutable
4. Tensors can be compiled into **optimized graphs** (tf.function)
5. TensorFlow tensors are designed for **deployment** (serving, mobile, web)

### Q2: What is Eager Execution? Why did TF 2.x make it default?
**Answer**: Eager execution means operations execute immediately, returning concrete values instead of building a graph. TF 2.x made it default because:
- Easier debugging (use Python debugger, print statements)
- More intuitive (behaves like regular Python/NumPy)
- Lower barrier to entry
- Still get graph performance via `@tf.function`

### Q3: What is tf.function and when should you use it?
**Answer**: `@tf.function` converts a Python function into a TensorFlow computation graph for performance. Use it when:
- Training loops need speed optimization
- Preparing models for deployment (SavedModel)
- Running on GPUs/TPUs (graph mode is faster)
- **Don't use** for debugging or prototyping

### Q4: Explain GradientTape. Why is it important?
**Answer**: `tf.GradientTape` records operations during the forward pass and computes gradients (derivatives) in the backward pass using the chain rule. It's the foundation of training neural networks — without it, you'd have to manually derive and code gradient formulas for every operation.

### Q5: What's the difference between tf.Variable and tf.constant?
**Answer**: 
- `tf.Variable`: mutable, tracked by GradientTape, used for model weights
- `tf.constant`: immutable, not tracked by default, used for data/hyperparameters
- During training, weights must be Variables so they can be updated via `assign_sub()`

### Q6: What is broadcasting? Give an example.
**Answer**: Broadcasting allows operations between tensors of different shapes by virtually expanding the smaller tensor. E.g., adding a bias vector `(128,)` to a batch output `(32, 128)` — the bias is "broadcast" across all 32 samples without copying data.

### Q7: How do you make TensorFlow training reproducible?
**Answer**: Set three seeds:
```python
import random, numpy as np, tensorflow as tf
random.seed(42)
np.random.seed(42)
tf.random.set_seed(42)
```
Also: use deterministic ops (`tf.config.experimental.enable_op_determinism()`), and note that GPU floating-point operations may have tiny non-deterministic variations.

---

## 16. Quick Reference

### Tensor Creation

| Function | Description | Example |
|----------|-------------|---------|
| `tf.constant()` | Create from values | `tf.constant([1,2,3])` |
| `tf.zeros()` | All zeros | `tf.zeros((3,4))` |
| `tf.ones()` | All ones | `tf.ones((2,3))` |
| `tf.fill()` | Fill with value | `tf.fill((2,3), 5.0)` |
| `tf.eye()` | Identity matrix | `tf.eye(4)` |
| `tf.range()` | Range of values | `tf.range(0, 10, 2)` |
| `tf.random.normal()` | Normal distribution | `tf.random.normal((3,3))` |
| `tf.random.uniform()` | Uniform distribution | `tf.random.uniform((3,3))` |
| `tf.Variable()` | Mutable tensor | `tf.Variable([1.0, 2.0])` |

### Tensor Operations

| Operation | Code |
|-----------|------|
| Matrix multiply | `tf.matmul(A, B)` or `A @ B` |
| Element-wise multiply | `tf.multiply(a, b)` or `a * b` |
| Transpose | `tf.transpose(M)` |
| Reshape | `tf.reshape(t, new_shape)` |
| Concatenate | `tf.concat([a, b], axis=0)` |
| Stack | `tf.stack([a, b], axis=0)` |
| Sum | `tf.reduce_sum(t, axis=0)` |
| Mean | `tf.reduce_mean(t, axis=0)` |
| Max | `tf.reduce_max(t, axis=0)` |
| Argmax | `tf.argmax(t, axis=0)` |
| Cast dtype | `tf.cast(t, tf.float32)` |
| Expand dims | `tf.expand_dims(t, axis=0)` |
| Squeeze | `tf.squeeze(t)` |
| Gather | `tf.gather(t, indices)` |
| Where | `tf.where(condition, x, y)` |

### Key Concepts

| Concept | What It Does | When to Use |
|---------|--------------|-------------|
| Eager Execution | Ops run immediately | Debugging, prototyping |
| `@tf.function` | Compiles to graph | Training, deployment |
| `GradientTape` | Computes gradients | Custom training loops |
| `tf.Variable` | Mutable tensor | Model weights |
| `tf.constant` | Immutable tensor | Data, hyperparameters |
| Broadcasting | Auto shape matching | Batch operations |

---

*Next Chapter: [02-Keras-Sequential-and-Functional-API](02-Keras-Sequential-and-Functional-API.md)*
