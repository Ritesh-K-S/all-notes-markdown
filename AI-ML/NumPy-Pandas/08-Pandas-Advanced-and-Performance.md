# Chapter 08: Pandas Advanced and Performance

## Table of Contents
- [MultiIndex (Hierarchical Indexing)](#multiindex-hierarchical-indexing)
- [Categorical Data Type](#categorical-data-type)
- [eval() and query() — Expression Evaluation](#eval-and-query--expression-evaluation)
- [Chunked Processing for Large Files](#chunked-processing-for-large-files)
- [Memory Optimization](#memory-optimization)
- [Vectorization and Performance Patterns](#vectorization-and-performance-patterns)
- [Alternative File Formats](#alternative-file-formats)
- [Method Chaining and Pipe](#method-chaining-and-pipe)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## MultiIndex (Hierarchical Indexing)

### What It Is
A MultiIndex is an index with multiple levels — like a tree structure for row/column labels. Think of a library catalog organized first by genre, then by author, then by title. Each level narrows down the search. In SQL terms, it's like a composite primary key.

### Why It Matters
- Represents higher-dimensional data in a 2D DataFrame (panel data)
- Enables powerful cross-section analysis: "sales by region by product by quarter"
- GroupBy results naturally create MultiIndex — you need to know how to work with them
- Unlocks `.xs()`, `.unstack()`, `.stack()` for reshaping data

### How It Works

```
Single Index:               MultiIndex (2 levels):
┌───────┬───────┐           ┌───────────────┬───────┐
│ Index │ Value │           │ Level0│Level1 │ Value │
├───────┼───────┤           ├───────┼───────┼───────┤
│   0   │  100  │           │ East  │  Q1   │  100  │
│   1   │  200  │           │       │  Q2   │  150  │
│   2   │  300  │           │ West  │  Q1   │  200  │
│   3   │  400  │           │       │  Q2   │  250  │
└───────┴───────┘           └───────┴───────┴───────┘
```

### Code Examples

```python
import pandas as pd
import numpy as np

# =============================================
# CREATING MULTIINDEX
# =============================================

# Method 1: From tuples
index = pd.MultiIndex.from_tuples([
    ('East', 'Q1'), ('East', 'Q2'), ('East', 'Q3'), ('East', 'Q4'),
    ('West', 'Q1'), ('West', 'Q2'), ('West', 'Q3'), ('West', 'Q4')
], names=['region', 'quarter'])

df = pd.DataFrame({
    'sales': [100, 150, 200, 180, 120, 160, 210, 190],
    'profit': [20, 35, 50, 40, 25, 38, 55, 42]
}, index=index)

# Method 2: From product (all combinations)
regions = ['East', 'West', 'South']
quarters = ['Q1', 'Q2', 'Q3', 'Q4']
index = pd.MultiIndex.from_product([regions, quarters], names=['region', 'quarter'])

# Method 3: From DataFrame columns (most common — after groupby)
data = pd.DataFrame({
    'region': ['East', 'East', 'West', 'West'] * 4,
    'product': ['A', 'B'] * 8,
    'quarter': (['Q1'] * 4 + ['Q2'] * 4 + ['Q3'] * 4 + ['Q4'] * 4),
    'sales': np.random.randint(50, 300, 16)
})

grouped = data.groupby(['region', 'product', 'quarter'])['sales'].sum()
# This returns a Series with a 3-level MultiIndex

# Method 4: set_index with multiple columns
df_multi = data.set_index(['region', 'quarter'])

# =============================================
# ACCESSING DATA IN MULTIINDEX
# =============================================

# Using the 2-level DataFrame from Method 1:
# df has index levels: region, quarter

# Select all data for 'East'
df.loc['East']
# Output:
#          sales  profit
# quarter
# Q1         100      20
# Q2         150      35
# Q3         200      50
# Q4         180      40

# Select specific (region, quarter)
df.loc[('East', 'Q1')]
# Output: sales=100, profit=20

# Select multiple outer-level values
df.loc[['East', 'West']]

# Slice within levels
df.loc['East':'West']  # all regions from East to West (inclusive)

# --- Cross-Section with .xs() ---
# Select all Q1 data across all regions
df.xs('Q1', level='quarter')
# Output:
#         sales  profit
# region
# East      100      20
# West      120      25

# .xs() with multiple levels
# df.xs(('East', 'Q1'), level=['region', 'quarter'])

# --- Using IndexSlice for complex slicing ---
idx = pd.IndexSlice

# All East data, Q1 to Q2
df.loc[idx['East', 'Q1':'Q2'], :]

# All regions, only Q1
df.loc[idx[:, 'Q1'], :]

# =============================================
# MANIPULATING MULTIINDEX
# =============================================

# Reset index (MultiIndex → regular columns)
df_flat = df.reset_index()
# Now region and quarter are regular columns

# Reset only one level
df.reset_index(level='quarter')

# Swap levels
df_swapped = df.swaplevel()
# Now quarter is level 0, region is level 1

# Sort the index (important for performance!)
df_sorted = df.sort_index()

# Rename index levels
df.index = df.index.set_names(['area', 'period'])

# Drop a level
df.droplevel('quarter')  # or droplevel(1) by position

# =============================================
# STACK AND UNSTACK
# =============================================

# Unstack: move an index level to columns
wide = df.unstack(level='quarter')
# Creates columns: (sales, Q1), (sales, Q2), ..., (profit, Q1), ...

# Stack: move column level to index (reverse of unstack)
long = wide.stack()

# Unstack specific level
data_grouped = data.groupby(['region', 'product'])['sales'].sum()
pivot_view = data_grouped.unstack('product')
# Output:
# product    A    B
# region
# East     ...  ...
# West     ...  ...

# =============================================
# AGGREGATION WITH MULTIINDEX
# =============================================

# Sum across a level
df.groupby(level='region').sum()      # total per region (across quarters)
df.groupby(level='quarter').mean()    # average per quarter (across regions)

# Or use sum with level parameter (deprecated in newer versions, use groupby)
# df.sum(level='region')  # deprecated

# MultiIndex columns
wide = df.unstack()
# Access: wide['sales']['Q1'] or wide[('sales', 'Q1')]
```

> **Pro Tip**: Always call `.sort_index()` after creating a MultiIndex. Unsorted MultiIndex causes performance warnings and can make `.loc[]` slicing return wrong results.

---

## Categorical Data Type

### What It Is
The `category` dtype stores data as integer codes that map to a fixed set of values, rather than repeating the full string for every row. Think of it like a color-by-number painting: instead of writing "Cadmium Red" a thousand times, you write "3" and have a legend that says "3 = Cadmium Red."

### Why It Matters
- **Memory savings**: A column with 1M rows but only 5 unique values stores 5 strings + 1M small integers instead of 1M full strings. Typical savings: 50-90%.
- **Performance**: GroupBy and sort on categorical columns are faster
- **Ordinal encoding**: Categories can have a defined order (S < M < L < XL)
- **Data validation**: Setting categories restricts what values a column can hold

### How It Works

```
Object dtype (string):          Category dtype:
┌─────────────────┐            ┌──────┐   Legend:
│ "Engineering"   │            │  0   │   0 = "Engineering"
│ "Sales"         │            │  1   │   1 = "Sales"
│ "Engineering"   │            │  0   │   2 = "Marketing"
│ "Marketing"     │            │  2   │
│ "Sales"         │            │  1   │
│ "Engineering"   │            │  0   │
└─────────────────┘            └──────┘
  6 × ~12 bytes                 6 × 1 byte + 3 strings
  = ~72 bytes                   = ~42 bytes (42% less)
  (scales MUCH worse at 1M rows)
```

### Code Examples

```python
import pandas as pd
import numpy as np

# =============================================
# CREATING CATEGORICAL COLUMNS
# =============================================

# From existing column
df = pd.DataFrame({
    'color': ['red', 'blue', 'green', 'red', 'blue'] * 200000,
    'size': ['S', 'M', 'L', 'XL', 'M'] * 200000,
    'price': np.random.uniform(10, 100, 1000000)
})

# Convert to category
df['color'] = df['color'].astype('category')

# Check memory savings
print(df['color'].memory_usage(deep=True))  # Much less than object

# --- Ordered categories (ordinal) ---
size_order = pd.CategoricalDtype(
    categories=['XS', 'S', 'M', 'L', 'XL', 'XXL'],
    ordered=True
)
df['size'] = df['size'].astype(size_order)

# Now comparisons work!
large_items = df[df['size'] > 'M']   # L and XL
df['size'].min()                       # 'S' (smallest in data)
df['size'].max()                       # 'XL'
df.sort_values('size')                 # sorts S, M, L, XL (not alphabetically!)

# =============================================
# WORKING WITH CATEGORIES
# =============================================

cat_col = df['color']

# Access category information
print(cat_col.cat.categories)     # Index(['blue', 'green', 'red'], dtype='object')
print(cat_col.cat.codes.head())   # integer codes: [2, 0, 1, 2, 0, ...]
print(cat_col.cat.ordered)        # False

# Add new categories (must do this before assigning new values!)
cat_col = cat_col.cat.add_categories(['yellow', 'purple'])

# Remove unused categories
cat_col = cat_col.cat.remove_unused_categories()

# Rename categories
df['color'] = df['color'].cat.rename_categories({
    'red': 'Red', 'blue': 'Blue', 'green': 'Green'
})

# Reorder categories
df['color'] = df['color'].cat.reorder_categories(['Blue', 'Green', 'Red'])

# =============================================
# MEMORY COMPARISON
# =============================================

n = 1_000_000
df_test = pd.DataFrame({
    'dept_object': np.random.choice(['Engineering', 'Sales', 'Marketing', 'HR', 'Finance'], n),
})
df_test['dept_category'] = df_test['dept_object'].astype('category')

obj_mem = df_test['dept_object'].memory_usage(deep=True) / 1e6
cat_mem = df_test['dept_category'].memory_usage(deep=True) / 1e6
print(f"Object: {obj_mem:.1f} MB")      # ~65 MB
print(f"Category: {cat_mem:.1f} MB")     # ~1 MB  (65x less!)

# =============================================
# PERFORMANCE COMPARISON
# =============================================

# GroupBy on categorical is faster
import time

# Object groupby
start = time.time()
df_test.groupby('dept_object').size()
obj_time = time.time() - start

# Category groupby
start = time.time()
df_test.groupby('dept_category').size()
cat_time = time.time() - start

print(f"Object GroupBy: {obj_time:.4f}s")
print(f"Category GroupBy: {cat_time:.4f}s")
# Category is typically 2-5x faster

# =============================================
# GOTCHA: ASSIGNING VALUES NOT IN CATEGORIES
# =============================================

df['color'] = df['color'].astype('category')
# df.loc[0, 'color'] = 'orange'  # ValueError! 'orange' not in categories

# Fix: add category first
df['color'] = df['color'].cat.add_categories('orange')
df.loc[0, 'color'] = 'orange'  # now works
```

### When to Use Categorical

| Condition | Use Category? | Reason |
|-----------|---------------|--------|
| < 50 unique values in 100K+ rows | Yes | Huge memory savings |
| Need ordinal comparison (S < M < L) | Yes | Only way to do this |
| High cardinality (e.g., user IDs) | No | No savings, adds overhead |
| Column used in groupby frequently | Yes | Faster groupby |
| Column values change frequently | No | Managing categories is tedious |

---

## eval() and query() — Expression Evaluation

### What It Is
`eval()` and `query()` let you write expressions as strings instead of verbose Pandas syntax. They use the `numexpr` library under the hood to evaluate expressions efficiently, avoiding creation of large intermediate arrays.

### Why It Matters
- **Memory efficient**: Avoids temporary arrays for complex expressions
- **Faster** on large DataFrames (1M+ rows) for arithmetic operations
- **Readable**: `df.query('age > 30 and city == "NYC"')` is cleaner than `df[(df['age'] > 30) & (df['city'] == 'NYC')]`
- **Column names with spaces** work easily: `` df.query('`Column Name` > 5') ``

### How It Works

Regular Pandas: `df['C'] = df['A'] + df['B']` creates a temporary array for `df['A'] + df['B']`, then assigns it. For 100M rows of float64, that's 800MB of temporary memory.

`eval()`: `df.eval('C = A + B')` computes element-by-element without creating a full temporary array, using chunks and `numexpr` for efficient CPU cache usage.

### Code Examples

```python
import pandas as pd
import numpy as np

n = 1_000_000
df = pd.DataFrame({
    'a': np.random.randn(n),
    'b': np.random.randn(n),
    'c': np.random.randn(n),
    'category': np.random.choice(['X', 'Y', 'Z'], n),
    'value': np.random.uniform(0, 100, n)
})

# =============================================
# pd.eval() — Top-level expression evaluation
# =============================================

# Arithmetic on DataFrames
result = pd.eval('df.a + df.b * df.c')

# =============================================
# df.eval() — DataFrame-level (column names directly)
# =============================================

# Create new columns
df.eval('d = a + b', inplace=True)
df.eval('ratio = a / (b + 1)', inplace=True)

# Complex expressions
df.eval('score = (a * 0.3 + b * 0.5 + c * 0.2) * 100', inplace=True)

# Multiple assignments (one per line)
df.eval('''
    sum_ab = a + b
    diff_ab = a - b
    product = a * b
''', inplace=True)

# Using local variables with @
threshold = 0.5
df.eval('above_threshold = value > @threshold', inplace=True)

# Boolean operations
df.eval('flag = (a > 0) & (b < 0)', inplace=True)

# =============================================
# df.query() — Filter rows with string expressions
# =============================================

# Simple filter
result = df.query('a > 0')

# Multiple conditions
result = df.query('a > 0 and b < 1')
result = df.query('a > 0 or b < -1')
result = df.query('not (a > 0)')

# Using variables
min_val = 0.5
result = df.query('value > @min_val')

# String matching
result = df.query('category == "X"')
result = df.query('category in ["X", "Y"]')
result = df.query('category != "Z"')

# Column names with spaces or special characters
df2 = df.rename(columns={'a': 'Column A'})
result = df2.query('`Column A` > 0')  # backtick quoting

# Chaining with other operations
result = (
    df.query('a > 0 and category == "X"')
    .sort_values('value', ascending=False)
    .head(100)
)

# =============================================
# PERFORMANCE: eval/query vs regular Pandas
# =============================================

# For LARGE DataFrames (> 100K rows), eval() can be faster
# because it avoids temporary arrays

import time

# Regular Pandas
start = time.time()
result1 = df['a'] + df['b'] * df['c'] - df['value'] / (df['a'] + 1)
regular_time = time.time() - start

# eval()
start = time.time()
result2 = df.eval('a + b * c - value / (a + 1)')
eval_time = time.time() - start

print(f"Regular: {regular_time:.4f}s")
print(f"eval(): {eval_time:.4f}s")
# eval() is often 2-5x faster for complex multi-column expressions
# AND uses less memory (no temporary arrays)
```

### eval() Limitations

| Supported | NOT Supported |
|-----------|---------------|
| Arithmetic: `+`, `-`, `*`, `/`, `**`, `%` | Custom functions: `np.log()`, `.str.contains()` |
| Comparison: `>`, `<`, `==`, `!=`, `>=`, `<=` | Method calls: `.mean()`, `.apply()` |
| Boolean: `and`, `or`, `not`, `&`, `\|`, `~` | Indexing: `df.iloc[...]` |
| Variable reference: `@variable` | Multi-line logic / if-else |
| `in` / `not in` for membership | Aggregations |

> **Pro Tip**: Use `eval()`/`query()` for large DataFrames (>100K rows) with arithmetic/comparison operations. For small DataFrames, regular Pandas is actually faster due to eval's parsing overhead.

---

## Chunked Processing for Large Files

### What It Is
Chunked processing reads and processes a file in small pieces instead of loading the entire thing into memory. Think of it like eating a cake one slice at a time instead of trying to swallow it whole.

### Why It Matters
- Files larger than available RAM cannot be loaded with `pd.read_csv()`
- A 10GB CSV needs ~30-50GB of RAM to load (Pandas overhead)
- Chunking lets you process arbitrarily large files with fixed memory
- Common in production ETL pipelines and data engineering

### How It Works

```
                     10GB CSV File
                    ┌─────────────┐
                    │             │
Chunk 1 (100K rows) │ ████████   │ → Process → Aggregate
                    │             │
Chunk 2 (100K rows) │ ████████   │ → Process → Aggregate
                    │             │
                    │    ...      │     ...        ...
                    │             │
Chunk N             │ ████████   │ → Process → Aggregate
                    └─────────────┘
                                         ↓
                                   Combine Results
                                   (final output fits in memory)
```

### Code Examples

```python
import pandas as pd
import numpy as np

# =============================================
# BASIC CHUNKED READING
# =============================================

# Read CSV in chunks of 100,000 rows
chunk_size = 100_000

# Method 1: Iterator pattern
chunks = pd.read_csv('large_file.csv', chunksize=chunk_size)

results = []
for chunk in chunks:
    # Process each chunk
    filtered = chunk[chunk['value'] > 100]
    results.append(filtered)

# Combine all results
final = pd.concat(results, ignore_index=True)

# Method 2: Aggregation pattern (memory-efficient)
total_sum = 0
total_count = 0

for chunk in pd.read_csv('large_file.csv', chunksize=chunk_size):
    total_sum += chunk['value'].sum()
    total_count += len(chunk)

overall_mean = total_sum / total_count

# Method 3: GroupBy aggregation across chunks
group_sums = {}
group_counts = {}

for chunk in pd.read_csv('large_file.csv', chunksize=chunk_size):
    for name, group in chunk.groupby('category'):
        group_sums[name] = group_sums.get(name, 0) + group['value'].sum()
        group_counts[name] = group_counts.get(name, 0) + len(group)

group_means = {k: group_sums[k] / group_counts[k] for k in group_sums}

# =============================================
# OPTIMIZED CSV READING (reduce what you load)
# =============================================

# Only read specific columns
df = pd.read_csv('large_file.csv', usecols=['id', 'name', 'value'])

# Specify dtypes upfront (avoid type inference overhead)
dtypes = {
    'id': 'int32',
    'name': 'category',      # categorical for low-cardinality strings
    'value': 'float32',      # float32 instead of float64
    'flag': 'bool'
}
df = pd.read_csv('large_file.csv', dtype=dtypes)

# Skip rows
df = pd.read_csv('large_file.csv', skiprows=range(1, 1000))  # skip first 999 data rows
df = pd.read_csv('large_file.csv', nrows=10000)                # read only first 10K rows

# Parse dates during read (faster than converting later)
df = pd.read_csv('large_file.csv', parse_dates=['date_column'])

# =============================================
# WRITING LARGE FILES IN CHUNKS
# =============================================

# Write chunks to CSV (append mode)
for i, chunk in enumerate(pd.read_csv('input.csv', chunksize=100_000)):
    processed = chunk[chunk['value'] > 0]  # some processing
    mode = 'w' if i == 0 else 'a'
    header = i == 0
    processed.to_csv('output.csv', mode=mode, header=header, index=False)

# =============================================
# USING SQL FOR LARGE DATA (alternative to chunking)
# =============================================

import sqlite3

# Load CSV into SQLite (can handle larger-than-RAM)
conn = sqlite3.connect('data.db')
for chunk in pd.read_csv('large_file.csv', chunksize=100_000):
    chunk.to_sql('my_table', conn, if_exists='append', index=False)

# Query with SQL (only loads result into memory)
result = pd.read_sql_query(
    "SELECT category, AVG(value) FROM my_table GROUP BY category",
    conn
)
conn.close()
```

> **Pro Tip**: For truly massive datasets (>10GB), consider using **Polars**, **DuckDB**, or **PySpark** instead of Pandas. They're designed for out-of-core processing and are 10-100x faster for large data operations.

---

## Memory Optimization

### What It Is
Memory optimization reduces the RAM footprint of your DataFrames. By choosing appropriate dtypes and representations, you can fit 3-10x more data into the same memory.

### Why It Matters
- A 2GB DataFrame with optimized dtypes might become 200MB
- More data in memory = fewer disk reads = faster processing
- Avoids `MemoryError` crashes
- Critical for production deployments with limited resources

### Code Examples

```python
import pandas as pd
import numpy as np

# =============================================
# DIAGNOSE MEMORY USAGE
# =============================================

df = pd.DataFrame({
    'id': np.arange(1_000_000),
    'category': np.random.choice(['A', 'B', 'C', 'D', 'E'], 1_000_000),
    'value': np.random.uniform(0, 100, 1_000_000),
    'flag': np.random.choice([0, 1], 1_000_000),
    'date': pd.date_range('2020-01-01', periods=1_000_000, freq='min')
})

# Check memory usage
print(df.info(memory_usage='deep'))

# Per-column memory
mem = df.memory_usage(deep=True) / 1e6  # in MB
print(mem)
# id          8.0 MB (int64)
# category   65.0 MB (object — strings!)
# value       8.0 MB (float64)
# flag        8.0 MB (int64 for 0/1 — wasteful!)
# date        8.0 MB (datetime64)

total_before = df.memory_usage(deep=True).sum() / 1e6
print(f"Total: {total_before:.1f} MB")

# =============================================
# OPTIMIZE EACH COLUMN
# =============================================

def optimize_dataframe(df):
    """Automatically optimize DataFrame dtypes to reduce memory."""
    df_optimized = df.copy()
    
    for col in df_optimized.columns:
        col_type = df_optimized[col].dtype
        
        if col_type == 'object':
            # Convert low-cardinality strings to category
            n_unique = df_optimized[col].nunique()
            n_total = len(df_optimized[col])
            if n_unique / n_total < 0.5:  # less than 50% unique
                df_optimized[col] = df_optimized[col].astype('category')
        
        elif col_type == 'int64':
            c_min = df_optimized[col].min()
            c_max = df_optimized[col].max()
            
            if c_min >= 0:  # unsigned
                if c_max <= 255:
                    df_optimized[col] = df_optimized[col].astype('uint8')
                elif c_max <= 65535:
                    df_optimized[col] = df_optimized[col].astype('uint16')
                elif c_max <= 4294967295:
                    df_optimized[col] = df_optimized[col].astype('uint32')
            else:  # signed
                if c_min >= -128 and c_max <= 127:
                    df_optimized[col] = df_optimized[col].astype('int8')
                elif c_min >= -32768 and c_max <= 32767:
                    df_optimized[col] = df_optimized[col].astype('int16')
                elif c_min >= -2147483648 and c_max <= 2147483647:
                    df_optimized[col] = df_optimized[col].astype('int32')
        
        elif col_type == 'float64':
            c_min = df_optimized[col].min()
            c_max = df_optimized[col].max()
            # Check if float32 precision is sufficient
            if c_min >= np.finfo('float32').min and c_max <= np.finfo('float32').max:
                df_optimized[col] = df_optimized[col].astype('float32')
    
    return df_optimized

df_opt = optimize_dataframe(df)

total_after = df_opt.memory_usage(deep=True).sum() / 1e6
print(f"Before: {total_before:.1f} MB")
print(f"After:  {total_after:.1f} MB")
print(f"Savings: {(1 - total_after/total_before)*100:.0f}%")
# Typical savings: 60-85%

# =============================================
# MANUAL OPTIMIZATION EXAMPLES
# =============================================

# 1. Integer downcasting
df['id'] = df['id'].astype('int32')               # 8 → 4 bytes
df['flag'] = df['flag'].astype('int8')             # 8 → 1 byte
# Or for 0/1: df['flag'] = df['flag'].astype(bool)  # 8 → 1 byte

# 2. Float downcasting
df['value'] = df['value'].astype('float32')        # 8 → 4 bytes

# 3. String → Category
df['category'] = df['category'].astype('category') # 65MB → 1MB

# 4. Sparse data (mostly zeros or mostly one value)
sparse_col = pd.array([0, 0, 0, 1, 0, 0, 0, 0, 0, 2], dtype=pd.SparseDtype('int64', 0))
# Only stores non-zero values + their positions

# =============================================
# MEMORY-EFFICIENT PATTERNS
# =============================================

# Delete columns you don't need (frees memory immediately)
df.drop(columns=['unnecessary_col1', 'unnecessary_col2'], inplace=True)

# Delete the DataFrame itself when done
import gc
del df
gc.collect()  # force garbage collection

# Read only needed columns from CSV
df = pd.read_csv('file.csv', usecols=['col1', 'col2', 'col3'])

# Specify dtypes during read (avoid memory spike from type inference)
df = pd.read_csv('file.csv', dtype={
    'id': 'int32',
    'name': 'category',
    'price': 'float32'
})
```

### Memory Optimization Checklist

| Check | Action | Typical Savings |
|-------|--------|-----------------|
| `int64` for small integers | Downcast to `int8`/`int16`/`int32` | 50-87% |
| `float64` where `float32` suffices | Downcast to `float32` | 50% |
| Repeated strings | Convert to `category` | 90-99% |
| Boolean stored as int | Convert to `bool` | 87% |
| Sparse data (mostly zeros) | Use `SparseDtype` | 80-99% |
| Unused columns | Drop them | 100% of that column |
| Loading from CSV | Specify `dtype=` and `usecols=` | Prevents spikes |

---

## Vectorization and Performance Patterns

### What It Is
Vectorization means operating on entire arrays at once instead of looping through elements one by one. NumPy and Pandas operations are vectorized — they're implemented in C and operate on memory-contiguous blocks, making them orders of magnitude faster than Python loops.

### Why It Matters
- The #1 performance rule in Pandas: **avoid Python loops**
- A vectorized operation on 1M rows: ~5ms. A Python loop: ~5000ms. That's 1000x.
- Understanding when to vectorize is what separates junior from senior data scientists
- Interview question: "How would you optimize this slow Pandas code?"

### Code Examples

```python
import pandas as pd
import numpy as np

n = 1_000_000
df = pd.DataFrame({
    'a': np.random.randn(n),
    'b': np.random.randn(n),
    'category': np.random.choice(['X', 'Y', 'Z'], n),
    'value': np.random.uniform(0, 100, n)
})

# =============================================
# THE PERFORMANCE PYRAMID (fastest → slowest)
# =============================================

# TIER 1: Vectorized NumPy/Pandas operations (FASTEST)
df['result'] = df['a'] + df['b'] * 2

# TIER 2: .map() with dict (fast for categorical transformations)
mapping = {'X': 1, 'Y': 2, 'Z': 3}
df['cat_code'] = df['category'].map(mapping)

# TIER 3: np.where / np.select (vectorized conditionals)
df['flag'] = np.where(df['a'] > 0, 'positive', 'negative')

conditions = [
    df['value'] < 25,
    df['value'] < 50,
    df['value'] < 75
]
choices = ['low', 'medium', 'high']
df['tier'] = np.select(conditions, choices, default='very_high')

# TIER 4: .apply() on Series (moderate — calls Python func per element)
df['a_rounded'] = df['a'].apply(lambda x: round(x, 2))
# But PREFER: df['a_rounded'] = df['a'].round(2)  ← vectorized!

# TIER 5: .apply(axis=1) on DataFrame (SLOW — iterates rows in Python)
# AVOID this pattern unless absolutely necessary
df['custom'] = df.apply(lambda row: row['a'] + row['b'], axis=1)
# PREFER: df['custom'] = df['a'] + df['b']

# TIER 6: iterrows / itertuples (SLOWEST — pure Python loop)
# NEVER DO THIS for large DataFrames

# =============================================
# REPLACING COMMON SLOW PATTERNS
# =============================================

# --- Pattern 1: Conditional assignment ---
# SLOW:
# for i, row in df.iterrows():
#     if row['a'] > 0:
#         df.at[i, 'sign'] = 'positive'
#     else:
#         df.at[i, 'sign'] = 'negative'

# FAST:
df['sign'] = np.where(df['a'] > 0, 'positive', 'negative')

# --- Pattern 2: Multiple conditions ---
# SLOW:
# def categorize(row):
#     if row['value'] > 90: return 'A'
#     elif row['value'] > 70: return 'B'
#     elif row['value'] > 50: return 'C'
#     else: return 'D'
# df['grade'] = df.apply(categorize, axis=1)

# FAST:
df['grade'] = np.select(
    [df['value'] > 90, df['value'] > 70, df['value'] > 50],
    ['A', 'B', 'C'],
    default='D'
)

# --- Pattern 3: String operations ---
# SLOW:
# df['upper'] = df['category'].apply(lambda x: x.lower())

# FAST:
df['lower'] = df['category'].str.lower()

# --- Pattern 4: Row-wise computation with multiple columns ---
# SLOW:
# df['distance'] = df.apply(
#     lambda r: np.sqrt(r['a']**2 + r['b']**2), axis=1
# )

# FAST:
df['distance'] = np.sqrt(df['a']**2 + df['b']**2)

# --- Pattern 5: Lookup / mapping ---
# SLOW:
# lookup = {'X': 100, 'Y': 200, 'Z': 300}
# df['mapped'] = df['category'].apply(lambda x: lookup[x])

# FAST:
df['mapped'] = df['category'].map({'X': 100, 'Y': 200, 'Z': 300})

# =============================================
# WHEN apply() IS ACTUALLY NEEDED
# =============================================

# Complex logic that truly can't be vectorized:
# - Calling external APIs per row
# - Complex stateful computations
# - When you need try/except per row
# Even then, consider: Cython, Numba, or converting to NumPy arrays first

# Using raw=True for speed (passes NumPy array instead of Series)
df['result'] = df[['a', 'b']].apply(np.sum, axis=1, raw=True)
# raw=True is 2-3x faster than raw=False for numeric operations

# =============================================
# NUMBA JIT COMPILATION (when vectorization isn't possible)
# =============================================

# from numba import jit
# 
# @jit(nopython=True)
# def custom_calc(a, b):
#     result = np.empty(len(a))
#     for i in range(len(a)):
#         if a[i] > 0:
#             result[i] = a[i] ** 2 + b[i]
#         else:
#             result[i] = a[i] - b[i] ** 2
#     return result
#
# df['result'] = custom_calc(df['a'].values, df['b'].values)
# 10-100x faster than apply, nearly as fast as vectorized
```

### Performance Benchmark (1M rows)

| Method | Time | Relative Speed |
|--------|------|---------------|
| Vectorized NumPy | ~5ms | 1x (baseline) |
| `pd.eval()` | ~8ms | 1.6x |
| `.map()` with dict | ~30ms | 6x |
| `np.where()` | ~10ms | 2x |
| `.apply()` on Series | ~200ms | 40x |
| `.apply(axis=1)` | ~5,000ms | 1,000x |
| `itertuples()` | ~3,000ms | 600x |
| `iterrows()` | ~30,000ms | 6,000x |

---

## Alternative File Formats

### What It Is
CSV is the most common data format, but it's terrible for performance. Binary formats like Parquet, Feather, and HDF5 are 5-50x faster to read/write and 2-10x smaller on disk.

### Why It Matters
- A 1GB CSV takes ~30 seconds to read. The same data in Parquet: ~2 seconds.
- Parquet preserves dtypes (no re-parsing), supports compression, and enables column pruning
- Industry standard in data engineering: data lakes use Parquet, not CSV
- Interview question: "What file format would you use for a large dataset and why?"

### Code Examples

```python
import pandas as pd
import numpy as np

# Create sample data
n = 1_000_000
df = pd.DataFrame({
    'id': np.arange(n),
    'name': np.random.choice(['Alice', 'Bob', 'Charlie', 'Diana'], n),
    'value': np.random.randn(n),
    'date': pd.date_range('2020-01-01', periods=n, freq='min'),
    'category': np.random.choice(['A', 'B', 'C'], n)
})

# =============================================
# PARQUET (recommended for most use cases)
# =============================================
# pip install pyarrow (or fastparquet)

# Write
df.to_parquet('data.parquet', engine='pyarrow', compression='snappy')

# Read (all columns)
df_parquet = pd.read_parquet('data.parquet')

# Read specific columns only (HUGE performance win)
df_subset = pd.read_parquet('data.parquet', columns=['id', 'value'])

# Filter during read (pushdown predicate — avoids loading unwanted rows)
# Requires partitioned dataset for best results
df_filtered = pd.read_parquet('data.parquet', 
                              filters=[('category', '==', 'A')])

# =============================================
# FEATHER (fastest for Pandas ↔ Pandas or Pandas ↔ R)
# =============================================

df.to_feather('data.feather')
df_feather = pd.read_feather('data.feather')
# Feather is FASTEST for read/write but no compression

# =============================================
# HDF5 (good for scientific computing, supports append)
# =============================================

# Write
df.to_hdf('data.h5', key='my_dataset', mode='w')

# Read
df_hdf = pd.read_hdf('data.h5', key='my_dataset')

# Append to existing file (unique to HDF5!)
# df.to_hdf('data.h5', key='my_dataset', mode='a', append=True)

# Query during read (very efficient for large files)
# df_subset = pd.read_hdf('data.h5', key='my_dataset', where='id > 500000')

# =============================================
# PICKLE (Python-specific, fastest for complex objects)
# =============================================

df.to_pickle('data.pkl')
df_pickle = pd.read_pickle('data.pkl')
# Warning: pickle files can execute arbitrary code — never unpickle untrusted files!

# =============================================
# CSV OPTIMIZATION TIPS
# =============================================

# If you must use CSV, optimize the read:
df = pd.read_csv('data.csv',
    dtype={'id': 'int32', 'name': 'category', 'value': 'float32'},
    usecols=['id', 'name', 'value'],     # only needed columns
    parse_dates=['date'],                 # parse dates
    engine='c',                           # C parser (default, fastest)
    low_memory=False                      # consistent dtype inference
)
```

### File Format Comparison (1M rows, 5 columns)

| Format | Write Time | Read Time | File Size | Preserves dtypes | Column Pruning |
|--------|-----------|-----------|-----------|-------------------|----------------|
| CSV | ~3s | ~3s | ~100 MB | No | No |
| Parquet (snappy) | ~0.5s | ~0.3s | ~15 MB | Yes | Yes |
| Feather | ~0.2s | ~0.1s | ~40 MB | Yes | Yes |
| HDF5 | ~0.5s | ~0.4s | ~45 MB | Yes | Partial |
| Pickle | ~0.3s | ~0.2s | ~50 MB | Yes | No |

> **Pro Tip**: Use **Parquet** as your default format for data at rest. It's compressed, preserves types, supports column selection, and is the industry standard for data lakes (Spark, AWS, Snowflake all use it natively).

---

## Method Chaining and Pipe

### What It Is
Method chaining writes multiple DataFrame operations as a single flowing expression, each step transforming the result of the previous step. The `.pipe()` method lets you include custom functions in the chain. Think of it like a factory assembly line — each station transforms the product.

### Why It Matters
- **Readability**: one clear pipeline instead of 10 intermediate variables
- **Debugging**: easy to comment out or add steps
- **No intermediate variables**: less memory, less namespace pollution
- **Functional style**: immutable-like transformations are easier to reason about

### Code Examples

```python
import pandas as pd
import numpy as np

# Sample raw data
df = pd.DataFrame({
    'Name': ['  alice smith  ', 'BOB JONES', '  Charlie Brown'],
    'Age': [25, -1, 35],
    'Salary': ['$70,000', '$85,000', '$120,000'],
    'Department': ['engineering', 'SALES', 'Engineering'],
    'Hire Date': ['2020-01-15', '2019-06-01', '2018-03-20']
})

# =============================================
# WITHOUT CHAINING (procedural style)
# =============================================

# df1 = df.copy()
# df1['Name'] = df1['Name'].str.strip().str.title()
# df1['Department'] = df1['Department'].str.lower()
# df1['Salary'] = df1['Salary'].str.replace('[$,]', '', regex=True).astype(float)
# df1['Hire Date'] = pd.to_datetime(df1['Hire Date'])
# df1 = df1[df1['Age'] > 0]
# df1 = df1.sort_values('Salary', ascending=False)
# df1 = df1.reset_index(drop=True)

# =============================================
# WITH CHAINING (functional style — preferred!)
# =============================================

result = (
    df
    .assign(
        Name=lambda x: x['Name'].str.strip().str.title(),
        Department=lambda x: x['Department'].str.lower(),
        Salary=lambda x: x['Salary'].str.replace('[$,]', '', regex=True).astype(float),
        Hire_Date=lambda x: pd.to_datetime(x['Hire Date'])
    )
    .drop(columns=['Hire Date'])
    .query('Age > 0')
    .sort_values('Salary', ascending=False)
    .reset_index(drop=True)
)

# =============================================
# USING .pipe() FOR CUSTOM FUNCTIONS
# =============================================

def add_tenure_column(df, reference_date='2024-06-01'):
    """Add years of tenure based on hire date."""
    ref = pd.Timestamp(reference_date)
    df = df.assign(
        Tenure_Years=lambda x: (ref - x['Hire_Date']).dt.days / 365.25
    )
    return df

def add_salary_band(df, col='Salary'):
    """Classify salary into bands."""
    df = df.assign(
        Salary_Band=lambda x: pd.cut(
            x[col],
            bins=[0, 60000, 90000, 150000, np.inf],
            labels=['Junior', 'Mid', 'Senior', 'Executive']
        )
    )
    return df

def log_shape(df, step_name=''):
    """Debugging helper — print shape and return df unchanged."""
    print(f"[{step_name}] Shape: {df.shape}")
    return df

# Full pipeline with pipe
result = (
    df
    .assign(
        Name=lambda x: x['Name'].str.strip().str.title(),
        Department=lambda x: x['Department'].str.lower(),
        Salary=lambda x: x['Salary'].str.replace('[$,]', '', regex=True).astype(float),
        Hire_Date=lambda x: pd.to_datetime(x['Hire Date'])
    )
    .drop(columns=['Hire Date'])
    .query('Age > 0')
    .pipe(log_shape, 'After filtering')       # debugging step
    .pipe(add_tenure_column, '2024-06-01')    # custom function
    .pipe(add_salary_band)                     # another custom function
    .sort_values('Salary', ascending=False)
    .reset_index(drop=True)
)

# =============================================
# KEY CHAINING METHODS
# =============================================

# .assign() — add or modify columns (returns new DataFrame)
df.assign(new_col=lambda x: x['a'] + x['b'])

# .query() — filter rows
df.query('age > 30')

# .rename() — rename columns
df.rename(columns={'old_name': 'new_name'})

# .drop() — remove columns/rows
df.drop(columns=['unwanted'])

# .sort_values() — sort
df.sort_values('col', ascending=False)

# .reset_index() — reset index
df.reset_index(drop=True)

# .pipe() — apply any function
df.pipe(my_custom_function, arg1, arg2)

# .astype() — change dtypes
df.astype({'col': 'int32'})
```

> **Pro Tip**: Use `.assign()` instead of `df['col'] = ...` in chains. `.assign()` returns a new DataFrame (chainable), while direct assignment returns `None` and modifies in place (breaks the chain).

---

## Common Mistakes

### 1. Not Sorting MultiIndex Before Slicing
```python
# WRONG — may get incorrect results or PerformanceWarning
df.loc[('East', 'Q1'):('West', 'Q2')]

# RIGHT — sort first
df = df.sort_index()
df.loc[('East', 'Q1'):('West', 'Q2')]
```

### 2. Assuming category Saves Memory for High-Cardinality Columns
```python
# WRONG — converting user_ids (1M unique values) to category
df['user_id'] = df['user_id'].astype('category')
# Actually INCREASES memory! (stores the full mapping table)

# Category only helps when: unique values << total rows
# Rule of thumb: < 50% unique values
```

### 3. Using eval/query for Small DataFrames
```python
# WRONG — eval() overhead > benefit for small DataFrames
small_df.eval('c = a + b')  # slower than small_df['c'] = small_df['a'] + small_df['b']

# eval/query shine only for DataFrames with > 100K rows
```

### 4. Concatenating Inside a Loop
```python
# WRONG — O(n²) memory and time
result = pd.DataFrame()
for chunk in chunks:
    result = pd.concat([result, processed_chunk])

# RIGHT — O(n) memory and time
processed = [process(chunk) for chunk in chunks]
result = pd.concat(processed, ignore_index=True)
```

### 5. Not Specifying dtypes When Reading CSV
```python
# WRONG — Pandas infers types by reading ALL data first, wasting time and memory
df = pd.read_csv('big_file.csv')

# RIGHT — specify upfront
df = pd.read_csv('big_file.csv', dtype={
    'id': 'int32', 'name': 'category', 'price': 'float32'
})
```

### 6. Using pickle for Untrusted Data
```python
# DANGEROUS — pickle can execute arbitrary code!
df = pd.read_pickle('untrusted_file.pkl')  # security risk!

# SAFE — use Parquet for data exchange
df = pd.read_parquet('data.parquet')
```

---

## Interview Questions

### Q1: How would you reduce a 5GB DataFrame to fit in 2GB of RAM?
**Answer**:
1. **Downcast numerics**: `int64` → `int8/16/32`, `float64` → `float32` (saves 50-87%)
2. **Convert low-cardinality strings to category** (saves 90%+)
3. **Drop unnecessary columns** with `usecols` during read
4. **Use Parquet** instead of CSV (preserves dtypes, no re-inference overhead)
5. **Process in chunks** with `pd.read_csv(chunksize=...)` if it still doesn't fit
6. If all else fails, use **Polars** or **DuckDB** for out-of-core processing

### Q2: What is a MultiIndex and when would you use it?
**Answer**: A MultiIndex (hierarchical index) has multiple levels, like a composite key. Use it when data has natural hierarchies (region → city → store) or when GroupBy produces multi-level results. It enables `.xs()` for cross-section selection and `.unstack()` for reshaping. Always sort it for correct slicing behavior.

### Q3: Compare Parquet vs CSV for storing large datasets.
**Answer**:
| Feature | CSV | Parquet |
|---------|-----|---------|
| Read speed | Slow (text parsing) | 10-50x faster (binary) |
| File size | Large | 5-10x smaller (compressed) |
| Type preservation | No (everything is text) | Yes |
| Column selection | Must read all columns | Can read specific columns |
| Ecosystem | Universal | Big data standard |
| Human readable | Yes | No |

Use Parquet for production/storage, CSV only for data exchange with non-technical users.

### Q4: What's the fastest way to apply a conditional transformation to a column?
**Answer**: Use `np.where()` for binary conditions or `np.select()` for multiple conditions. Both are fully vectorized. Avoid `.apply(axis=1)` which is 1000x slower.
```python
df['label'] = np.where(df['score'] > 50, 'pass', 'fail')
df['grade'] = np.select(
    [df['score'] > 90, df['score'] > 70, df['score'] > 50],
    ['A', 'B', 'C'], default='F'
)
```

### Q5: How does the `category` dtype improve performance?
**Answer**: Category stores data as small integer codes + a lookup table of unique values. This reduces memory by 90%+ for low-cardinality string columns (e.g., 5 unique values in 1M rows). It also speeds up `groupby()`, `sort_values()`, and `merge()` because operations work on integers instead of strings. It additionally supports ordered categories for ordinal comparisons.

### Q6: Explain method chaining in Pandas. Why is it preferred?
**Answer**: Method chaining writes transformations as a single expression: `df.query(...).assign(...).sort_values(...)`. Benefits:
- No intermediate variables cluttering the namespace
- Each step is a pure transformation — easy to read, debug, reorder
- `.pipe()` integrates custom functions into the chain
- Use `.assign()` instead of direct column assignment to stay chainable

### Q7: When would you use `iterrows()` vs vectorized operations?
**Answer**: Almost never use `iterrows()`. It's 1000-6000x slower than vectorized operations. The only valid use cases are:
- Calling an external API per row
- Complex stateful logic that depends on previous iterations
- Debugging (inspecting a few rows)
Even then, consider `itertuples()` (3-5x faster than `iterrows()`) or `apply()` first.

---

## Quick Reference

| Operation | Code | Notes |
|-----------|------|-------|
| Create MultiIndex | `pd.MultiIndex.from_product([a, b])` | Or `.from_tuples()` |
| Select level | `df.xs('Q1', level='quarter')` | Cross-section |
| Reset MultiIndex | `df.reset_index()` | MultiIndex → columns |
| Sort MultiIndex | `df.sort_index()` | Always do this! |
| Stack/Unstack | `df.unstack('level')` / `df.stack()` | Reshape index ↔ columns |
| To category | `df['col'].astype('category')` | Low-cardinality strings |
| Ordered category | `pd.CategoricalDtype([...], ordered=True)` | Enables `<`, `>` |
| Category codes | `df['col'].cat.codes` | Integer encoding |
| eval() | `df.eval('c = a + b')` | Fast arithmetic |
| query() | `df.query('a > 0 and b < 1')` | Clean filtering |
| Read chunks | `pd.read_csv(f, chunksize=N)` | Iterator of DataFrames |
| Memory usage | `df.memory_usage(deep=True)` | Per-column bytes |
| Downcast int | `pd.to_numeric(s, downcast='integer')` | Auto smallest int |
| Read Parquet | `pd.read_parquet(f, columns=[...])` | Column pruning |
| Write Parquet | `df.to_parquet(f, compression='snappy')` | Standard format |
| Method chain | `df.assign(...).query(...).pipe(fn)` | Functional style |
| np.where | `np.where(cond, true_val, false_val)` | Vectorized if-else |
| np.select | `np.select(conditions, choices, default)` | Vectorized multi-if |

---

*Previous: [07-Pandas-Time-Series](07-Pandas-Time-Series.md) | Back to: [00-INDEX](00-INDEX.md)*
