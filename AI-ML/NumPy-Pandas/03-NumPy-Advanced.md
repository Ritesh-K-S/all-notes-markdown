# Chapter 03: NumPy Advanced

## Table of Contents
- [Structured Arrays](#structured-arrays)
- [Memory Layout — C vs Fortran Order](#memory-layout--c-vs-fortran-order)
- [Strides Deep Dive](#strides-deep-dive)
- [NumPy Random Module](#numpy-random-module)
- [Performance Tips and Optimization](#performance-tips-and-optimization)
- [Memory-Mapped Files](#memory-mapped-files)
- [Custom ufuncs with np.vectorize and np.frompyfunc](#custom-ufuncs)
- [String and DateTime Operations](#string-and-datetime-operations)
- [Masked Arrays](#masked-arrays)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Structured Arrays

### What They Are

Structured arrays are NumPy arrays where each element is a **record** (like a row in a database table) with named fields of potentially different types. Think of it as a lightweight alternative to Pandas DataFrames when you need the speed of NumPy but want named columns.

**Analogy:** A regular NumPy array is like a spreadsheet where every cell must be the same type (all numbers). A structured array is like a spreadsheet where each column can be a different type — one column for names (strings), one for ages (integers), one for salaries (floats).

### Why They Matter

- Store heterogeneous data in a single array (name + age + salary)
- Used internally by many scientific libraries (astropy, h5py)
- Faster than Pandas for simple record access on large datasets
- Direct memory compatibility with C structs (important for interfacing with C code)

### How They Work

```python
import numpy as np

# ─── Method 1: Define dtype with a list of tuples ───
dt = np.dtype([('name', 'U20'),      # Unicode string, max 20 chars
               ('age', np.int32),      # 32-bit integer
               ('salary', np.float64)]) # 64-bit float

employees = np.array([
    ('Alice', 30, 75000.0),
    ('Bob', 25, 65000.0),
    ('Charlie', 35, 90000.0),
    ('Diana', 28, 72000.0)
], dtype=dt)

print(employees)
# [('Alice', 30, 75000.) ('Bob', 25, 65000.) ('Charlie', 35, 90000.) ('Diana', 28, 72000.)]

# ─── Access by field name (like a column) ───
print(employees['name'])     # ['Alice' 'Bob' 'Charlie' 'Diana']
print(employees['salary'])   # [75000. 65000. 90000. 72000.]

# ─── Access by index (like a row) ───
print(employees[0])          # ('Alice', 30, 75000.)
print(employees[0]['name'])  # Alice

# ─── Filtering works naturally ───
high_earners = employees[employees['salary'] > 70000]
print(high_earners['name'])  # ['Alice' 'Charlie' 'Diana']

# ─── Vectorized operations on fields ───
print(f"Average salary: ${employees['salary'].mean():,.0f}")  # $75,500
print(f"Average age: {employees['age'].mean():.1f}")           # 29.5
```

### Method 2: Dictionary-Style dtype

```python
import numpy as np

# More readable for complex dtypes
dt = np.dtype({
    'names': ['x', 'y', 'z', 'label'],
    'formats': [np.float64, np.float64, np.float64, 'U10']
})

points = np.array([
    (1.0, 2.0, 3.0, 'A'),
    (4.0, 5.0, 6.0, 'B'),
    (7.0, 8.0, 9.0, 'A'),
], dtype=dt)

# Get all 'A' labeled points
a_points = points[points['label'] == 'A']
print(a_points['x'])  # [1. 7.]
```

### Nested Structured Arrays

```python
import numpy as np

# Nested dtype — struct within struct
address_dt = np.dtype([('street', 'U50'), ('city', 'U30'), ('zip', 'U10')])
person_dt = np.dtype([('name', 'U30'), ('age', np.int32), ('address', address_dt)])

people = np.array([
    ('Alice', 30, ('123 Main St', 'New York', '10001')),
    ('Bob', 25, ('456 Oak Ave', 'Boston', '02101')),
], dtype=person_dt)

print(people['address']['city'])  # ['New York' 'Boston']
```

> **When to use Structured Arrays vs Pandas:** Use structured arrays when you need raw speed and simple column access on large datasets. Use Pandas when you need groupby, merge, pivot, time series, or complex data manipulation.

---

## Memory Layout — C vs Fortran Order

### What It Is

When a 2D array is stored in memory (which is 1-dimensional), the elements must be laid out in some linear order. There are two conventions:

- **C order (row-major):** Rows are stored contiguously. Element `[0,0], [0,1], [0,2], [1,0], [1,1], [1,2]...`
- **Fortran order (column-major):** Columns are stored contiguously. Element `[0,0], [1,0], [2,0], [0,1], [1,1], [2,1]...`

```
Array:        C order (row-major):        Fortran order (column-major):
┌───┬───┬───┐
│ 1 │ 2 │ 3 │  Memory: [1,2,3,4,5,6]     Memory: [1,4,2,5,3,6]
├───┼───┼───┤
│ 4 │ 5 │ 6 │  Row 0 together, then       Col 0 together, then
└───┴───┴───┘  Row 1 together              Col 1 together
```

### Why It Matters

- **Cache efficiency**: Accessing elements in memory order is faster (CPU cache lines)
- **Interoperability**: C libraries expect C order; Fortran/MATLAB expect Fortran order
- **Performance**: Iterating along the "wrong" axis can be 2-10x slower

```python
import numpy as np
import time

# Create arrays in different orders
c_arr = np.zeros((10000, 10000), order='C')    # Row-major (default)
f_arr = np.zeros((10000, 10000), order='F')    # Column-major

# ─── Row-wise sum: C order is faster ───
start = time.time()
_ = c_arr.sum(axis=1)  # Sum across columns (within each row)
print(f"C order, row sum: {time.time() - start:.4f}s")

start = time.time()
_ = f_arr.sum(axis=1)
print(f"F order, row sum: {time.time() - start:.4f}s")

# ─── Column-wise sum: F order is faster ───
start = time.time()
_ = c_arr.sum(axis=0)  # Sum down rows (within each column)
print(f"C order, col sum: {time.time() - start:.4f}s")

start = time.time()
_ = f_arr.sum(axis=0)
print(f"F order, col sum: {time.time() - start:.4f}s")
```

### Checking Memory Layout

```python
import numpy as np

arr = np.array([[1, 2, 3], [4, 5, 6]])

print(arr.flags)
# C_CONTIGUOUS : True    ← stored in C order
# F_CONTIGUOUS : False
# OWNDATA : True
# WRITEABLE : True
# ALIGNED : True

# After transpose — it's Fortran order!
t = arr.T
print(t.flags['C_CONTIGUOUS'])  # False
print(t.flags['F_CONTIGUOUS'])  # True

# Force a copy in C order (useful when passing to C libraries)
c_copy = np.ascontiguousarray(t)
print(c_copy.flags['C_CONTIGUOUS'])  # True
```

> **Pro Tip:** NumPy defaults to C order. If you're working with Fortran or MATLAB data, use `order='F'`. If you need to pass array data to a C extension, ensure it's contiguous with `np.ascontiguousarray()`.

---

## Strides Deep Dive

### What They Are

Strides tell NumPy how many **bytes** to skip in memory to move to the next element along each dimension. They're the mechanism that makes views, transposition, and broadcasting work without copying data.

**Analogy:** Imagine a long hallway of lockers (memory). Strides tell you: "To get to the next row, skip 24 lockers. To get to the next column, skip 8 lockers."

### How They Work

```python
import numpy as np

arr = np.array([[1, 2, 3],
                [4, 5, 6]], dtype=np.int64)  # 8 bytes per element

print(arr.strides)  # (24, 8)
# (24, 8) means:
#   - To move to the next ROW: skip 24 bytes (3 elements × 8 bytes)
#   - To move to the next COL: skip 8 bytes (1 element × 8 bytes)

# Memory layout (C order):
# Byte:  0    8   16   24   32   40
# Value: [1]  [2]  [3]  [4]  [5]  [6]
#        ←─ row 0 ─→    ←─ row 1 ─→
```

### How Transpose Works (Zero Copy!)

```python
import numpy as np

arr = np.array([[1, 2, 3],
                [4, 5, 6]], dtype=np.int64)
print(f"Original strides: {arr.strides}")  # (24, 8)

t = arr.T
print(f"Transposed strides: {t.strides}")  # (8, 24)
# Just SWAPPED the strides! No data was moved!
# Now "next row" = 8 bytes (was next col)
# And "next col" = 24 bytes (was next row)
print(t.base is arr)  # True — shares the same data!
```

### Stride Tricks (Advanced)

```python
import numpy as np
from numpy.lib.stride_tricks import as_strided

# ─── Create a sliding window view (no copying!) ───
arr = np.array([1, 2, 3, 4, 5, 6, 7, 8], dtype=np.int64)
window_size = 3

# Calculate: we need (n - window + 1) windows, each of size window_size
n_windows = len(arr) - window_size + 1
new_shape = (n_windows, window_size)
new_strides = (arr.strides[0], arr.strides[0])  # (8, 8) — same stride both ways

windows = as_strided(arr, shape=new_shape, strides=new_strides)
print(windows)
# [[1 2 3]
#  [2 3 4]
#  [3 4 5]
#  [4 5 6]
#  [5 6 7]
#  [6 7 8]]
# Each row is a sliding window — and NO data was copied!

# Safer alternative in modern NumPy:
windows_safe = np.lib.stride_tricks.sliding_window_view(arr, window_size)
print(windows_safe)  # Same result, but safer
```

> **Warning:** `as_strided` is powerful but dangerous — wrong strides can read arbitrary memory. Prefer `sliding_window_view` in NumPy 1.20+.

---

## NumPy Random Module

### What It Is

NumPy's random module generates pseudo-random numbers — numbers that appear random but are produced by a deterministic algorithm. This is essential for ML (initializing weights, shuffling data, sampling).

### Why It Matters

- **Reproducibility**: Setting a seed ensures same results every run
- **Simulation**: Monte Carlo methods, bootstrapping
- **Data augmentation**: Random transforms for training data
- **Model initialization**: Neural network weight initialization

### Old vs New API

```python
import numpy as np

# ─── OLD API (legacy, still works) ───
np.random.seed(42)           # Global seed
x = np.random.rand(3)       # Uniform [0, 1)
y = np.random.randn(3)      # Normal(0, 1)

# ─── NEW API (recommended since NumPy 1.17) ───
rng = np.random.default_rng(42)  # Create a Generator with seed
x = rng.random(3)               # Uniform [0, 1)
y = rng.standard_normal(3)      # Normal(0, 1)
```

> **Pro Tip:** Always use the new `default_rng()` API. The old API uses a global state that can cause bugs in multi-threaded code and makes reproducibility harder.

### Common Distributions

```python
import numpy as np

rng = np.random.default_rng(42)

# ─── Uniform Distribution ───
# All values equally likely in [low, high)
uniform = rng.uniform(low=0, high=10, size=(3, 3))
print(uniform.round(2))

# ─── Normal (Gaussian) Distribution ───
# Bell curve: most values near mean, fewer far away
normal = rng.normal(loc=0, scale=1, size=1000)  # mean=0, std=1
print(f"Mean: {normal.mean():.3f}, Std: {normal.std():.3f}")

# ─── Integer Random ───
integers = rng.integers(low=0, high=10, size=5)  # [0, 10)
print(integers)  # e.g., [0 7 6 4 4]

# ─── Binomial ───
# Number of successes in n trials with probability p
flips = rng.binomial(n=10, p=0.5, size=5)  # 10 coin flips, 5 experiments
print(flips)  # e.g., [4 7 3 5 6]

# ─── Poisson ───
# Number of events in a fixed interval (avg rate = lam)
events = rng.poisson(lam=5, size=10)
print(events)  # e.g., [6 4 3 8 5 5 7 4 3 6]

# ─── Exponential ───
# Time between events in a Poisson process
wait_times = rng.exponential(scale=2.0, size=5)  # mean wait = 2.0
print(wait_times.round(2))

# ─── Choice (sampling from array) ───
items = np.array(['apple', 'banana', 'cherry', 'date'])
sampled = rng.choice(items, size=3, replace=False)  # Without replacement
print(sampled)

# Weighted sampling
weights = [0.1, 0.5, 0.3, 0.1]  # banana most likely
sampled = rng.choice(items, size=10, replace=True, p=weights)
print(sampled)
```

### Shuffling and Permutation

```python
import numpy as np

rng = np.random.default_rng(42)

# ─── Shuffle (in-place) ───
arr = np.array([1, 2, 3, 4, 5])
rng.shuffle(arr)
print(arr)  # e.g., [3 5 1 4 2]

# ─── Permutation (returns new array) ───
arr = np.array([1, 2, 3, 4, 5])
perm = rng.permutation(arr)
print(arr)   # [1 2 3 4 5] — unchanged!
print(perm)  # [3 5 1 4 2] — shuffled copy

# ─── Shuffle rows of a dataset (keeping row integrity) ───
X = np.array([[1, 2], [3, 4], [5, 6], [7, 8], [9, 10]])
y = np.array([0, 1, 0, 1, 0])

indices = rng.permutation(len(X))
X_shuffled = X[indices]
y_shuffled = y[indices]
# Now X_shuffled and y_shuffled are shuffled together correctly
```

### Reproducibility Patterns

```python
import numpy as np

# ─── Pattern 1: Global reproducibility ───
rng = np.random.default_rng(42)
print(rng.random(3))  # Always: [0.773... 0.438... 0.858...]

# ─── Pattern 2: Per-function reproducibility ───
def train_model(seed=42):
    rng = np.random.default_rng(seed)
    weights = rng.normal(0, 0.01, size=(10, 5))
    return weights

# Same seed → same weights every time
w1 = train_model(42)
w2 = train_model(42)
print(np.array_equal(w1, w2))  # True

# ─── Pattern 3: Spawning independent streams ───
# For parallel processing — each worker gets independent stream
parent_rng = np.random.default_rng(42)
child_rngs = parent_rng.spawn(4)  # 4 independent generators
for i, child in enumerate(child_rngs):
    print(f"Worker {i}: {child.random(3).round(3)}")
```

---

## Performance Tips and Optimization

### 1. Vectorize — Avoid Python Loops

```python
import numpy as np
import time

size = 1_000_000

# SLOW: Python loop
arr = np.random.rand(size)
start = time.time()
result = np.empty(size)
for i in range(size):
    result[i] = arr[i] ** 2 + 2 * arr[i] + 1
print(f"Loop: {time.time() - start:.4f}s")

# FAST: Vectorized
start = time.time()
result = arr ** 2 + 2 * arr + 1
print(f"Vectorized: {time.time() - start:.4f}s")
# Typically 50-200x faster
```

### 2. Use In-Place Operations

```python
import numpy as np

arr = np.random.rand(10_000_000)

# SLOW: creates new arrays for each operation
# result = arr * 2 + 1  # allocates 2 temporary arrays

# FASTER: in-place operations
arr *= 2    # No new array
arr += 1    # No new array

# Or use the out parameter
a = np.random.rand(10_000_000)
b = np.random.rand(10_000_000)
out = np.empty_like(a)
np.add(a, b, out=out)      # Write directly to pre-allocated array
np.multiply(out, 2, out=out)  # Reuse same buffer
```

### 3. Choose the Right dtype

```python
import numpy as np

# float64 (default) — 8 bytes per element
big = np.zeros((10000, 10000), dtype=np.float64)
print(f"float64: {big.nbytes / 1e6:.0f} MB")  # 800 MB

# float32 — 4 bytes, enough for most ML
small = np.zeros((10000, 10000), dtype=np.float32)
print(f"float32: {small.nbytes / 1e6:.0f} MB")  # 400 MB

# float16 — 2 bytes, for inference/GPU
tiny = np.zeros((10000, 10000), dtype=np.float16)
print(f"float16: {tiny.nbytes / 1e6:.0f} MB")   # 200 MB

# For boolean masks, use bool_ (1 byte) not int (8 bytes)
mask = np.zeros(1_000_000, dtype=np.bool_)
print(f"bool: {mask.nbytes / 1e6:.1f} MB")  # 1.0 MB
```

### 4. Avoid Unnecessary Copies

```python
import numpy as np

arr = np.arange(1_000_000)

# Creates a COPY (uses 2x memory)
subset_copy = arr[[0, 1, 2, 3, 4]]  # Fancy indexing = copy

# Creates a VIEW (shares memory)
subset_view = arr[0:5]  # Slicing = view

# ravel() returns view when possible, flatten() always copies
flat_view = arr.ravel()     # ✓ View
flat_copy = arr.flatten()   # ✗ Copy (slower, more memory)
```

### 5. Use np.einsum for Complex Operations

```python
import numpy as np

# Einstein summation — compact, optimized multi-dimensional ops
A = np.random.rand(100, 50)
B = np.random.rand(50, 80)

# Matrix multiplication
C = np.einsum('ij,jk->ik', A, B)  # Same as A @ B

# Batch matrix multiply
batch_A = np.random.rand(32, 4, 5)
batch_B = np.random.rand(32, 5, 3)
batch_C = np.einsum('bij,bjk->bik', batch_A, batch_B)
print(batch_C.shape)  # (32, 4, 3)

# Trace
M = np.random.rand(5, 5)
print(np.einsum('ii->', M))  # Same as np.trace(M)

# Diagonal
print(np.einsum('ii->i', M))  # Same as np.diag(M)

# Element-wise multiply and sum (dot product generalized)
a = np.random.rand(100)
b = np.random.rand(100)
print(np.einsum('i,i->', a, b))  # Same as np.dot(a, b)
```

### 6. Preallocate Arrays

```python
import numpy as np
import time

n = 100_000

# SLOW: growing a list, then converting
start = time.time()
results = []
for i in range(n):
    results.append(i ** 2)
arr = np.array(results)
print(f"Append + convert: {time.time() - start:.4f}s")

# FAST: preallocate and fill
start = time.time()
arr = np.empty(n, dtype=np.int64)
for i in range(n):
    arr[i] = i ** 2
print(f"Preallocate: {time.time() - start:.4f}s")

# FASTEST: vectorize (no loop at all)
start = time.time()
arr = np.arange(n) ** 2
print(f"Vectorized: {time.time() - start:.4f}s")
```

### Performance Comparison Table

| Technique | Speedup | Memory Impact |
|-----------|---------|---------------|
| Vectorization | 50-200x | Same |
| In-place operations | 1.5-3x | Saves 50%+ |
| float32 vs float64 | 1.2-2x | Saves 50% |
| Preallocate arrays | 2-5x | Same |
| `np.einsum` for complex | 1-3x | Less temp arrays |
| `ravel()` vs `flatten()` | 1-10x | View vs copy |
| `np.where` vs loop | 20-100x | Same |

---

## Memory-Mapped Files

### What They Are

Memory-mapped files let you work with arrays **larger than RAM** by mapping a file on disk to a virtual memory address. NumPy reads/writes only the parts you access — like having a window that slides over a massive dataset.

**Analogy:** Imagine a library with 10 million books. Instead of carrying all books home, you just carry a library card. When you need a specific book, you request it — it appears magically. That's memory mapping.

### How They Work

```python
import numpy as np
import os

# ─── Create a large memory-mapped file ───
filename = 'large_data.dat'
shape = (10000, 1000)

# Create the file (write mode)
mmap_write = np.memmap(filename, dtype=np.float32, mode='w+', shape=shape)
mmap_write[:100] = np.random.rand(100, 1000).astype(np.float32)  # Write some data
mmap_write.flush()  # Ensure data is written to disk
del mmap_write  # Close

# Read back (read-only mode for safety)
mmap_read = np.memmap(filename, dtype=np.float32, mode='r', shape=shape)
print(mmap_read[:2, :5])  # Access just the data you need
print(f"File size: {os.path.getsize(filename) / 1e6:.1f} MB")

# Clean up
del mmap_read
os.remove(filename)
```

### When to Use Memory-Mapped Files

| Scenario | Use memmap? |
|----------|-------------|
| Dataset larger than RAM | ✓ Yes |
| Random access to large file | ✓ Yes |
| Shared data between processes | ✓ Yes |
| Small arrays (< 1 GB) | ✗ No (load into memory) |
| Sequential streaming | ✗ No (use chunks) |

---

## Custom ufuncs

### np.vectorize — Easy but NOT Fast

```python
import numpy as np

# Wraps a scalar function to work on arrays
# WARNING: This does NOT actually vectorize — it's just a convenience wrapper!
def my_func(x):
    if x > 0:
        return x ** 2
    else:
        return 0

vectorized_func = np.vectorize(my_func)
arr = np.array([-2, -1, 0, 1, 2, 3])
print(vectorized_func(arr))  # [0 0 0 1 4 9]

# FASTER alternative: use np.where
result = np.where(arr > 0, arr ** 2, 0)
print(result)  # [0 0 0 1 4 9] — same result, much faster
```

> **Important:** `np.vectorize` is a convenience, not a performance tool. It still loops internally. Always prefer built-in NumPy operations or `np.where`.

### np.frompyfunc — Slightly More Flexible

```python
import numpy as np

def categorize(x):
    if x < 0:
        return 'negative'
    elif x == 0:
        return 'zero'
    else:
        return 'positive'

# Returns object array (slower but flexible)
cat_func = np.frompyfunc(categorize, 1, 1)
arr = np.array([-3, -1, 0, 2, 5])
print(cat_func(arr))  # ['negative' 'negative' 'zero' 'positive' 'positive']
```

### True Vectorization with np.piecewise

```python
import numpy as np

x = np.array([-3, -1, 0, 1, 3, 5])

# Define a piecewise function (truly vectorized)
result = np.piecewise(
    x,
    [x < 0, x == 0, x > 0],            # conditions
    [lambda x: -x, 0, lambda x: x**2]  # corresponding functions/values
)
print(result)  # [3 1 0 1 9 25]
```

---

## String and DateTime Operations

### String Arrays

```python
import numpy as np

# Fixed-length strings
names = np.array(['Alice', 'Bob', 'Charlie', 'Diana'], dtype='U20')

# Use np.char for vectorized string operations
print(np.char.upper(names))       # ['ALICE' 'BOB' 'CHARLIE' 'DIANA']
print(np.char.lower(names))       # ['alice' 'bob' 'charlie' 'diana']
print(np.char.str_len(names))     # [5 3 7 5]
print(np.char.startswith(names, 'A'))  # [True False False False]
print(np.char.replace(names, 'a', '@'))  # ['Alice' 'Bob' 'Ch@rlie' 'Di@n@']
print(np.char.add(names, ' Smith'))  # ['Alice Smith' 'Bob Smith' ...]

# String comparison
print(np.char.equal(names, 'Bob'))  # [False True False False]
```

### DateTime Arrays

```python
import numpy as np

# NumPy has built-in datetime64 type
dates = np.array(['2024-01-15', '2024-06-22', '2024-12-31'], dtype='datetime64')
print(dates)
# ['2024-01-15' '2024-06-22' '2024-12-31']

# Date arithmetic
print(dates + np.timedelta64(7, 'D'))   # Add 7 days
print(dates - dates[0])                  # Differences: [0, 159, 351] days

# Generate date ranges
date_range = np.arange('2024-01', '2024-07', dtype='datetime64[M]')
print(date_range)  # ['2024-01' '2024-02' '2024-03' '2024-04' '2024-05' '2024-06']

# Business days
business_days = np.busday_count('2024-01-01', '2024-01-31')
print(f"Business days in Jan 2024: {business_days}")  # 23

# Check if a date is a business day
print(np.is_busday('2024-01-15'))  # True (Monday)
print(np.is_busday('2024-01-14'))  # False (Sunday)
```

> **Note:** For complex date/time operations, Pandas is far superior. NumPy's datetime64 is best for simple date storage and arithmetic.

---

## Masked Arrays

### What They Are

Masked arrays let you mark certain elements as "invalid" or "missing" without removing them. Operations automatically skip masked elements.

```python
import numpy as np
import numpy.ma as ma

# Sensor data with some invalid readings (-999)
raw_data = np.array([23.5, 24.1, -999, 22.8, -999, 25.3, 24.7])

# Create masked array (mask the invalid values)
data = ma.masked_equal(raw_data, -999)
print(data)
# [23.5 24.1 -- 22.8 -- 25.3 24.7]

# Operations automatically ignore masked values
print(f"Mean: {data.mean():.2f}")    # 24.08 (only valid values)
print(f"Max: {data.max():.2f}")      # 25.3
print(f"Count valid: {data.count()}")  # 5

# Mask by condition
temps = np.array([20, 45, 22, 100, 23, -50, 24])
valid_temps = ma.masked_outside(temps, -10, 50)  # Mask unreasonable values
print(valid_temps)       # [20 45 22 -- 23 -- 24]
print(valid_temps.mean())  # 26.8

# Fill masked values with a specific value
filled = data.filled(fill_value=0)
print(filled)  # [23.5 24.1 0. 22.8 0. 25.3 24.7]

# Fill with mean
filled_mean = data.filled(fill_value=data.mean())
print(filled_mean.round(2))  # [23.5 24.1 24.08 22.8 24.08 25.3 24.7]
```

### Masked Arrays vs NaN

| Feature | Masked Array | NaN approach |
|---------|-------------|--------------|
| Memory | Extra bool mask array | No overhead |
| Aggregation | Auto-skip in all ops | Need `nan`-functions |
| Integer arrays | ✓ Works | ✗ NaN is float-only |
| Pandas compat | Less common | Standard approach |
| Speed | Slightly slower | Slightly faster |

---

## Common Mistakes

### 1. Using the old random API in production

```python
import numpy as np

# WRONG: global state, not reproducible in parallel code
np.random.seed(42)
x = np.random.rand(5)

# CORRECT: explicit generator, thread-safe
rng = np.random.default_rng(42)
x = rng.random(5)
```

### 2. Misunderstanding memory layout after transpose

```python
import numpy as np

arr = np.array([[1, 2, 3], [4, 5, 6]])
t = arr.T

# t is a VIEW — NOT contiguous in C order
print(t.flags['C_CONTIGUOUS'])  # False

# Some operations may silently copy or give wrong results
# FIX: make contiguous if needed
t_contiguous = np.ascontiguousarray(t)
```

### 3. Using np.vectorize for performance

```python
import numpy as np

# WRONG: thinking this is fast
@np.vectorize
def slow_relu(x):
    return max(0, x)

# CORRECT: use built-in NumPy ops
def fast_relu(x):
    return np.maximum(0, x)

arr = np.random.randn(1_000_000)
# fast_relu is 100x faster than slow_relu
```

### 4. Not using random seeds for reproducibility

```python
import numpy as np

# WRONG: different results every run
weights = np.random.randn(100, 50)

# CORRECT: reproducible
rng = np.random.default_rng(42)
weights = rng.standard_normal((100, 50))
```

### 5. Ignoring dtype overflow

```python
import numpy as np

# uint8 overflow (common with images)
img = np.array([250, 200, 180], dtype=np.uint8)
brightened = img + np.uint8(50)
print(brightened)  # [44 250 230] — 250+50=300 wraps to 44!

# FIX: use np.clip or cast first
brightened = np.clip(img.astype(np.int16) + 50, 0, 255).astype(np.uint8)
print(brightened)  # [255 250 230] ✓
```

---

## Interview Questions

### Q1: Explain memory layout (C order vs Fortran order) and when it matters.
**Answer:** C order (row-major) stores rows contiguously; Fortran order (column-major) stores columns contiguously. It matters for performance: accessing elements in storage order is faster due to CPU cache lines. If your inner loop iterates over columns, Fortran order is faster; if over rows, C order is faster. It also matters for interoperability with C and Fortran libraries.

### Q2: What are strides in NumPy and how does transpose work internally?
**Answer:** Strides are the number of bytes to skip in memory to move to the next element along each axis. For a 2D `float64` array of shape `(3, 4)`, strides are `(32, 8)` — skip 32 bytes for next row, 8 for next column. Transpose works by simply swapping the strides (and shape) without moving any data. It's $O(1)$ time and zero memory cost.

### Q3: How would you work with a dataset larger than RAM using NumPy?
**Answer:** Use `np.memmap` (memory-mapped files) — maps a file on disk to virtual memory, loading only accessed portions. Alternatively, process data in chunks using `np.load` with `mmap_mode='r'` for `.npy` files. For very large datasets, consider HDF5 (h5py) or Zarr.

### Q4: Explain the difference between np.random.seed() and np.random.default_rng().
**Answer:** `np.random.seed()` sets a global random state shared across all code — it's thread-unsafe and hard to reason about. `default_rng()` creates an independent Generator object with its own state — it's thread-safe, composable (can spawn child generators), and uses a better algorithm (PCG64 vs Mersenne Twister). Always prefer `default_rng()`.

### Q5: What is np.einsum and when would you use it?
**Answer:** `np.einsum` performs Einstein summation — a compact notation for specifying multi-dimensional array operations (dot products, matrix multiplication, batch operations, traces, etc.) in a single function call. Use it when you need complex tensor operations that would otherwise require multiple intermediate arrays. It can be faster because it fuses operations and avoids temporaries.

### Q6: What are structured arrays and when are they useful?
**Answer:** Structured arrays have named fields with potentially different dtypes per field (like a database record). They're useful when you need heterogeneous tabular data with NumPy performance, C struct interop (shared memory with C programs), or lightweight record storage without Pandas overhead. They're faster than Pandas for simple column access on large datasets but lack Pandas' rich data manipulation features.

### Q7: How do you optimize NumPy code for production?
**Answer:**
1. Vectorize — eliminate Python loops
2. Use appropriate dtypes (float32 vs float64)
3. In-place operations to avoid temporary arrays
4. Preallocate output arrays
5. Use `ravel()` over `flatten()`
6. `np.einsum` for complex operations
7. Ensure contiguous memory for C extensions
8. Use `np.memmap` for data > RAM

---

## Quick Reference

### Random Number Generation (New API)

| Function | Distribution | Example |
|----------|-------------|---------|
| `rng.random(size)` | Uniform [0, 1) | `rng.random((3,3))` |
| `rng.integers(lo, hi, size)` | Uniform integers | `rng.integers(0, 10, 5)` |
| `rng.standard_normal(size)` | Normal(0, 1) | `rng.standard_normal(100)` |
| `rng.normal(mu, sigma, size)` | Normal(μ, σ) | `rng.normal(5, 2, 100)` |
| `rng.uniform(lo, hi, size)` | Uniform [lo, hi) | `rng.uniform(-1, 1, 100)` |
| `rng.binomial(n, p, size)` | Binomial | `rng.binomial(10, 0.5, 100)` |
| `rng.poisson(lam, size)` | Poisson | `rng.poisson(5, 100)` |
| `rng.exponential(scale, size)` | Exponential | `rng.exponential(2, 100)` |
| `rng.choice(arr, size)` | Sample from array | `rng.choice(arr, 5)` |
| `rng.shuffle(arr)` | Shuffle in-place | `rng.shuffle(arr)` |
| `rng.permutation(arr)` | Shuffled copy | `rng.permutation(arr)` |

### Memory Layout Checklist

| Check | Command |
|-------|---------|
| Is C contiguous? | `arr.flags['C_CONTIGUOUS']` |
| Is Fortran contiguous? | `arr.flags['F_CONTIGUOUS']` |
| View strides | `arr.strides` |
| Force C contiguous | `np.ascontiguousarray(arr)` |
| Force Fortran contiguous | `np.asfortranarray(arr)` |
| Memory size (bytes) | `arr.nbytes` |
| Shares memory? | `np.shares_memory(a, b)` |

### Performance Quick Guide

| Situation | Do This | Don't Do This |
|-----------|---------|---------------|
| Element-wise math | Vectorized ops (`arr * 2`) | `for` loop |
| Conditional | `np.where(cond, a, b)` | `if/else` in loop |
| Random numbers | `default_rng(seed)` | `np.random.seed()` |
| Type conversion | `.astype(np.float32)` | Keep default float64 |
| Flatten array | `.ravel()` | `.flatten()` |
| Pre-allocate | `np.empty(n)` then fill | Append to list |
| Complex tensor ops | `np.einsum(...)` | Multiple matmuls |

---

## Key Takeaways

1. **Structured arrays** = lightweight DataFrames within NumPy (fast, typed, named fields)
2. **Memory layout matters** — C order for row operations, Fortran order for column operations
3. **Strides** are the secret behind views, transpose, and broadcasting (zero-copy magic)
4. **Always use `default_rng()`** — thread-safe, reproducible, better algorithms
5. **`np.vectorize` is NOT fast** — it's just syntactic sugar for a loop
6. **Optimize in this order**: vectorize → right dtype → in-place → preallocate → einsum
7. **Memory-mapped files** let you work with datasets larger than RAM

---

*Next Chapter: [04-Pandas-Series-and-DataFrame](04-Pandas-Series-and-DataFrame.md) — Series, DataFrame, Creation, Indexing, Selection, dtypes*
