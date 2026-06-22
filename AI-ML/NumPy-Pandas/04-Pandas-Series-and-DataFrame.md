# Chapter 04: Pandas Series and DataFrame

## Table of Contents
- [What is Pandas?](#what-is-pandas)
- [Why Pandas Matters](#why-pandas-matters)
- [Installation and Import](#installation-and-import)
- [Series — The 1D Labeled Array](#series--the-1d-labeled-array)
- [DataFrame — The 2D Labeled Table](#dataframe--the-2d-labeled-table)
- [Creating DataFrames](#creating-dataframes)
- [Reading and Writing Data](#reading-and-writing-data)
- [Indexing and Selection](#indexing-and-selection)
- [Data Types in Pandas](#data-types-in-pandas)
- [Basic Operations](#basic-operations)
- [Inspecting DataFrames](#inspecting-dataframes)
- [Modifying DataFrames](#modifying-dataframes)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Pandas?

**Simple explanation:** Pandas is a Python library that gives you a supercharged spreadsheet inside your code. Just like Excel lets you organize data in rows and columns, Pandas gives you a **DataFrame** — a table where every column can hold a different type of data (numbers, text, dates), and you can filter, sort, group, and analyze it all with simple commands.

**Analogy:** If NumPy is a calculator (great at crunching numbers), Pandas is a full spreadsheet application (great at organizing, labeling, cleaning, and analyzing data). NumPy works with raw grids of numbers; Pandas works with labeled tables where rows and columns have names.

### Pandas vs NumPy

| Feature | NumPy | Pandas |
|---------|-------|--------|
| Data structure | ndarray (homogeneous) | Series/DataFrame (heterogeneous) |
| Labels | Integer indices only | Named rows and columns |
| Missing data | No native support | Built-in NaN handling |
| Data types | Single dtype per array | Different dtype per column |
| File I/O | `.npy`, `.npz` | CSV, Excel, SQL, JSON, Parquet, ... |
| Operations | Matrix math, broadcasting | GroupBy, merge, pivot, time series |
| Best for | Numerical computation | Data analysis and manipulation |

---

## Why Pandas Matters

### Real-World Use Cases
- **Data Science**: 90%+ of data preprocessing happens in Pandas
- **Business Analytics**: Sales reports, customer analysis, KPI dashboards
- **Machine Learning**: Feature engineering, data cleaning before model training
- **Finance**: Stock analysis, portfolio management, risk calculation
- **Research**: Experiment data processing, survey analysis

### The Data Science Workflow

```
Raw Data → [Load with Pandas] → [Clean] → [Transform] → [Analyze] → [Visualize/Model]
              read_csv()        dropna()    groupby()     describe()    matplotlib/sklearn
              read_excel()      fillna()    merge()       corr()
              read_sql()        astype()    pivot()       value_counts()
```

---

## Installation and Import

```python
# Installation
# pip install pandas

# Standard import convention — ALWAYS use 'pd'
import pandas as pd
import numpy as np  # Often used together

# Check version
print(pd.__version__)  # e.g., '2.2.0'
```

> **Important:** Always import as `pd`. This is the universal convention — any data science code you'll ever read uses it.

---

## Series — The 1D Labeled Array

### What It Is

A **Series** is a one-dimensional labeled array. Think of it as a single column in a spreadsheet — it has values and an index (labels for each value).

```
Index │ Value
──────┼──────
  a   │  10
  b   │  20
  c   │  30
  d   │  40
```

### Creating a Series

```python
import pandas as pd
import numpy as np

# ─── From a list ───
s1 = pd.Series([10, 20, 30, 40])
print(s1)
# 0    10
# 1    20
# 2    30
# 3    40
# dtype: int64

# ─── With custom index ───
s2 = pd.Series([10, 20, 30, 40], index=['a', 'b', 'c', 'd'])
print(s2)
# a    10
# b    20
# c    30
# d    40
# dtype: int64

# ─── From a dictionary (keys become index) ───
s3 = pd.Series({'Alice': 85, 'Bob': 92, 'Charlie': 78, 'Diana': 95})
print(s3)
# Alice      85
# Bob        92
# Charlie    78
# Diana      95
# dtype: int64

# ─── From a NumPy array ───
s4 = pd.Series(np.random.randn(5), index=['a', 'b', 'c', 'd', 'e'])
print(s4.round(2))

# ─── From a scalar (broadcasts to all indices) ───
s5 = pd.Series(42, index=['x', 'y', 'z'])
print(s5)
# x    42
# y    42
# z    42
# dtype: int64
```

### Accessing Series Data

```python
import pandas as pd

scores = pd.Series({'Alice': 85, 'Bob': 92, 'Charlie': 78, 'Diana': 95})

# ─── By label ───
print(scores['Alice'])         # 85
print(scores[['Alice', 'Diana']])  # Multiple labels → Series

# ─── By position ───
print(scores.iloc[0])          # 85 (first element)
print(scores.iloc[-1])         # 95 (last element)
print(scores.iloc[1:3])        # Bob=92, Charlie=78

# ─── Boolean indexing ───
print(scores[scores > 80])
# Alice    85
# Bob      92
# Diana    95

# ─── Attributes ───
print(scores.index)     # Index(['Alice', 'Bob', 'Charlie', 'Diana'])
print(scores.values)    # array([85, 92, 78, 95])
print(scores.dtype)     # int64
print(scores.name)      # None
scores.name = 'Exam Scores'
print(scores.name)      # 'Exam Scores'
```

### Series Operations

```python
import pandas as pd

s = pd.Series([10, 20, 30, 40, 50])

# ─── Vectorized arithmetic (like NumPy) ───
print(s + 5)      # [15, 25, 35, 45, 55]
print(s * 2)      # [20, 40, 60, 80, 100]
print(s ** 2)     # [100, 400, 900, 1600, 2500]

# ─── Aggregations ───
print(s.sum())     # 150
print(s.mean())    # 30.0
print(s.std())     # 15.81
print(s.min())     # 10
print(s.max())     # 50
print(s.median())  # 30.0

# ─── Alignment by index (CRITICAL feature!) ───
a = pd.Series([1, 2, 3], index=['x', 'y', 'z'])
b = pd.Series([10, 20, 30], index=['y', 'z', 'w'])
result = a + b
print(result)
# w     NaN    ← 'w' only in b, not in a
# x     NaN    ← 'x' only in a, not in b
# y    12.0    ← 2 + 10
# z    23.0    ← 3 + 20
# dtype: float64

# This auto-alignment is one of Pandas' killer features!
# Use fill_value to handle missing indices
result = a.add(b, fill_value=0)
print(result)
# w    30.0
# x     1.0
# y    12.0
# z    23.0
```

> **Key Insight:** When you perform operations between two Series, Pandas automatically aligns them by index. Unmatched indices become NaN. This prevents silent data misalignment bugs that plague spreadsheet work.

### Series String Methods

```python
import pandas as pd

names = pd.Series(['Alice Smith', 'bob jones', 'CHARLIE BROWN', '  diana  '])

# .str accessor gives vectorized string operations
print(names.str.lower())       # ['alice smith', 'bob jones', 'charlie brown', '  diana  ']
print(names.str.upper())       # All uppercase
print(names.str.title())       # ['Alice Smith', 'Bob Jones', 'Charlie Brown', '  Diana  ']
print(names.str.strip())       # Remove whitespace
print(names.str.len())         # [11, 9, 13, 9]
print(names.str.contains('a', case=False))  # [True, False, True, True]
print(names.str.split(' '))    # Lists of words
print(names.str.replace(' ', '_'))  # Replace spaces

# Extract first name
print(names.str.strip().str.split(' ').str[0])
# ['Alice', 'bob', 'CHARLIE', 'diana']
```

---

## DataFrame — The 2D Labeled Table

### What It Is

A **DataFrame** is a 2-dimensional labeled data structure — essentially a table (like a spreadsheet or SQL table) with rows and columns. Each column is a Series, and all columns share the same row index.

```
         Name      Age   Salary    Department
─────────────────────────────────────────────
  0  │  Alice   │  30  │ 75000  │  Engineering
  1  │  Bob     │  25  │ 65000  │  Marketing
  2  │  Charlie │  35  │ 90000  │  Engineering
  3  │  Diana   │  28  │ 72000  │  Sales
         ↑          ↑       ↑         ↑
       Series    Series  Series    Series
       (str)     (int)   (int)     (str)
```

### How It Relates to Series

```python
import pandas as pd

# A DataFrame is essentially a dict of Series sharing the same index
df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie'],
    'age': [30, 25, 35],
    'salary': [75000, 65000, 90000]
})

# Each column IS a Series
print(type(df['name']))  # <class 'pandas.core.series.Series'>
print(df['name'])
# 0      Alice
# 1        Bob
# 2    Charlie
# Name: name, dtype: object
```

---

## Creating DataFrames

### From a Dictionary (Most Common)

```python
import pandas as pd

# ─── Dict of lists ───
df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie', 'Diana'],
    'age': [30, 25, 35, 28],
    'salary': [75000, 65000, 90000, 72000],
    'department': ['Engineering', 'Marketing', 'Engineering', 'Sales']
})
print(df)
#       name  age  salary   department
# 0    Alice   30   75000  Engineering
# 1      Bob   25   65000    Marketing
# 2  Charlie   35   90000  Engineering
# 3    Diana   28   72000        Sales

# ─── With custom index ───
df = pd.DataFrame({
    'age': [30, 25, 35],
    'salary': [75000, 65000, 90000]
}, index=['Alice', 'Bob', 'Charlie'])
print(df)
#          age  salary
# Alice     30   75000
# Bob       25   65000
# Charlie   35   90000
```

### From a List of Dictionaries

```python
import pandas as pd

# Each dict = one row (common with JSON/API data)
records = [
    {'name': 'Alice', 'age': 30, 'city': 'NYC'},
    {'name': 'Bob', 'age': 25, 'city': 'Boston'},
    {'name': 'Charlie', 'age': 35}  # Missing 'city' → NaN
]

df = pd.DataFrame(records)
print(df)
#       name  age    city
# 0    Alice   30     NYC
# 1      Bob   25  Boston
# 2  Charlie   35     NaN
```

### From a List of Lists/Tuples

```python
import pandas as pd

# ─── List of lists + column names ───
data = [
    ['Alice', 30, 75000],
    ['Bob', 25, 65000],
    ['Charlie', 35, 90000]
]

df = pd.DataFrame(data, columns=['name', 'age', 'salary'])
print(df)

# ─── List of tuples ───
data = [
    ('Alice', 30, 'NYC'),
    ('Bob', 25, 'Boston'),
    ('Charlie', 35, 'LA')
]
df = pd.DataFrame(data, columns=['name', 'age', 'city'])
```

### From a NumPy Array

```python
import pandas as pd
import numpy as np

# NumPy array → DataFrame
arr = np.random.randn(5, 3)
df = pd.DataFrame(arr, columns=['feature_1', 'feature_2', 'feature_3'])
print(df.round(2))
#    feature_1  feature_2  feature_3
# 0       0.50      -0.14       0.65
# 1      -0.23       1.58      -0.23
# 2       0.77      -0.47       0.54
# ...

# Common ML pattern: features array → labeled DataFrame
from sklearn.datasets import load_iris  # Example
# X = load_iris().data
# df = pd.DataFrame(X, columns=load_iris().feature_names)
```

### From a Dictionary of Series

```python
import pandas as pd

# Series with different indices → auto-aligned with NaN fill
s1 = pd.Series([1, 2, 3], index=['a', 'b', 'c'])
s2 = pd.Series([4, 5, 6], index=['b', 'c', 'd'])

df = pd.DataFrame({'col1': s1, 'col2': s2})
print(df)
#    col1  col2
# a   1.0   NaN
# b   2.0   4.0
# c   3.0   5.0
# d   NaN   6.0
```

---

## Reading and Writing Data

### CSV Files (Most Common)

```python
import pandas as pd

# ─── Reading CSV ───
df = pd.read_csv('data.csv')                    # Basic
df = pd.read_csv('data.csv', index_col=0)       # First column as index
df = pd.read_csv('data.csv', header=None)       # No header row
df = pd.read_csv('data.csv', 
                 names=['a', 'b', 'c'])         # Custom column names
df = pd.read_csv('data.csv', 
                 usecols=['name', 'age'])        # Only load specific columns
df = pd.read_csv('data.csv', 
                 nrows=100)                      # First 100 rows only
df = pd.read_csv('data.csv',
                 dtype={'age': 'int32', 
                        'name': 'string'})       # Specify dtypes
df = pd.read_csv('data.csv', 
                 parse_dates=['date_col'])        # Parse date columns
df = pd.read_csv('data.csv', 
                 na_values=['N/A', 'missing', '-']) # Custom NaN markers

# ─── Reading large files in chunks ───
chunks = pd.read_csv('huge_file.csv', chunksize=10000)
for chunk in chunks:
    # Process each 10k-row chunk
    process(chunk)

# ─── Writing CSV ───
df.to_csv('output.csv', index=False)             # Without row index
df.to_csv('output.csv', columns=['name', 'age']) # Specific columns
df.to_csv('output.csv', sep='\t')                # Tab-separated
```

### Other Formats

```python
import pandas as pd

# ─── Excel ───
df = pd.read_excel('data.xlsx', sheet_name='Sheet1')
df.to_excel('output.xlsx', index=False, sheet_name='Results')

# ─── JSON ───
df = pd.read_json('data.json')
df.to_json('output.json', orient='records')

# ─── Parquet (fast, compressed — preferred for large datasets) ───
df = pd.read_parquet('data.parquet')
df.to_parquet('output.parquet', engine='pyarrow')

# ─── SQL ───
import sqlite3
conn = sqlite3.connect('database.db')
df = pd.read_sql('SELECT * FROM users', conn)
df.to_sql('users_copy', conn, if_exists='replace', index=False)

# ─── From clipboard (paste from Excel/web) ───
# df = pd.read_clipboard()

# ─── From URL ───
url = 'https://example.com/data.csv'
# df = pd.read_csv(url)
```

### File Format Comparison

| Format | Read Speed | Write Speed | File Size | Human Readable | Best For |
|--------|-----------|-------------|-----------|---------------|----------|
| CSV | Slow | Slow | Large | ✓ Yes | Sharing, compatibility |
| Parquet | Fast | Fast | Small | ✗ No | Production, large data |
| Feather | Very fast | Very fast | Small | ✗ No | Temporary, inter-process |
| Excel | Very slow | Very slow | Large | ✓ Yes | Business reports |
| JSON | Slow | Slow | Large | ✓ Yes | API data, nested |
| HDF5 | Fast | Fast | Small | ✗ No | Scientific data |

> **Pro Tip:** For any dataset > 100MB, use **Parquet** instead of CSV. It's 2-10x faster to read, 3-5x smaller files, and preserves dtypes (CSV loses them).

---

## Indexing and Selection

### This is the MOST Important Section

Pandas has multiple ways to select data. Understanding when to use each is critical.

### Column Selection

```python
import pandas as pd

df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie', 'Diana'],
    'age': [30, 25, 35, 28],
    'salary': [75000, 65000, 90000, 72000],
    'dept': ['Eng', 'Mkt', 'Eng', 'Sales']
})

# ─── Single column (returns Series) ───
print(df['name'])           # Bracket notation (always works)
print(df.name)              # Dot notation (doesn't work if column name
                            # has spaces or conflicts with methods)

# ─── Multiple columns (returns DataFrame) ───
print(df[['name', 'age']])  # Double brackets = list of columns

# ─── Which to use? ───
# ALWAYS use bracket notation df['col']
# Dot notation breaks with: df.count (method!), df.my column (space!)
```

### .loc — Label-Based Selection

```python
import pandas as pd

df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie', 'Diana'],
    'age': [30, 25, 35, 28],
    'salary': [75000, 65000, 90000, 72000]
}, index=['emp1', 'emp2', 'emp3', 'emp4'])

# ─── Single row by label ───
print(df.loc['emp1'])
# name      Alice
# age          30
# salary    75000
# Name: emp1, dtype: object

# ─── Multiple rows ───
print(df.loc[['emp1', 'emp3']])
#       name  age  salary
# emp1  Alice   30   75000
# emp3  Charlie 35   90000

# ─── Row + column ───
print(df.loc['emp1', 'name'])       # 'Alice' (single value)
print(df.loc['emp1', ['name', 'age']])  # Series

# ─── Slicing with loc (INCLUSIVE on both ends!) ───
print(df.loc['emp1':'emp3'])  # emp1, emp2, AND emp3 included!
#       name     age  salary
# emp1  Alice     30   75000
# emp2  Bob       25   65000
# emp3  Charlie   35   90000

# ─── Row slice + column slice ───
print(df.loc['emp1':'emp3', 'name':'age'])
#       name     age
# emp1  Alice     30
# emp2  Bob       25
# emp3  Charlie   35

# ─── Boolean selection with loc ───
print(df.loc[df['age'] > 28])
#        name  age  salary
# emp1  Alice   30   75000
# emp3  Charlie 35   90000

# ─── Boolean + specific columns ───
print(df.loc[df['age'] > 28, ['name', 'salary']])
#        name  salary
# emp1  Alice   75000
# emp3  Charlie 90000
```

### .iloc — Position-Based Selection

```python
import pandas as pd

df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie', 'Diana'],
    'age': [30, 25, 35, 28],
    'salary': [75000, 65000, 90000, 72000]
})

# ─── Single row by position ───
print(df.iloc[0])      # First row
print(df.iloc[-1])     # Last row

# ─── Multiple rows ───
print(df.iloc[[0, 2]])  # Rows 0 and 2

# ─── Slicing (EXCLUSIVE end, like Python) ───
print(df.iloc[1:3])    # Rows 1 and 2 (NOT 3)

# ─── Row + column ───
print(df.iloc[0, 1])           # 30 (row 0, col 1)
print(df.iloc[0:2, 0:2])       # First 2 rows, first 2 cols
#     name  age
# 0  Alice   30
# 1    Bob   25

# ─── Every other row ───
print(df.iloc[::2])    # Rows 0, 2
```

### .loc vs .iloc — The Critical Difference

```
┌─────────┬──────────────────────┬──────────────────────┐
│         │       .loc           │       .iloc          │
├─────────┼──────────────────────┼──────────────────────┤
│ Uses    │ Labels (names)       │ Positions (integers) │
│ Slicing │ INCLUSIVE both ends  │ EXCLUSIVE end        │
│ Example │ df.loc['a':'c']      │ df.iloc[0:3]         │
│         │ Returns a, b, c     │ Returns 0, 1, 2      │
│ Error   │ KeyError if missing  │ IndexError if OOB    │
└─────────┴──────────────────────┴──────────────────────┘
```

```python
import pandas as pd

df = pd.DataFrame({'val': [10, 20, 30, 40, 50]}, 
                  index=[100, 200, 300, 400, 500])

print(df.loc[200])    # Label 200 → val=20
print(df.iloc[1])     # Position 1 → val=20 (same row, different accessor)

# CRITICAL: With integer index, loc uses LABELS, not positions!
print(df.loc[100:300])   # Rows with labels 100, 200, 300 (inclusive!)
print(df.iloc[0:3])      # Rows at positions 0, 1, 2 (exclusive end)
```

### Boolean Indexing (Filtering)

```python
import pandas as pd

df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie', 'Diana', 'Eve'],
    'age': [30, 25, 35, 28, 32],
    'salary': [75000, 65000, 90000, 72000, 80000],
    'dept': ['Eng', 'Mkt', 'Eng', 'Sales', 'Eng']
})

# ─── Simple filter ───
young = df[df['age'] < 30]
print(young)

# ─── Multiple conditions (use & for AND, | for OR) ───
# MUST use parentheses around each condition!
result = df[(df['age'] > 25) & (df['dept'] == 'Eng')]
print(result)
#       name  age  salary dept
# 0    Alice   30   75000  Eng
# 2  Charlie   35   90000  Eng
# 4      Eve   32   80000  Eng

# ─── NOT condition ───
result = df[~(df['dept'] == 'Eng')]  # Everyone NOT in Engineering

# ─── isin() for multiple values ───
result = df[df['dept'].isin(['Eng', 'Sales'])]
print(result)

# ─── between() for range ───
result = df[df['age'].between(28, 32)]  # 28 <= age <= 32
print(result)

# ─── String conditions ───
result = df[df['name'].str.startswith('A')]
result = df[df['name'].str.contains('li', case=False)]

# ─── query() method — SQL-like syntax ───
result = df.query('age > 28 and dept == "Eng"')
print(result)
# Same as df[(df['age'] > 28) & (df['dept'] == 'Eng')]
# Cleaner for complex conditions!

# Using variables in query
min_age = 28
result = df.query('age > @min_age')  # @ prefix for Python variables
```

### Setting Values

```python
import pandas as pd

df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie'],
    'age': [30, 25, 35],
    'salary': [75000, 65000, 90000]
})

# ─── Set a single value ───
df.loc[0, 'salary'] = 80000      # By label
df.iloc[0, 2] = 80000            # By position

# ─── Set entire column ───
df['salary'] = [80000, 70000, 95000]

# ─── Conditional setting ───
df.loc[df['age'] > 30, 'salary'] = 100000

# ─── Create new column from existing ───
df['bonus'] = df['salary'] * 0.1

# ─── WARNING: Chained indexing — DON'T do this! ───
# df[df['age'] > 30]['salary'] = 100000  # May not work! SettingWithCopyWarning
# ALWAYS use .loc instead:
df.loc[df['age'] > 30, 'salary'] = 100000
```

> **Critical Rule:** Never use chained indexing (`df[...][...]`) to set values. Always use `.loc[rows, cols]`. Chained indexing may create a copy instead of modifying the original, leading to the dreaded `SettingWithCopyWarning`.

---

## Data Types in Pandas

### Understanding dtypes

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie'],
    'age': [30, 25, 35],
    'salary': [75000.0, 65000.0, 90000.0],
    'is_manager': [True, False, True],
    'start_date': pd.to_datetime(['2020-01-15', '2021-06-01', '2019-03-20'])
})

print(df.dtypes)
# name           object       ← strings (legacy)
# age             int64
# salary        float64
# is_manager       bool
# start_date  datetime64[ns]

print(df.info())
# Shows dtypes + memory usage + non-null counts
```

### Pandas dtype Reference

| Pandas dtype | Python type | Description | Use When |
|-------------|-------------|-------------|----------|
| `int64` | int | 64-bit integer | Counts, IDs |
| `float64` | float | 64-bit float | Measurements, money |
| `bool` | bool | True/False | Flags, filters |
| `object` | str/mixed | Python objects | Text (legacy) |
| `string` | str | String type | Text (modern, preferred) |
| `category` | — | Categorical | Low-cardinality strings |
| `datetime64[ns]` | datetime | Timestamps | Dates and times |
| `timedelta64[ns]` | timedelta | Time differences | Durations |
| `Int64` (capital I) | int | Nullable integer | Integers with NaN |
| `Float64` (capital F) | float | Nullable float | — |
| `boolean` | bool | Nullable boolean | Booleans with NaN |

### Type Conversion

```python
import pandas as pd

df = pd.DataFrame({
    'id': ['1', '2', '3'],          # Strings that should be ints
    'price': ['10.5', '20.3', '30.1'],  # Strings that should be floats
    'active': [1, 0, 1],            # Ints that should be bools
    'date': ['2024-01-15', '2024-06-22', '2024-12-31']  # Strings → dates
})

# ─── Convert types ───
df['id'] = df['id'].astype(int)
df['price'] = df['price'].astype(float)
df['active'] = df['active'].astype(bool)
df['date'] = pd.to_datetime(df['date'])

print(df.dtypes)
# id         int64 (or int32)
# price    float64
# active      bool
# date    datetime64[ns]

# ─── Handle errors in conversion ───
messy = pd.Series(['1', '2', 'three', '4', 'N/A'])

# errors='coerce' → invalid values become NaN
clean = pd.to_numeric(messy, errors='coerce')
print(clean)
# 0    1.0
# 1    2.0
# 2    NaN  ← 'three' couldn't convert
# 3    4.0
# 4    NaN  ← 'N/A' couldn't convert

# ─── Nullable integer type (supports NaN!) ───
s = pd.array([1, 2, None, 4], dtype=pd.Int64Dtype())
print(s)
# [1, 2, <NA>, 4]
# dtype: Int64  ← capital I = nullable
```

### Category dtype (Memory Optimization)

```python
import pandas as pd

# Regular object column
df = pd.DataFrame({
    'color': ['red', 'blue', 'red', 'green', 'blue'] * 100000
})
print(f"Object memory: {df['color'].memory_usage(deep=True) / 1e6:.1f} MB")
# ~2.8 MB

# Convert to category
df['color'] = df['color'].astype('category')
print(f"Category memory: {df['color'].memory_usage(deep=True) / 1e6:.1f} MB")
# ~0.5 MB — 5x smaller!

# Categories are stored as integers internally with a mapping
print(df['color'].cat.categories)  # Index(['blue', 'green', 'red'])
print(df['color'].cat.codes[:5])   # [2 0 2 1 0] — integer codes
```

> **Pro Tip:** Convert any column with fewer than ~50 unique values to `category` dtype. Especially effective for columns like 'country', 'status', 'department', 'color', etc.

---

## Basic Operations

### Arithmetic and Statistical

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    'A': [10, 20, 30, 40, 50],
    'B': [5, 15, 25, 35, 45],
    'C': [100, 200, 300, 400, 500]
})

# ─── Column arithmetic ───
df['D'] = df['A'] + df['B']          # Column addition
df['ratio'] = df['A'] / df['C']      # Division
df['A_squared'] = df['A'] ** 2       # Power

# ─── Apply function to column ───
df['A_log'] = np.log(df['A'])

# ─── Descriptive statistics ───
print(df.describe())
# Shows count, mean, std, min, 25%, 50%, 75%, max for each numeric column

print(df['A'].describe())
# count     5.0
# mean     30.0
# std      15.81
# min      10.0
# 25%      20.0
# 50%      30.0
# 75%      40.0
# max      50.0

# ─── Correlation matrix ───
print(df[['A', 'B', 'C']].corr())
#      A    B    C
# A  1.0  1.0  1.0
# B  1.0  1.0  1.0
# C  1.0  1.0  1.0
# (Perfect correlation because they're all linear)

# ─── Value counts (for categorical data) ───
status = pd.Series(['active', 'inactive', 'active', 'active', 'inactive', 'pending'])
print(status.value_counts())
# active      3
# inactive    2
# pending     1
# dtype: int64

# Normalized (percentages)
print(status.value_counts(normalize=True))
# active      0.500
# inactive    0.333
# pending     0.167
```

### apply() — Custom Functions

```python
import pandas as pd

df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie'],
    'salary': [75000, 65000, 90000],
    'years': [5, 3, 8]
})

# ─── Apply to a column (Series) ───
df['tax_bracket'] = df['salary'].apply(
    lambda x: 'high' if x > 80000 else 'standard'
)

# ─── Apply with a named function ───
def calculate_bonus(row):
    if row['years'] > 5:
        return row['salary'] * 0.15
    else:
        return row['salary'] * 0.10

df['bonus'] = df.apply(calculate_bonus, axis=1)  # axis=1 → row-wise
print(df)
#       name  salary  years tax_bracket    bonus
# 0    Alice   75000      5    standard   7500.0
# 1      Bob   65000      3    standard   6500.0
# 2  Charlie   90000      8        high  13500.0

# ─── Apply to entire DataFrame (column-wise) ───
numeric_df = df[['salary', 'years']]
print(numeric_df.apply(np.mean, axis=0))  # Mean of each column
# salary    76666.67
# years         5.33
```

> **Pro Tip:** `apply()` is a Python loop under the hood — it's slow for large datasets. Always prefer vectorized operations when possible:
> - Instead of `df['x'].apply(lambda x: x**2)`, use `df['x'] ** 2`
> - Instead of `apply(lambda row: ...)`, use `np.where()` or vectorized conditions

### map() and replace()

```python
import pandas as pd

# ─── map() — transform Series values using a dict or function ───
grades = pd.Series(['A', 'B', 'C', 'A', 'B'])

# Dict mapping
numeric = grades.map({'A': 4.0, 'B': 3.0, 'C': 2.0})
print(numeric)  # [4.0, 3.0, 2.0, 4.0, 3.0]

# Function mapping
print(grades.map(str.lower))  # ['a', 'b', 'c', 'a', 'b']

# ─── replace() — replace specific values ───
df = pd.DataFrame({'status': ['Y', 'N', 'Y', 'N', 'Y']})
df['status'] = df['status'].replace({'Y': 'Yes', 'N': 'No'})
print(df)
#   status
# 0    Yes
# 1     No
# 2    Yes
# 3     No
# 4    Yes
```

---

## Inspecting DataFrames

### Essential Inspection Tools

```python
import pandas as pd
import numpy as np

# Create sample dataset
df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie', 'Diana', 'Eve'],
    'age': [30, 25, 35, 28, None],
    'salary': [75000, 65000, 90000, 72000, 80000],
    'dept': ['Eng', 'Mkt', 'Eng', 'Sales', 'Eng']
})

# ─── First/Last rows ───
print(df.head())       # First 5 rows (default)
print(df.head(2))      # First 2 rows
print(df.tail(3))      # Last 3 rows

# ─── Shape and size ───
print(df.shape)        # (5, 4) — 5 rows, 4 columns
print(len(df))         # 5 — number of rows
print(df.size)         # 20 — total cells (5 × 4)

# ─── Column names ───
print(df.columns)      # Index(['name', 'age', 'salary', 'dept'])
print(df.columns.tolist())  # ['name', 'age', 'salary', 'dept']

# ─── Index ───
print(df.index)        # RangeIndex(start=0, stop=5, step=1)

# ─── Data types ───
print(df.dtypes)

# ─── Info (the most useful single command) ───
print(df.info())
# <class 'pandas.core.DataFrame'>
# RangeIndex: 5 entries, 0 to 4
# Data columns (total 4 columns):
#  #   Column  Non-Null Count  Dtype
# ---  ------  --------------  -----
#  0   name    5 non-null      object
#  1   age     4 non-null      float64  ← 1 missing!
#  2   salary  5 non-null      int64
#  3   dept    5 non-null      object
# dtypes: float64(1), int64(1), object(2)
# memory usage: 288.0+ bytes

# ─── Statistical summary ───
print(df.describe())             # Numeric columns only
print(df.describe(include='all'))  # All columns

# ─── Unique values ───
print(df['dept'].unique())        # ['Eng' 'Mkt' 'Sales']
print(df['dept'].nunique())       # 3

# ─── Missing values ───
print(df.isnull().sum())
# name      0
# age       1   ← 1 missing value
# salary    0
# dept      0

# Missing value percentage
print((df.isnull().sum() / len(df) * 100).round(1))

# ─── Memory usage ───
print(df.memory_usage(deep=True))
# Index     128
# name      320
# age        40
# salary     40
# dept      299
# Total memory in MB:
print(f"{df.memory_usage(deep=True).sum() / 1e6:.2f} MB")

# ─── Random sample ───
print(df.sample(2))      # 2 random rows
print(df.sample(frac=0.5))  # 50% of rows
```

---

## Modifying DataFrames

### Adding and Removing Columns

```python
import pandas as pd

df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie'],
    'age': [30, 25, 35],
    'salary': [75000, 65000, 90000]
})

# ─── Add columns ───
df['bonus'] = df['salary'] * 0.1                # From calculation
df['department'] = ['Eng', 'Mkt', 'Eng']        # From list
df['is_senior'] = df['age'] >= 30               # From condition
df['id'] = range(1001, 1004)                    # From range

# ─── Insert at specific position ───
df.insert(0, 'emp_id', ['E001', 'E002', 'E003'])  # Insert at position 0

# ─── Rename columns ───
df = df.rename(columns={'name': 'full_name', 'age': 'years_old'})

# Rename all columns at once
# df.columns = ['a', 'b', 'c', 'd', 'e', 'f', 'g']

# ─── Remove columns ───
df = df.drop(columns=['bonus', 'is_senior'])       # Drop specific columns
# Or: df.drop(['bonus', 'is_senior'], axis=1)

# ─── Remove rows ───
df = df.drop(index=[0, 2])  # Drop rows by index label
```

### Adding and Removing Rows

```python
import pandas as pd

df = pd.DataFrame({
    'name': ['Alice', 'Bob'],
    'age': [30, 25]
})

# ─── Add rows ───
# Method 1: concat (preferred)
new_row = pd.DataFrame({'name': ['Charlie'], 'age': [35]})
df = pd.concat([df, new_row], ignore_index=True)

# Method 2: loc with new index
df.loc[len(df)] = ['Diana', 28]

print(df)
#       name  age
# 0    Alice   30
# 1      Bob   25
# 2  Charlie   35
# 3    Diana   28

# ─── Remove rows by condition ───
df = df[df['age'] >= 28]  # Keep only age >= 28
print(df)
#       name  age
# 0    Alice   30
# 2  Charlie   35
# 3    Diana   28

# ─── Reset index after removing rows ───
df = df.reset_index(drop=True)
print(df)
#       name  age
# 0    Alice   30
# 1  Charlie   35
# 2    Diana   28
```

### Sorting

```python
import pandas as pd

df = pd.DataFrame({
    'name': ['Charlie', 'Alice', 'Bob', 'Diana'],
    'age': [35, 30, 25, 30],
    'salary': [90000, 75000, 65000, 72000]
})

# ─── Sort by single column ───
df_sorted = df.sort_values('age')
print(df_sorted)

# ─── Sort descending ───
df_sorted = df.sort_values('salary', ascending=False)

# ─── Sort by multiple columns ───
df_sorted = df.sort_values(['age', 'salary'], ascending=[True, False])
# First by age ascending, then by salary descending within same age

# ─── Sort by index ───
df_sorted = df.sort_index()

# ─── Get top N ───
top3 = df.nlargest(3, 'salary')
bottom2 = df.nsmallest(2, 'age')
```

### Setting and Resetting Index

```python
import pandas as pd

df = pd.DataFrame({
    'emp_id': ['E001', 'E002', 'E003'],
    'name': ['Alice', 'Bob', 'Charlie'],
    'age': [30, 25, 35]
})

# ─── Set a column as index ───
df = df.set_index('emp_id')
print(df)
#        name  age
# emp_id
# E001  Alice   30
# E002    Bob   25
# E003  Charlie 35

# Now you can use .loc with emp_id
print(df.loc['E002'])

# ─── Reset index (move index back to column) ───
df = df.reset_index()
print(df)
#   emp_id     name  age
# 0   E001    Alice   30
# 1   E002      Bob   25
# 2   E003  Charlie   35
```

### Copy vs Reference

```python
import pandas as pd

df = pd.DataFrame({'A': [1, 2, 3], 'B': [4, 5, 6]})

# ─── Assignment creates a reference (NOT a copy) ───
df2 = df
df2['A'] = 99
print(df['A'])  # [99, 99, 99] — original changed!

# ─── .copy() creates an independent copy ───
df = pd.DataFrame({'A': [1, 2, 3], 'B': [4, 5, 6]})
df3 = df.copy()
df3['A'] = 99
print(df['A'])  # [1, 2, 3] — original safe!
```

> **Rule:** Always use `.copy()` when you want to create an independent DataFrame from a slice or selection. This avoids the `SettingWithCopyWarning` and subtle bugs.

---

## Common Mistakes

### 1. Chained Indexing (SettingWithCopyWarning)

```python
import pandas as pd

df = pd.DataFrame({
    'name': ['Alice', 'Bob', 'Charlie'],
    'age': [30, 25, 35]
})

# WRONG — may not modify df!
# df[df['age'] > 28]['age'] = 99  # SettingWithCopyWarning!

# CORRECT — always use .loc
df.loc[df['age'] > 28, 'age'] = 99
```

### 2. Confusing .loc and .iloc with integer index

```python
import pandas as pd

# DataFrame with integer index that doesn't start at 0
df = pd.DataFrame({'val': [10, 20, 30]}, index=[5, 10, 15])

# .loc uses LABELS
print(df.loc[5])    # val=10 (label 5)

# .iloc uses POSITIONS
print(df.iloc[0])   # val=10 (position 0)

# CAREFUL: df.loc[0] would raise KeyError (no label 0)!
```

### 3. Forgetting inplace is deprecated / error-prone

```python
import pandas as pd

df = pd.DataFrame({'A': [3, 1, 2], 'B': [6, 4, 5]})

# OLD style (many methods had inplace=True, now discouraged)
# df.sort_values('A', inplace=True)  # Modifies df, returns None

# PREFERRED — explicit reassignment
df = df.sort_values('A')  # Returns new DataFrame, assign to df

# This is cleaner and avoids confusion with method chaining
```

### 4. Using == to check for NaN

```python
import pandas as pd
import numpy as np

s = pd.Series([1, np.nan, 3])

# WRONG — NaN != NaN in IEEE 754
print(s == np.nan)  # [False, False, False] — always False!

# CORRECT
print(s.isna())     # [False, True, False]
print(s.isnull())   # Same thing (alias)
print(pd.isna(s))   # Also works
```

### 5. Not specifying dtypes when reading CSV

```python
import pandas as pd

# WRONG — Pandas guesses types (often wrong)
# df = pd.read_csv('data.csv')
# '00123' becomes 123 (loses leading zeros!)
# '2024-01-15' stays as string (not parsed as date)

# CORRECT — specify dtypes
# df = pd.read_csv('data.csv', 
#                   dtype={'zip_code': str, 'id': str},
#                   parse_dates=['date_column'])
```

### 6. Modifying while iterating

```python
import pandas as pd

df = pd.DataFrame({'A': [1, 2, 3], 'B': [4, 5, 6]})

# WRONG — modifying during iteration is unreliable
# for idx, row in df.iterrows():
#     df.loc[idx, 'C'] = row['A'] + row['B']

# CORRECT — vectorized operation
df['C'] = df['A'] + df['B']

# If you MUST iterate (complex logic), use apply:
# df['C'] = df.apply(lambda row: row['A'] + row['B'], axis=1)
```

---

## Interview Questions

### Q1: What's the difference between a Series and a DataFrame?
**Answer:** A Series is a 1D labeled array (single column with an index). A DataFrame is a 2D labeled data structure (table with rows and columns). A DataFrame is essentially a dict of Series sharing the same index. Each column in a DataFrame is a Series.

### Q2: Explain the difference between `.loc` and `.iloc`.
**Answer:** `.loc` selects data by **labels** (index names/column names) and slicing is inclusive on both ends. `.iloc` selects by **integer positions** and slicing is exclusive on the end (like Python). With integer indices, `.loc[5]` finds label 5, while `.iloc[5]` finds position 5 — these may be different rows.

### Q3: What is the `SettingWithCopyWarning` and how do you fix it?
**Answer:** It occurs when Pandas can't determine if you're modifying a view or a copy of the DataFrame. Happens with chained indexing like `df[condition]['col'] = val`. Fix: always use `.loc` for combined row/column selection: `df.loc[condition, 'col'] = val`. In Pandas 2.0+, Copy-on-Write mode eliminates this issue entirely.

### Q4: How does Pandas handle missing data?
**Answer:** Pandas uses `NaN` (Not a Number) for missing float data and `pd.NA` for nullable types. Key functions: `isna()`/`isnull()` to detect, `dropna()` to remove, `fillna()` to replace. NaN comparisons are always False (`NaN != NaN`), so use `isna()` not `==`. Integer columns with NaN are auto-upcast to float64 unless using nullable `Int64` dtype.

### Q5: When should you use `category` dtype?
**Answer:** Use category for columns with low cardinality (few unique values relative to total rows). Examples: gender, country, status, department. Benefits: 3-10x memory reduction, faster groupby/sort operations. Convert with `df['col'].astype('category')`. Don't use for high-cardinality columns (e.g., email addresses, UUIDs).

### Q6: What's the most efficient way to read a large CSV file?
**Answer:**
1. Specify `dtype` to avoid inference overhead
2. Use `usecols` to load only needed columns
3. Use `nrows` for sampling
4. Read in `chunksize` for processing in batches
5. Better yet, convert to Parquet format for 2-10x faster reads
6. Use `engine='pyarrow'` for fastest CSV parsing

### Q7: How do you handle the alignment behavior of Pandas operations?
**Answer:** Pandas auto-aligns Series/DataFrames by index during operations. Unmatched indices become NaN. Use `fill_value` parameter (e.g., `a.add(b, fill_value=0)`) to handle missing alignments. Use `reset_index()` if you want position-based alignment instead. This auto-alignment prevents data misalignment bugs but can introduce unexpected NaN values.

### Q8: Difference between `apply()`, `map()`, and `applymap()`?
**Answer:**
- `map()` — Series only; maps each value through a dict or function
- `apply()` — Works on both Series (element-wise) and DataFrame (row/column-wise with `axis`)
- `applymap()` — DataFrame only; applies function to every single element (deprecated in Pandas 2.1+, use `map()` instead)

For performance: vectorized ops > `map()` > `apply()` > `iterrows()`

---

## Quick Reference

### DataFrame Creation

| Method | Code | Notes |
|--------|------|-------|
| From dict of lists | `pd.DataFrame({'a': [1,2], 'b': [3,4]})` | Most common |
| From list of dicts | `pd.DataFrame([{'a':1}, {'a':2}])` | From JSON/API |
| From NumPy array | `pd.DataFrame(arr, columns=['a','b'])` | From ML |
| From CSV | `pd.read_csv('file.csv')` | From file |
| From dict of Series | `pd.DataFrame({'a': s1, 'b': s2})` | Auto-aligns |

### Selection Cheat Sheet

| What | How | Returns |
|------|-----|---------|
| Single column | `df['col']` | Series |
| Multiple columns | `df[['col1', 'col2']]` | DataFrame |
| Row by label | `df.loc['label']` | Series |
| Row by position | `df.iloc[0]` | Series |
| Cell by label | `df.loc['row', 'col']` | Scalar |
| Cell by position | `df.iloc[0, 1]` | Scalar |
| Rows by condition | `df[df['col'] > 5]` | DataFrame |
| Rows + cols by label | `df.loc[rows, cols]` | DataFrame |
| Rows + cols by position | `df.iloc[r1:r2, c1:c2]` | DataFrame |

### Inspection Cheat Sheet

| Command | What It Shows |
|---------|---------------|
| `df.head(n)` | First n rows |
| `df.tail(n)` | Last n rows |
| `df.shape` | (rows, cols) |
| `df.dtypes` | Column data types |
| `df.info()` | Types + non-null counts + memory |
| `df.describe()` | Statistical summary |
| `df.columns` | Column names |
| `df.index` | Row index |
| `df.isnull().sum()` | Missing values per column |
| `df['col'].unique()` | Unique values |
| `df['col'].nunique()` | Count of unique values |
| `df['col'].value_counts()` | Frequency of each value |
| `df.sample(n)` | Random n rows |
| `df.memory_usage(deep=True)` | Memory per column |

### Modification Cheat Sheet

| Task | Code |
|------|------|
| Add column | `df['new'] = values` |
| Rename columns | `df.rename(columns={'old': 'new'})` |
| Drop columns | `df.drop(columns=['col1', 'col2'])` |
| Drop rows | `df.drop(index=[0, 1])` |
| Sort by column | `df.sort_values('col')` |
| Set index | `df.set_index('col')` |
| Reset index | `df.reset_index()` |
| Change dtype | `df['col'].astype(int)` |
| Replace values | `df['col'].replace({'old': 'new'})` |
| Copy DataFrame | `df.copy()` |

---

## Key Takeaways

1. **Series = 1D labeled array**, **DataFrame = 2D labeled table** (dict of Series)
2. **`.loc` for labels, `.iloc` for positions** — never confuse them
3. **Never use chained indexing** to set values — always use `.loc[rows, cols] = value`
4. **`df.info()`** is the single most useful inspection command
5. **Specify dtypes** when reading CSV for correctness and performance
6. **Use `category` dtype** for low-cardinality string columns (huge memory savings)
7. **Prefer Parquet over CSV** for any serious data work (faster, smaller, preserves types)
8. **Auto-alignment by index** is a feature, not a bug — but be aware of it
9. **Vectorize operations** — `apply()` is a loop; use built-in Pandas/NumPy ops when possible
10. **Always `.copy()`** when creating independent subsets to avoid mutation bugs

---

*Next Chapter: [05-Pandas-Data-Manipulation](05-Pandas-Data-Manipulation.md) — Filtering, Sorting, GroupBy, Merge, Join, Concat, Pivot*
