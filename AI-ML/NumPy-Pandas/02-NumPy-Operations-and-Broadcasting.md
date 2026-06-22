# Chapter 02: NumPy Operations and Broadcasting

## Table of Contents
- [Vectorized Operations](#vectorized-operations)
- [Universal Functions (ufuncs)](#universal-functions-ufuncs)
- [Broadcasting](#broadcasting)
- [Aggregation Functions](#aggregation-functions)
- [Comparison and Logic Operations](#comparison-and-logic-operations)
- [Sorting and Searching](#sorting-and-searching)
- [Linear Algebra Basics](#linear-algebra-basics)
- [Set Operations](#set-operations)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Vectorized Operations

### What It Is

Vectorization means performing operations on entire arrays at once instead of looping through elements one by one. It's like telling 1000 workers to each do their task simultaneously, instead of having one worker do 1000 tasks sequentially.

### Why It Matters

- **Speed**: 10-100x faster than Python loops
- **Readability**: `arr * 2` is cleaner than a for loop
- **Correctness**: Fewer chances for off-by-one errors
- **GPU-ready**: Vectorized code translates easily to GPU computing

### How It Works

```python
import numpy as np

a = np.array([1, 2, 3, 4, 5])
b = np.array([10, 20, 30, 40, 50])

# ─── Arithmetic Operations (element-wise) ───
print(a + b)     # [11 22 33 44 55]
print(a - b)     # [-9 -18 -27 -36 -45]
print(a * b)     # [10 40 90 160 250]
print(a / b)     # [0.1 0.1 0.1 0.1 0.1]
print(a ** 2)    # [1 4 9 16 25]
print(a % 2)     # [1 0 1 0 1]  (modulo)
print(a // 2)    # [0 1 1 2 2]  (floor division)

# ─── Scalar Operations (broadcast scalar to array) ───
print(a + 10)    # [11 12 13 14 15]
print(a * 3)     # [3 6 9 12 15]
print(1 / a)     # [1.   0.5  0.33 0.25 0.2]
```

### Performance: Vectorized vs Loop

```python
import numpy as np
import time

size = 1_000_000
a = np.random.rand(size)
b = np.random.rand(size)

# METHOD 1: Python loop (SLOW)
start = time.time()
result = np.empty(size)
for i in range(size):
    result[i] = a[i] + b[i]
loop_time = time.time() - start

# METHOD 2: Vectorized (FAST)
start = time.time()
result = a + b
vec_time = time.time() - start

print(f"Loop: {loop_time:.4f}s")       # ~0.35s
print(f"Vectorized: {vec_time:.4f}s")  # ~0.001s
print(f"Speedup: {loop_time/vec_time:.0f}x")  # ~300x
```

> **Rule of Thumb:** If you're writing a `for` loop over array elements in NumPy, you're probably doing it wrong. There's almost always a vectorized alternative.

---

## Universal Functions (ufuncs)

### What They Are

Universal functions (ufuncs) are NumPy functions that operate element-wise on arrays. They're the building blocks of vectorization — optimized C implementations that handle broadcasting, type casting, and output allocation automatically.

### Math ufuncs

```python
import numpy as np

arr = np.array([1, 4, 9, 16, 25], dtype=np.float64)

# ─── Basic Math ───
print(np.sqrt(arr))      # [1. 2. 3. 4. 5.]
print(np.square(arr))    # [1. 16. 81. 256. 625.]
print(np.abs(np.array([-1, -2, 3])))  # [1 2 3]
print(np.sign(np.array([-5, 0, 3])))  # [-1  0  1]

# ─── Exponential and Logarithmic ───
x = np.array([1, 2, 3])
print(np.exp(x))         # [2.718  7.389  20.086]
print(np.log(x))         # [0.    0.693  1.099] — natural log (ln)
print(np.log2(x))        # [0.    1.     1.585]
print(np.log10(x))       # [0.    0.301  0.477]
print(np.exp2(x))        # [2.  4.  8.] — 2^x

# ─── Trigonometric ───
angles = np.array([0, np.pi/6, np.pi/4, np.pi/3, np.pi/2])
print(np.sin(angles))    # [0.    0.5   0.707 0.866 1.   ]
print(np.cos(angles))    # [1.    0.866 0.707 0.5   0.   ]
print(np.tan(angles[:4]))# [0.    0.577 1.    1.732]

# ─── Rounding ───
decimals = np.array([1.2, 2.5, 3.7, 4.4, 5.9])
print(np.floor(decimals))   # [1. 2. 3. 4. 5.]
print(np.ceil(decimals))    # [2. 3. 4. 5. 6.]
print(np.round(decimals))   # [1. 2. 4. 4. 6.] — banker's rounding!
print(np.trunc(decimals))   # [1. 2. 3. 4. 5.] — toward zero
```

### Special ufuncs for ML

```python
import numpy as np

# Sigmoid function (logistic)
def sigmoid(x):
    return 1 / (1 + np.exp(-x))

x = np.array([-3, -1, 0, 1, 3])
print(sigmoid(x))  # [0.047 0.269 0.5 0.731 0.953]

# Softmax (for classification)
def softmax(x):
    # Subtract max for numerical stability
    e_x = np.exp(x - np.max(x))
    return e_x / e_x.sum()

logits = np.array([2.0, 1.0, 0.1])
print(softmax(logits))  # [0.659 0.242 0.099] — sums to 1.0

# ReLU (Rectified Linear Unit)
def relu(x):
    return np.maximum(0, x)

print(relu(np.array([-2, -1, 0, 1, 2])))  # [0 0 0 1 2]

# Clip values (gradient clipping, pixel values)
arr = np.array([-5, 3, 8, 15, 22])
print(np.clip(arr, 0, 10))  # [0 3 8 10 10]
```

### Binary ufuncs

```python
import numpy as np

a = np.array([1, 5, 3, 8, 2])
b = np.array([4, 2, 6, 1, 9])

# Element-wise min/max
print(np.maximum(a, b))   # [4 5 6 8 9]
print(np.minimum(a, b))   # [1 2 3 1 2]

# Power
print(np.power(a, 2))     # [1 25 9 64 4]

# Remainder
print(np.mod(a, 3))       # [1 2 0 2 2]
print(np.remainder(a, 3)) # Same as mod

# Division with separate quotient and remainder
print(np.divmod(a, 3))    # (array([0,1,1,2,0]), array([1,2,0,2,2]))
```

### The `out` Parameter (Zero-Copy Operations)

```python
import numpy as np

# Normal: creates a new array (allocates memory)
a = np.array([1.0, 2.0, 3.0])
result = np.sqrt(a)  # New array allocated

# With out: writes result into existing array (no allocation)
output = np.empty(3)
np.sqrt(a, out=output)  # Result written into 'output'
print(output)  # [1.  1.414  1.732]

# Pro Tip: In tight loops, pre-allocate and use out=
buffer = np.empty(1_000_000)
data = np.random.rand(1_000_000)
np.multiply(data, 2, out=buffer)  # Faster than buffer = data * 2
```

---

## Broadcasting

### What It Is

Broadcasting is NumPy's way of performing operations on arrays with **different shapes**. It's like a smart auto-expansion system — smaller arrays are "stretched" to match larger ones without actually copying data.

**Analogy:** Imagine you have a spreadsheet with 1000 rows. You want to multiply every row by the same set of weights `[0.5, 0.3, 0.2]`. You don't copy those weights 1000 times — you just apply them row by row. That's broadcasting.

### Why It Matters

- Eliminates the need for explicit loops
- Saves memory (no actual copying happens)
- Makes code concise and readable
- Essential for ML (batch operations, normalization)

### Broadcasting Rules

NumPy compares shapes element-wise, **starting from the trailing (rightmost) dimension**:

1. If dimensions are equal → compatible
2. If one of them is 1 → compatible (that dimension gets "stretched")
3. If neither condition is met → **error**

```
Rule applied right-to-left:

Shape A:     (4, 3)
Shape B:        (3,)  ← padded to (1, 3) then broadcast to (4, 3)
Result:      (4, 3)  ✓

Shape A:     (4, 3)
Shape B:     (4, 1)  ← broadcast to (4, 3)
Result:      (4, 3)  ✓

Shape A:     (4, 3)
Shape B:     (4,)    ← NOT compatible! 3 ≠ 4
Result:      ERROR   ✗
```

### Visual Examples

```
Example 1: Array + Scalar
┌───┬───┬───┐       ┌───┬───┬───┐       ┌───┬───┬───┐
│ 1 │ 2 │ 3 │   +   │ 10│ 10│ 10│   =   │11 │12 │13 │
│ 4 │ 5 │ 6 │       │ 10│ 10│ 10│       │14 │15 │16 │
└───┴───┴───┘       └───┴───┴───┘       └───┴───┴───┘
  (2,3)          scalar 10 → (2,3)          (2,3)


Example 2: Matrix + Row Vector
┌───┬───┬───┐       ┌───┬───┬───┐       ┌───┬───┬───┐
│ 1 │ 2 │ 3 │       │10 │20 │30 │       │11 │22 │33 │
│ 4 │ 5 │ 6 │   +   │10 │20 │30 │   =   │14 │25 │36 │
│ 7 │ 8 │ 9 │       │10 │20 │30 │       │17 │28 │39 │
└───┴───┴───┘       └───┴───┴───┘       └───┴───┴───┘
  (3,3)            (3,) → (3,3)              (3,3)


Example 3: Column Vector + Row Vector
┌───┐             ┌───┬───┬───┐       ┌───┬───┬───┐
│ 1 │             │10 │20 │30 │       │11 │21 │31 │
│ 2 │      +      │10 │20 │30 │   =   │12 │22 │32 │
│ 3 │             │10 │20 │30 │       │13 │23 │33 │
└───┘             └───┴───┴───┘       └───┴───┴───┘
(3,1)→(3,3)         (3,)→(3,3)            (3,3)
```

### Code Examples

```python
import numpy as np

# ─── Example 1: Scalar broadcast ───
arr = np.array([[1, 2, 3],
                [4, 5, 6]])
print(arr + 10)
# [[11 12 13]
#  [14 15 16]]

# ─── Example 2: Row vector broadcast ───
matrix = np.array([[1, 2, 3],
                   [4, 5, 6],
                   [7, 8, 9]])
row = np.array([10, 20, 30])   # shape (3,)
print(matrix + row)
# [[11 22 33]
#  [14 25 36]
#  [17 28 39]]

# ─── Example 3: Column vector broadcast ───
col = np.array([[100],
                [200],
                [300]])        # shape (3, 1)
print(matrix + col)
# [[101 102 103]
#  [204 205 206]
#  [307 308 309]]

# ─── Example 4: Outer product via broadcasting ───
a = np.array([1, 2, 3])        # shape (3,)
b = np.array([10, 20, 30, 40]) # shape (4,)
outer = a[:, np.newaxis] * b[np.newaxis, :]  # (3,1) * (1,4) = (3,4)
print(outer)
# [[10  20  30  40]
#  [20  40  60  80]
#  [30  60  90 120]]
```

### Real-World ML Example: Feature Normalization

```python
import numpy as np

# Dataset: 5 samples, 3 features
data = np.array([[170, 65, 30],    # height(cm), weight(kg), age
                 [180, 80, 25],
                 [160, 55, 35],
                 [175, 70, 28],
                 [165, 60, 40]])

# Z-score normalization: (x - mean) / std
# Broadcasting makes this a one-liner!
mean = data.mean(axis=0)   # shape (3,) — mean of each feature
std = data.std(axis=0)     # shape (3,) — std of each feature

normalized = (data - mean) / std  # (5,3) - (3,) / (3,) → broadcasts!
print(f"Means after normalization: {normalized.mean(axis=0)}")
# [0. 0. 0.] — approximately zero (floating point)
print(f"Stds after normalization: {normalized.std(axis=0)}")
# [1. 1. 1.] — all standardized

# Min-Max normalization: (x - min) / (max - min)
min_val = data.min(axis=0)
max_val = data.max(axis=0)
scaled = (data - min_val) / (max_val - min_val)  # All values in [0, 1]
```

### Broadcasting Compatibility Quick Check

```python
import numpy as np

# Quick mental check: Can these broadcast?
# Read shapes RIGHT to LEFT, each pair must be equal or one must be 1

# (256, 256, 3) and (3,)        → YES: 3==3, 256→256, 256→256
# (256, 256, 3) and (256, 1)    → YES: 3&1, 256==256, 256→256
# (256, 256, 3) and (256, 3)    → ERROR: 256≠3 at second-from-right

# Use np.broadcast_shapes to check programmatically
print(np.broadcast_shapes((5, 3), (3,)))      # (5, 3)
print(np.broadcast_shapes((5, 1), (1, 3)))    # (5, 3)
print(np.broadcast_shapes((3, 1), (1, 4)))    # (3, 4)

# This will raise:
# np.broadcast_shapes((3, 4), (5, 4))  → ValueError
```

---

## Aggregation Functions

### What They Are

Aggregation (reduction) functions collapse array values into summary statistics. They can operate on the entire array or along a specific axis.

### Understanding Axes

```
For a 2D array (matrix):
- axis=0 → operate DOWN columns (collapse rows)
- axis=1 → operate ACROSS rows (collapse columns)
- axis=None → operate on ALL elements (flatten first)

Think of it as: "the axis that DISAPPEARS"

     axis=1 →
     ┌───┬───┬───┐
  a  │ 1 │ 2 │ 3 │ → sum(axis=1) = [6, 15, 24]
  x  ├───┼───┼───┤
  i  │ 4 │ 5 │ 6 │
  s  ├───┼───┼───┤
  =  │ 7 │ 8 │ 9 │
  0  └───┴───┴───┘
  ↓    ↓   ↓   ↓
      [12, 15, 18] = sum(axis=0)
```

### Core Aggregations

```python
import numpy as np

arr = np.array([[1, 2, 3],
                [4, 5, 6],
                [7, 8, 9]])

# ─── Sum ───
print(np.sum(arr))          # 45 (all elements)
print(np.sum(arr, axis=0))  # [12 15 18] (sum each column)
print(np.sum(arr, axis=1))  # [6 15 24] (sum each row)

# ─── Mean ───
print(np.mean(arr))          # 5.0
print(np.mean(arr, axis=0))  # [4. 5. 6.]
print(np.mean(arr, axis=1))  # [2. 5. 8.]

# ─── Standard Deviation and Variance ───
print(np.std(arr))           # 2.582 (population std, ddof=0)
print(np.std(arr, ddof=1))   # 2.739 (sample std)
print(np.var(arr))           # 6.667

# ─── Min, Max ───
print(np.min(arr))           # 1
print(np.max(arr, axis=1))   # [3 6 9] (max of each row)

# ─── Argmin, Argmax (index of min/max) ───
print(np.argmin(arr))         # 0 (flattened index)
print(np.argmax(arr, axis=0)) # [2 2 2] (row index of max in each col)
print(np.argmax(arr, axis=1)) # [2 2 2] (col index of max in each row)

# ─── Cumulative Sum / Product ───
a = np.array([1, 2, 3, 4, 5])
print(np.cumsum(a))   # [1  3  6 10 15]
print(np.cumprod(a))  # [1  2  6 24 120]

# ─── Percentile and Median ───
data = np.array([1, 3, 5, 7, 9, 11, 13])
print(np.median(data))            # 7.0
print(np.percentile(data, 25))    # 4.0 (Q1)
print(np.percentile(data, 75))    # 10.0 (Q3)
print(np.quantile(data, 0.5))     # 7.0 (same as median)
```

### keepdims — Preserving Dimensions

```python
import numpy as np

arr = np.array([[1, 2, 3],
                [4, 5, 6]])

# Without keepdims: shape collapses
mean_no_keep = arr.mean(axis=1)
print(mean_no_keep.shape)  # (2,)

# With keepdims: dimension preserved (as size 1)
mean_keep = arr.mean(axis=1, keepdims=True)
print(mean_keep.shape)  # (2, 1)

# Why it matters: broadcasting works correctly
# Center the data (subtract row means)
centered = arr - arr.mean(axis=1, keepdims=True)
print(centered)
# [[-1.  0.  1.]
#  [-1.  0.  1.]]

# Without keepdims, this would fail or give wrong result!
```

> **Pro Tip:** Always use `keepdims=True` when you plan to broadcast the result back against the original array. It's a lifesaver in ML normalization code.

---

## Comparison and Logic Operations

### Element-wise Comparisons

```python
import numpy as np

a = np.array([1, 2, 3, 4, 5])
b = np.array([5, 4, 3, 2, 1])

# Comparison operators return boolean arrays
print(a > b)     # [False False False  True  True]
print(a == b)    # [False False  True False False]
print(a >= 3)    # [False False  True  True  True]
print(a != b)    # [ True  True False  True  True]

# Combining conditions
mask = (a > 1) & (a < 5)   # AND
print(mask)  # [False  True  True  True False]

mask = (a < 2) | (a > 4)   # OR
print(mask)  # [ True False False False  True]

mask = ~(a > 3)             # NOT
print(mask)  # [ True  True  True False False]
```

### Aggregating Boolean Results

```python
import numpy as np

arr = np.array([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])

# How many elements satisfy a condition?
print(np.sum(arr > 5))       # 5 (True=1, False=0)
print(np.count_nonzero(arr > 5))  # 5 (slightly faster)

# Are any/all elements satisfying condition?
print(np.any(arr > 9))       # True (at least one > 9)
print(np.all(arr > 0))       # True (all positive)
print(np.all(arr > 5))       # False (not all > 5)

# Useful for validation
data = np.array([1.0, 2.0, np.nan, 4.0])
print(np.any(np.isnan(data)))    # True — has NaN!
print(np.all(np.isfinite(data))) # False
```

### np.where — Conditional Selection

```python
import numpy as np

arr = np.array([1, -2, 3, -4, 5])

# np.where(condition, value_if_true, value_if_false)
result = np.where(arr > 0, arr, 0)  # ReLU!
print(result)  # [1 0 3 0 5]

# Replace negatives with their absolute value
result = np.where(arr < 0, -arr, arr)
print(result)  # [1 2 3 4 5]

# np.where with just condition — returns INDICES
indices = np.where(arr > 0)
print(indices)  # (array([0, 2, 4]),) — indices where condition is True
print(arr[indices])  # [1 3 5]

# 2D example
matrix = np.array([[1, 2], [3, 4]])
rows, cols = np.where(matrix > 2)
print(f"Rows: {rows}, Cols: {cols}")  # Rows: [1 1], Cols: [0 1]
```

### np.select — Multiple Conditions

```python
import numpy as np

scores = np.array([95, 82, 67, 45, 73, 88, 55])

# Assign letter grades based on multiple conditions
conditions = [
    scores >= 90,    # A
    scores >= 80,    # B
    scores >= 70,    # C
    scores >= 60,    # D
]
choices = ['A', 'B', 'C', 'D']

grades = np.select(conditions, choices, default='F')
print(grades)  # ['A' 'B' 'D' 'F' 'C' 'B' 'F']
```

---

## Sorting and Searching

### Sorting

```python
import numpy as np

arr = np.array([3, 1, 4, 1, 5, 9, 2, 6])

# np.sort — returns SORTED COPY (original unchanged)
sorted_arr = np.sort(arr)
print(sorted_arr)  # [1 1 2 3 4 5 6 9]
print(arr)         # [3 1 4 1 5 9 2 6] — unchanged

# arr.sort() — sorts IN-PLACE (modifies original)
arr.sort()
print(arr)  # [1 1 2 3 4 5 6 9] — modified!

# Reverse sort
arr = np.array([3, 1, 4, 1, 5])
print(np.sort(arr)[::-1])  # [5 4 3 1 1]

# Sort 2D array
matrix = np.array([[3, 1, 2],
                   [6, 4, 5]])
print(np.sort(matrix, axis=1))  # Sort each row
# [[1 2 3]
#  [4 5 6]]
print(np.sort(matrix, axis=0))  # Sort each column
# [[3 1 2]
#  [6 4 5]]

# ─── argsort — get indices that WOULD sort the array ───
arr = np.array([30, 10, 40, 20])
indices = np.argsort(arr)
print(indices)       # [1 3 0 2] — index 1 is smallest, etc.
print(arr[indices])  # [10 20 30 40] — sorted using indices
```

### Partial Sorting (for Top-K)

```python
import numpy as np

arr = np.array([50, 10, 80, 30, 60, 20, 70, 40])

# np.partition: put k smallest elements first (unordered among themselves)
# Much faster than full sort for large arrays!
partitioned = np.partition(arr, 3)
print(partitioned[:3])  # [10 20 30] — 3 smallest (may not be sorted)
print(partitioned[3:])  # [...remaining...] — rest in arbitrary order

# Get indices of top-5 largest
top5_idx = np.argpartition(arr, -5)[-5:]
print(arr[top5_idx])  # Top 5 values (unsorted among themselves)

# If you need them sorted:
top5_sorted_idx = top5_idx[np.argsort(arr[top5_idx])[::-1]]
print(arr[top5_sorted_idx])  # Top 5, sorted descending
```

### Searching

```python
import numpy as np

arr = np.array([10, 20, 30, 40, 50])

# searchsorted — find insertion point in sorted array
# Binary search — O(log n)
idx = np.searchsorted(arr, 25)
print(idx)  # 2 — insert at index 2 to maintain sorted order

# Multiple values
indices = np.searchsorted(arr, [15, 35, 55])
print(indices)  # [1 3 5]

# Find unique elements
arr = np.array([3, 1, 2, 3, 1, 4, 2])
unique = np.unique(arr)
print(unique)  # [1 2 3 4]

# Unique with counts
unique, counts = np.unique(arr, return_counts=True)
print(dict(zip(unique, counts)))  # {1: 2, 2: 2, 3: 2, 4: 1}
```

---

## Linear Algebra Basics

### Why Linear Algebra in NumPy?

Linear algebra is the **language of ML**. Neural networks, PCA, SVD, linear regression — all linear algebra. NumPy provides optimized BLAS/LAPACK implementations.

### Matrix Multiplication

```python
import numpy as np

A = np.array([[1, 2],
              [3, 4]])
B = np.array([[5, 6],
              [7, 8]])

# ─── Matrix multiplication (3 equivalent ways) ───
C = A @ B              # Preferred (Python 3.5+)
C = np.matmul(A, B)    # Explicit function
C = np.dot(A, B)       # Works but less clear for matrices

print(C)
# [[19 22]
#  [43 50]]

# HOW: C[i,j] = sum(A[i,:] * B[:,j])
# C[0,0] = 1*5 + 2*7 = 19
# C[0,1] = 1*6 + 2*8 = 22
# C[1,0] = 3*5 + 4*7 = 43
# C[1,1] = 3*6 + 4*8 = 50
```

> **Warning:** `*` is element-wise multiplication, `@` is matrix multiplication. This is the most common confusion!

```python
import numpy as np

A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])

print(A * B)   # Element-wise: [[5,12], [21,32]]
print(A @ B)   # Matrix mult:  [[19,22], [43,50]]
```

### Vector Operations

```python
import numpy as np

# Dot product: sum of element-wise products
a = np.array([1, 2, 3])
b = np.array([4, 5, 6])
print(np.dot(a, b))  # 1*4 + 2*5 + 3*6 = 32

# Cross product (3D vectors only)
print(np.cross(a, b))  # [-3  6 -3]

# Vector norm (magnitude/length)
v = np.array([3, 4])
print(np.linalg.norm(v))  # 5.0 = sqrt(9 + 16)

# L1 norm (Manhattan distance)
print(np.linalg.norm(v, ord=1))  # 7.0 = |3| + |4|

# Normalize vector (unit vector)
unit_v = v / np.linalg.norm(v)
print(unit_v)                    # [0.6 0.8]
print(np.linalg.norm(unit_v))   # 1.0
```

### Matrix Operations

```python
import numpy as np

A = np.array([[1, 2],
              [3, 4]])

# Transpose
print(A.T)
# [[1 3]
#  [2 4]]

# Determinant
det = np.linalg.det(A)
print(f"det(A) = {det}")  # -2.0

# Inverse
A_inv = np.linalg.inv(A)
print(A_inv)
# [[-2.   1. ]
#  [ 1.5 -0.5]]

# Verify: A @ A_inv = Identity
print(A @ A_inv)  # [[1. 0.], [0. 1.]] (approximately)

# Trace (sum of diagonal)
print(np.trace(A))  # 5 (1 + 4)

# Rank
print(np.linalg.matrix_rank(A))  # 2

# Eigenvalues and Eigenvectors
eigenvalues, eigenvectors = np.linalg.eig(A)
print(f"Eigenvalues: {eigenvalues}")    # [-0.372  5.372]
print(f"Eigenvectors:\n{eigenvectors}") # Each column is an eigenvector
```

### Solving Linear Systems

$$Ax = b$$

```python
import numpy as np

# Solve: 2x + y = 5, x + 3y = 7
A = np.array([[2, 1],
              [1, 3]])
b = np.array([5, 7])

x = np.linalg.solve(A, b)
print(x)  # [1.6 1.8] → x=1.6, y=1.8

# Verify
print(A @ x)  # [5. 7.] ✓

# Least squares (for overdetermined systems)
# Fit y = mx + c through points
x_data = np.array([1, 2, 3, 4, 5])
y_data = np.array([2.1, 3.9, 6.2, 7.8, 10.1])

# Design matrix [x, 1]
A = np.column_stack([x_data, np.ones(len(x_data))])
# Solve in least-squares sense
result = np.linalg.lstsq(A, y_data, rcond=None)
m, c = result[0]
print(f"y = {m:.2f}x + {c:.2f}")  # y ≈ 2.0x + 0.0
```

---

## Set Operations

```python
import numpy as np

a = np.array([1, 2, 3, 4, 5])
b = np.array([3, 4, 5, 6, 7])

# Intersection (elements in both)
print(np.intersect1d(a, b))  # [3 4 5]

# Union (all unique elements)
print(np.union1d(a, b))      # [1 2 3 4 5 6 7]

# Difference (in a but not in b)
print(np.setdiff1d(a, b))    # [1 2]

# Symmetric difference (in either but not both)
print(np.setxor1d(a, b))     # [1 2 6 7]

# Membership test
print(np.isin(a, [2, 4, 6]))  # [False True False True False]

# Unique values (already covered but important)
arr = np.array([3, 1, 2, 3, 1, 4, 2])
print(np.unique(arr))  # [1 2 3 4]
```

---

## Common Mistakes

### 1. Confusing `*` and `@` for matrix multiplication

```python
import numpy as np

A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])

# WRONG (element-wise)
print(A * B)  # [[5, 12], [21, 32]]

# CORRECT (matrix multiplication)
print(A @ B)  # [[19, 22], [43, 50]]
```

### 2. Broadcasting shape mismatch

```python
import numpy as np

# This silently gives wrong results!
data = np.random.rand(100, 3)  # 100 samples, 3 features
weights = np.array([0.5, 0.3])  # Only 2 weights — MISMATCH!

# This would raise an error:
# result = data * weights  # ValueError: shapes (100,3) and (2,) not aligned

# FIX: ensure dimensions match
weights = np.array([0.5, 0.3, 0.2])  # 3 weights for 3 features
result = data * weights  # ✓ broadcasts correctly
```

### 3. Forgetting axis parameter

```python
import numpy as np

# Common bug: getting a scalar when you wanted per-column
data = np.array([[1, 2, 3],
                 [4, 5, 6]])

wrong = np.mean(data)          # 3.5 — single number!
right = np.mean(data, axis=0)  # [2.5, 3.5, 4.5] — per column
```

### 4. Integer overflow

```python
import numpy as np

# Small dtypes can overflow!
a = np.array([200, 200], dtype=np.uint8)
print(a + a)  # [144 144] — WRONG! 400 wraps around for uint8

# FIX: cast before operation
a = a.astype(np.int32)
print(a + a)  # [400 400] ✓
```

### 5. NaN propagation

```python
import numpy as np

arr = np.array([1.0, 2.0, np.nan, 4.0, 5.0])

print(np.sum(arr))     # nan — NaN poisons everything!
print(np.mean(arr))    # nan

# FIX: use nan-safe functions
print(np.nansum(arr))  # 12.0
print(np.nanmean(arr)) # 3.0
```

---

## Interview Questions

### Q1: What is broadcasting? Give an example where it's useful.
**Answer:** Broadcasting is NumPy's mechanism for performing operations on arrays with different shapes by virtually expanding the smaller array. Rules: compare shapes right-to-left; dimensions must be equal or one must be 1. Example: normalizing a dataset — subtract mean vector (shape `(n_features,)`) from data matrix (shape `(n_samples, n_features)`) in one operation.

### Q2: What's the difference between `np.dot()`, `np.matmul()` (`@`), and `*`?
**Answer:**
- `*` → element-wise multiplication (Hadamard product)
- `np.dot()` → dot product for 1D vectors; matrix multiplication for 2D; has special behavior for higher dims
- `np.matmul()` / `@` → matrix multiplication; no scalar multiplication; handles batches correctly for 3D+

### Q3: How would you find the top-k elements efficiently in a large array?
**Answer:** Use `np.argpartition(arr, -k)[-k:]` — it's $O(n)$ average time (partial sort), much faster than `np.argsort()` which is $O(n \log n)$. If you need them sorted, argsort just the k elements.

### Q4: Explain the `axis` parameter in NumPy aggregations.
**Answer:** The axis parameter specifies which dimension gets **collapsed** (reduced). For a 2D array: `axis=0` collapses rows (operates down columns, result has shape `(n_cols,)`), `axis=1` collapses columns (operates across each row, result has shape `(n_rows,)`). `axis=None` operates on flattened array.

### Q5: What are ufuncs and why are they important?
**Answer:** Universal functions (ufuncs) are vectorized wrappers for element-wise operations implemented in compiled C code. They're important because: (1) they're much faster than Python loops, (2) they handle broadcasting automatically, (3) they support the `out` parameter for zero-allocation operations, (4) they support reduction and accumulation methods.

### Q6: How do you handle NaN values in NumPy calculations?
**Answer:** NaN propagates through all standard operations (sum, mean, etc. all return NaN). Use nan-safe alternatives: `np.nansum`, `np.nanmean`, `np.nanstd`, `np.nanmax`, etc. Detect NaN with `np.isnan()`. Replace with `np.nan_to_num()` or boolean indexing: `arr[np.isnan(arr)] = 0`.

### Q7: What does `keepdims=True` do and when should you use it?
**Answer:** `keepdims=True` preserves the reduced axis as a dimension of size 1 instead of removing it. This is crucial when you want to broadcast the aggregation result back against the original array. Example: centering data with `data - data.mean(axis=0, keepdims=True)`.

---

## Quick Reference

### Arithmetic Operations

| Operation | Syntax | Notes |
|-----------|--------|-------|
| Add | `a + b` or `np.add(a, b)` | Element-wise |
| Subtract | `a - b` or `np.subtract(a, b)` | Element-wise |
| Multiply | `a * b` or `np.multiply(a, b)` | Element-wise |
| Divide | `a / b` or `np.divide(a, b)` | Returns float |
| Floor divide | `a // b` | Integer division |
| Power | `a ** n` or `np.power(a, n)` | Element-wise |
| Matrix multiply | `a @ b` or `np.matmul(a, b)` | True matrix mult |

### Aggregation Functions

| Function | Description | Nan-safe version |
|----------|-------------|-----------------|
| `np.sum()` | Sum of elements | `np.nansum()` |
| `np.mean()` | Arithmetic mean | `np.nanmean()` |
| `np.std()` | Standard deviation | `np.nanstd()` |
| `np.var()` | Variance | `np.nanvar()` |
| `np.min()` | Minimum | `np.nanmin()` |
| `np.max()` | Maximum | `np.nanmax()` |
| `np.argmin()` | Index of minimum | `np.nanargmin()` |
| `np.argmax()` | Index of maximum | `np.nanargmax()` |
| `np.median()` | Median value | `np.nanmedian()` |
| `np.percentile()` | Percentile | `np.nanpercentile()` |
| `np.cumsum()` | Cumulative sum | `np.nancumsum()` |
| `np.prod()` | Product | `np.nanprod()` |

### Broadcasting Rules Summary

```
1. Compare shapes from RIGHT to LEFT
2. Dimensions must be EQUAL or ONE of them is 1
3. Missing dimensions are treated as 1 (left-padded)

Examples:
  (5, 3)  +  (3,)    → (5, 3)  ✓
  (5, 3)  +  (5, 1)  → (5, 3)  ✓
  (5, 3)  +  (5,)    → ERROR   ✗
  (1, 3)  +  (5, 1)  → (5, 3)  ✓
  (8,1,6,1) + (7,1,5) → (8,7,6,5) ✓
```

### Linear Algebra Cheat Sheet

| Function | Description |
|----------|-------------|
| `A @ B` | Matrix multiplication |
| `A.T` | Transpose |
| `np.linalg.det(A)` | Determinant |
| `np.linalg.inv(A)` | Inverse |
| `np.linalg.eig(A)` | Eigenvalues & vectors |
| `np.linalg.norm(v)` | Vector/matrix norm |
| `np.linalg.solve(A, b)` | Solve Ax = b |
| `np.linalg.lstsq(A, b)` | Least squares |
| `np.linalg.svd(A)` | Singular value decomposition |
| `np.linalg.matrix_rank(A)` | Matrix rank |
| `np.trace(A)` | Sum of diagonal |

---

## Key Takeaways

1. **Vectorize everything** — avoid Python loops over array elements
2. **Broadcasting** is powerful but requires understanding shape alignment
3. **axis parameter** = "the dimension that disappears"
4. **`keepdims=True`** prevents shape bugs when broadcasting aggregation results
5. **`@` for matrix mult, `*` for element-wise** — never confuse them
6. **NaN propagates** — use `nansum`, `nanmean`, etc. when data has missing values
7. **`np.partition`** for top-k is faster than full sorting

---

*Next Chapter: [03-NumPy-Advanced](03-NumPy-Advanced.md) — Structured Arrays, Memory Layout, Performance Tips, Random Module*
