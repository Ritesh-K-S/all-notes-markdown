# Chapter 01: PyTorch Fundamentals

## Table of Contents
- [1. Introduction to PyTorch](#1-introduction-to-pytorch)
- [2. Tensors — The Core Data Structure](#2-tensors--the-core-data-structure)
- [3. Tensor Operations](#3-tensor-operations)
- [4. Autograd — Automatic Differentiation](#4-autograd--automatic-differentiation)
- [5. GPU Acceleration with CUDA](#5-gpu-acceleration-with-cuda)
- [6. NumPy Bridge](#6-numpy-bridge)
- [7. Common Mistakes](#7-common-mistakes)
- [8. Interview Questions](#8-interview-questions)
- [9. Quick Reference](#9-quick-reference)

---

## 1. Introduction to PyTorch

### What It Is
PyTorch is an open-source deep learning framework developed by Meta (Facebook) AI Research. Think of it as a supercharged NumPy that can run on GPUs and automatically compute gradients (derivatives) — the two things you need to train neural networks.

### Why It Matters
- **#1 framework in research** — over 80% of papers at top AI conferences use PyTorch
- **Pythonic and intuitive** — feels like writing normal Python, not a separate language
- **Dynamic computation graphs** — you build the graph as you run (eager execution), making debugging trivial
- **Industry adoption** — Tesla, OpenAI, Meta, Microsoft all use PyTorch in production
- **Ecosystem** — Hugging Face, Lightning, torchvision, torchaudio all built on PyTorch

### How It Works — The Big Picture

```
┌─────────────────────────────────────────────────────┐
│                   PyTorch Stack                       │
├─────────────────────────────────────────────────────┤
│  Your Code (Python)                                  │
│       ↓                                              │
│  torch.Tensor  ←→  Autograd Engine                   │
│       ↓                                              │
│  ATen (C++ Tensor Library)                           │
│       ↓                                              │
│  CUDA / cuDNN / MPS / CPU backends                   │
└─────────────────────────────────────────────────────┘
```

PyTorch's core idea: **Tensors + Automatic Differentiation + GPU = Deep Learning**

### Installation

```python
# CPU only
# pip install torch torchvision torchaudio

# CUDA 12.1 (NVIDIA GPU)
# pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121

# Verify installation
import torch
print(f"PyTorch version: {torch.__version__}")
print(f"CUDA available: {torch.cuda.is_available()}")
print(f"CUDA version: {torch.version.cuda}")
```

---

## 2. Tensors — The Core Data Structure

### What It Is
A **tensor** is a multi-dimensional array — the fundamental data container in PyTorch. It's like a NumPy array on steroids: it can live on a GPU and track its own gradients.

**Real-world analogy:** Think of tensors like containers of different shapes:
- **Scalar (0D):** A single number — temperature reading (23.5°C)
- **Vector (1D):** A list of numbers — stock prices for a week [150, 152, 148, ...]
- **Matrix (2D):** A spreadsheet — a grayscale image (height × width)
- **3D Tensor:** A cube — a color image (3 channels × height × width)
- **4D Tensor:** A batch — 32 color images (batch × channels × height × width)

```
Scalar      Vector       Matrix         3D Tensor
  5         [1,2,3]     [[1,2,3],      [[[1,2],
                          [4,5,6]]       [3,4]],
                                        [[5,6],
 (0-D)      (1-D)       (2-D)           [7,8]]]
                                        (3-D)
```

### Why It Matters
Every piece of data in deep learning — images, text, audio, video — gets converted to tensors before a model can process it. Understanding tensors is like understanding letters before writing essays.

### Creating Tensors

```python
import torch

# === From Python data ===
# Scalar (0-dimensional)
scalar = torch.tensor(7)
print(f"Scalar: {scalar}, ndim: {scalar.ndim}, shape: {scalar.shape}")
# Output: Scalar: tensor(7), ndim: 0, shape: torch.Size([])

# Vector (1-dimensional)
vector = torch.tensor([1, 2, 3, 4])
print(f"Vector: {vector}, ndim: {vector.ndim}, shape: {vector.shape}")
# Output: Vector: tensor([1, 2, 3, 4]), ndim: 1, shape: torch.Size([4])

# Matrix (2-dimensional)
matrix = torch.tensor([[1, 2, 3],
                       [4, 5, 6]])
print(f"Matrix shape: {matrix.shape}")  # torch.Size([2, 3])

# 3D Tensor
tensor_3d = torch.tensor([[[1, 2], [3, 4]],
                           [[5, 6], [7, 8]]])
print(f"3D Tensor shape: {tensor_3d.shape}")  # torch.Size([2, 2, 2])

# === Factory functions (most common in practice) ===
# Zeros and Ones
zeros = torch.zeros(3, 4)          # 3×4 matrix of zeros
ones = torch.ones(2, 3, 4)        # 2×3×4 tensor of ones

# Random tensors (used for weight initialization)
rand_uniform = torch.rand(3, 3)    # Uniform [0, 1)
rand_normal = torch.randn(3, 3)   # Standard normal (mean=0, std=1)

# Range tensors
arange = torch.arange(0, 10, 2)   # tensor([0, 2, 4, 6, 8])
linspace = torch.linspace(0, 1, 5) # tensor([0.0, 0.25, 0.5, 0.75, 1.0])

# Identity matrix
eye = torch.eye(3)                 # 3×3 identity matrix

# Full (fill with a specific value)
full = torch.full((2, 3), fill_value=3.14)

# Like functions (create tensor with same shape/dtype as another)
x = torch.tensor([[1, 2], [3, 4]], dtype=torch.float32)
zeros_like_x = torch.zeros_like(x)  # Same shape, filled with 0
ones_like_x = torch.ones_like(x)    # Same shape, filled with 1
rand_like_x = torch.rand_like(x)    # Same shape, random values

print(f"rand_uniform:\n{rand_uniform}")
print(f"arange: {arange}")
print(f"linspace: {linspace}")
```

### Tensor Attributes (The Big 3)

Every tensor has three critical properties:

```python
import torch

t = torch.rand(3, 4)

# 1. Shape — dimensions of the tensor
print(f"Shape: {t.shape}")          # torch.Size([3, 4])
print(f"Size: {t.size()}")          # Same as .shape (method vs property)

# 2. Dtype — data type of elements
print(f"Dtype: {t.dtype}")          # torch.float32 (default for torch.rand)

# 3. Device — where the tensor lives (CPU or GPU)
print(f"Device: {t.device}")        # cpu
```

### Data Types (dtypes)

| dtype | Description | Use Case |
|-------|-------------|----------|
| `torch.float32` (default) | 32-bit float | Model weights, general computation |
| `torch.float16` | 16-bit float | Mixed precision training |
| `torch.bfloat16` | Brain float 16 | Training on TPUs/modern GPUs |
| `torch.float64` | 64-bit float | High precision (rarely needed) |
| `torch.int32` | 32-bit integer | Indices, counting |
| `torch.int64` (`torch.long`) | 64-bit integer | Class labels, indices in NLP |
| `torch.bool` | Boolean | Masks, conditions |

```python
# Specifying dtype at creation
x = torch.tensor([1, 2, 3], dtype=torch.float32)
y = torch.zeros(3, 4, dtype=torch.int64)

# Casting (converting) dtypes
x_int = x.to(torch.int32)       # Method 1: .to()
x_int = x.int()                  # Method 2: shorthand
x_float = x_int.float()         # int → float

# Pro Tip: Class labels MUST be torch.long (int64) for loss functions
labels = torch.tensor([0, 1, 2, 1], dtype=torch.long)
```

> **⚠️ Important:** PyTorch defaults to `float32` for floating-point and `int64` for integers. Always verify dtypes match when combining tensors — mismatches cause silent bugs or errors.

### Tensor Shape Manipulation

```python
import torch

x = torch.arange(12)  # tensor([0, 1, 2, ..., 11])
print(f"Original: {x.shape}")  # torch.Size([12])

# === Reshape ===
# Rearranges elements into new shape (total elements must match)
reshaped = x.reshape(3, 4)     # 12 elements → 3 rows × 4 cols
reshaped = x.reshape(2, 2, 3)  # 12 elements → 2×2×3
reshaped = x.reshape(3, -1)    # -1 means "figure it out" → 3×4

# === View (memory-efficient reshape) ===
# Same as reshape but REQUIRES contiguous memory (shares data!)
viewed = x.view(3, 4)  # No copy — changes to viewed affect x!

# === Squeeze and Unsqueeze ===
# squeeze: Remove dimensions of size 1
a = torch.zeros(1, 3, 1, 4)
print(f"Before squeeze: {a.shape}")    # torch.Size([1, 3, 1, 4])
print(f"After squeeze: {a.squeeze().shape}")  # torch.Size([3, 4])
print(f"Squeeze dim 0: {a.squeeze(0).shape}") # torch.Size([3, 1, 4])

# unsqueeze: Add a dimension of size 1
b = torch.zeros(3, 4)
print(f"Before unsqueeze: {b.shape}")         # torch.Size([3, 4])
print(f"Unsqueeze dim 0: {b.unsqueeze(0).shape}")  # torch.Size([1, 3, 4])  — batch dim
print(f"Unsqueeze dim -1: {b.unsqueeze(-1).shape}") # torch.Size([3, 4, 1])

# === Transpose and Permute ===
c = torch.rand(2, 3, 4)
# transpose: swap exactly TWO dimensions
transposed = c.transpose(0, 2)  # Shape: [4, 3, 2]

# permute: reorder ALL dimensions (more general)
permuted = c.permute(2, 0, 1)   # Shape: [4, 2, 3]

# Common use: converting image formats
# PyTorch uses: [C, H, W] — channels first
# NumPy/PIL uses: [H, W, C] — channels last
img_pytorch = torch.rand(3, 224, 224)         # C, H, W
img_numpy = img_pytorch.permute(1, 2, 0)     # H, W, C

# === Flatten ===
d = torch.rand(2, 3, 4)
flat = d.flatten()              # → shape [24]
flat_from_1 = d.flatten(1)     # Flatten from dim 1 → shape [2, 12]

# === Stack and Cat (combining tensors) ===
t1 = torch.tensor([1, 2, 3])
t2 = torch.tensor([4, 5, 6])

# cat: concatenate along EXISTING dimension
catted = torch.cat([t1, t2], dim=0)  # tensor([1,2,3,4,5,6]) — shape [6]

# stack: stack along NEW dimension
stacked = torch.stack([t1, t2], dim=0)  # tensor([[1,2,3],[4,5,6]]) — shape [2,3]

# Pro Tip: stack = unsqueeze + cat
# Use cat to make sequences longer, stack to create batches
```

> **Pro Tip:** `view()` vs `reshape()` — Use `view()` when you want to guarantee no data copy (faster, but requires contiguous memory). Use `reshape()` when you don't care about copies and just want it to work.

---

## 3. Tensor Operations

### What It Is
Tensor operations are the mathematical building blocks of neural networks. Every forward pass, backward pass, and data transformation is a sequence of tensor operations.

### Why It Matters
Understanding operations lets you:
- Build custom layers and architectures
- Debug shape mismatches (the #1 PyTorch error)
- Write efficient code (vectorized operations vs loops)
- Understand what happens inside `nn.Linear`, `nn.Conv2d`, etc.

### Arithmetic Operations

```python
import torch

a = torch.tensor([1.0, 2.0, 3.0])
b = torch.tensor([4.0, 5.0, 6.0])

# Element-wise operations (same shape required, or broadcastable)
add = a + b           # tensor([5., 7., 9.])    — same as torch.add(a, b)
sub = a - b           # tensor([-3., -3., -3.]) — same as torch.sub(a, b)
mul = a * b           # tensor([4., 10., 18.])  — same as torch.mul(a, b)
div = a / b           # tensor([0.25, 0.4, 0.5])— same as torch.div(a, b)
power = a ** 2        # tensor([1., 4., 9.])    — same as torch.pow(a, 2)

# In-place operations (modifies tensor directly — saves memory)
# Convention: trailing underscore _ means in-place
a.add_(1)             # a is now [2., 3., 4.] — NO new tensor created
# ⚠️ Warning: in-place ops can break autograd! Avoid during training.
```

### Matrix Operations

```python
import torch

# Matrix multiplication — THE most important operation in deep learning
A = torch.tensor([[1., 2.], [3., 4.]])  # 2×2
B = torch.tensor([[5., 6.], [7., 8.]])  # 2×2

# Three ways to do matrix multiplication:
result1 = A @ B                    # Operator (cleanest)
result2 = torch.matmul(A, B)      # Function (most general)
result3 = torch.mm(A, B)          # Only for 2D matrices
# All give: tensor([[19., 22.], [43., 50.]])

# Matrix-vector multiplication
x = torch.tensor([1., 2.])        # Vector of size 2
result = A @ x                     # tensor([5., 11.])

# Batch matrix multiplication (for batched operations)
# Shape: (batch, n, m) @ (batch, m, p) → (batch, n, p)
batch_A = torch.rand(32, 10, 20)  # 32 matrices of 10×20
batch_B = torch.rand(32, 20, 5)   # 32 matrices of 20×5
batch_result = torch.bmm(batch_A, batch_B)  # Shape: [32, 10, 5]

# Dot product (1D tensors only)
v1 = torch.tensor([1., 2., 3.])
v2 = torch.tensor([4., 5., 6.])
dot = torch.dot(v1, v2)           # tensor(32.) = 1*4 + 2*5 + 3*6

# Transpose
print(A.T)                         # Shorthand for 2D transpose
print(A.t())                       # Same thing

# Determinant and Inverse
det = torch.linalg.det(A)         # Determinant
inv = torch.linalg.inv(A)         # Inverse

# Einstein summation (advanced but powerful)
# "ij,jk->ik" means matrix multiplication
result = torch.einsum('ij,jk->ik', A, B)
```

> **Key Rule:** For `A @ B` to work, the last dimension of A must equal the second-to-last dimension of B:
> `(m, n) @ (n, p) → (m, p)`

### Broadcasting

Broadcasting allows operations between tensors of different shapes by automatically expanding dimensions.

```python
import torch

# Rule: Dimensions are compared from right to left
# Compatible if: dimensions are equal OR one of them is 1

# Example 1: Scalar broadcast
a = torch.tensor([[1, 2, 3],
                  [4, 5, 6]])      # Shape: [2, 3]
b = torch.tensor(10)               # Shape: [] (scalar)
result = a + b                     # b broadcasts to [[10,10,10],[10,10,10]]
# Result: tensor([[11, 12, 13], [14, 15, 16]])

# Example 2: Vector broadcast
a = torch.tensor([[1, 2, 3],
                  [4, 5, 6]])      # Shape: [2, 3]
b = torch.tensor([10, 20, 30])    # Shape: [3]
result = a + b                     # b broadcasts to [[10,20,30],[10,20,30]]
# Result: tensor([[11, 22, 33], [14, 25, 36]])

# Example 3: Column broadcast
a = torch.tensor([[1, 2, 3],
                  [4, 5, 6]])      # Shape: [2, 3]
b = torch.tensor([[10], [20]])    # Shape: [2, 1]
result = a + b                     # b broadcasts along columns
# Result: tensor([[11, 12, 13], [24, 25, 26]])

# Visual: Broadcasting rules
#   Shape of a:  2 × 3
#   Shape of b:      3     ← automatically becomes 1 × 3, then expands to 2 × 3
#   Result:      2 × 3
```

```
Broadcasting Visualization:

a (2×3):        b (3,):          a + b (2×3):
┌───┬───┬───┐   [10,20,30]      ┌────┬────┬────┐
│ 1 │ 2 │ 3 │        ↓          │ 11 │ 22 │ 33 │
├───┼───┼───┤   ┌──┬──┬──┐      ├────┼────┼────┤
│ 4 │ 5 │ 6 │   │10│20│30│      │ 14 │ 25 │ 36 │
└───┴───┴───┘   │10│20│30│      └────┴────┴────┘
                └──┴──┴──┘
```

### Reduction Operations

```python
import torch

x = torch.tensor([[1., 2., 3.],
                  [4., 5., 6.]])  # Shape: [2, 3]

# Sum
print(x.sum())           # tensor(21.) — sum of all elements
print(x.sum(dim=0))      # tensor([5., 7., 9.]) — sum along rows (collapse dim 0)
print(x.sum(dim=1))      # tensor([6., 15.]) — sum along columns (collapse dim 1)

# Mean
print(x.mean())          # tensor(3.5)
print(x.mean(dim=0))     # tensor([2.5, 3.5, 4.5])
print(x.mean(dim=1))     # tensor([2., 5.])

# Max and Min
print(x.max())           # tensor(6.)
print(x.max(dim=1))      # values=tensor([3., 6.]), indices=tensor([2, 2])
print(x.argmax())        # tensor(5) — index of max in flattened tensor
print(x.argmax(dim=1))   # tensor([2, 2]) — index of max per row

# Standard deviation and Variance
print(x.std())           # Standard deviation
print(x.var())           # Variance

# keepdim — preserve dimensions (useful for broadcasting back)
sum_keepdim = x.sum(dim=1, keepdim=True)  # Shape: [2, 1] instead of [2]
# Now you can do: x - sum_keepdim (broadcasts correctly!)

# Pro Tip: Manual softmax using operations
logits = torch.tensor([2.0, 1.0, 0.1])
exp_logits = torch.exp(logits)
softmax = exp_logits / exp_logits.sum()  # tensor([0.6590, 0.2424, 0.0986])
# In practice: use torch.softmax(logits, dim=0)
```

### Comparison and Logical Operations

```python
import torch

x = torch.tensor([1, 2, 3, 4, 5])

# Comparison (returns boolean tensor)
print(x > 3)             # tensor([False, False, False, True, True])
print(x == 3)            # tensor([False, False, True, False, False])
print(x >= 3)            # tensor([False, False, True, True, True])

# Boolean indexing (like NumPy fancy indexing)
mask = x > 3
filtered = x[mask]       # tensor([4, 5])

# Where (conditional selection)
result = torch.where(x > 3, x, torch.zeros_like(x))
# tensor([0, 0, 0, 4, 5]) — keep values > 3, replace others with 0

# All and Any
print((x > 0).all())     # tensor(True) — all elements > 0?
print((x > 4).any())     # tensor(True) — any element > 4?

# Clamp (clip values to range)
clamped = torch.clamp(x.float(), min=2.0, max=4.0)
# tensor([2., 2., 3., 4., 4.])
```

### Indexing and Slicing

```python
import torch

x = torch.tensor([[1, 2, 3, 4],
                  [5, 6, 7, 8],
                  [9, 10, 11, 12]])  # Shape: [3, 4]

# Basic indexing (same as NumPy)
print(x[0])              # tensor([1, 2, 3, 4]) — first row
print(x[0, 2])           # tensor(3) — row 0, col 2
print(x[:, 0])           # tensor([1, 5, 9]) — all rows, first col
print(x[1:, :2])         # tensor([[5, 6], [9, 10]]) — rows 1+, first 2 cols

# Advanced indexing
rows = torch.tensor([0, 2])
cols = torch.tensor([1, 3])
print(x[rows, cols])     # tensor([2, 12]) — elements at (0,1) and (2,3)

# Boolean indexing
mask = x > 5
print(x[mask])           # tensor([6, 7, 8, 9, 10, 11, 12])

# Assign values
x_copy = x.clone()       # Always clone before modifying!
x_copy[0, 0] = 99
x_copy[:, -1] = 0        # Set last column to 0
```

---

## 4. Autograd — Automatic Differentiation

### What It Is
**Autograd** is PyTorch's automatic differentiation engine. It automatically computes gradients (derivatives) of any computation — the mathematical backbone of training neural networks via backpropagation.

**Analogy:** Imagine you're tracking a chain of recipe steps. Autograd is like a camera that records every step, so you can later rewind and figure out "if I changed ingredient X by a tiny amount, how would the final dish change?"

### Why It Matters
Training a neural network requires computing:

$$\frac{\partial \text{Loss}}{\partial \text{weight}}$$

for every single weight in the network (millions to billions!). Autograd does this automatically by recording operations on tensors and applying the chain rule:

$$\frac{\partial L}{\partial x} = \frac{\partial L}{\partial y} \cdot \frac{\partial y}{\partial x}$$

Without autograd, you'd have to manually derive and code gradients for every operation — impractical for networks with millions of parameters.

### How It Works

```
Forward Pass (Recorded by Autograd):
┌───┐     ┌───┐     ┌───┐     ┌──────┐
│ x │ ──→ │ × │ ──→ │ + │ ──→ │ Loss │
└───┘  w  └───┘  b  └───┘     └──────┘
              ↑          ↑
              w          b

Backward Pass (Chain Rule Applied):
┌──────┐     ┌───┐     ┌───┐     ┌───┐
│ Loss │ ──→ │ + │ ──→ │ × │ ──→ │ x │
└──────┘  ∂L └───┘  ∂L └───┘  ∂L └───┘
          /∂(x*w+b) /∂(x*w)   /∂x
```

The **computational graph** is built dynamically during the forward pass. Each operation creates a node. During `.backward()`, gradients flow backward through the graph.

### Basic Autograd Usage

```python
import torch

# === Step 1: Create tensor with requires_grad=True ===
# This tells PyTorch: "Track all operations on this tensor"
x = torch.tensor(3.0, requires_grad=True)
w = torch.tensor(2.0, requires_grad=True)
b = torch.tensor(1.0, requires_grad=True)

# === Step 2: Perform operations (forward pass) ===
# y = w*x + b = 2*3 + 1 = 7
y = w * x + b
print(f"y = {y}")  # tensor(7., grad_fn=<AddBackward0>)
# Notice: grad_fn tells you this tensor was created by an operation

# === Step 3: Compute gradients (backward pass) ===
y.backward()  # Computes dy/dx, dy/dw, dy/db

# === Step 4: Access gradients ===
print(f"dy/dx = {x.grad}")  # tensor(2.) — dy/dx = w = 2
print(f"dy/dw = {w.grad}")  # tensor(3.) — dy/dw = x = 3
print(f"dy/db = {b.grad}")  # tensor(1.) — dy/db = 1
```

### Gradient Computation with Multiple Steps

```python
import torch

x = torch.tensor(2.0, requires_grad=True)

# Complex computation: y = x³ + 2x² + x
y = x**3 + 2*x**2 + x
# dy/dx = 3x² + 4x + 1 = 3(4) + 4(2) + 1 = 21
y.backward()
print(f"dy/dx at x=2: {x.grad}")  # tensor(21.)

# === Gradient of vector outputs ===
x = torch.tensor([1.0, 2.0, 3.0], requires_grad=True)
y = x ** 2  # y = [1, 4, 9]

# backward() only works on scalar outputs
# For vector outputs, provide a gradient vector (Jacobian-vector product)
y.backward(torch.ones_like(y))  # Equivalent to summing first
print(f"Gradients: {x.grad}")  # tensor([2., 4., 6.]) — d(x²)/dx = 2x
```

### Practical Training Example

```python
import torch

# Simple linear regression: y = 2x + 1 (we want to learn w=2, b=1)
torch.manual_seed(42)

# Generate synthetic data
X = torch.linspace(0, 10, 100)
y_true = 2 * X + 1 + torch.randn(100) * 0.5  # Add noise

# Initialize parameters (random starting point)
w = torch.tensor(0.0, requires_grad=True)
b = torch.tensor(0.0, requires_grad=True)

learning_rate = 0.01

# Training loop
for epoch in range(100):
    # Forward pass
    y_pred = w * X + b
    
    # Compute loss (MSE)
    loss = ((y_pred - y_true) ** 2).mean()
    
    # Backward pass — compute gradients
    loss.backward()  # Computes dloss/dw and dloss/db
    
    # Update parameters (gradient descent)
    with torch.no_grad():  # Don't track these operations!
        w -= learning_rate * w.grad
        b -= learning_rate * b.grad
    
    # Zero gradients (CRUCIAL — gradients accumulate by default!)
    w.grad.zero_()
    b.grad.zero_()
    
    if epoch % 20 == 0:
        print(f"Epoch {epoch}: loss={loss.item():.4f}, w={w.item():.4f}, b={b.item():.4f}")

# Final: w≈2.0, b≈1.0
print(f"\nLearned: y = {w.item():.3f}x + {b.item():.3f}")
```

### Controlling Gradient Tracking

```python
import torch

x = torch.tensor(5.0, requires_grad=True)

# === Method 1: torch.no_grad() context manager ===
# Most common — used during inference/evaluation
with torch.no_grad():
    y = x * 2  # No gradient tracked
    print(y.requires_grad)  # False

# === Method 2: .detach() ===
# Creates a new tensor that shares data but has no grad history
y = x * 2
y_detached = y.detach()  # Breaks the computation graph at this point
print(y_detached.requires_grad)  # False

# === Method 3: torch.inference_mode() (PyTorch 2.0+) ===
# Faster than no_grad — use for pure inference
with torch.inference_mode():
    y = x * 2
    # Even more restricted — tensor can't be used in grad computations later

# === When to use each: ===
# torch.no_grad()      → Validation loop, parameter updates
# .detach()            → When you need to use the value but stop gradients
# torch.inference_mode() → Production inference (fastest)
```

### Gradient Accumulation (Important Concept)

```python
import torch

x = torch.tensor(3.0, requires_grad=True)

# Gradients ACCUMULATE (they're added, not replaced!)
y1 = x ** 2
y1.backward()
print(f"After first backward: {x.grad}")  # tensor(6.) = 2*3

y2 = x ** 3
y2.backward()
print(f"After second backward: {x.grad}")  # tensor(33.) = 6 + 27 = 6 + 3*3²

# This is BY DESIGN — useful for gradient accumulation with large batches
# But usually you want to zero gradients between iterations:
x.grad.zero_()
print(f"After zeroing: {x.grad}")  # tensor(0.)
```

> **⚠️ Critical:** Always call `optimizer.zero_grad()` (or `param.grad.zero_()`) before each backward pass, unless you intentionally want gradient accumulation.

---

## 5. GPU Acceleration with CUDA

### What It Is
CUDA is NVIDIA's parallel computing platform. PyTorch can move tensors and models to GPU memory for 10-100x faster computation, because GPUs have thousands of cores optimized for parallel math.

**Analogy:** CPU is like one brilliant mathematician solving problems one by one. GPU is like 10,000 average calculators working simultaneously — for matrix math, the swarm wins.

### Why It Matters
- Training ResNet on ImageNet: 2 weeks on CPU → 2 hours on GPU
- Large language models: impossible without GPUs
- Real-time inference: milliseconds vs seconds

### Device Management

```python
import torch

# === Check GPU availability ===
print(f"CUDA available: {torch.cuda.is_available()}")
print(f"Number of GPUs: {torch.cuda.device_count()}")
if torch.cuda.is_available():
    print(f"GPU name: {torch.cuda.get_device_name(0)}")
    print(f"GPU memory: {torch.cuda.get_device_properties(0).total_mem / 1e9:.1f} GB")

# === Device-agnostic code (BEST PRACTICE) ===
# This works whether you have a GPU or not
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(f"Using device: {device}")

# For Apple Silicon Macs:
# device = torch.device("mps" if torch.backends.mps.is_available() else "cpu")

# === Moving tensors to device ===
x = torch.rand(3, 3)              # Created on CPU
x_gpu = x.to(device)              # Moved to GPU (or stays on CPU)
x_gpu = x.cuda()                  # Explicitly move to GPU (CUDA only)
x_cpu = x_gpu.cpu()               # Move back to CPU

# === Creating tensors directly on device ===
y = torch.rand(3, 3, device=device)  # Created directly on GPU — faster!

# === IMPORTANT: Tensors must be on same device for operations ===
a = torch.rand(3, 3, device="cpu")
b = torch.rand(3, 3, device=device)
# c = a + b  # ERROR if device is cuda! Cannot mix devices.
c = a.to(device) + b  # ✓ Move a to GPU first
```

### GPU Memory Management

```python
import torch

# Check memory usage
if torch.cuda.is_available():
    print(f"Allocated: {torch.cuda.memory_allocated() / 1e6:.1f} MB")
    print(f"Cached: {torch.cuda.memory_reserved() / 1e6:.1f} MB")
    
    # Free unused cached memory
    torch.cuda.empty_cache()
    
    # Pro Tip: Use this in training loops to prevent OOM
    # del large_tensor
    # torch.cuda.empty_cache()

# === Monitoring GPU memory in training ===
def print_gpu_memory(tag=""):
    if torch.cuda.is_available():
        allocated = torch.cuda.memory_allocated() / 1e9
        reserved = torch.cuda.memory_reserved() / 1e9
        print(f"[{tag}] Allocated: {allocated:.2f} GB, Reserved: {reserved:.2f} GB")
```

### Device-Agnostic Code Pattern

```python
import torch
import torch.nn as nn

# The gold standard pattern for device handling:
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# Model to device
model = nn.Linear(10, 5).to(device)

# Data to device
x = torch.randn(32, 10).to(device)

# Forward pass (everything on same device)
output = model(x)

# When getting values back to Python/NumPy:
value = output[0, 0].item()           # Single value → Python float
numpy_array = output.detach().cpu().numpy()  # Tensor → NumPy (must be on CPU!)
```

---

## 6. NumPy Bridge

### What It Is
PyTorch provides seamless conversion between PyTorch tensors and NumPy arrays. They can even **share memory** (zero-copy), making integration with the broader Python scientific computing ecosystem effortless.

### Why It Matters
- Load data from NumPy/Pandas/scikit-learn → convert to tensors for training
- Use trained model outputs in NumPy/Matplotlib for visualization
- Interop with any library that uses NumPy arrays

### Conversions

```python
import torch
import numpy as np

# === NumPy → Tensor ===
np_array = np.array([1.0, 2.0, 3.0])

# Method 1: torch.from_numpy() — SHARES memory (no copy!)
tensor_shared = torch.from_numpy(np_array)
np_array[0] = 99                         # This changes the tensor too!
print(tensor_shared)                     # tensor([99., 2., 3.])

# Method 2: torch.tensor() — COPIES data (independent)
tensor_copy = torch.tensor(np_array)     # Safe copy
np_array[0] = 1                          # Doesn't affect tensor_copy

# === Tensor → NumPy ===
t = torch.tensor([1.0, 2.0, 3.0])

# .numpy() — shares memory (tensor must be on CPU!)
np_shared = t.numpy()
t[0] = 99
print(np_shared)                         # [99. 2. 3.] — changed!

# Safe copy:
np_copy = t.detach().cpu().numpy().copy()

# === For GPU tensors — must move to CPU first ===
if torch.cuda.is_available():
    gpu_tensor = torch.rand(3, device="cuda")
    # gpu_tensor.numpy()  # ERROR! Can't convert CUDA tensor to numpy
    np_from_gpu = gpu_tensor.detach().cpu().numpy()  # ✓

# === Data type mapping ===
# NumPy float64 → torch.float64 (by default)
# NumPy float32 → torch.float32
# NumPy int64   → torch.int64
np_f32 = np.array([1.0, 2.0], dtype=np.float32)
t_f32 = torch.from_numpy(np_f32)
print(t_f32.dtype)  # torch.float32
```

> **Pro Tip:** Use `torch.from_numpy()` for zero-copy when you won't modify the original array. Use `torch.tensor()` when you need an independent copy.

### Reproducibility (Seeds)

```python
import torch
import numpy as np
import random

def set_seed(seed=42):
    """Set all seeds for reproducibility."""
    torch.manual_seed(seed)              # CPU
    torch.cuda.manual_seed(seed)         # Current GPU
    torch.cuda.manual_seed_all(seed)     # All GPUs
    np.random.seed(seed)                 # NumPy
    random.seed(seed)                    # Python
    
    # For CUDA determinism (may reduce performance)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False

set_seed(42)

# Now all random operations are reproducible
a = torch.rand(3)
print(a)  # Always the same values with seed 42
```

---

## 7. Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Not zeroing gradients | Gradients accumulate across iterations | Call `optimizer.zero_grad()` before `.backward()` |
| Mixing devices | `RuntimeError: tensors on different devices` | Use `device = torch.device(...)` pattern consistently |
| Forgetting `.item()` | Storing tensor objects in lists → memory leak | Use `loss.item()` to extract scalar Python number |
| NumPy on GPU tensor | `Can't convert CUDA tensor to numpy` | Always `.detach().cpu().numpy()` |
| In-place ops + autograd | `RuntimeError: in-place operation` | Avoid `tensor.add_()` on tensors needing gradients |
| Wrong dtype for labels | `CrossEntropyLoss` expects `torch.long` | Cast labels: `labels.long()` |
| `view()` on non-contiguous | `RuntimeError: view size not compatible` | Use `.contiguous().view()` or `.reshape()` |
| Not using `torch.no_grad()` during eval | Wasted memory tracking gradients | Wrap inference in `with torch.no_grad():` |

---

## 8. Interview Questions

### Conceptual Questions

**Q1: What is a tensor? How is it different from a NumPy array?**
> A tensor is a multi-dimensional array that can: (1) run on GPU, (2) track gradients for autograd. NumPy arrays are CPU-only and have no gradient support. Under the hood, PyTorch tensors use ATen (C++ library) while NumPy uses its own C extensions.

**Q2: Explain dynamic vs static computation graphs.**
> PyTorch uses **dynamic** graphs (define-by-run): the graph is built fresh each forward pass. TensorFlow 1.x used **static** graphs (define-and-run): graph built once, then executed. Dynamic = easier debugging, conditional logic, variable-length inputs. Static = easier optimization, deployment.

**Q3: Why do we need to call `zero_grad()`?**
> PyTorch accumulates gradients by design (useful for gradient accumulation with large effective batch sizes). If you don't zero them, gradients from previous iterations add to current ones, giving wrong updates.

**Q4: What's the difference between `.detach()` and `torch.no_grad()`?**
> `.detach()` creates a new tensor disconnected from the computation graph (stops gradient flow at that point). `torch.no_grad()` is a context manager that prevents ALL operations inside it from being tracked. Use `.detach()` for specific tensors, `no_grad()` for entire blocks.

**Q5: Explain broadcasting. When does it fail?**
> Broadcasting expands tensor dimensions for element-wise operations. Rules (from right): dimensions must be equal or one must be 1. Fails when dimensions are different and neither is 1. Example: `[3,4] + [5]` fails because 4≠5 and neither is 1.

### Coding Questions

**Q6: Write code to manually implement a linear layer forward pass.**
```python
def linear_forward(x, weight, bias):
    # x: (batch, in_features)
    # weight: (out_features, in_features)
    # bias: (out_features,)
    return x @ weight.T + bias  # Broadcasting handles batch dim
```

**Q7: How would you move an entire model + data pipeline to GPU efficiently?**
```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = model.to(device)          # Move once
for batch_x, batch_y in dataloader:
    batch_x = batch_x.to(device)  # Move each batch
    batch_y = batch_y.to(device)
    output = model(batch_x)
```

---

## 9. Quick Reference

### Tensor Creation Cheat Sheet

| Function | Description | Example |
|----------|-------------|---------|
| `torch.tensor(data)` | From Python list/array | `torch.tensor([1,2,3])` |
| `torch.zeros(shape)` | All zeros | `torch.zeros(3, 4)` |
| `torch.ones(shape)` | All ones | `torch.ones(2, 3)` |
| `torch.rand(shape)` | Uniform [0,1) | `torch.rand(5, 5)` |
| `torch.randn(shape)` | Normal(0,1) | `torch.randn(5, 5)` |
| `torch.arange(start, end, step)` | Range | `torch.arange(0, 10, 2)` |
| `torch.linspace(start, end, steps)` | Linear space | `torch.linspace(0, 1, 100)` |
| `torch.eye(n)` | Identity | `torch.eye(3)` |
| `torch.full(shape, val)` | Fill value | `torch.full((2,3), 7)` |
| `torch.*_like(tensor)` | Same shape | `torch.zeros_like(x)` |

### Shape Operations Cheat Sheet

| Operation | Description | Shape Change |
|-----------|-------------|--------------|
| `.reshape(a, b)` | Reshape | → (a, b) |
| `.view(a, b)` | Reshape (no copy) | → (a, b) |
| `.squeeze()` | Remove dims of 1 | (1,3,1,4) → (3,4) |
| `.unsqueeze(dim)` | Add dim of 1 | (3,4) → (1,3,4) |
| `.transpose(d1, d2)` | Swap 2 dims | Swap d1↔d2 |
| `.permute(dims)` | Reorder all dims | Custom order |
| `.flatten(start)` | Flatten from dim | (2,3,4) → (2,12) |
| `torch.cat([t], dim)` | Concatenate | Extend dim |
| `torch.stack([t], dim)` | Stack (new dim) | Add dim |

### Key Autograd Rules

1. Set `requires_grad=True` on learnable parameters
2. Forward pass builds the graph
3. `.backward()` computes all gradients
4. Access via `.grad` attribute
5. **Always** zero gradients between iterations
6. Use `torch.no_grad()` for inference
7. Use `.detach()` to stop gradient flow at a point
8. `.item()` extracts Python number from scalar tensor

---

*Next: [Chapter 02 — Building Neural Networks](02-Building-Neural-Networks.md)*
