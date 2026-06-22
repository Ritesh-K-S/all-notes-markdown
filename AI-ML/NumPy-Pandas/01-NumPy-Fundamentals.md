# Chapter 01: NumPy Fundamentals

## Table of Contents
- [What is NumPy?](#what-is-numpy)
- [Why NumPy Matters](#why-numpy-matters)
- [Installation and Import](#installation-and-import)
- [NumPy Arrays — The Core Data Structure](#numpy-arrays--the-core-data-structure)
- [Array Creation Methods](#array-creation-methods)
- [Data Types (dtypes)](#data-types-dtypes)
- [Indexing and Slicing](#indexing-and-slicing)
- [Reshaping Arrays](#reshaping-arrays)
- [Copying vs Viewing](#copying-vs-viewing)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is NumPy?

**Simple explanation:** NumPy (Numerical Python) is a Python library that gives you a supercharged list called an **array**. While Python lists are like general-purpose boxes that can hold anything, NumPy arrays are like specially designed containers where every item is the same type — making math operations incredibly fast.

**Analogy:** Think of a Python list as a mixed bag of marbles (different sizes, colors, materials). NumPy arrays are like a perfectly organized egg carton — every slot is the same size, evenly spaced, and you know exactly where everything is. This organization makes it 10-100x faster to do bulk operations.

---

## Why NumPy Matters

### Real-World Relevance
- **Every ML/AI library** (TensorFlow, PyTorch, Scikit-learn) is built on NumPy
- **Image processing** — every image is a NumPy array (height × width × channels)
- **Financial modeling** — time series data, portfolio calculations
- **Scientific computing** — physics simulations, signal processing
- **Data Science** — Pandas DataFrames use NumPy arrays internally

### Performance Comparison

```python
import numpy as np
import time

# Python list operation
python_list = list(range(1_000_000))
start = time.time()
result = [x * 2 for x in python_list]
print(f"Python list: {time.time() - start:.4f}s")

# NumPy array operation
numpy_array = np.arange(1_000_000)
start = time.time()
result = numpy_array * 2
print(f"NumPy array: {time.time() - start:.4f}s")

# Output:
# Python list: 0.0850s
# NumPy array: 0.0012s  ← ~70x faster!
```

### Why is NumPy faster?

| Factor | Python List | NumPy Array |
|--------|------------|-------------|
| Memory | Scattered (pointers to objects) | Contiguous block |
| Type checking | Every operation checks type | All same type, no checking |
| Implementation | Pure Python (interpreted) | C/Fortran (compiled) |
| Vectorization | Loop item by item | SIMD CPU instructions |

```
Python List Memory Layout:
┌───┐    ┌─────────┐
│ * │───→│ int: 1  │
├───┤    └─────────┘
│ * │───→│ int: 2  │    (scattered in memory)
├───┤    └─────────┘
│ * │───→│ int: 3  │
└───┘    └─────────┘

NumPy Array Memory Layout:
┌───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │  (contiguous block)
└───┴───┴───┴───┴───┘
```

---

## Installation and Import

```python
# Installation
# pip install numpy

# Standard import convention — ALWAYS use 'np'
import numpy as np

# Check version
print(np.__version__)  # e.g., '1.26.4'
```

> **Important:** Never do `from numpy import *`. Always use `np.` prefix for clarity and to avoid namespace collisions.

---

## NumPy Arrays — The Core Data Structure

### What is an ndarray?

The `ndarray` (N-dimensional array) is NumPy's primary data structure. It's a grid of values, all of the **same type**, indexed by a tuple of non-negative integers.

### Key Terminology

| Term | Meaning | Example |
|------|---------|---------|
| **ndim** | Number of dimensions (axes) | 2D array has ndim=2 |
| **shape** | Size along each dimension | (3, 4) = 3 rows, 4 cols |
| **size** | Total number of elements | 3×4 = 12 |
| **dtype** | Data type of elements | float64, int32 |
| **itemsize** | Bytes per element | float64 = 8 bytes |
| **strides** | Bytes to step in each dimension | (32, 8) for a 4-col float64 |

```python
import numpy as np

# Create a 2D array
arr = np.array([[1, 2, 3, 4],
                [5, 6, 7, 8],
                [9, 10, 11, 12]])

print(f"ndim:     {arr.ndim}")      # 2
print(f"shape:    {arr.shape}")     # (3, 4)
print(f"size:     {arr.size}")      # 12
print(f"dtype:    {arr.dtype}")     # int64 (or int32 on Windows)
print(f"itemsize: {arr.itemsize}")  # 8 bytes
print(f"strides:  {arr.strides}")   # (32, 8) — 4 items × 8 bytes to next row
```

### Visualizing Dimensions

```
0D (Scalar):     42

1D (Vector):     [1, 2, 3, 4, 5]
                  ─── axis 0 ───→

2D (Matrix):     [[1, 2, 3],
                   [4, 5, 6],
                   [7, 8, 9]]
                  │  axis 1 →
                  ↓ axis 0

3D (Tensor):     [[[1, 2], [3, 4]],
                   [[5, 6], [7, 8]]]
                  │ │  axis 2 →
                  │ ↓ axis 1
                  ↓ axis 0
```

---

## Array Creation Methods

### From Python Objects

```python
import numpy as np

# From a list
arr1 = np.array([1, 2, 3, 4, 5])
print(arr1)  # [1 2 3 4 5]

# From nested lists (2D)
arr2 = np.array([[1, 2, 3], [4, 5, 6]])
print(arr2)
# [[1 2 3]
#  [4 5 6]]

# Specifying dtype explicitly
arr3 = np.array([1, 2, 3], dtype=np.float32)
print(arr3)  # [1. 2. 3.]
print(arr3.dtype)  # float32

# From tuple
arr4 = np.array((10, 20, 30))
print(arr4)  # [10 20 30]
```

### Using Built-in Functions

```python
import numpy as np

# ─── Zeros and Ones ───
zeros = np.zeros((3, 4))        # 3×4 matrix of 0.0 (float64 by default)
ones = np.ones((2, 3))          # 2×3 matrix of 1.0
full = np.full((2, 2), 7)      # 2×2 matrix filled with 7

print(zeros)
# [[0. 0. 0. 0.]
#  [0. 0. 0. 0.]
#  [0. 0. 0. 0.]]

# ─── Ranges ───
arange = np.arange(0, 10, 2)       # [0, 2, 4, 6, 8] — like range() but returns array
linspace = np.linspace(0, 1, 5)    # [0.  0.25 0.5 0.75 1.] — 5 evenly spaced points

# ─── Identity and Diagonal ───
eye = np.eye(3)                     # 3×3 identity matrix
diag = np.diag([1, 2, 3])          # Diagonal matrix from list

print(eye)
# [[1. 0. 0.]
#  [0. 1. 0.]
#  [0. 0. 1.]]

# ─── Random Arrays ───
rand_uniform = np.random.rand(3, 3)       # Uniform [0, 1)
rand_normal = np.random.randn(3, 3)       # Standard normal (mean=0, std=1)
rand_int = np.random.randint(0, 10, (3, 3))  # Random integers [0, 10)

# ─── Like Functions (match shape of existing array) ───
template = np.array([[1, 2], [3, 4]])
zeros_like = np.zeros_like(template)   # Same shape, filled with zeros
ones_like = np.ones_like(template)     # Same shape, filled with ones
```

### arange vs linspace

```python
import numpy as np

# arange: you specify the STEP size
# Pitfall: may not include the endpoint
a = np.arange(0, 1, 0.3)
print(a)  # [0.  0.3 0.6 0.9] — endpoint 1.0 NOT included

# linspace: you specify the NUMBER of points
# Always includes both endpoints (by default)
b = np.linspace(0, 1, 5)
print(b)  # [0.   0.25 0.5  0.75 1.  ] — endpoint 1.0 IS included

# Pro Tip: Use linspace for floating point ranges to avoid
# accumulation errors that arange can have
```

> **Pro Tip:** Always prefer `np.linspace()` over `np.arange()` for floating-point ranges. `arange` with float steps can produce unexpected results due to floating-point precision.

---

## Data Types (dtypes)

### Why dtypes Matter

NumPy's power comes from **homogeneous** arrays — every element is the same type stored in the same number of bytes. This enables:
- Predictable memory usage
- Fast vectorized operations
- Direct compatibility with C/Fortran libraries

### Common dtypes

| dtype | Description | Size | Range |
|-------|-------------|------|-------|
| `np.int8` | Signed integer | 1 byte | -128 to 127 |
| `np.int16` | Signed integer | 2 bytes | -32,768 to 32,767 |
| `np.int32` | Signed integer | 4 bytes | ±2.1 billion |
| `np.int64` | Signed integer | 8 bytes | ±9.2 × 10¹⁸ |
| `np.uint8` | Unsigned integer | 1 byte | 0 to 255 |
| `np.float16` | Half precision | 2 bytes | ±6.5 × 10⁴ |
| `np.float32` | Single precision | 4 bytes | ±3.4 × 10³⁸ |
| `np.float64` | Double precision | 8 bytes | ±1.8 × 10³⁰⁸ |
| `np.bool_` | Boolean | 1 byte | True/False |
| `np.complex64` | Complex | 8 bytes | Two float32 |
| `np.str_` | Fixed-length string | varies | — |

### Working with dtypes

```python
import numpy as np

# Automatic dtype inference
a = np.array([1, 2, 3])           # int64 (int32 on Windows)
b = np.array([1.0, 2.0, 3.0])    # float64
c = np.array([True, False, True]) # bool

# Explicit dtype
d = np.array([1, 2, 3], dtype=np.float32)
print(d.dtype)  # float32

# Type casting (creates a NEW array)
e = a.astype(np.float64)
print(e)        # [1. 2. 3.]
print(e.dtype)  # float64

# Check memory usage
arr = np.zeros((1000, 1000), dtype=np.float64)
print(f"float64: {arr.nbytes / 1024 / 1024:.1f} MB")  # 7.6 MB

arr32 = np.zeros((1000, 1000), dtype=np.float32)
print(f"float32: {arr32.nbytes / 1024 / 1024:.1f} MB")  # 3.8 MB
```

### dtype Upcasting Rules

```python
import numpy as np

# NumPy automatically upcasts to prevent data loss
a = np.array([1, 2, 3], dtype=np.int32)
b = np.array([1.5, 2.5, 3.5], dtype=np.float64)

result = a + b
print(result.dtype)  # float64 — int32 was upcast to float64

# Upcasting hierarchy:
# bool → int8 → int16 → int32 → int64 → float16 → float32 → float64
```

> **Pro Tip (Images):** Images loaded with OpenCV/PIL are `uint8` (0-255). When doing math, cast to `float32` first to avoid overflow: `img = img.astype(np.float32) / 255.0`

---

## Indexing and Slicing

### 1D Array Indexing

```python
import numpy as np

arr = np.array([10, 20, 30, 40, 50, 60, 70, 80, 90])

# Basic indexing (0-based)
print(arr[0])    # 10 (first element)
print(arr[-1])   # 90 (last element)
print(arr[-2])   # 80 (second to last)

# Slicing: arr[start:stop:step]
print(arr[2:5])    # [30 40 50] — index 2, 3, 4 (stop is exclusive)
print(arr[:3])     # [10 20 30] — first 3 elements
print(arr[5:])     # [60 70 80 90] — from index 5 onwards
print(arr[::2])    # [10 30 50 70 90] — every other element
print(arr[::-1])   # [90 80 70 60 50 40 30 20 10] — reversed
```

### 2D Array Indexing

```python
import numpy as np

arr = np.array([[1,  2,  3,  4],
                [5,  6,  7,  8],
                [9,  10, 11, 12]])

# Single element: arr[row, col]
print(arr[0, 0])   # 1  (top-left)
print(arr[2, 3])   # 12 (bottom-right)
print(arr[-1, -1]) # 12 (same as above)

# Entire row or column
print(arr[1])       # [5 6 7 8] — second row
print(arr[:, 2])    # [3 7 11] — third column

# Sub-matrix (slicing both axes)
print(arr[0:2, 1:3])
# [[2 3]
#  [6 7]]

# Multiple specific rows/columns
print(arr[[0, 2]])      # Rows 0 and 2
print(arr[:, [0, 3]])   # Columns 0 and 3
```

```
Visual: arr[0:2, 1:3]

    col0  col1  col2  col3
    ┌────┬─────┬─────┬────┐
row0│ 1  │[ 2  │  3 ]│  4 │  ← row 0, cols 1-2
    ├────┼─────┼─────┼────┤
row1│ 5  │[ 6  │  7 ]│  8 │  ← row 1, cols 1-2
    ├────┼─────┼─────┼────┤
row2│ 9  │ 10  │ 11  │ 12 │
    └────┴─────┴─────┴────┘
```

### Boolean (Mask) Indexing

```python
import numpy as np

arr = np.array([15, 22, 8, 45, 30, 5, 67, 12])

# Create a boolean mask
mask = arr > 20
print(mask)  # [False  True False  True  True False  True False]

# Use mask to filter
print(arr[mask])       # [22 45 30 67]
print(arr[arr > 20])   # Same thing, inline

# Combine conditions (use & for AND, | for OR, ~ for NOT)
# MUST use parentheses around each condition!
result = arr[(arr > 10) & (arr < 40)]
print(result)  # [15 22 30 12]

# Modify elements matching condition
arr[arr < 10] = 0
print(arr)  # [15 22  0 45 30  0 67 12]
```

> **Warning:** Use `&`, `|`, `~` for element-wise boolean operations on arrays, NOT `and`, `or`, `not` (those are Python's scalar operators and will raise errors on arrays).

### Fancy (Integer Array) Indexing

```python
import numpy as np

arr = np.array([10, 20, 30, 40, 50])

# Index with an array of indices
indices = np.array([0, 3, 4])
print(arr[indices])  # [10 40 50]

# Useful for reordering
order = np.array([4, 3, 2, 1, 0])
print(arr[order])  # [50 40 30 20 10]

# 2D fancy indexing
matrix = np.array([[1, 2], [3, 4], [5, 6]])
rows = np.array([0, 1, 2])
cols = np.array([1, 0, 1])
print(matrix[rows, cols])  # [2 3 6] — elements at (0,1), (1,0), (2,1)
```

### Key Difference: Slice vs Fancy Index

```python
import numpy as np

arr = np.array([10, 20, 30, 40, 50])

# Slicing returns a VIEW (shares memory)
slice_view = arr[1:4]
slice_view[0] = 999
print(arr)  # [10 999 30 40 50] ← original changed!

# Fancy indexing returns a COPY (independent)
arr = np.array([10, 20, 30, 40, 50])
fancy_copy = arr[[1, 2, 3]]
fancy_copy[0] = 999
print(arr)  # [10 20 30 40 50] ← original unchanged!
```

---

## Reshaping Arrays

### Core Reshaping Operations

```python
import numpy as np

arr = np.arange(12)  # [0, 1, 2, ..., 11]

# reshape: change shape without changing data
reshaped = arr.reshape(3, 4)
print(reshaped)
# [[ 0  1  2  3]
#  [ 4  5  6  7]
#  [ 8  9 10 11]]

# Use -1 to auto-calculate one dimension
auto = arr.reshape(4, -1)   # 12 / 4 = 3 columns
print(auto.shape)  # (4, 3)

auto2 = arr.reshape(-1, 6)  # 12 / 6 = 2 rows
print(auto2.shape)  # (2, 6)

# flatten: always returns a COPY (1D)
flat = reshaped.flatten()
print(flat)  # [0 1 2 3 4 5 6 7 8 9 10 11]

# ravel: returns a VIEW if possible (1D)
raveled = reshaped.ravel()
print(raveled)  # Same output, but shares memory when possible
```

### Transpose

```python
import numpy as np

arr = np.array([[1, 2, 3],
                [4, 5, 6]])
print(f"Original shape: {arr.shape}")  # (2, 3)

# Transpose swaps rows and columns
transposed = arr.T
print(f"Transposed shape: {transposed.shape}")  # (3, 2)
print(transposed)
# [[1 4]
#  [2 5]
#  [3 6]]

# For higher dimensions, specify axis order
arr3d = np.arange(24).reshape(2, 3, 4)
print(arr3d.shape)  # (2, 3, 4)

transposed3d = arr3d.transpose(1, 0, 2)  # Swap first two axes
print(transposed3d.shape)  # (3, 2, 4)
```

### Adding/Removing Dimensions

```python
import numpy as np

arr = np.array([1, 2, 3])  # shape: (3,)

# Add dimension with np.newaxis or np.expand_dims
row_vector = arr[np.newaxis, :]     # shape: (1, 3)
col_vector = arr[:, np.newaxis]     # shape: (3, 1)

# Equivalent using expand_dims
row_vector2 = np.expand_dims(arr, axis=0)  # shape: (1, 3)
col_vector2 = np.expand_dims(arr, axis=1)  # shape: (3, 1)

# Remove dimensions of size 1
a = np.array([[[1, 2, 3]]])  # shape: (1, 1, 3)
squeezed = a.squeeze()        # shape: (3,)

# This is CRITICAL for ML — models often expect (batch, features)
# Single sample: shape (5,) → must become (1, 5)
sample = np.array([0.1, 0.2, 0.3, 0.4, 0.5])
batch = sample.reshape(1, -1)   # or sample[np.newaxis, :]
print(batch.shape)  # (1, 5) — ready for model.predict()
```

### Stacking and Splitting

```python
import numpy as np

a = np.array([1, 2, 3])
b = np.array([4, 5, 6])

# Stacking
print(np.hstack([a, b]))       # [1 2 3 4 5 6] — horizontal
print(np.vstack([a, b]))       # [[1 2 3], [4 5 6]] — vertical
print(np.column_stack([a, b])) # [[1 4], [2 5], [3 6]] — as columns

# General concatenate
c = np.array([[1, 2], [3, 4]])
d = np.array([[5, 6], [7, 8]])
print(np.concatenate([c, d], axis=0))  # Stack vertically
# [[1 2]
#  [3 4]
#  [5 6]
#  [7 8]]
print(np.concatenate([c, d], axis=1))  # Stack horizontally
# [[1 2 5 6]
#  [3 4 7 8]]

# Splitting
arr = np.arange(12).reshape(3, 4)
parts = np.hsplit(arr, 2)  # Split into 2 halves horizontally
print(parts[0])
# [[0 1]
#  [4 5]
#  [8 9]]
```

---

## Copying vs Viewing

This is one of the most **critical** concepts in NumPy. Getting it wrong leads to subtle bugs.

### The Three Levels

```python
import numpy as np

original = np.array([1, 2, 3, 4, 5])

# 1. NO COPY — just another name for same object
alias = original
alias[0] = 99
print(original[0])  # 99 — same object!
print(alias is original)  # True

# 2. VIEW (shallow copy) — different object, SHARED data
original = np.array([1, 2, 3, 4, 5])
view = original[1:4]  # Slicing creates a view
view[0] = 99
print(original)  # [1 99 3 4 5] — original changed!
print(view.base is original)  # True — view's base IS original

# 3. COPY (deep copy) — completely independent
original = np.array([1, 2, 3, 4, 5])
copy = original.copy()
copy[0] = 99
print(original)  # [1 2 3 4 5] — original unchanged!
print(copy.base is None)  # True — no base, independent
```

### When Does NumPy Create a View vs Copy?

| Operation | Result |
|-----------|--------|
| `arr[2:5]` (basic slicing) | **View** |
| `arr[[0, 2, 4]]` (fancy indexing) | **Copy** |
| `arr[arr > 3]` (boolean indexing) | **Copy** |
| `arr.reshape(2, 3)` | **View** (usually) |
| `arr.T` (transpose) | **View** |
| `arr.flatten()` | **Copy** |
| `arr.ravel()` | **View** (if possible) |
| `arr.copy()` | **Copy** |

```python
import numpy as np

# How to check if something is a view
arr = np.array([1, 2, 3, 4, 5])
s = arr[1:4]
print(s.base is not None)  # True → it's a view
print(np.shares_memory(arr, s))  # True → shares memory
```

> **Pro Tip:** When in doubt, use `.copy()` explicitly. The small memory cost is worth avoiding hours of debugging subtle mutation bugs.

---

## Common Mistakes

### 1. Forgetting that slices are views

```python
import numpy as np

# BUG: modifying a slice modifies the original
data = np.array([1, 2, 3, 4, 5])
subset = data[2:]
subset[0] = 99
print(data)  # [1 2 99 4 5] — oops!

# FIX: explicitly copy
subset = data[2:].copy()
subset[0] = 99
print(data)  # [1 2 3 4 5] — safe
```

### 2. Using `and`/`or` instead of `&`/`|`

```python
import numpy as np

arr = np.array([1, 2, 3, 4, 5])

# WRONG — raises ValueError
# result = arr[(arr > 2) and (arr < 5)]

# CORRECT — use bitwise operators with parentheses
result = arr[(arr > 2) & (arr < 5)]
print(result)  # [3 4]
```

### 3. Shape mismatch in operations

```python
import numpy as np

# WRONG — confusing 1D array with 2D column
a = np.array([1, 2, 3])      # shape: (3,)
b = np.array([[1], [2], [3]]) # shape: (3, 1)
# These are NOT the same! a + b broadcasts to (3, 3)!

print((a + b).shape)  # (3, 3) — probably not what you wanted!

# FIX — be explicit about dimensions
a_col = a.reshape(-1, 1)  # shape: (3, 1)
print((a_col + b).shape)  # (3, 1) — correct element-wise
```

### 4. Integer division with integer dtypes

```python
import numpy as np

# This loses precision!
a = np.array([1, 2, 3, 4, 5])
result = a / 2
print(result)        # [0.5 1.  1.5 2.  2.5] — actually fine in NumPy
print(result.dtype)  # float64 — NumPy auto-upcasts with /

# But floor division keeps int
result2 = a // 2
print(result2)       # [0 1 1 2 2]
print(result2.dtype) # int64
```

### 5. Not understanding that np.array() copies by default

```python
import numpy as np

original_list = [1, 2, 3]
arr = np.array(original_list)  # This COPIES the data
arr[0] = 99
print(original_list)  # [1, 2, 3] — list is independent

# If you DON'T want a copy (rare):
arr2 = np.asarray(original_list)  # May not copy if already an array
```

---

## Interview Questions

### Q1: What's the difference between a Python list and a NumPy array?
**Answer:**
- NumPy arrays are **homogeneous** (same dtype), lists are heterogeneous
- NumPy uses **contiguous memory**, lists use scattered pointers
- NumPy supports **vectorized operations**, lists require explicit loops
- NumPy is **10-100x faster** for numerical operations
- NumPy supports **broadcasting**, lists don't

### Q2: Explain the difference between `.copy()` and a view in NumPy.
**Answer:** A view shares memory with the original array — modifications to the view affect the original. A copy is independent. Slicing creates views; fancy/boolean indexing creates copies. Use `np.shares_memory(a, b)` to check.

### Q3: What is the difference between `reshape()` and `resize()`?
**Answer:**
- `reshape()` returns a view (if possible) with a new shape; total size must match
- `resize()` modifies the array **in-place** and can change total size (pads with zeros or truncates)

### Q4: How would you efficiently create a large array of zeros?
**Answer:** `np.zeros(shape, dtype=np.float32)` — specify float32 instead of float64 to halve memory. For even larger arrays, consider memory-mapped files with `np.memmap`.

### Q5: What happens when you add an int array and a float array?
**Answer:** NumPy performs **type promotion** (upcasting). The int array is automatically converted to float for the operation. The result will be float dtype. The original arrays remain unchanged.

### Q6: How do you add a new axis to an array?
**Answer:** Three ways:
- `arr[np.newaxis, :]` or `arr[None, :]` — adds axis at position 0
- `arr[:, np.newaxis]` — adds axis at position 1
- `np.expand_dims(arr, axis=0)` — most explicit

### Q7: What's the difference between `flatten()` and `ravel()`?
**Answer:** Both return 1D arrays. `flatten()` always returns a copy. `ravel()` returns a view when possible (faster, less memory), but may return a copy if the array is non-contiguous. Use `ravel()` for performance, `flatten()` for safety.

---

## Quick Reference

### Array Creation Cheat Sheet

| Function | Description | Example |
|----------|-------------|---------|
| `np.array(list)` | From Python list | `np.array([1,2,3])` |
| `np.zeros(shape)` | All zeros | `np.zeros((3,4))` |
| `np.ones(shape)` | All ones | `np.ones((2,3))` |
| `np.full(shape, val)` | Fill with value | `np.full((2,2), 7)` |
| `np.eye(n)` | Identity matrix | `np.eye(3)` |
| `np.arange(start, stop, step)` | Range array | `np.arange(0,10,2)` |
| `np.linspace(start, stop, n)` | Evenly spaced | `np.linspace(0,1,5)` |
| `np.random.rand(shape)` | Uniform [0,1) | `np.random.rand(3,3)` |
| `np.random.randn(shape)` | Normal(0,1) | `np.random.randn(3,3)` |
| `np.random.randint(lo,hi,shape)` | Random ints | `np.random.randint(0,10,(3,3))` |
| `np.empty(shape)` | Uninitialized | `np.empty((2,3))` |

### Indexing Cheat Sheet

| Syntax | Meaning |
|--------|---------|
| `arr[2]` | Element at index 2 |
| `arr[-1]` | Last element |
| `arr[1:4]` | Elements 1, 2, 3 (view) |
| `arr[::2]` | Every other element |
| `arr[::-1]` | Reversed |
| `arr[arr > 5]` | Boolean filter (copy) |
| `arr[[0,2,4]]` | Fancy index (copy) |
| `arr[row, col]` | 2D element access |
| `arr[:, 0]` | First column |
| `arr[0, :]` | First row |

### Reshaping Cheat Sheet

| Function | Returns | Notes |
|----------|---------|-------|
| `.reshape(shape)` | View* | Total size must match |
| `.flatten()` | Copy | Always 1D |
| `.ravel()` | View* | 1D, may copy |
| `.T` | View | Transpose |
| `.squeeze()` | View | Remove size-1 dims |
| `np.expand_dims()` | View | Add dimension |
| `np.newaxis` | View | Add dimension via indexing |

*View when possible, copy when memory layout doesn't allow it.

---

## Key Takeaways

1. **NumPy arrays are fast** because of contiguous memory, homogeneous dtypes, and C implementation
2. **Views vs Copies** — understand when slicing shares memory (views) vs creates independent data (copies)
3. **dtype matters** — choose the smallest type that fits your data for memory efficiency
4. **Use vectorized operations** — never loop over array elements manually
5. **-1 in reshape** — lets NumPy auto-calculate one dimension
6. **Boolean indexing** is your best friend for filtering data

---

*Next Chapter: [02-NumPy-Operations-and-Broadcasting](02-NumPy-Operations-and-Broadcasting.md) — Vectorization, Broadcasting, Universal Functions*
