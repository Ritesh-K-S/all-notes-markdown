# Chapter 05: Pandas Data Manipulation

## Table of Contents
- [Filtering Data](#filtering-data)
- [Sorting Data](#sorting-data)
- [GroupBy Operations](#groupby-operations)
- [Merge and Join](#merge-and-join)
- [Concat and Append](#concat-and-append)
- [Pivot Tables and Reshaping](#pivot-tables-and-reshaping)
- [Apply, Map, and Transform](#apply-map-and-transform)
- [Window Functions](#window-functions)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Filtering Data

### What It Is
Filtering is selecting specific rows from a DataFrame based on conditions — like using a sieve to keep only the data you care about. Think of it as asking your data a yes/no question and keeping only the "yes" rows.

### Why It Matters
- Every data analysis starts with filtering: "Show me customers from New York" or "Find transactions above $1000"
- SQL's `WHERE` clause equivalent in Pandas
- Used in feature engineering, data cleaning, and exploratory analysis
- Critical for building training sets in ML (e.g., filtering out outliers)

### How It Works

The core mechanism is **Boolean indexing**:
1. You create a condition that produces a Series of `True`/`False` values (one per row)
2. You pass this Boolean Series into `df[...]`
3. Pandas keeps only rows where the value is `True`

```
DataFrame:        Boolean Mask:       Result:
┌─────┬─────┐    ┌───────┐          ┌─────┬─────┐
│ A   │ B   │    │ True  │    →     │ 10  │ 'x' │
│ 10  │ 'x' │    │ False │          │ 30  │ 'z' │
│ 20  │ 'y' │    │ True  │          └─────┴─────┘
│ 30  │ 'z' │    └───────┘
└─────┴─────┘
```

### Code Examples

```python
import pandas as pd
import numpy as np

# Create sample DataFrame
df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie', 'Diana', 'Eve'],
    'age': [25, 30, 35, 28, 42],
    'city': ['NYC', 'LA', 'NYC', 'Chicago', 'LA'],
    'salary': [70000, 85000, 120000, 65000, 95000]
})

# --- Basic Filtering ---

# Single condition
adults_over_30 = df[df['age'] > 30]
# Output:
#       name  age   city  salary
# 2  Charlie   35    NYC  120000
# 4      Eve   42     LA   95000

# Multiple conditions with & (AND) — MUST use parentheses!
nyc_high_earners = df[(df['city'] == 'NYC') & (df['salary'] > 80000)]
# Output:
#       name  age city  salary
# 2  Charlie   35  NYC  120000

# Multiple conditions with | (OR)
nyc_or_la = df[(df['city'] == 'NYC') | (df['city'] == 'LA')]

# Negation with ~
not_nyc = df[~(df['city'] == 'NYC')]

# --- Using .isin() for multiple values ---
cities_of_interest = ['NYC', 'LA']
filtered = df[df['city'].isin(cities_of_interest)]

# --- Using .between() for ranges ---
mid_salary = df[df['salary'].between(70000, 100000)]  # inclusive by default

# --- String filtering ---
df_text = pd.DataFrame({'product': ['Apple iPhone', 'Samsung Galaxy', 'Apple Watch', 'Google Pixel']})
apple_products = df_text[df_text['product'].str.contains('Apple')]
starts_with_s = df_text[df_text['product'].str.startswith('Samsung')]

# --- Using .query() method (cleaner syntax) ---
result = df.query('age > 30 and city == "NYC"')
# Equivalent to: df[(df['age'] > 30) & (df['city'] == 'NYC')]

# Using variables in query with @
min_age = 30
result = df.query('age > @min_age')

# --- .loc[] for label-based filtering with column selection ---
result = df.loc[df['age'] > 30, ['name', 'salary']]
# Output:
#       name  salary
# 2  Charlie  120000
# 4      Eve   95000

# --- np.where() for conditional column creation ---
df['category'] = np.where(df['salary'] > 80000, 'high', 'low')
```

> **Pro Tip**: `.query()` is faster for large DataFrames because it uses `numexpr` under the hood, avoiding creation of intermediate Boolean arrays. Use it when you have millions of rows.

---

## Sorting Data

### What It Is
Sorting arranges your rows (or columns) in a specific order — ascending or descending by one or more columns. Like organizing a deck of cards by suit then by number.

### Why It Matters
- Finding top/bottom N values (top customers, worst performers)
- Required before certain operations (merge_asof needs sorted data)
- Debugging: sorted data is easier to visually inspect
- Presentation: reports need sorted data

### How It Works
Pandas uses **Timsort** (hybrid merge sort + insertion sort) internally, which is $O(n \log n)$ on average. For multiple columns, it sorts by the first column, then breaks ties with the second, and so on.

### Code Examples

```python
import pandas as pd

df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie', 'Diana'],
    'department': ['Engineering', 'Sales', 'Engineering', 'Sales'],
    'salary': [90000, 75000, 120000, 85000],
    'experience': [5, 8, 12, 3]
})

# --- Sort by single column ---
by_salary = df.sort_values('salary')                    # ascending (default)
by_salary_desc = df.sort_values('salary', ascending=False)  # descending

# --- Sort by multiple columns ---
# First by department (A-Z), then by salary (high to low) within each dept
sorted_df = df.sort_values(
    by=['department', 'salary'],
    ascending=[True, False]  # different direction per column
)
# Output:
#       name   department  salary  experience
# 2  Charlie  Engineering  120000          12
# 0    Alice  Engineering   90000           5
# 3    Diana        Sales   85000           3
# 1      Bob        Sales   75000           8

# --- Sort by index ---
df_shuffled = df.sample(frac=1)  # shuffle rows
df_restored = df_shuffled.sort_index()  # restore original order

# --- nlargest / nsmallest (optimized for top/bottom N) ---
top_2_salary = df.nlargest(2, 'salary')
# Much faster than sort_values().head(2) for large DataFrames
# Uses partial sort: O(n) instead of O(n log n)

bottom_2_exp = df.nsmallest(2, 'experience')

# --- Sorting with NaN handling ---
df_with_nan = pd.DataFrame({'value': [3, np.nan, 1, np.nan, 2]})
df_with_nan.sort_values('value', na_position='first')   # NaN at top
df_with_nan.sort_values('value', na_position='last')    # NaN at bottom (default)

# --- Stable sort for preserving original order of ties ---
df.sort_values('department', kind='mergesort')  # stable sort

# --- Reset index after sorting ---
sorted_clean = df.sort_values('salary').reset_index(drop=True)
```

> **Pro Tip**: Use `nlargest()`/`nsmallest()` when you need only top/bottom N rows. For N << len(df), it's significantly faster than full sorting because it uses a heap-based partial sort internally.

---

## GroupBy Operations

### What It Is
GroupBy is **split-apply-combine**: you split your data into groups based on some criteria, apply a function to each group independently, then combine results back together. Think of it like sorting students into classrooms, having each classroom take a test, then collecting all test scores.

### Why It Matters
- Foundation of SQL-like aggregation in Python
- Used in 90%+ of data analysis tasks: "average salary by department", "total sales by region"
- Powers feature engineering in ML: group-level statistics become features
- Equivalent to SQL's `GROUP BY` + aggregate functions

### How It Works

```
         SPLIT                    APPLY                 COMBINE
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  Full DataFrame  │    │ Group A → func() │    │  Result with     │
│                  │ →  │ Group B → func() │ →  │  one row per     │
│  A, B, A, B, A  │    │                  │    │  group           │
└──────────────────┘    └──────────────────┘    └──────────────────┘
```

Internally, Pandas:
1. Computes a mapping of row → group label
2. Creates a `GroupBy` object (lazy — no computation yet)
3. When you call `.mean()`, `.sum()`, etc., it iterates through groups and applies the function

### Code Examples

```python
import pandas as pd
import numpy as np

# Sample data: sales records
df = pd.DataFrame({
    'region': ['East', 'West', 'East', 'West', 'East', 'South'],
    'product': ['A', 'A', 'B', 'B', 'A', 'A'],
    'revenue': [100, 150, 200, 80, 120, 90],
    'quantity': [10, 15, 8, 5, 12, 9],
    'date': pd.to_datetime(['2024-01-01', '2024-01-01', '2024-01-02', 
                            '2024-01-02', '2024-01-03', '2024-01-03'])
})

# --- Basic GroupBy + Aggregation ---
# Single column, single aggregation
avg_by_region = df.groupby('region')['revenue'].mean()
# Output:
# region
# East     140.0
# South     90.0
# West     115.0
# Name: revenue, dtype: float64

# --- Multiple aggregations on same column ---
revenue_stats = df.groupby('region')['revenue'].agg(['mean', 'sum', 'count', 'std'])

# --- Different aggregations for different columns ---
result = df.groupby('region').agg(
    total_revenue=('revenue', 'sum'),
    avg_quantity=('quantity', 'mean'),
    num_orders=('revenue', 'count'),
    max_revenue=('revenue', 'max')
)
# Output:
#        total_revenue  avg_quantity  num_orders  max_revenue
# region
# East             420          10.0           3          200
# South             90           9.0           1           90
# West             230          10.0           2          150

# --- GroupBy with multiple columns ---
multi_group = df.groupby(['region', 'product'])['revenue'].sum()
# Returns a Series with MultiIndex
# Output:
# region  product
# East    A          220
#         B          200
# South   A           90
# West    A          150
#         B           80

# Unstack to pivot
multi_group.unstack(fill_value=0)

# --- Custom aggregation functions ---
def revenue_range(x):
    return x.max() - x.min()

df.groupby('region')['revenue'].agg(revenue_range)

# Using lambda
df.groupby('region')['revenue'].agg(lambda x: x.quantile(0.75))

# --- .transform() — return same-size output (broadcast back) ---
# Add group mean as a new column (useful for feature engineering)
df['region_avg_revenue'] = df.groupby('region')['revenue'].transform('mean')
# Each row gets its group's mean — DataFrame keeps original shape!

# Normalize within groups (z-score per group)
df['revenue_zscore'] = df.groupby('region')['revenue'].transform(
    lambda x: (x - x.mean()) / x.std()
)

# --- .filter() — keep/discard entire groups ---
# Keep only regions with more than 1 order
big_regions = df.groupby('region').filter(lambda x: len(x) > 1)

# --- .apply() — most flexible (any function) ---
# Custom function that takes a group DataFrame, returns anything
def top_product(group):
    return group.nlargest(1, 'revenue')

df.groupby('region').apply(top_product, include_groups=False)

# --- Iterating over groups ---
for name, group in df.groupby('region'):
    print(f"Region: {name}, Count: {len(group)}")

# --- Named aggregation (clean and explicit) ---
result = df.groupby('region').agg(
    total_rev=pd.NamedAgg(column='revenue', aggfunc='sum'),
    avg_qty=pd.NamedAgg(column='quantity', aggfunc='mean')
)

# --- Percentile rank within group ---
df['revenue_rank'] = df.groupby('region')['revenue'].rank(pct=True)

# --- Cumulative operations within groups ---
df['cumulative_revenue'] = df.groupby('region')['revenue'].cumsum()
df['expanding_mean'] = df.groupby('region')['revenue'].expanding().mean().reset_index(level=0, drop=True)
```

> **Pro Tip**: Never loop over groups manually with `for name, group in groupby`. Use `.transform()`, `.apply()`, or `.agg()` — they're vectorized and 10-100x faster.

### GroupBy Performance Comparison

| Method | Use Case | Returns |
|--------|----------|---------|
| `.agg()` | Reduce each group to single value | Smaller DataFrame |
| `.transform()` | Compute group stat, broadcast back | Same-size DataFrame |
| `.filter()` | Keep/drop entire groups | Subset of original |
| `.apply()` | Anything (flexible but slower) | Depends on function |

---

## Merge and Join

### What It Is
Merging combines two DataFrames side-by-side based on shared column values (like matching puzzle pieces). It's exactly like SQL's `JOIN` operation. Think of it as a VLOOKUP on steroids.

### Why It Matters
- Real data lives in multiple tables (customers, orders, products)
- Data enrichment: adding features from different sources
- ETL pipelines always involve joining datasets
- Interview favorite: "What's the difference between inner, left, right, and outer join?"

### How It Works

```
LEFT DataFrame          RIGHT DataFrame
┌─────┬───────┐        ┌─────┬──────┐
│ id  │ name  │        │ id  │ dept │
├─────┼───────┤        ├─────┼──────┤
│  1  │ Alice │        │  1  │ Eng  │
│  2  │ Bob   │        │  3  │ HR   │
│  3  │ Carol │        │  4  │ Mkt  │
└─────┴───────┘        └─────┴──────┘

INNER JOIN (id):        LEFT JOIN (id):
┌─────┬───────┬──────┐ ┌─────┬───────┬──────┐
│ id  │ name  │ dept │ │ id  │ name  │ dept │
├─────┼───────┼──────┤ ├─────┼───────┼──────┤
│  1  │ Alice │ Eng  │ │  1  │ Alice │ Eng  │
│  3  │ Carol │ HR   │ │  2  │ Bob   │ NaN  │
└─────┴───────┴──────┘ │  3  │ Carol │ HR   │
                        └─────┴───────┴──────┘
```

Join types:
- **Inner**: Only rows with matching keys in BOTH tables
- **Left**: All rows from left + matching from right (NaN if no match)
- **Right**: All rows from right + matching from left
- **Outer**: ALL rows from both (NaN where no match)
- **Cross**: Cartesian product (every row paired with every other row)

### Code Examples

```python
import pandas as pd

# Create sample tables
customers = pd.DataFrame({
    'customer_id': [1, 2, 3, 4],
    'name': ['Alice', 'Bob', 'Charlie', 'Diana'],
    'city': ['NYC', 'LA', 'Chicago', 'NYC']
})

orders = pd.DataFrame({
    'order_id': [101, 102, 103, 104, 105],
    'customer_id': [1, 2, 2, 5, 3],
    'amount': [250, 150, 300, 75, 500]
})

products = pd.DataFrame({
    'order_id': [101, 102, 103, 104, 105],
    'product': ['Laptop', 'Phone', 'Tablet', 'Mouse', 'Monitor']
})

# --- Inner Join (default) ---
inner = pd.merge(customers, orders, on='customer_id', how='inner')
# Only customers 1, 2, 3 appear (they have orders)
# Customer 4 (no orders) and order from customer 5 (not in customers) excluded

# --- Left Join ---
left = pd.merge(customers, orders, on='customer_id', how='left')
# All customers appear; Diana gets NaN for order columns

# --- Right Join ---
right = pd.merge(customers, orders, on='customer_id', how='right')
# All orders appear; order from customer_id=5 gets NaN for name, city

# --- Outer Join ---
outer = pd.merge(customers, orders, on='customer_id', how='outer')
# Everything from both sides; NaN fills the gaps

# --- Merging on different column names ---
df_a = pd.DataFrame({'emp_id': [1, 2], 'name': ['Alice', 'Bob']})
df_b = pd.DataFrame({'employee_number': [1, 2], 'salary': [90000, 75000]})

merged = pd.merge(df_a, df_b, left_on='emp_id', right_on='employee_number')

# --- Merging on multiple keys ---
sales = pd.DataFrame({
    'store': ['A', 'A', 'B', 'B'],
    'product': ['X', 'Y', 'X', 'Y'],
    'revenue': [100, 200, 150, 250]
})

targets = pd.DataFrame({
    'store': ['A', 'A', 'B', 'B'],
    'product': ['X', 'Y', 'X', 'Y'],
    'target': [120, 180, 160, 240]
})

merged = pd.merge(sales, targets, on=['store', 'product'])

# --- Handling duplicate column names with suffixes ---
df1 = pd.DataFrame({'id': [1, 2], 'value': [10, 20]})
df2 = pd.DataFrame({'id': [1, 2], 'value': [100, 200]})

merged = pd.merge(df1, df2, on='id', suffixes=('_left', '_right'))
# Columns: id, value_left, value_right

# --- Chaining multiple merges ---
full_data = (
    orders
    .merge(customers, on='customer_id', how='left')
    .merge(products, on='order_id', how='left')
)

# --- Merge with indicator to see where rows came from ---
result = pd.merge(customers, orders, on='customer_id', how='outer', indicator=True)
# _merge column: 'left_only', 'right_only', or 'both'

# Find customers with no orders:
no_orders = result[result['_merge'] == 'left_only']

# --- Index-based join (DataFrame.join) ---
# .join() uses index by default
df_a = pd.DataFrame({'name': ['Alice', 'Bob']}, index=[1, 2])
df_b = pd.DataFrame({'dept': ['Eng', 'Sales']}, index=[1, 2])
joined = df_a.join(df_b)  # joins on index

# --- merge_asof for approximate matching (time series) ---
trades = pd.DataFrame({
    'time': pd.to_datetime(['10:01', '10:03', '10:05']),
    'price': [100, 102, 101]
})
quotes = pd.DataFrame({
    'time': pd.to_datetime(['10:00', '10:02', '10:04']),
    'bid': [99, 101, 100]
})

# Match each trade with the most recent quote
result = pd.merge_asof(trades.sort_values('time'), 
                       quotes.sort_values('time'), 
                       on='time')
```

> **Pro Tip**: Always check for unexpected row multiplication after merging. If your merge produces more rows than expected, you likely have duplicate keys. Use `df.duplicated(subset=['key_col']).sum()` to check before merging.

### Merge Complexity

| Join Type | Time Complexity | Notes |
|-----------|----------------|-------|
| Hash Join (default) | $O(n + m)$ | Used for equality joins |
| Sort-Merge Join | $O(n\log n + m\log m)$ | Used by merge_asof |
| Cross Join | $O(n \times m)$ | Avoid on large data |

---

## Concat and Append

### What It Is
Concatenation stacks DataFrames either vertically (adding more rows) or horizontally (adding more columns). Think of it like stacking papers on top of each other (axis=0) or gluing them side by side (axis=1).

### Why It Matters
- Combining monthly data files into one dataset
- Stacking results from multiple API calls
- Building datasets from multiple sources with the same schema
- Reassembling data after parallel processing

### How It Works

```
axis=0 (vertical stack):          axis=1 (horizontal stack):
┌───────┐                         ┌───────┬───────┐
│  DF1  │                         │       │       │
├───────┤   →   ┌───────┐        │  DF1  │  DF2  │
│  DF2  │       │ Result│        │       │       │
└───────┘       └───────┘        └───────┴───────┘
```

### Code Examples

```python
import pandas as pd
import numpy as np

# --- Vertical concatenation (stacking rows) ---
jan = pd.DataFrame({'product': ['A', 'B'], 'sales': [100, 200]})
feb = pd.DataFrame({'product': ['A', 'B'], 'sales': [110, 190]})
mar = pd.DataFrame({'product': ['A', 'B', 'C'], 'sales': [120, 180, 50]})

# Stack all months
all_months = pd.concat([jan, feb, mar], ignore_index=True)
# ignore_index=True resets the index (0, 1, 2, 3, ...)

# Add identifier for source
all_months = pd.concat([jan, feb, mar], keys=['Jan', 'Feb', 'Mar'])
# Creates a MultiIndex: (Jan, 0), (Jan, 1), (Feb, 0), ...

# --- Handling mismatched columns ---
df1 = pd.DataFrame({'A': [1, 2], 'B': [3, 4]})
df2 = pd.DataFrame({'A': [5, 6], 'C': [7, 8]})

# join='outer' (default): keeps all columns, fills NaN
result_outer = pd.concat([df1, df2], ignore_index=True)
#    A    B    C
# 0  1  3.0  NaN
# 1  2  4.0  NaN
# 2  5  NaN  7.0
# 3  6  NaN  8.0

# join='inner': keeps only shared columns
result_inner = pd.concat([df1, df2], join='inner', ignore_index=True)
#    A
# 0  1
# 1  2
# 2  5
# 3  6

# --- Horizontal concatenation (adding columns) ---
names = pd.DataFrame({'name': ['Alice', 'Bob']})
scores = pd.DataFrame({'score': [95, 87]})
combined = pd.concat([names, scores], axis=1)

# --- Efficient concatenation pattern (avoid in-loop concat!) ---

# BAD - O(n²) complexity, creates new DataFrame each iteration
# result = pd.DataFrame()
# for file in files:
#     df = pd.read_csv(file)
#     result = pd.concat([result, df])  # SLOW!

# GOOD - O(n) complexity, single concat at the end
dfs = []  # collect all DataFrames in a list
for i in range(100):
    dfs.append(pd.DataFrame({'value': [i]}))
result = pd.concat(dfs, ignore_index=True)  # one concat at the end

# BEST - list comprehension
import glob
# all_data = pd.concat(
#     [pd.read_csv(f) for f in glob.glob('data/*.csv')],
#     ignore_index=True
# )

# --- Verify no duplicate indices after concat ---
result = pd.concat([df1, df2])
assert not result.index.duplicated().any(), "Duplicate indices found!"
# Or just use ignore_index=True
```

> **Warning**: `DataFrame.append()` was deprecated in Pandas 1.4 and removed in 2.0. Always use `pd.concat()` instead.

---

## Pivot Tables and Reshaping

### What It Is
Reshaping transforms data between "long" (tidy) format and "wide" format. Think of a pivot table like an Excel pivot table — it summarizes data by rearranging rows into columns. Melting does the reverse.

### Why It Matters
- Reporting: pivot tables create summary views
- ML feature engineering: wide format needed for many algorithms
- Data cleaning: raw data often comes in the wrong shape
- Visualization libraries often need specific formats

### How It Works

```
LONG FORMAT (tidy):              WIDE FORMAT (pivoted):
┌────────┬─────────┬───────┐    ┌────────┬──────┬──────┐
│ date   │ product │ sales │    │ date   │ A    │ B    │
├────────┼─────────┼───────┤    ├────────┼──────┼──────┤
│ Jan    │ A       │ 100   │    │ Jan    │ 100  │ 200  │
│ Jan    │ B       │ 200   │    │ Feb    │ 110  │ 190  │
│ Feb    │ A       │ 110   │    └────────┴──────┴──────┘
│ Feb    │ B       │ 190   │
└────────┴─────────┴───────┘
        pivot() →
        ← melt()
```

### Code Examples

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    'date': ['2024-01', '2024-01', '2024-01', '2024-02', '2024-02', '2024-02'],
    'region': ['East', 'West', 'East', 'West', 'East', 'West'],
    'product': ['A', 'A', 'B', 'B', 'A', 'A'],
    'revenue': [100, 150, 200, 80, 120, 90],
    'quantity': [10, 15, 8, 5, 12, 9]
})

# --- pivot_table() — handles duplicates with aggregation ---
pivot = df.pivot_table(
    values='revenue',         # what to aggregate
    index='date',             # row labels
    columns='region',         # column labels
    aggfunc='sum',            # how to aggregate duplicates
    fill_value=0              # fill NaN with 0
)
# Output:
# region    East  West
# date
# 2024-01    300   150
# 2024-02    120    90

# Multiple aggregation functions
pivot_multi = df.pivot_table(
    values='revenue',
    index='date',
    columns='region',
    aggfunc=['sum', 'mean', 'count']
)

# Multiple values
pivot_values = df.pivot_table(
    values=['revenue', 'quantity'],
    index='date',
    columns='region',
    aggfunc='sum'
)

# Add margins (totals)
pivot_margins = df.pivot_table(
    values='revenue',
    index='date',
    columns='region',
    aggfunc='sum',
    margins=True,          # adds 'All' row and column
    margins_name='Total'
)

# --- pivot() — no aggregation (must have unique index/column pairs) ---
simple_df = pd.DataFrame({
    'date': ['Mon', 'Mon', 'Tue', 'Tue'],
    'city': ['NYC', 'LA', 'NYC', 'LA'],
    'temp': [72, 85, 68, 88]
})
wide = simple_df.pivot(index='date', columns='city', values='temp')
# Output:
# city  LA  NYC
# date
# Mon   85   72
# Tue   88   68

# --- melt() — wide to long (unpivot) ---
wide_df = pd.DataFrame({
    'student': ['Alice', 'Bob'],
    'math': [90, 85],
    'science': [88, 92],
    'english': [95, 78]
})

long_df = pd.melt(
    wide_df,
    id_vars=['student'],           # columns to keep as-is
    value_vars=['math', 'science', 'english'],  # columns to unpivot
    var_name='subject',            # name for the former column headers
    value_name='score'             # name for the values
)
# Output:
#   student  subject  score
# 0   Alice     math     90
# 1     Bob     math     85
# 2   Alice  science     88
# 3     Bob  science     92
# 4   Alice  english     95
# 5     Bob  english     78

# --- stack() and unstack() ---
# stack: columns → index level (wide → long)
# unstack: index level → columns (long → wide)

multi_idx = df.groupby(['date', 'region'])['revenue'].sum()
# Unstack region to columns
unstacked = multi_idx.unstack(fill_value=0)

# Stack it back
stacked = unstacked.stack()

# --- crosstab() — frequency table / contingency table ---
survey = pd.DataFrame({
    'gender': ['M', 'F', 'M', 'F', 'M', 'F', 'M', 'M'],
    'preference': ['A', 'B', 'A', 'A', 'B', 'B', 'A', 'A']
})

ct = pd.crosstab(survey['gender'], survey['preference'])
# Output:
# preference  A  B
# gender
# F           1  2
# M           4  1

# With normalization (percentages)
ct_pct = pd.crosstab(survey['gender'], survey['preference'], normalize='index')
# Shows row percentages (each row sums to 1.0)
```

> **Pro Tip**: Use `pivot_table()` over `pivot()` — it handles duplicate entries gracefully via aggregation. `pivot()` will raise an error if you have duplicate index-column pairs.

---

## Apply, Map, and Transform

### What It Is
These are methods to apply custom functions to your data:
- **`apply()`**: Apply any function along an axis (rows or columns)
- **`map()`**: Element-wise transformation on a Series
- **`applymap()`/`map()`**: Element-wise on entire DataFrame (Pandas 2.1+: use `DataFrame.map()`)
- **`transform()`**: Like apply, but must return same-size output

### Why It Matters
- When built-in functions don't cover your use case
- Feature engineering: creating complex derived features
- Data cleaning: custom parsing, transformations
- Critical to understand performance implications — vectorized > apply > loops

### Code Examples

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    'name': ['Alice Smith', 'Bob Jones', 'Charlie Brown'],
    'salary': [70000, 85000, 120000],
    'bonus_pct': [0.10, 0.15, 0.20],
    'start_date': pd.to_datetime(['2020-01-15', '2019-06-01', '2018-03-20'])
})

# --- Series.map() — element-wise, for Series only ---
# With a function
df['salary_k'] = df['salary'].map(lambda x: f"${x/1000:.0f}K")

# With a dictionary (like VLOOKUP)
grade_map = {70000: 'Junior', 85000: 'Mid', 120000: 'Senior'}
df['grade'] = df['salary'].map(grade_map)

# --- Series.apply() — similar to map but more features ---
df['first_name'] = df['name'].apply(lambda x: x.split()[0])

# With extra arguments
def add_tax(salary, rate):
    return salary * (1 + rate)

df['after_tax'] = df['salary'].apply(add_tax, args=(0.30,))
# Or with keyword arguments:
df['after_tax'] = df['salary'].apply(add_tax, rate=0.30)

# --- DataFrame.apply() — apply function to each row or column ---

# Apply to each column (axis=0, default)
numeric_df = df[['salary', 'bonus_pct']]
col_ranges = numeric_df.apply(lambda col: col.max() - col.min())

# Apply to each row (axis=1) — access multiple columns
df['total_comp'] = df.apply(
    lambda row: row['salary'] * (1 + row['bonus_pct']), axis=1
)

# But PREFER vectorized operations when possible!
# This is 10-100x faster:
df['total_comp'] = df['salary'] * (1 + df['bonus_pct'])

# --- When apply() IS necessary (complex logic) ---
def categorize_employee(row):
    if row['salary'] > 100000 and row['bonus_pct'] >= 0.15:
        return 'Executive'
    elif row['salary'] > 80000:
        return 'Senior'
    else:
        return 'Standard'

df['category'] = df.apply(categorize_employee, axis=1)

# Better alternative with np.select (vectorized):
conditions = [
    (df['salary'] > 100000) & (df['bonus_pct'] >= 0.15),
    df['salary'] > 80000
]
choices = ['Executive', 'Senior']
df['category'] = np.select(conditions, choices, default='Standard')

# --- DataFrame.map() (Pandas 2.1+) — element-wise on entire DataFrame ---
numeric_df = df[['salary', 'bonus_pct']]
formatted = numeric_df.map(lambda x: f"{x:,.2f}" if isinstance(x, (int, float)) else x)

# --- Performance hierarchy (fastest to slowest) ---
# 1. Vectorized NumPy/Pandas operations  (BEST)
# 2. .map() with dict
# 3. .apply() on Series
# 4. .apply() with axis=1 on DataFrame  (SLOWEST — iterates rows)

# Example: all equivalent, but VASTLY different performance
# FASTEST: vectorized
df['bonus'] = df['salary'] * df['bonus_pct']

# SLOW: apply on each row
df['bonus'] = df.apply(lambda r: r['salary'] * r['bonus_pct'], axis=1)

# SLOWEST: Python loop
# for i, row in df.iterrows():
#     df.at[i, 'bonus'] = row['salary'] * row['bonus_pct']
```

### Performance Comparison (1M rows)

| Method | Time | Speed |
|--------|------|-------|
| Vectorized | ~5ms | 1x (baseline) |
| `.map()` with dict | ~50ms | ~10x slower |
| `.apply()` on Series | ~200ms | ~40x slower |
| `.apply(axis=1)` on DataFrame | ~5000ms | ~1000x slower |
| `iterrows()` loop | ~30000ms | ~6000x slower |

> **Pro Tip**: If you find yourself using `apply(axis=1)`, stop and ask: "Can I express this with vectorized operations, `np.where()`, or `np.select()`?" The answer is usually yes.

---

## Window Functions

### What It Is
Window functions compute results over a sliding window of rows — like a moving average. Instead of collapsing groups into one row, they compute a result for EACH row based on its neighbors. Think of looking at data through a sliding magnifying glass.

### Why It Matters
- Time series analysis: moving averages, trend detection
- Anomaly detection: compare each value to its local context
- Finance: rolling volatility, cumulative returns
- Feature engineering: lag features, rolling statistics

### Code Examples

```python
import pandas as pd
import numpy as np

# Time series data
dates = pd.date_range('2024-01-01', periods=20, freq='D')
df = pd.DataFrame({
    'date': dates,
    'price': np.random.randn(20).cumsum() + 100,
    'volume': np.random.randint(1000, 5000, 20)
})

# --- Rolling window ---
df['ma_7'] = df['price'].rolling(window=7).mean()       # 7-day moving average
df['ma_3'] = df['price'].rolling(window=3).mean()       # 3-day moving average
df['rolling_std'] = df['price'].rolling(window=7).std() # rolling volatility

# Min periods: allow partial windows at the start
df['ma_7_partial'] = df['price'].rolling(window=7, min_periods=1).mean()

# Center the window (look both forward and backward)
df['ma_centered'] = df['price'].rolling(window=5, center=True).mean()

# --- Expanding window (cumulative) ---
df['cumulative_mean'] = df['price'].expanding().mean()  # mean of all data so far
df['cumulative_max'] = df['price'].expanding().max()    # running maximum
df['cumulative_sum'] = df['volume'].expanding().sum()   # running total

# --- Exponentially Weighted Moving Average (EWM) ---
# Recent values matter more (exponential decay)
df['ewma'] = df['price'].ewm(span=7).mean()   # span = "effective window"
df['ewm_std'] = df['price'].ewm(span=7).std()

# --- Shift and Lag ---
df['prev_price'] = df['price'].shift(1)           # previous day's price
df['next_price'] = df['price'].shift(-1)          # next day's price
df['daily_return'] = df['price'].pct_change()     # percentage change from prev

# Difference
df['price_diff'] = df['price'].diff()             # price[t] - price[t-1]

# --- Rolling with custom function ---
df['rolling_range'] = df['price'].rolling(7).apply(lambda x: x.max() - x.min())

# --- Rank within rolling window ---
df['rolling_rank'] = df['price'].rolling(7).rank()

# --- Multiple columns at once ---
df[['price_ma', 'volume_ma']] = df[['price', 'volume']].rolling(7).mean()
```

---

## Common Mistakes

### 1. Forgetting Parentheses in Boolean Filtering
```python
# WRONG — operator precedence issue
df[df['age'] > 30 & df['city'] == 'NYC']  # Error or wrong result!

# CORRECT
df[(df['age'] > 30) & (df['city'] == 'NYC')]
```

### 2. Using `and`/`or` Instead of `&`/`|`
```python
# WRONG — Python's and/or work on scalars, not arrays
df[(df['age'] > 30) and (df['city'] == 'NYC')]  # ValueError!

# CORRECT — use bitwise operators for element-wise comparison
df[(df['age'] > 30) & (df['city'] == 'NYC')]
```

### 3. Merge Explosion (Unexpected Row Multiplication)
```python
# If both DataFrames have duplicate keys, merge creates cartesian product for those keys
left = pd.DataFrame({'key': ['A', 'A'], 'val': [1, 2]})
right = pd.DataFrame({'key': ['A', 'A'], 'val': [3, 4]})
pd.merge(left, right, on='key')  # 4 rows! (2 × 2)

# Always check: len(merged) vs len(left) and len(right)
```

### 4. Modifying a Copy Instead of Original (SettingWithCopyWarning)
```python
# WRONG — may modify a copy
subset = df[df['age'] > 30]
subset['new_col'] = 'value'  # Warning! May not modify df

# CORRECT — use .loc on the original
df.loc[df['age'] > 30, 'new_col'] = 'value'

# Or make an explicit copy
subset = df[df['age'] > 30].copy()
subset['new_col'] = 'value'  # Safe, modifying the copy intentionally
```

### 5. Concatenating in a Loop
```python
# WRONG — O(n²) time complexity
result = pd.DataFrame()
for chunk in data_chunks:
    result = pd.concat([result, chunk])  # Copies entire result each time!

# CORRECT — O(n) time complexity
chunks = [process(chunk) for chunk in data_chunks]
result = pd.concat(chunks, ignore_index=True)
```

### 6. GroupBy Forgetting reset_index()
```python
# After groupby, result has group columns as index
grouped = df.groupby('region')['revenue'].sum()
# grouped is a Series with 'region' as index

# If you want region as a column:
grouped = df.groupby('region')['revenue'].sum().reset_index()
# Or: df.groupby('region', as_index=False)['revenue'].sum()
```

---

## Interview Questions

### Q1: What's the difference between `merge()`, `join()`, and `concat()`?
**Answer**: 
- `merge()`: Combines DataFrames on column values (like SQL JOIN). Supports all join types.
- `join()`: Convenience method that merges on index by default. Equivalent to `merge()` with `left_index=True`/`right_index=True`.
- `concat()`: Stacks DataFrames along an axis (row-wise or column-wise). No key matching — just physical stacking.

### Q2: How does `transform()` differ from `apply()` in GroupBy?
**Answer**: `transform()` MUST return a result the same size as the input (one value per original row), enabling broadcasting back to the original DataFrame. `apply()` can return anything — a scalar, a Series, or a DataFrame of any shape. `transform()` is also more optimized internally.

### Q3: You have a DataFrame with 100M rows and need to add a column with the group mean. How?
**Answer**: Use `df['group_mean'] = df.groupby('group_col')['value'].transform('mean')`. This is $O(n)$ and doesn't require a separate merge step. Never use `apply` + merge for this.

### Q4: What happens when you merge two DataFrames that both have duplicate keys?
**Answer**: You get a many-to-many join — the Cartesian product of matching rows. If left has 3 rows with key 'A' and right has 2 rows with key 'A', you get 6 rows in the result. Use `validate='one_to_many'` parameter to catch this.

### Q5: How would you find rows in DataFrame A that are NOT in DataFrame B?
**Answer**:
```python
# Method 1: merge with indicator
merged = pd.merge(df_a, df_b, on='key', how='left', indicator=True)
only_in_a = merged[merged['_merge'] == 'left_only']

# Method 2: isin
only_in_a = df_a[~df_a['key'].isin(df_b['key'])]
```

### Q6: Explain the Split-Apply-Combine pattern.
**Answer**: GroupBy splits data into groups based on key values, applies a function to each group independently, then combines results. This enables vectorized per-group computation without explicit loops.

### Q7: When would you use `pivot_table()` vs `groupby().unstack()`?
**Answer**: They produce equivalent results. `pivot_table()` is more readable for complex reshaping with multiple aggregations. `groupby().unstack()` is useful when you already have a GroupBy object and want to reshape one level to columns.

---

## Quick Reference

| Operation | Code | Notes |
|-----------|------|-------|
| Filter rows | `df[df['col'] > val]` | Use `&`, `\|`, `~` for compound |
| Filter with query | `df.query('col > val')` | Faster for large DataFrames |
| Filter with isin | `df[df['col'].isin(list)]` | Multiple value match |
| Sort by column | `df.sort_values('col')` | `ascending=False` for desc |
| Top N | `df.nlargest(n, 'col')` | Faster than sort+head |
| GroupBy + agg | `df.groupby('g')['v'].mean()` | Split-apply-combine |
| Named agg | `df.groupby('g').agg(name=('col','func'))` | Clean named output |
| Transform | `df.groupby('g')['v'].transform('mean')` | Same-size output |
| Inner merge | `pd.merge(a, b, on='key')` | Only matching rows |
| Left merge | `pd.merge(a, b, on='key', how='left')` | All from left |
| Concat rows | `pd.concat([df1, df2])` | Stack vertically |
| Concat cols | `pd.concat([df1, df2], axis=1)` | Stack horizontally |
| Pivot | `df.pivot_table(values, index, columns)` | Long → wide |
| Melt | `pd.melt(df, id_vars, value_vars)` | Wide → long |
| Rolling mean | `df['col'].rolling(7).mean()` | Moving average |
| Shift/lag | `df['col'].shift(1)` | Previous row value |
| Pct change | `df['col'].pct_change()` | Return from previous |

---

*Next Chapter: [06-Pandas-Data-Cleaning](06-Pandas-Data-Cleaning.md)*
