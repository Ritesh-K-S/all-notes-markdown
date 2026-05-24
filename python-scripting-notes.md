# Python Scripting — Complete Notes

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Comments & Docstrings](#2-comments--docstrings)
3. [Variables & Data Types](#3-variables--data-types)
4. [Numbers & Arithmetic](#4-numbers--arithmetic)
5. [Strings](#5-strings)
6. [Lists](#6-lists)
7. [Tuples](#7-tuples)
8. [Dictionaries](#8-dictionaries)
9. [Sets](#9-sets)
10. [Control Flow](#10-control-flow)
11. [Loops](#11-loops)
12. [Functions](#12-functions)
13. [Classes & OOP](#13-classes--oop)
14. [Error Handling](#14-error-handling)
15. [File Handling](#15-file-handling)
16. [Modules & Packages](#16-modules--packages)
17. [Regular Expressions](#17-regular-expressions)
18. [Subprocess — Running System Commands](#18-subprocess--running-system-commands)
19. [System & OS Interaction](#19-system--os-interaction)
20. [JSON](#20-json)
21. [CSV](#21-csv)
22. [Datetime](#22-datetime)
23. [Logging](#23-logging)
24. [Concurrency](#24-concurrency)
25. [Networking](#25-networking)
26. [Database](#26-database)
27. [Itertools](#27-itertools)
28. [Collections Module](#28-collections-module)
29. [Functools Module](#29-functools-module)
30. [Web Scraping](#30-web-scraping)
31. [Testing](#31-testing)
32. [Useful Standard Library Modules](#32-useful-standard-library-modules)
33. [Script Template](#33-script-template)
34. [Common Recipes & Patterns](#34-common-recipes--patterns)
35. [Quick Reference Cheat Sheet](#35-quick-reference-cheat-sheet)

---

## 1. INTRODUCTION

### What is Python?
- **High-level**, **interpreted**, **general-purpose** programming language.
- Created by **Guido van Rossum** (1991).
- Uses **indentation** instead of braces for code blocks.
- **Dynamically typed** — no need to declare variable types.

### Why Python for Scripting?
- Easy to read and write
- Huge standard library — "batteries included"
- Cross-platform (Linux, macOS, Windows)
- Better error handling than shell scripts
- Great for automation, data processing, web scraping, APIs

### Running Python
```bash
# Check version
python3 --version

# Interactive shell
python3

# Run a script
python3 script.py

# Run one-liner
python3 -c "print('Hello')"

# Run as module
python3 -m http.server 8080
```

### Your First Script
```python
#!/usr/bin/env python3
"""My first Python script."""

print("Hello, World!")
```
```bash
chmod +x hello.py
./hello.py
```

### Shebang Line
```python
#!/usr/bin/env python3    # portable (recommended)
#!/usr/bin/python3        # absolute path
```

---

## 2. COMMENTS & DOCSTRINGS

```python
# Single-line comment

x = 10  # Inline comment

# Multi-line comment (just multiple single-lines)
# This is line 1
# This is line 2

"""
This is a docstring.
Used for documenting modules, classes, and functions.
Not technically a comment — it's a string literal.
"""

def greet(name):
    """Return a greeting message.

    Args:
        name: The person's name.

    Returns:
        A greeting string.
    """
    return f"Hello, {name}"

# Access docstring
print(greet.__doc__)
```

---

## 3. VARIABLES & DATA TYPES

### Variable Assignment
```python
# No declaration keyword needed
name = "Ritesh"         # str
age = 25                # int
height = 5.9            # float
is_active = True        # bool
value = None            # NoneType

# Multiple assignment
a, b, c = 1, 2, 3
x = y = z = 0

# Swap
a, b = b, a

# Type checking
type(name)         # <class 'str'>
isinstance(age, int)   # True
```

### Data Types Overview
| Type | Example | Mutable |
|------|---------|---------|
| `int` | `42`, `0xFF`, `0b1010`, `0o17` | No |
| `float` | `3.14`, `2.5e10`, `float('inf')` | No |
| `complex` | `3+4j` | No |
| `bool` | `True`, `False` | No |
| `str` | `"hello"`, `'world'`, `"""multi"""` | No |
| `bytes` | `b"hello"` | No |
| `bytearray` | `bytearray(b"hello")` | Yes |
| `list` | `[1, 2, 3]` | Yes |
| `tuple` | `(1, 2, 3)` | No |
| `set` | `{1, 2, 3}` | Yes |
| `frozenset` | `frozenset({1, 2, 3})` | No |
| `dict` | `{"a": 1, "b": 2}` | Yes |
| `NoneType` | `None` | - |

### Type Conversion
```python
int("42")            # 42
int(3.9)             # 3 (truncates)
float("3.14")        # 3.14
str(42)              # "42"
bool(0)              # False
bool("")             # False
bool([])             # False
bool(None)           # False
bool(1)              # True
bool("hello")        # True
list("abc")          # ['a', 'b', 'c']
tuple([1, 2, 3])     # (1, 2, 3)
set([1, 2, 2, 3])   # {1, 2, 3}
dict([("a", 1), ("b", 2)])  # {'a': 1, 'b': 2}
```

### Falsy Values
```python
# All of these evaluate to False:
False, None, 0, 0.0, 0j, "", [], (), {}, set(), frozenset(), range(0)
```

---

## 4. NUMBERS & ARITHMETIC

### Integer
```python
a = 42
b = 0xFF         # 255 (hex)
c = 0b1010       # 10 (binary)
d = 0o17         # 15 (octal)
e = 1_000_000    # 1000000 (underscores for readability)
# Python ints have unlimited precision — no overflow!
big = 10 ** 100  # works fine
```

### Float
```python
f = 3.14
g = 2.5e10       # 25000000000.0
h = float('inf') # infinity
i = float('-inf')
j = float('nan') # not a number

import math
math.isnan(j)    # True
math.isinf(h)    # True
```

### Operators
```python
# Arithmetic
a + b        # Addition
a - b        # Subtraction
a * b        # Multiplication
a / b        # Division (always returns float)
a // b       # Floor division (integer division)
a % b        # Modulus
a ** b       # Exponentiation
-a           # Negation

# Augmented assignment
a += 5       # a = a + 5
a -= 5
a *= 5
a /= 5
a //= 5
a %= 5
a **= 5

# Built-in functions
abs(-5)          # 5
round(3.7)       # 4
round(3.14159, 2)  # 3.14
pow(2, 10)       # 1024
pow(2, 10, 100)  # 24 (2^10 % 100)
divmod(17, 5)    # (3, 2) → (quotient, remainder)
min(1, 2, 3)     # 1
max(1, 2, 3)     # 3
sum([1, 2, 3])   # 6
```

### Math Module
```python
import math

math.pi          # 3.141592653589793
math.e           # 2.718281828459045
math.tau         # 6.283185307179586
math.inf         # infinity

math.sqrt(144)   # 12.0
math.floor(3.7)  # 3
math.ceil(3.2)   # 4
math.trunc(3.9)  # 3
math.factorial(5)  # 120
math.gcd(12, 8)  # 4
math.lcm(4, 6)   # 12  (Python 3.9+)
math.log(100)    # 4.605 (natural)
math.log10(100)  # 2.0
math.log2(8)     # 3.0
math.sin(math.pi/2)  # 1.0
math.cos(0)      # 1.0
math.degrees(math.pi)  # 180.0
math.radians(180)      # 3.14159...
math.isclose(0.1+0.2, 0.3)  # True
```

### Random Module
```python
import random

random.random()              # float [0.0, 1.0)
random.uniform(1, 10)        # float [1.0, 10.0]
random.randint(1, 100)       # int [1, 100]
random.randrange(0, 100, 5)  # random from range (step=5)
random.choice([1, 2, 3])     # random element
random.choices([1,2,3], k=5) # k random with replacement
random.sample([1,2,3,4,5], k=3)  # k random without replacement
random.shuffle(my_list)      # shuffle in-place
random.seed(42)              # set seed for reproducibility
```

---

## 5. STRINGS

### Creating Strings
```python
s1 = "Hello"
s2 = 'World'
s3 = """Multi
line string"""
s4 = '''Also
multi-line'''
s5 = r"raw\nstring"     # no escape processing
s6 = b"bytes string"    # bytes
s7 = f"formatted {s1}"  # f-string
```

### String Methods
```python
s = "Hello, World!"

# Case
s.upper()            # "HELLO, WORLD!"
s.lower()            # "hello, world!"
s.capitalize()       # "Hello, world!"
s.title()            # "Hello, World!"
s.swapcase()         # "hELLO, wORLD!"
s.casefold()         # "hello, world!" (aggressive lowercase, handles unicode)

# Search
s.find("World")      # 7 (index, or -1 if not found)
s.rfind("l")         # 10 (from right)
s.index("World")     # 7 (like find but raises ValueError)
s.rindex("l")        # 10
s.count("l")         # 3
s.startswith("Hell") # True
s.endswith("!")      # True
"World" in s         # True

# Replace & Modify
s.replace("World", "Python")   # "Hello, Python!"
s.replace("l", "L", 2)        # "HeLLo, World!" (max 2 replacements)

# Strip whitespace
"  hello  ".strip()     # "hello"
"  hello  ".lstrip()    # "hello  "
"  hello  ".rstrip()    # "  hello"
"xxhelloxx".strip("x")  # "hello"

# Split & Join
"a,b,c".split(",")          # ['a', 'b', 'c']
"hello world".split()       # ['hello', 'world'] (splits on whitespace)
"a,b,c".split(",", 1)       # ['a', 'b,c'] (max splits)
"a\nb\nc".splitlines()      # ['a', 'b', 'c']
",".join(["a", "b", "c"])   # "a,b,c"
" ".join(["Hello", "World"])  # "Hello World"

# Padding & Alignment
"hi".ljust(10)        # "hi        "
"hi".rjust(10)        # "        hi"
"hi".center(10)       # "    hi    "
"hi".ljust(10, '-')   # "hi--------"
"42".zfill(5)         # "00042"

# Check type
"abc".isalpha()       # True
"123".isdigit()       # True
"abc123".isalnum()    # True
"   ".isspace()       # True
"Hello".istitle()     # True
"HELLO".isupper()     # True
"hello".islower()     # True
"hello123".isidentifier()  # True (valid variable name?)
"print".iskeyword()   # False (use keyword.iskeyword())

# Encode/Decode
"hello".encode("utf-8")           # b'hello'
b"hello".decode("utf-8")          # "hello"

# String partition
"hello=world".partition("=")      # ('hello', '=', 'world')
"hello=world=foo".rpartition("=") # ('hello=world', '=', 'foo')

# Translate & maketrans
table = str.maketrans("aeiou", "12345")
"hello".translate(table)           # "h2ll4"

# Remove prefix/suffix (Python 3.9+)
"HelloWorld".removeprefix("Hello")   # "World"
"HelloWorld".removesuffix("World")   # "Hello"
```

### String Formatting

```python
name = "Ritesh"
age = 25

# 1. f-strings (Python 3.6+ — RECOMMENDED)
f"Name: {name}, Age: {age}"
f"Sum: {5 + 3}"
f"Upper: {name.upper()}"
f"Pi: {3.14159:.2f}"
f"Hex: {255:#x}"              # 0xff
f"Padded: {42:05d}"            # 00042
f"Left: {'hi':<10}"            # "hi        "
f"Right: {'hi':>10}"           # "        hi"
f"Center: {'hi':^10}"          # "    hi    "
f"Percent: {0.85:.1%}"         # "85.0%"
f"Comma: {1000000:,}"          # "1,000,000"
f"{'yes' if age >= 18 else 'no'}"  # conditional expression

# Debug format (Python 3.8+)
f"{name=}"                      # "name='Ritesh'"
f"{age=}, {name=}"              # "age=25, name='Ritesh'"

# 2. .format() method
"Name: {}, Age: {}".format(name, age)
"Name: {0}, Age: {1}".format(name, age)
"Name: {n}, Age: {a}".format(n=name, a=age)
"{:>10}".format("hi")          # "        hi"

# 3. % operator (old style — avoid)
"Name: %s, Age: %d" % (name, age)
"Pi: %.2f" % 3.14159
```

### Format Spec Mini-Language
```
{value:fill align sign # 0 width , .precision type}
```
| Part | Options |
|------|---------|
| fill | any character |
| align | `<` left, `>` right, `^` center, `=` pad after sign |
| sign | `+` always, `-` neg only (default), ` ` space for positive |
| `#` | alternate form (0x, 0o, 0b prefix) |
| `0` | zero-pad |
| width | minimum width |
| `,` | thousands separator |
| `.precision` | decimal places (float) or max length (string) |
| type | `d` int, `f` float, `e` scientific, `x` hex, `o` octal, `b` binary, `s` string, `%` percentage |

### Escape Characters
```python
\\    # Backslash
\'    # Single quote
\"    # Double quote
\n    # Newline
\t    # Tab
\r    # Carriage return
\b    # Backspace
\f    # Form feed
\0    # Null
\a    # Bell
\ooo  # Octal value
\xhh  # Hex value
\uxxxx   # Unicode 16-bit
\Uxxxxxxxx  # Unicode 32-bit
\N{name}    # Unicode by name: \N{snowman} → ☃
```

### Slicing
```python
s = "Hello, World!"
#     0123456789...

s[0]       # 'H'
s[-1]      # '!'
s[0:5]     # 'Hello'
s[7:]      # 'World!'
s[:5]      # 'Hello'
s[-6:]     # 'orld!'
s[::2]     # 'Hlo ol!'  (every 2nd char)
s[::-1]    # '!dlroW ,olleH'  (reverse)
s[1:10:2]  # 'el,Wr'
```

---

## 6. LISTS

### Creating Lists
```python
empty = []
nums = [1, 2, 3, 4, 5]
mixed = [1, "two", 3.0, True, None]
nested = [[1, 2], [3, 4]]
from_range = list(range(10))        # [0,1,2,...,9]
from_string = list("hello")        # ['h','e','l','l','o']
repeated = [0] * 5                  # [0, 0, 0, 0, 0]
```

### Accessing
```python
nums = [10, 20, 30, 40, 50]
nums[0]        # 10
nums[-1]       # 50
nums[1:3]      # [20, 30]
nums[::-1]     # [50, 40, 30, 20, 10]
```

### Modifying
```python
nums = [1, 2, 3]

# Add
nums.append(4)             # [1, 2, 3, 4]
nums.insert(1, 1.5)        # [1, 1.5, 2, 3, 4]
nums.extend([5, 6])        # [1, 1.5, 2, 3, 4, 5, 6]
nums += [7, 8]             # append via +

# Remove
nums.remove(1.5)           # removes first occurrence
popped = nums.pop()        # removes & returns last
popped = nums.pop(0)       # removes & returns index 0
del nums[0]                # delete by index
del nums[1:3]              # delete slice
nums.clear()               # remove all

# Modify
nums[0] = 99
nums[1:3] = [20, 30]      # replace slice
```

### List Methods
```python
nums = [3, 1, 4, 1, 5, 9, 2, 6]

nums.sort()                    # sort in-place → [1,1,2,3,4,5,6,9]
nums.sort(reverse=True)        # descending
nums.sort(key=len)             # sort by function
nums.sort(key=lambda x: x%3)  # sort by custom key

sorted_nums = sorted(nums)    # returns new sorted list
sorted(nums, reverse=True)
sorted(nums, key=abs)

nums.reverse()                 # reverse in-place
list(reversed(nums))           # returns new reversed

nums.index(4)                  # index of first 4
nums.index(4, 2)               # search from index 2
nums.count(1)                  # count occurrences
nums.copy()                    # shallow copy
```

### List Comprehension
```python
# Basic
squares = [x**2 for x in range(10)]           # [0,1,4,9,16,25,36,49,64,81]

# With condition
evens = [x for x in range(20) if x % 2 == 0]  # [0,2,4,6,...,18]

# With else
labels = ["even" if x%2==0 else "odd" for x in range(5)]

# Nested
matrix = [[1,2,3],[4,5,6],[7,8,9]]
flat = [x for row in matrix for x in row]      # [1,2,3,4,5,6,7,8,9]

# With function
words = ["hello", "world"]
upper = [w.upper() for w in words]

# Nested comprehension (2D)
matrix = [[i*3+j for j in range(3)] for i in range(3)]
# [[0,1,2],[3,4,5],[6,7,8]]
```

### Unpacking
```python
a, b, c = [1, 2, 3]

# Star unpacking
first, *rest = [1, 2, 3, 4, 5]    # first=1, rest=[2,3,4,5]
first, *mid, last = [1,2,3,4,5]   # first=1, mid=[2,3,4], last=5
*start, last = [1, 2, 3, 4]       # start=[1,2,3], last=4
```

### Useful Patterns
```python
# Check if list is empty
if not my_list:
    print("empty")

# Flatten nested list
import itertools
flat = list(itertools.chain.from_iterable(nested))

# Remove duplicates (preserving order)
unique = list(dict.fromkeys(my_list))

# Zip two lists
names = ["a", "b", "c"]
scores = [90, 80, 70]
pairs = list(zip(names, scores))   # [('a',90), ('b',80), ('c',70)]

# Enumerate
for i, val in enumerate(my_list):
    print(f"{i}: {val}")

for i, val in enumerate(my_list, start=1):  # start from 1
    print(f"{i}: {val}")

# Map and Filter
doubled = list(map(lambda x: x*2, nums))
evens = list(filter(lambda x: x%2==0, nums))

# any / all
any([False, False, True])    # True
all([True, True, True])      # True
all([True, False, True])     # False

# Chunk list
def chunks(lst, n):
    for i in range(0, len(lst), n):
        yield lst[i:i+n]
list(chunks([1,2,3,4,5,6,7], 3))  # [[1,2,3],[4,5,6],[7]]
```

---

## 7. TUPLES

```python
# Creating
empty = ()
single = (1,)          # comma is required for single-element tuple!
t = (1, 2, 3)
t = 1, 2, 3           # parentheses optional
t = tuple([1, 2, 3])

# Accessing (same as list)
t[0]         # 1
t[-1]        # 3
t[1:3]       # (2, 3)

# Methods
t.count(2)   # 1
t.index(3)   # 2

# Unpacking
a, b, c = (1, 2, 3)
a, *rest = (1, 2, 3, 4)

# Immutable — cannot modify
# t[0] = 99  → TypeError

# Named Tuple
from collections import namedtuple
Point = namedtuple('Point', ['x', 'y'])
p = Point(3, 4)
p.x          # 3
p.y          # 4
p[0]         # 3 (also indexable)

# When to use tuple vs list?
# Tuple: fixed data, dict keys, function returns, immutable collections
# List: dynamic data, need to modify, ordered collection
```

---

## 8. DICTIONARIES

### Creating
```python
empty = {}
d = {"name": "Ritesh", "age": 25, "city": "Delhi"}
d = dict(name="Ritesh", age=25)
d = dict([("name", "Ritesh"), ("age", 25)])
d = dict.fromkeys(["a", "b", "c"], 0)    # {'a':0, 'b':0, 'c':0}
```

### Accessing
```python
d = {"name": "Ritesh", "age": 25}

d["name"]              # "Ritesh"
d.get("name")          # "Ritesh"
d.get("phone", "N/A")  # "N/A" (default if missing)
# d["phone"]           # KeyError!

"name" in d            # True (checks keys)
"Ritesh" in d.values() # True (checks values)
```

### Modifying
```python
d["email"] = "r@example.com"    # add/update
d.update({"age": 26, "phone": "123"})  # bulk update
d.update(x=1, y=2)             # keyword args

# Remove
del d["phone"]                  # delete key (KeyError if missing)
val = d.pop("email")           # remove & return value
val = d.pop("missing", None)   # with default (no error)
key, val = d.popitem()          # remove & return last pair
d.clear()                       # remove all

# Setdefault — get or set
d.setdefault("country", "India")  # set if key doesn't exist, return value
```

### Iterating
```python
d = {"a": 1, "b": 2, "c": 3}

# Keys
for key in d:
    print(key)
for key in d.keys():
    print(key)

# Values
for val in d.values():
    print(val)

# Key-Value pairs
for key, val in d.items():
    print(f"{key}: {val}")
```

### Dict Comprehension
```python
squares = {x: x**2 for x in range(6)}
# {0:0, 1:1, 2:4, 3:9, 4:16, 5:25}

# Filter
evens = {k: v for k, v in d.items() if v % 2 == 0}

# Swap keys and values
inverted = {v: k for k, v in d.items()}

# From two lists
keys = ["a", "b", "c"]
vals = [1, 2, 3]
d = dict(zip(keys, vals))
```

### Merging Dicts
```python
a = {"x": 1, "y": 2}
b = {"y": 3, "z": 4}

# Python 3.9+ — merge operator
merged = a | b           # {'x':1, 'y':3, 'z':4}
a |= b                   # update a in-place

# Python 3.5+ — unpacking
merged = {**a, **b}

# Older
merged = a.copy()
merged.update(b)
```

### Useful Patterns
```python
# Count occurrences
from collections import Counter
counts = Counter("abracadabra")  # Counter({'a':5,'b':2,'r':2,'c':1,'d':1})
counts.most_common(3)            # [('a',5),('b',2),('r',2)]

# Default values
from collections import defaultdict
dd = defaultdict(list)
dd["fruits"].append("apple")     # no KeyError!
dd["fruits"].append("banana")

dd = defaultdict(int)
dd["count"] += 1                 # starts at 0

# Ordered Dict (maintains insertion order — default in Python 3.7+)
from collections import OrderedDict

# Pretty print
import json
print(json.dumps(d, indent=2))
```

---

## 9. SETS

```python
# Creating
s = {1, 2, 3}
s = set([1, 2, 2, 3])    # {1, 2, 3} — duplicates removed
empty = set()              # NOT {} (that's a dict!)

# Add/Remove
s.add(4)                   # add element
s.update([5, 6, 7])        # add multiple
s.remove(3)                # remove (KeyError if missing)
s.discard(99)              # remove (no error if missing)
s.pop()                    # remove & return arbitrary element
s.clear()                  # remove all

# Operations
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

a | b           # Union: {1,2,3,4,5,6}
a & b           # Intersection: {3,4}
a - b           # Difference: {1,2}
a ^ b           # Symmetric difference: {1,2,5,6}

a.union(b)
a.intersection(b)
a.difference(b)
a.symmetric_difference(b)

# In-place
a |= b          # a.update(b)
a &= b          # a.intersection_update(b)
a -= b          # a.difference_update(b)
a ^= b          # a.symmetric_difference_update(b)

# Tests
a.issubset(b)       # a <= b
a.issuperset(b)     # a >= b
a.isdisjoint(b)     # no common elements

# Set comprehension
evens = {x for x in range(20) if x % 2 == 0}

# Frozen set (immutable)
fs = frozenset([1, 2, 3])   # can be used as dict key or set element
```

---

## 10. CONTROL FLOW

### if / elif / else
```python
age = 18

if age < 13:
    print("child")
elif age < 18:
    print("teen")
elif age < 65:
    print("adult")
else:
    print("senior")

# One-liner (ternary)
status = "adult" if age >= 18 else "minor"

# Chained comparison
if 0 < age < 120:
    print("valid age")

# Truthy/Falsy
if my_list:        # True if non-empty
    print("has items")

# Walrus operator (Python 3.8+)
if (n := len(my_list)) > 5:
    print(f"List is long: {n} items")
```

### match / case (Python 3.10+)
```python
command = "start"

match command:
    case "start":
        print("Starting...")
    case "stop":
        print("Stopping...")
    case "restart":
        print("Restarting...")
    case _:
        print("Unknown command")

# With patterns
match point:
    case (0, 0):
        print("Origin")
    case (x, 0):
        print(f"On x-axis at {x}")
    case (0, y):
        print(f"On y-axis at {y}")
    case (x, y):
        print(f"At ({x}, {y})")

# With guard
match age:
    case n if n < 0:
        print("Invalid")
    case n if n < 18:
        print("Minor")
    case _:
        print("Adult")

# With class patterns
match event:
    case {"type": "click", "x": x, "y": y}:
        print(f"Click at ({x}, {y})")
    case {"type": "keypress", "key": key}:
        print(f"Key: {key}")
```

---

## 11. LOOPS

### for Loop
```python
# Over list
for item in [1, 2, 3]:
    print(item)

# Over string
for ch in "hello":
    print(ch)

# Over range
for i in range(5):            # 0,1,2,3,4
    print(i)
for i in range(2, 10):        # 2,3,...,9
    print(i)
for i in range(0, 20, 3):     # 0,3,6,9,12,15,18
    print(i)
for i in range(10, 0, -1):    # 10,9,...,1
    print(i)

# Over dict
for key in my_dict:
    print(key)
for key, val in my_dict.items():
    print(key, val)

# Enumerate
for i, val in enumerate(my_list):
    print(i, val)

# Zip
for a, b in zip(list1, list2):
    print(a, b)

# Nested
for i in range(3):
    for j in range(3):
        print(i, j)
```

### while Loop
```python
count = 0
while count < 5:
    print(count)
    count += 1

# Infinite loop
while True:
    user_input = input("Enter (q to quit): ")
    if user_input == "q":
        break
```

### Loop Control
```python
# break — exit loop
for i in range(10):
    if i == 5:
        break
    print(i)    # 0,1,2,3,4

# continue — skip iteration
for i in range(10):
    if i % 2 == 0:
        continue
    print(i)    # 1,3,5,7,9

# else clause (runs if loop completed WITHOUT break)
for i in range(5):
    if i == 99:
        break
else:
    print("Loop completed normally")   # this prints

for i in range(5):
    if i == 3:
        break
else:
    print("This won't print")          # skipped due to break

# pass — placeholder (do nothing)
for i in range(10):
    pass    # TODO: implement later
```

### Comprehensions (Loop shortcuts)
```python
# List comprehension
[x**2 for x in range(10)]

# Dict comprehension
{k: v for k, v in pairs}

# Set comprehension
{x % 3 for x in range(10)}

# Generator expression (lazy — doesn't create list in memory)
gen = (x**2 for x in range(1000000))
sum(x**2 for x in range(10))   # no brackets needed as function arg
```

---

## 12. FUNCTIONS

### Defining & Calling
```python
def greet(name):
    """Return a greeting."""
    return f"Hello, {name}!"

result = greet("Ritesh")   # "Hello, Ritesh!"
```

### Parameters & Arguments

```python
# Positional arguments
def add(a, b):
    return a + b
add(3, 5)

# Default values
def greet(name, greeting="Hello"):
    return f"{greeting}, {name}!"
greet("Ritesh")              # "Hello, Ritesh!"
greet("Ritesh", "Hi")        # "Hi, Ritesh!"

# Keyword arguments
greet(greeting="Hey", name="Ritesh")

# *args — variable positional arguments (tuple)
def total(*args):
    return sum(args)
total(1, 2, 3, 4)   # 10

# **kwargs — variable keyword arguments (dict)
def info(**kwargs):
    for k, v in kwargs.items():
        print(f"{k}: {v}")
info(name="Ritesh", age=25)

# Combined (ORDER MATTERS)
def func(pos, /, pos_or_kw, *, kw_only):
    pass
# pos         → positional-only (before /)
# pos_or_kw   → either way
# kw_only     → keyword-only (after *)

# Real example
def create_user(name, age, *, admin=False, active=True):
    pass
create_user("Ritesh", 25, admin=True)

# Positional-only (Python 3.8+)
def func(a, b, /, c, d):
    pass
func(1, 2, 3, d=4)      # OK
# func(a=1, b=2, 3, 4)  # Error — a, b must be positional

# Unpack into function
def add(a, b, c):
    return a + b + c
args = [1, 2, 3]
add(*args)               # unpack list
kwargs = {"a": 1, "b": 2, "c": 3}
add(**kwargs)             # unpack dict
```

### Return Values
```python
# Single value
def square(x):
    return x ** 2

# Multiple values (returns tuple)
def divide(a, b):
    return a // b, a % b
quotient, remainder = divide(17, 5)

# None (default return)
def greet(name):
    print(f"Hello, {name}")    # implicitly returns None
```

### Lambda (Anonymous Functions)
```python
square = lambda x: x ** 2
square(5)    # 25

add = lambda a, b: a + b

# Common with sort, map, filter
sorted(names, key=lambda x: x.lower())
list(map(lambda x: x*2, [1,2,3]))
list(filter(lambda x: x > 0, [-1, 0, 1, 2]))
```

### Scope
```python
x = "global"

def outer():
    x = "outer"

    def inner():
        x = "inner"
        print(x)       # inner

    inner()
    print(x)           # outer

print(x)               # global

# global keyword — modify global variable
count = 0
def increment():
    global count
    count += 1

# nonlocal keyword — modify enclosing function's variable
def outer():
    x = 10
    def inner():
        nonlocal x
        x = 20
    inner()
    print(x)    # 20
```

### Closures
```python
def make_multiplier(n):
    def multiplier(x):
        return x * n
    return multiplier

double = make_multiplier(2)
triple = make_multiplier(3)
double(5)    # 10
triple(5)    # 15
```

### Decorators
```python
# Basic decorator
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("Before")
        result = func(*args, **kwargs)
        print("After")
        return result
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

say_hello()
# Before
# Hello!
# After

# Decorator with arguments
def repeat(n):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for _ in range(n):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def greet():
    print("Hi!")

# Preserve function metadata
from functools import wraps

def my_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

# Common built-in decorators
@staticmethod
@classmethod
@property
@functools.lru_cache
@functools.wraps
@dataclasses.dataclass
```

### Generators
```python
# Generator function (uses yield)
def countdown(n):
    while n > 0:
        yield n
        n -= 1

for val in countdown(5):
    print(val)     # 5, 4, 3, 2, 1

# Generator expression
squares = (x**2 for x in range(10))
next(squares)    # 0
next(squares)    # 1

# yield from (delegate to sub-generator)
def chain(*iterables):
    for it in iterables:
        yield from it

list(chain([1,2], [3,4], [5,6]))   # [1,2,3,4,5,6]

# Two-way communication with send()
def accumulator():
    total = 0
    while True:
        value = yield total
        if value is None:
            break
        total += value

gen = accumulator()
next(gen)          # 0 (prime the generator)
gen.send(10)       # 10
gen.send(20)       # 30
gen.send(5)        # 35

# Use cases:
# - Large data processing (memory efficient)
# - Infinite sequences
# - Pipeline processing
# - Lazy evaluation
```

### Type Hints (Python 3.5+)
```python
def greet(name: str) -> str:
    return f"Hello, {name}"

def add(a: int, b: int = 0) -> int:
    return a + b

def process(items: list[str]) -> dict[str, int]:   # Python 3.9+
    return {item: len(item) for item in items}

# Complex types
from typing import Optional, Union, Any, Callable, Iterator

def find(items: list[int], target: int) -> Optional[int]:
    """Return index or None."""
    ...

def apply(func: Callable[[int], int], value: int) -> int:
    return func(value)

x: Union[int, str] = 42      # int or str
x: int | str = 42             # Python 3.10+ syntax
```

---

## 13. CLASSES & OOP

### Basic Class
```python
class Dog:
    # Class variable (shared by all instances)
    species = "Canis familiaris"

    # Constructor
    def __init__(self, name, age):
        # Instance variables
        self.name = name
        self.age = age

    # Instance method
    def bark(self):
        return f"{self.name} says Woof!"

    # String representation
    def __repr__(self):
        return f"Dog('{self.name}', {self.age})"

    def __str__(self):
        return f"{self.name} ({self.age} years)"

# Create instance
dog = Dog("Buddy", 5)
dog.bark()        # "Buddy says Woof!"
print(dog)        # "Buddy (5 years)"
Dog.species       # "Canis familiaris"
```

### Inheritance
```python
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        raise NotImplementedError

class Dog(Animal):
    def speak(self):
        return f"{self.name} says Woof!"

class Cat(Animal):
    def speak(self):
        return f"{self.name} says Meow!"

# Multiple inheritance
class FlyingDog(Dog, FlyingMixin):
    pass

# Check inheritance
isinstance(dog, Dog)      # True
isinstance(dog, Animal)   # True
issubclass(Dog, Animal)   # True

# Method Resolution Order
Dog.__mro__
# or
Dog.mro()

# super() — call parent method
class Puppy(Dog):
    def __init__(self, name, age, tricks):
        super().__init__(name, age)
        self.tricks = tricks
```

### Class Methods & Static Methods
```python
class MyClass:
    count = 0

    def __init__(self, name):
        self.name = name
        MyClass.count += 1

    # Instance method — gets self
    def greet(self):
        return f"Hello, {self.name}"

    # Class method — gets cls (class itself)
    @classmethod
    def get_count(cls):
        return cls.count

    @classmethod
    def from_string(cls, s):
        name = s.split("-")[0]
        return cls(name)    # alternative constructor

    # Static method — gets nothing (utility)
    @staticmethod
    def is_valid(name):
        return len(name) > 0

MyClass.get_count()           # 0
obj = MyClass.from_string("Ritesh-25")
MyClass.is_valid("test")      # True
```

### Properties
```python
class Circle:
    def __init__(self, radius):
        self._radius = radius    # convention: _ means "private"

    @property
    def radius(self):
        """Getter."""
        return self._radius

    @radius.setter
    def radius(self, value):
        """Setter with validation."""
        if value < 0:
            raise ValueError("Radius cannot be negative")
        self._radius = value

    @radius.deleter
    def radius(self):
        del self._radius

    @property
    def area(self):
        """Computed property (read-only)."""
        import math
        return math.pi * self._radius ** 2

c = Circle(5)
c.radius         # 5 (uses getter)
c.radius = 10    # uses setter
c.area           # 314.159... (computed)
# c.area = 100   # AttributeError — no setter
```

### Magic/Dunder Methods
```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    # String representation
    def __repr__(self):         return f"Vector({self.x}, {self.y})"
    def __str__(self):          return f"({self.x}, {self.y})"

    # Arithmetic
    def __add__(self, other):   return Vector(self.x+other.x, self.y+other.y)
    def __sub__(self, other):   return Vector(self.x-other.x, self.y-other.y)
    def __mul__(self, scalar):  return Vector(self.x*scalar, self.y*scalar)
    def __neg__(self):          return Vector(-self.x, -self.y)
    def __abs__(self):          return (self.x**2 + self.y**2) ** 0.5

    # Comparison
    def __eq__(self, other):    return self.x==other.x and self.y==other.y
    def __lt__(self, other):    return abs(self) < abs(other)
    def __le__(self, other):    return abs(self) <= abs(other)
    def __hash__(self):         return hash((self.x, self.y))

    # Container
    def __len__(self):          return 2
    def __getitem__(self, i):   return (self.x, self.y)[i]
    def __iter__(self):         yield self.x; yield self.y
    def __contains__(self, v):  return v in (self.x, self.y)

    # Callable
    def __call__(self, scale):  return Vector(self.x*scale, self.y*scale)

    # Bool
    def __bool__(self):         return self.x != 0 or self.y != 0

    # Context manager
    def __enter__(self):        return self
    def __exit__(self, *args):  pass

v1 = Vector(3, 4)
v2 = Vector(1, 2)
v3 = v1 + v2        # Vector(4, 6)
abs(v1)              # 5.0
v1(10)               # Vector(30, 40) — callable
```

### Common Dunder Methods Reference
| Method | Triggers |
|--------|----------|
| `__init__` | Constructor |
| `__del__` | Destructor |
| `__repr__` | `repr(obj)`, debugger |
| `__str__` | `str(obj)`, `print()` |
| `__format__` | `format()`, f-strings |
| `__bool__` | `bool(obj)`, truthiness |
| `__len__` | `len(obj)` |
| `__getitem__` | `obj[key]` |
| `__setitem__` | `obj[key] = val` |
| `__delitem__` | `del obj[key]` |
| `__contains__` | `x in obj` |
| `__iter__` | `for x in obj` |
| `__next__` | `next(obj)` |
| `__call__` | `obj()` |
| `__add__` | `obj + other` |
| `__eq__` | `obj == other` |
| `__lt__` | `obj < other` |
| `__hash__` | `hash(obj)` |
| `__enter__/__exit__` | `with obj:` |
| `__getattr__` | Access missing attribute |
| `__setattr__` | Set any attribute |

### Dataclasses (Python 3.7+)
```python
from dataclasses import dataclass, field

@dataclass
class Point:
    x: float
    y: float
    z: float = 0.0

p = Point(1.0, 2.0)
print(p)              # Point(x=1.0, y=2.0, z=0.0)
p == Point(1.0, 2.0)  # True (auto __eq__)

@dataclass(order=True)    # adds comparison methods
class Student:
    name: str
    grade: float
    courses: list = field(default_factory=list)  # mutable default

    def __post_init__(self):
        self.name = self.name.title()

# Frozen (immutable)
@dataclass(frozen=True)
class Color:
    r: int
    g: int
    b: int
```

### Abstract Classes
```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass

    @abstractmethod
    def perimeter(self):
        pass

    def description(self):    # concrete method
        return f"Shape with area {self.area()}"

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius

    def area(self):
        return 3.14159 * self.radius ** 2

    def perimeter(self):
        return 2 * 3.14159 * self.radius

# shape = Shape()  → TypeError (can't instantiate abstract)
c = Circle(5)      # OK
```

### Slots (Memory Optimization)
```python
class Point:
    __slots__ = ('x', 'y')    # restrict attributes, save memory

    def __init__(self, x, y):
        self.x = x
        self.y = y

# p.z = 3  → AttributeError (no __dict__)
```

### Enums
```python
from enum import Enum, auto

class Color(Enum):
    RED = 1
    GREEN = 2
    BLUE = 3

class Direction(Enum):
    NORTH = auto()    # 1
    SOUTH = auto()    # 2
    EAST = auto()     # 3
    WEST = auto()     # 4

Color.RED             # <Color.RED: 1>
Color.RED.name        # 'RED'
Color.RED.value       # 1
Color(1)              # Color.RED
Color['RED']          # Color.RED
list(Color)           # all members

# Iteration
for c in Color:
    print(c.name, c.value)
```

---

## 14. ERROR HANDLING

### try / except / else / finally
```python
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Cannot divide by zero!")
except (TypeError, ValueError) as e:
    print(f"Error: {e}")
except Exception as e:
    print(f"Unexpected error: {e}")
else:
    print("Success — no error occurred")   # runs only if NO exception
finally:
    print("Always runs — cleanup here")    # runs ALWAYS
```

### Raising Exceptions
```python
raise ValueError("Invalid input")
raise TypeError("Expected string")
raise RuntimeError("Something went wrong")

# Re-raise current exception
try:
    risky_operation()
except Exception:
    log_error()
    raise    # re-raise the same exception

# Raise from (chaining)
try:
    x = int("abc")
except ValueError as e:
    raise RuntimeError("Parse failed") from e
```

### Custom Exceptions
```python
class AppError(Exception):
    """Base exception for our app."""
    pass

class ValidationError(AppError):
    """Raised when validation fails."""
    def __init__(self, field, message):
        self.field = field
        self.message = message
        super().__init__(f"{field}: {message}")

class NotFoundError(AppError):
    """Raised when resource not found."""
    pass

# Usage
raise ValidationError("email", "Invalid format")

try:
    validate_user(data)
except ValidationError as e:
    print(f"Validation error on {e.field}: {e.message}")
except AppError as e:
    print(f"App error: {e}")
```

### Built-in Exception Hierarchy
```
BaseException
├── KeyboardInterrupt
├── SystemExit
├── GeneratorExit
└── Exception
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   ├── OverflowError
    │   └── FloatingPointError
    ├── LookupError
    │   ├── IndexError
    │   └── KeyError
    ├── OSError (IOError, FileNotFoundError, PermissionError, ...)
    ├── ValueError
    ├── TypeError
    ├── AttributeError
    ├── NameError
    ├── ImportError (ModuleNotFoundError)
    ├── RuntimeError (RecursionError, NotImplementedError)
    ├── StopIteration
    ├── AssertionError
    └── UnicodeError
```

### Assertions
```python
# Debug-time checks (disabled with python3 -O)
assert x > 0, "x must be positive"
assert isinstance(name, str), "name must be a string"
assert len(items) > 0, "items cannot be empty"

# DON'T use for input validation — use exceptions
```

### Context Managers (with statement)
```python
# Automatic resource cleanup
with open("file.txt", "r") as f:
    content = f.read()
# f is automatically closed

# Multiple context managers
with open("in.txt") as fin, open("out.txt", "w") as fout:
    fout.write(fin.read())

# Custom context manager (class)
class Timer:
    def __enter__(self):
        import time
        self.start = time.time()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        import time
        self.elapsed = time.time() - self.start
        print(f"Elapsed: {self.elapsed:.2f}s")
        return False    # don't suppress exceptions

with Timer() as t:
    # do work
    pass

# Custom context manager (generator)
from contextlib import contextmanager

@contextmanager
def temp_dir():
    import tempfile, shutil
    d = tempfile.mkdtemp()
    try:
        yield d
    finally:
        shutil.rmtree(d)

with temp_dir() as d:
    print(f"Working in {d}")
```

---

## 15. FILE HANDLING

### Reading Files
```python
# Method 1: with statement (recommended)
with open("file.txt", "r") as f:
    content = f.read()           # entire file as string

with open("file.txt", "r") as f:
    lines = f.readlines()        # list of lines (with \n)

with open("file.txt", "r") as f:
    for line in f:               # line by line (memory efficient)
        print(line.strip())

with open("file.txt", "r") as f:
    first_line = f.readline()    # single line
```

### Writing Files
```python
# Write (overwrite)
with open("file.txt", "w") as f:
    f.write("Hello\n")
    f.write("World\n")

# Write list
with open("file.txt", "w") as f:
    lines = ["line1\n", "line2\n", "line3\n"]
    f.writelines(lines)

# Append
with open("file.txt", "a") as f:
    f.write("Appended line\n")

# Print to file
with open("file.txt", "w") as f:
    print("Hello", file=f)
    print("World", file=f)
```

### File Modes
| Mode | Description |
|------|-------------|
| `r` | Read (default) |
| `w` | Write (creates/truncates) |
| `a` | Append |
| `x` | Exclusive create (fail if exists) |
| `b` | Binary mode (add to others: `rb`, `wb`) |
| `t` | Text mode (default) |
| `+` | Read and write (`r+`, `w+`, `a+`) |

### Binary Files
```python
# Read binary
with open("image.png", "rb") as f:
    data = f.read()

# Write binary
with open("copy.png", "wb") as f:
    f.write(data)
```

### File Object Methods
```python
f.read()          # read all
f.read(100)       # read 100 bytes/chars
f.readline()      # read one line
f.readlines()     # read all lines as list
f.write(s)        # write string
f.writelines(lst) # write list of strings
f.seek(0)         # move to position
f.tell()          # current position
f.flush()         # flush buffer
f.close()         # close file
f.closed          # check if closed
f.name            # filename
f.mode            # file mode
```

### os & os.path Module
```python
import os

# Current directory
os.getcwd()                    # get current dir
os.chdir("/path/to/dir")      # change dir

# List files
os.listdir(".")                # list directory
os.scandir(".")                # iterator with DirEntry objects

# Create
os.mkdir("newdir")             # single directory
os.makedirs("a/b/c", exist_ok=True)   # recursive

# Delete
os.remove("file.txt")         # delete file
os.unlink("file.txt")         # same as remove
os.rmdir("emptydir")          # delete empty dir

# Rename/Move
os.rename("old.txt", "new.txt")
os.replace("src", "dst")      # atomic replace

# File info
os.path.exists("file.txt")    # True/False
os.path.isfile("file.txt")    # is regular file?
os.path.isdir("path")         # is directory?
os.path.islink("path")        # is symlink?
os.path.getsize("file.txt")   # size in bytes

# Path manipulation
os.path.join("dir", "file.txt")     # "dir/file.txt"
os.path.dirname("/a/b/c.txt")       # "/a/b"
os.path.basename("/a/b/c.txt")      # "c.txt"
os.path.splitext("file.tar.gz")     # ('file.tar', '.gz')
os.path.split("/a/b/c.txt")         # ('/a/b', 'c.txt')
os.path.abspath("file.txt")         # absolute path
os.path.expanduser("~/docs")        # expand ~
os.path.realpath("symlink")         # resolve symlinks

# Environment
os.environ["HOME"]             # get env var
os.environ.get("VAR", "default")
os.getenv("VAR", "default")

# Walk directory tree
for dirpath, dirnames, filenames in os.walk("."):
    for filename in filenames:
        filepath = os.path.join(dirpath, filename)
        print(filepath)
```

### pathlib Module (Modern — Python 3.4+)
```python
from pathlib import Path

# Create path
p = Path("dir/file.txt")
p = Path.home() / "documents" / "file.txt"
p = Path.cwd()

# Properties
p.name           # "file.txt"
p.stem           # "file"
p.suffix         # ".txt"
p.suffixes       # ['.tar', '.gz'] for file.tar.gz
p.parent         # Path("dir")
p.parents        # all parents
p.parts          # ('dir', 'file.txt')
p.anchor         # "/" on Unix, "C:\\" on Windows

# Tests
p.exists()
p.is_file()
p.is_dir()
p.is_symlink()
p.is_absolute()

# Read/Write
content = p.read_text()                   # read entire file
content = p.read_text(encoding="utf-8")
p.write_text("hello")
data = p.read_bytes()
p.write_bytes(data)

# Operations
p.mkdir(parents=True, exist_ok=True)
p.rmdir()           # remove empty dir
p.unlink()           # delete file
p.rename("new.txt")
p.replace("dest.txt")
p.touch()            # create empty / update timestamp

# Glob
list(Path(".").glob("*.txt"))          # all .txt files
list(Path(".").glob("**/*.py"))        # recursive
list(Path(".").rglob("*.py"))          # same as above

# Stat
info = p.stat()
info.st_size
info.st_mtime

# Resolve
p.resolve()          # absolute path
p.expanduser()       # expand ~

# Iterate directory
for child in Path(".").iterdir():
    print(child)
```

### shutil Module (High-level File Operations)
```python
import shutil

shutil.copy("src.txt", "dst.txt")        # copy file
shutil.copy2("src.txt", "dst.txt")       # copy with metadata
shutil.copytree("src_dir", "dst_dir")    # copy directory tree
shutil.move("src", "dst")                # move/rename
shutil.rmtree("dir")                     # delete directory tree

shutil.disk_usage("/")                   # disk usage (total, used, free)
shutil.which("python3")                  # find executable
shutil.make_archive("backup", "zip", "src_dir")  # create archive
shutil.unpack_archive("backup.zip", "dst_dir")   # extract
```

### tempfile Module
```python
import tempfile

# Temporary file (auto-deleted)
with tempfile.NamedTemporaryFile(mode='w', suffix='.txt', delete=True) as f:
    f.write("temp data")
    print(f.name)    # path to temp file

# Temporary directory
with tempfile.TemporaryDirectory() as tmpdir:
    print(tmpdir)    # auto-deleted when block exits

# Get temp dir path
tempfile.gettempdir()    # e.g., /tmp
```

---

## 16. MODULES & PACKAGES

### Importing
```python
# Import module
import os
import sys
import os.path

# Import specific items
from os import path, listdir
from os.path import join, exists
from math import pi, sqrt

# Import with alias
import numpy as np
from collections import defaultdict as dd

# Import all (avoid in general)
from math import *

# Relative import (inside packages)
from . import sibling_module
from .. import parent_module
from .utils import helper
```

### Creating Modules
```python
# utils.py
"""Utility functions."""

PI = 3.14159

def add(a, b):
    return a + b

def greet(name):
    return f"Hello, {name}"

class Calculator:
    pass

# main.py
import utils
utils.add(3, 5)
utils.PI
```

### Creating Packages
```
mypackage/
├── __init__.py      # makes it a package (can be empty)
├── module1.py
├── module2.py
└── subpackage/
    ├── __init__.py
    └── module3.py
```

```python
# __init__.py — controls what's exported
from .module1 import func1
from .module2 import func2

__all__ = ['func1', 'func2']   # controls * import

# Usage
from mypackage import func1
from mypackage.subpackage import module3
```

### `if __name__ == "__main__"`
```python
def main():
    print("Running as script")

if __name__ == "__main__":
    main()
# This block runs ONLY when the file is executed directly,
# NOT when imported as a module.
```

### Package Management (pip)
```bash
pip install requests              # install
pip install requests==2.28.0      # specific version
pip install "requests>=2.20"      # version constraint
pip install -r requirements.txt   # from file
pip uninstall requests
pip list                          # installed packages
pip show requests                 # package info
pip freeze > requirements.txt     # export
pip install --upgrade requests    # upgrade
pip install -e .                  # install in editable mode
```

### Virtual Environments
```bash
# Create
python3 -m venv myenv

# Activate
source myenv/bin/activate    # Linux/Mac
myenv\Scripts\activate       # Windows

# Deactivate
deactivate

# Alternative: using virtualenv
pip install virtualenv
virtualenv myenv
```

---

## 17. REGULAR EXPRESSIONS

```python
import re

text = "My phone is 123-456-7890 and email is user@example.com"
```

### Basic Functions
```python
# Search — find first match anywhere
match = re.search(r'\d{3}-\d{3}-\d{4}', text)
if match:
    print(match.group())    # "123-456-7890"
    print(match.start())    # start index
    print(match.end())      # end index
    print(match.span())     # (start, end)

# Match — match at BEGINNING only
match = re.match(r'My', text)

# Fullmatch — match ENTIRE string
re.fullmatch(r'\d+', "12345")

# Find all
phones = re.findall(r'\d{3}-\d{3}-\d{4}', text)   # list of strings

# Find all with groups
pairs = re.findall(r'(\d{3})-(\d{3})-(\d{4})', text)  # list of tuples

# Find iterator
for m in re.finditer(r'\d+', text):
    print(m.group(), m.span())

# Replace
result = re.sub(r'\d', '*', text)            # replace digits with *
result = re.sub(r'(\w+)@(\w+)', r'\2@\1', text)  # backreference
result = re.sub(r'\d+', lambda m: str(int(m.group())*2), text)  # function

# Replace with count
result = re.sub(r'\d', '*', text, count=3)   # first 3 only

# Split
parts = re.split(r'[,;\s]+', "a, b; c d")   # ['a','b','c','d']
parts = re.split(r'(\s+)', "hello world")    # keep separator with groups

# Compile for reuse
pattern = re.compile(r'\d{3}-\d{3}-\d{4}')
pattern.search(text)
pattern.findall(text)
```

### Regex Syntax Reference
| Pattern | Matches |
|---------|---------|
| `.` | Any char (except newline) |
| `\d` | Digit `[0-9]` |
| `\D` | Non-digit |
| `\w` | Word char `[a-zA-Z0-9_]` |
| `\W` | Non-word char |
| `\s` | Whitespace `[\t\n\r\f\v ]` |
| `\S` | Non-whitespace |
| `\b` | Word boundary |
| `^` | Start of string (or line with MULTILINE) |
| `$` | End of string (or line) |
| `*` | 0 or more |
| `+` | 1 or more |
| `?` | 0 or 1 |
| `{n}` | Exactly n |
| `{n,m}` | Between n and m |
| `{n,}` | n or more |
| `[abc]` | Character class |
| `[^abc]` | Negated class |
| `[a-z]` | Range |
| `(...)` | Capture group |
| `(?:...)` | Non-capture group |
| `(?P<name>...)` | Named group |
| `\1`, `\2` | Backreference |
| `(?=...)` | Lookahead (positive) |
| `(?!...)` | Lookahead (negative) |
| `(?<=...)` | Lookbehind (positive) |
| `(?<!...)` | Lookbehind (negative) |
| `a\|b` | Alternation (a or b) |
| `(?i)` | Inline case-insensitive flag |

### Flags
```python
re.IGNORECASE  # or re.I — case-insensitive
re.MULTILINE   # or re.M — ^ and $ match line boundaries
re.DOTALL      # or re.S — . matches newline
re.VERBOSE     # or re.X — allow comments in pattern
re.ASCII       # or re.A — ASCII-only matching

# Usage
re.search(r'hello', text, re.IGNORECASE)

# Combine flags
re.search(r'hello', text, re.I | re.M)

# Verbose pattern (for readability)
pattern = re.compile(r"""
    (\d{3})     # area code
    [-.\s]?     # separator
    (\d{3})     # prefix
    [-.\s]?     # separator
    (\d{4})     # line number
""", re.VERBOSE)
```

### Named Groups
```python
m = re.search(r'(?P<area>\d{3})-(?P<prefix>\d{3})-(?P<line>\d{4})', text)
m.group('area')      # '123'
m.group('prefix')    # '456'
m.groupdict()        # {'area': '123', 'prefix': '456', 'line': '7890'}
```

---

## 18. SUBPROCESS — RUNNING SYSTEM COMMANDS

```python
import subprocess
```

### subprocess.run() (Recommended — Python 3.5+)
```python
# Simple command
result = subprocess.run(["ls", "-la"], capture_output=True, text=True)
result.stdout         # output as string
result.stderr         # errors as string
result.returncode     # exit code

# Check for errors
result = subprocess.run(["ls", "/nonexistent"], capture_output=True, text=True, check=True)
# raises CalledProcessError if returncode != 0

# With shell
result = subprocess.run("echo $HOME", shell=True, capture_output=True, text=True)

# Input
result = subprocess.run(["grep", "hello"], input="hello world\nfoo bar\n",
                        capture_output=True, text=True)

# Timeout
result = subprocess.run(["sleep", "10"], timeout=5)  # TimeoutExpired

# Working directory
result = subprocess.run(["ls"], cwd="/tmp", capture_output=True, text=True)

# Environment
import os
env = os.environ.copy()
env["MY_VAR"] = "value"
result = subprocess.run(["printenv", "MY_VAR"], env=env, capture_output=True, text=True)
```

### Piping Commands
```python
# Pipe: ls | grep ".py"
p1 = subprocess.Popen(["ls"], stdout=subprocess.PIPE)
p2 = subprocess.Popen(["grep", ".py"], stdin=p1.stdout, stdout=subprocess.PIPE, text=True)
p1.stdout.close()
output = p2.communicate()[0]

# Or use shell=True (less safe)
result = subprocess.run("ls | grep .py", shell=True, capture_output=True, text=True)
```

### subprocess.Popen (Lower-level Control)
```python
proc = subprocess.Popen(
    ["python3", "server.py"],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True
)

# Communicate
stdout, stderr = proc.communicate(input="data", timeout=30)

# Poll
while proc.poll() is None:
    # process still running
    line = proc.stdout.readline()

# Kill
proc.terminate()    # SIGTERM
proc.kill()         # SIGKILL
proc.wait()         # wait for exit

# Return code
proc.returncode
```

### Security Note
```python
# NEVER pass user input directly to shell=True
# BAD (command injection vulnerability):
subprocess.run(f"echo {user_input}", shell=True)

# GOOD (safe):
subprocess.run(["echo", user_input])
```

---

## 19. SYSTEM & OS INTERACTION

### sys Module
```python
import sys

sys.argv              # command-line arguments list
sys.argv[0]           # script name
sys.exit(0)           # exit with code
sys.exit("Error msg") # exit with message to stderr

sys.version           # Python version string
sys.version_info      # (major, minor, micro, ...)
sys.platform          # 'linux', 'darwin', 'win32'
sys.executable        # path to Python binary
sys.path              # module search path
sys.modules           # loaded modules dict

sys.stdin             # standard input
sys.stdout            # standard output
sys.stderr            # standard error

sys.maxsize           # max int for platform
sys.float_info        # float limits
sys.getrecursionlimit()
sys.setrecursionlimit(10000)
sys.getsizeof(obj)    # memory size in bytes
```

### Argument Parsing — argparse
```python
import argparse

parser = argparse.ArgumentParser(
    description="Process some files.",
    epilog="Example: %(prog)s -v input.txt"
)

# Positional argument
parser.add_argument("filename", help="Input file")
parser.add_argument("output", nargs="?", default="out.txt")  # optional positional

# Optional arguments
parser.add_argument("-v", "--verbose", action="store_true", help="Verbose output")
parser.add_argument("-n", "--number", type=int, default=10, help="Number of items")
parser.add_argument("-f", "--format", choices=["json", "csv", "xml"], default="json")
parser.add_argument("-o", "--output", type=argparse.FileType('w'), default=sys.stdout)
parser.add_argument("--tags", nargs="+", help="One or more tags")
parser.add_argument("--coords", nargs=2, type=float, metavar=("X", "Y"))
parser.add_argument("--version", action="version", version="%(prog)s 1.0")

# Mutually exclusive
group = parser.add_mutually_exclusive_group()
group.add_argument("--start", action="store_true")
group.add_argument("--stop", action="store_true")

# Parse
args = parser.parse_args()
print(args.filename)
print(args.verbose)
print(args.number)

# Usage: python script.py input.txt -v -n 20 --format csv --tags a b c
```

### Signal Handling
```python
import signal
import sys

def handler(signum, frame):
    print(f"\nReceived signal {signum}")
    sys.exit(0)

signal.signal(signal.SIGINT, handler)    # Ctrl+C
signal.signal(signal.SIGTERM, handler)   # kill

# Ignore signal
signal.signal(signal.SIGINT, signal.SIG_IGN)

# Reset to default
signal.signal(signal.SIGINT, signal.SIG_DFL)

# Alarm (Unix only)
signal.alarm(30)    # SIGALRM after 30 seconds
```

### Platform Info
```python
import platform

platform.system()          # 'Linux', 'Darwin', 'Windows'
platform.release()         # kernel version
platform.version()         # detailed version
platform.machine()         # 'x86_64', 'arm64'
platform.processor()       # processor type
platform.node()            # hostname
platform.python_version()  # '3.11.5'
platform.uname()           # all system info
platform.platform()        # human-readable
```

---

## 20. JSON

```python
import json

# Python → JSON string
data = {"name": "Ritesh", "age": 25, "skills": ["Python", "Linux"]}
json_str = json.dumps(data)                          # compact
json_str = json.dumps(data, indent=2)                # pretty
json_str = json.dumps(data, sort_keys=True)          # sorted keys
json_str = json.dumps(data, ensure_ascii=False)      # allow unicode
json_str = json.dumps(data, separators=(',', ':'))   # compact separators

# JSON string → Python
data = json.loads(json_str)

# Write to file
with open("data.json", "w") as f:
    json.dump(data, f, indent=2)

# Read from file
with open("data.json", "r") as f:
    data = json.load(f)

# Custom encoder
class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, set):
            return list(obj)
        if isinstance(obj, datetime):
            return obj.isoformat()
        return super().default(obj)

json.dumps(data, cls=CustomEncoder)

# Type mapping
# Python        → JSON
# dict          → object {}
# list, tuple   → array []
# str           → string ""
# int, float    → number
# True          → true
# False         → false
# None          → null
```

---

## 21. CSV

```python
import csv

# Read CSV
with open("data.csv", "r") as f:
    reader = csv.reader(f)
    header = next(reader)         # skip header
    for row in reader:
        print(row)                # list of strings

# Read as dict
with open("data.csv", "r") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row["name"], row["age"])

# Write CSV
with open("output.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["name", "age", "city"])       # one row
    writer.writerows([                              # multiple
        ["Alice", 25, "NY"],
        ["Bob", 30, "LA"],
    ])

# Write dict
with open("output.csv", "w", newline="") as f:
    fields = ["name", "age", "city"]
    writer = csv.DictWriter(f, fieldnames=fields)
    writer.writeheader()
    writer.writerow({"name": "Alice", "age": 25, "city": "NY"})

# Custom delimiter
reader = csv.reader(f, delimiter='\t')       # TSV
reader = csv.reader(f, delimiter='|')

# Handle quoting
writer = csv.writer(f, quoting=csv.QUOTE_ALL)
```

---

## 22. DATETIME

```python
from datetime import datetime, date, time, timedelta, timezone

# Current date/time
now = datetime.now()                    # local
utc_now = datetime.now(timezone.utc)    # UTC
today = date.today()

# Create
dt = datetime(2026, 4, 14, 10, 30, 0)
d = date(2026, 4, 14)
t = time(10, 30, 0)

# Access components
dt.year, dt.month, dt.day
dt.hour, dt.minute, dt.second, dt.microsecond
dt.weekday()       # 0=Monday, 6=Sunday
dt.isoweekday()    # 1=Monday, 7=Sunday

# Format → String
dt.strftime("%Y-%m-%d %H:%M:%S")    # "2026-04-14 10:30:00"
dt.strftime("%B %d, %Y")             # "April 14, 2026"
dt.isoformat()                        # "2026-04-14T10:30:00"

# String → Datetime
dt = datetime.strptime("2026-04-14", "%Y-%m-%d")
dt = datetime.fromisoformat("2026-04-14T10:30:00")

# Format codes
# %Y  year (4 digit)     %y  year (2 digit)
# %m  month (01-12)      %B  month name       %b  abbreviated
# %d  day (01-31)
# %H  hour 24h           %I  hour 12h         %p  AM/PM
# %M  minute             %S  second
# %A  weekday name       %a  abbreviated
# %j  day of year        %U  week number (Sun)  %W  week number (Mon)
# %Z  timezone name      %z  UTC offset
# %%  literal %

# Arithmetic
delta = timedelta(days=30, hours=5, minutes=30)
future = dt + delta
past = dt - delta
diff = datetime(2026, 12, 31) - datetime(2026, 1, 1)
diff.days              # 364
diff.total_seconds()   # total seconds

# Compare
dt1 < dt2
dt1 == dt2

# Timestamp
ts = dt.timestamp()              # datetime → unix timestamp
dt = datetime.fromtimestamp(ts)  # unix timestamp → datetime
```

---

## 23. LOGGING

```python
import logging

# Basic setup
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
    filename="app.log",      # omit for console output
    filemode="a"
)

# Log levels (lowest → highest)
logging.debug("Detailed diagnostic info")
logging.info("General operational info")
logging.warning("Something unexpected")
logging.error("Error occurred")
logging.critical("System is unusable")

# Logger with name
logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)

# Multiple handlers
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.INFO)

file_handler = logging.FileHandler("app.log")
file_handler.setLevel(logging.DEBUG)

formatter = logging.Formatter("%(asctime)s [%(levelname)s] %(message)s")
console_handler.setFormatter(formatter)
file_handler.setFormatter(formatter)

logger.addHandler(console_handler)
logger.addHandler(file_handler)

# Log exception with traceback
try:
    1/0
except Exception:
    logger.exception("Division failed")    # includes traceback
    # or
    logger.error("Failed", exc_info=True)

# Rotating log files
from logging.handlers import RotatingFileHandler, TimedRotatingFileHandler

handler = RotatingFileHandler("app.log", maxBytes=5*1024*1024, backupCount=3)
handler = TimedRotatingFileHandler("app.log", when="midnight", backupCount=7)
```

---

## 24. CONCURRENCY

### Threading
```python
import threading

def worker(name, seconds):
    import time
    print(f"[{name}] Starting")
    time.sleep(seconds)
    print(f"[{name}] Done")

# Create and start threads
t1 = threading.Thread(target=worker, args=("A", 2))
t2 = threading.Thread(target=worker, args=("B", 3))
t1.start()
t2.start()
t1.join()     # wait for completion
t2.join()

# Daemon thread (dies with main thread)
t = threading.Thread(target=worker, args=("D", 5), daemon=True)

# Lock
lock = threading.Lock()
def safe_increment():
    with lock:
        # only one thread at a time
        shared_counter += 1

# Thread pool
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(worker, f"T{i}", 1) for i in range(10)]
    for future in futures:
        result = future.result()     # blocks until done
```

### Multiprocessing
```python
import multiprocessing

def cpu_task(n):
    return sum(i*i for i in range(n))

# Single process
p = multiprocessing.Process(target=cpu_task, args=(1000000,))
p.start()
p.join()

# Process pool
from concurrent.futures import ProcessPoolExecutor

with ProcessPoolExecutor(max_workers=4) as executor:
    results = executor.map(cpu_task, [10**6, 10**7, 10**8])
    for r in results:
        print(r)
```

### Asyncio (Python 3.5+)
```python
import asyncio

async def fetch_data(url, delay):
    print(f"Fetching {url}")
    await asyncio.sleep(delay)    # non-blocking sleep
    return f"Data from {url}"

async def main():
    # Run concurrently
    tasks = [
        asyncio.create_task(fetch_data("url1", 2)),
        asyncio.create_task(fetch_data("url2", 1)),
        asyncio.create_task(fetch_data("url3", 3)),
    ]
    results = await asyncio.gather(*tasks)
    print(results)

asyncio.run(main())

# Async context manager
async with aiohttp.ClientSession() as session:
    async with session.get(url) as response:
        data = await response.json()

# Async generator
async def async_range(n):
    for i in range(n):
        await asyncio.sleep(0.1)
        yield i

async for val in async_range(10):
    print(val)
```

---

## 25. NETWORKING

### HTTP — requests Library
```python
import requests

# GET
response = requests.get("https://api.example.com/data")
response.status_code    # 200
response.text           # response body as string
response.json()         # parse JSON response
response.headers        # response headers
response.cookies

# With parameters
response = requests.get("https://api.example.com/search", params={"q": "python"})

# POST
response = requests.post("https://api.example.com/users",
    json={"name": "Ritesh", "age": 25}
)

# Headers
response = requests.get(url, headers={"Authorization": "Bearer TOKEN"})

# Form data
response = requests.post(url, data={"username": "user", "password": "pass"})

# Upload file
with open("file.txt", "rb") as f:
    response = requests.post(url, files={"file": f})

# Session (persistent cookies)
session = requests.Session()
session.headers.update({"Authorization": "Bearer TOKEN"})
response = session.get(url)

# Timeout
response = requests.get(url, timeout=10)

# Error handling
response.raise_for_status()   # raises HTTPError for 4xx/5xx
```

### Socket Programming
```python
import socket

# TCP Client
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.connect(("example.com", 80))
    s.sendall(b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n")
    data = s.recv(4096)
    print(data.decode())

# TCP Server
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind(("0.0.0.0", 8080))
    s.listen(5)
    while True:
        conn, addr = s.accept()
        with conn:
            data = conn.recv(1024)
            conn.sendall(b"HTTP/1.1 200 OK\r\n\r\nHello")

# UDP
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.sendto(b"Hello", ("example.com", 12345))
data, addr = sock.recvfrom(1024)

# DNS lookup
socket.gethostbyname("example.com")    # IP address
socket.getfqdn()                         # fully qualified domain name
```

### Built-in HTTP Server
```bash
python3 -m http.server 8080                    # serve current dir
python3 -m http.server 8080 --directory /path  # serve specific dir
python3 -m http.server 8080 --bind 127.0.0.1   # bind to localhost
```

### Email
```python
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

msg = MIMEMultipart()
msg["From"] = "sender@example.com"
msg["To"] = "receiver@example.com"
msg["Subject"] = "Test Email"
msg.attach(MIMEText("Hello from Python!", "plain"))

with smtplib.SMTP("smtp.gmail.com", 587) as server:
    server.starttls()
    server.login("user", "password")
    server.send_message(msg)
```

---

## 26. DATABASE

### SQLite (Built-in)
```python
import sqlite3

# Connect (creates file if not exists)
conn = sqlite3.connect("mydb.db")
# OR in-memory
conn = sqlite3.connect(":memory:")

cursor = conn.cursor()

# Create table
cursor.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE,
        age INTEGER
    )
""")

# Insert — use parameterized queries (prevents SQL injection)
cursor.execute("INSERT INTO users (name, email, age) VALUES (?, ?, ?)",
               ("Ritesh", "r@example.com", 25))

# Insert many
users = [("Alice", "a@ex.com", 30), ("Bob", "b@ex.com", 28)]
cursor.executemany("INSERT INTO users (name, email, age) VALUES (?, ?, ?)", users)

# Commit
conn.commit()

# Select
cursor.execute("SELECT * FROM users WHERE age > ?", (20,))
rows = cursor.fetchall()       # list of tuples
row = cursor.fetchone()        # single row
rows = cursor.fetchmany(5)    # n rows

# Row as dict
conn.row_factory = sqlite3.Row
cursor = conn.cursor()
cursor.execute("SELECT * FROM users")
for row in cursor:
    print(row["name"], row["email"])

# Update
cursor.execute("UPDATE users SET age = ? WHERE name = ?", (26, "Ritesh"))

# Delete
cursor.execute("DELETE FROM users WHERE name = ?", ("Bob",))

# Using context manager
with sqlite3.connect("mydb.db") as conn:
    conn.execute("INSERT INTO users (name, email, age) VALUES (?, ?, ?)",
                 ("Charlie", "c@ex.com", 22))
    # auto-commits on success, auto-rollbacks on exception

# Close
conn.close()
```

---

## 27. ITERTOOLS

```python
import itertools

# Infinite iterators
itertools.count(10, 2)             # 10, 12, 14, 16, ...
itertools.cycle([1, 2, 3])        # 1, 2, 3, 1, 2, 3, ...
itertools.repeat("hello", 3)      # "hello", "hello", "hello"

# Combinatoric
list(itertools.product([1,2], [3,4]))        # [(1,3),(1,4),(2,3),(2,4)]
list(itertools.permutations([1,2,3], 2))     # all 2-length permutations
list(itertools.combinations([1,2,3,4], 2))   # all 2-length combinations
list(itertools.combinations_with_replacement([1,2,3], 2))

# Chain (combine iterables)
list(itertools.chain([1,2], [3,4], [5,6]))   # [1,2,3,4,5,6]
list(itertools.chain.from_iterable([[1,2],[3,4]]))

# Group by
data = [("A", 1), ("A", 2), ("B", 3), ("B", 4)]
for key, group in itertools.groupby(data, key=lambda x: x[0]):
    print(key, list(group))

# Accumulate
list(itertools.accumulate([1,2,3,4,5]))            # [1,3,6,10,15]
list(itertools.accumulate([1,2,3,4], lambda a,b: a*b))  # [1,2,6,24]

# Slice iterator
list(itertools.islice(range(100), 5, 20, 3))   # [5, 8, 11, 14, 17]

# Compress / filter
list(itertools.compress("ABCDE", [1,0,1,0,1]))  # ['A','C','E']
list(itertools.filterfalse(lambda x: x%2, range(10)))  # even: [0,2,4,6,8]
list(itertools.takewhile(lambda x: x<5, [1,3,5,2]))   # [1, 3]
list(itertools.dropwhile(lambda x: x<5, [1,3,5,2]))   # [5, 2]

# Starmap
list(itertools.starmap(pow, [(2,3), (3,2), (10,3)]))   # [8, 9, 1000]

# Tee (duplicate iterator)
a, b = itertools.tee(range(5), 2)

# Zip longest
list(itertools.zip_longest([1,2,3], [4,5], fillvalue=0))  # [(1,4),(2,5),(3,0)]

# Pairwise (Python 3.10+)
list(itertools.pairwise([1,2,3,4]))   # [(1,2),(2,3),(3,4)]

# Batched (Python 3.12+)
list(itertools.batched(range(10), 3))  # [(0,1,2),(3,4,5),(6,7,8),(9,)]
```

---

## 28. COLLECTIONS MODULE

```python
from collections import (
    Counter, defaultdict, OrderedDict,
    deque, namedtuple, ChainMap
)

# Counter — count occurrences
c = Counter("abracadabra")          # Counter({'a':5, 'b':2, 'r':2, ...})
c = Counter(["apple", "banana", "apple"])
c.most_common(2)                     # [('a', 5), ('b', 2)]
c["a"]                                # 5
c.total()                             # total count (Python 3.10+)
c1 + c2                               # add counts
c1 - c2                               # subtract (keeps positive)
c1 & c2                               # min of each
c1 | c2                               # max of each

# defaultdict — dict with default factory
dd = defaultdict(int)                  # default 0
dd["missing"] += 1                     # no KeyError!

dd = defaultdict(list)
dd["fruits"].append("apple")

dd = defaultdict(lambda: "unknown")

# deque — double-ended queue (fast append/pop both ends)
d = deque([1, 2, 3])
d.append(4)                # right
d.appendleft(0)            # left
d.pop()                    # right
d.popleft()                # left
d.rotate(1)                # rotate right
d.rotate(-1)               # rotate left
d.extend([4, 5])
d.extendleft([0, -1])      # note: reverses order
d = deque(maxlen=5)        # bounded — auto-discards old items

# namedtuple
Point = namedtuple('Point', ['x', 'y'])
p = Point(3, 4)
p.x, p.y                   # 3, 4
p._asdict()                 # {'x': 3, 'y': 4}
p._replace(x=10)            # Point(x=10, y=4) — new tuple

# OrderedDict (insertion order — default in dict since 3.7)
od = OrderedDict()
od.move_to_end("key")       # move to end
od.move_to_end("key", last=False)  # move to beginning

# ChainMap — combine multiple dicts
defaults = {"color": "red", "size": "medium"}
user_prefs = {"color": "blue"}
combined = ChainMap(user_prefs, defaults)
combined["color"]            # "blue" (first found)
combined["size"]             # "medium" (from defaults)
```

---

## 29. FUNCTOOLS MODULE

```python
from functools import (
    reduce, partial, lru_cache, wraps,
    total_ordering, cached_property, singledispatch
)

# reduce — fold list to single value
from functools import reduce
reduce(lambda a, b: a + b, [1, 2, 3, 4, 5])    # 15
reduce(lambda a, b: a * b, [1, 2, 3, 4, 5])    # 120
reduce(lambda a, b: a if a > b else b, [3,1,4,1,5])  # 5

# partial — fix some arguments
from functools import partial
def power(base, exp):
    return base ** exp
square = partial(power, exp=2)
cube = partial(power, exp=3)
square(5)    # 25
cube(3)      # 27

# lru_cache — memoization
@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
fibonacci(100)    # instant!
fibonacci.cache_info()
fibonacci.cache_clear()

# cached_property (Python 3.8+)
class Circle:
    def __init__(self, radius):
        self.radius = radius

    @cached_property
    def area(self):
        import math
        return math.pi * self.radius ** 2

# total_ordering — auto-generate comparison methods
@total_ordering
class Student:
    def __init__(self, grade):
        self.grade = grade
    def __eq__(self, other):
        return self.grade == other.grade
    def __lt__(self, other):
        return self.grade < other.grade
# Now <=, >, >= are auto-generated

# singledispatch — function overloading by type
@singledispatch
def process(data):
    raise TypeError(f"Unsupported type: {type(data)}")

@process.register(str)
def _(data):
    return data.upper()

@process.register(list)
def _(data):
    return [x*2 for x in data]

process("hello")    # "HELLO"
process([1, 2, 3])  # [2, 4, 6]
```

---

## 30. WEB SCRAPING

```python
# requests + BeautifulSoup
import requests
from bs4 import BeautifulSoup

response = requests.get("https://example.com")
soup = BeautifulSoup(response.text, "html.parser")

# Find elements
soup.title                      # <title>...</title>
soup.title.string               # text content
soup.find("h1")                 # first h1
soup.find("div", class_="main") # by class
soup.find("div", id="content")  # by id
soup.find("a", href=True)       # has href attribute
soup.find_all("p")              # all <p> tags
soup.find_all("a", limit=5)     # first 5
soup.select("div.main > p")     # CSS selector
soup.select_one("#content")     # single CSS selector

# Extract data
for link in soup.find_all("a"):
    url = link.get("href")
    text = link.get_text(strip=True)
    print(f"{text}: {url}")

# Navigate
tag.parent
tag.children
tag.next_sibling
tag.previous_sibling
tag.find_next("p")
tag.find_previous("h1")
```

---

## 31. TESTING

### unittest
```python
import unittest

class TestMath(unittest.TestCase):
    def setUp(self):
        """Runs before each test."""
        self.data = [1, 2, 3]

    def tearDown(self):
        """Runs after each test."""
        pass

    def test_add(self):
        self.assertEqual(1 + 1, 2)

    def test_list_len(self):
        self.assertEqual(len(self.data), 3)

    def test_in(self):
        self.assertIn(2, self.data)

    def test_raises(self):
        with self.assertRaises(ZeroDivisionError):
            1 / 0

    def test_almost(self):
        self.assertAlmostEqual(0.1 + 0.2, 0.3, places=5)

    @unittest.skip("Not implemented yet")
    def test_todo(self):
        pass

if __name__ == "__main__":
    unittest.main()
```

### pytest (recommended)
```python
# test_math.py

def test_add():
    assert 1 + 1 == 2

def test_string():
    assert "hello".upper() == "HELLO"

def test_list():
    assert 3 in [1, 2, 3]

def test_exception():
    import pytest
    with pytest.raises(ZeroDivisionError):
        1 / 0

# Fixtures
import pytest

@pytest.fixture
def sample_data():
    return {"name": "Ritesh", "age": 25}

def test_name(sample_data):
    assert sample_data["name"] == "Ritesh"

# Parametrize
@pytest.mark.parametrize("input,expected", [
    (1, 1), (2, 4), (3, 9), (4, 16)
])
def test_square(input, expected):
    assert input ** 2 == expected
```

```bash
# Run tests
pytest                        # all tests
pytest test_file.py           # specific file
pytest -v                     # verbose
pytest -k "test_add"          # by name pattern
pytest --tb=short             # short traceback
pytest -x                     # stop on first failure
pytest --cov=mymodule         # coverage
```

---

## 32. USEFUL STANDARD LIBRARY MODULES

### glob — File Pattern Matching
```python
import glob
glob.glob("*.txt")                  # all .txt files
glob.glob("**/*.py", recursive=True)  # recursive
glob.iglob("*.txt")                 # iterator version
```

### fnmatch — Filename Matching
```python
import fnmatch
fnmatch.fnmatch("file.txt", "*.txt")    # True
fnmatch.filter(os.listdir("."), "*.py")  # filter list
```

### hashlib — Hashing
```python
import hashlib
hashlib.md5(b"hello").hexdigest()
hashlib.sha256(b"hello").hexdigest()
hashlib.sha512(b"hello").hexdigest()

# Hash a file
def hash_file(path, algo="sha256"):
    h = hashlib.new(algo)
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()
```

### secrets — Secure Random
```python
import secrets
secrets.token_hex(16)         # random hex string
secrets.token_urlsafe(16)    # URL-safe token
secrets.token_bytes(16)       # random bytes
secrets.randbelow(100)        # random int [0, 100)
secrets.choice(["a", "b", "c"])
```

### uuid — Unique Identifiers
```python
import uuid
uuid.uuid4()                  # random UUID
str(uuid.uuid4())             # as string: 'a1b2c3d4-...'
uuid.uuid1()                  # based on host & time
```

### pickle — Object Serialization
```python
import pickle

# Serialize
data = {"key": "value", "list": [1, 2, 3]}
with open("data.pkl", "wb") as f:
    pickle.dump(data, f)

# Deserialize
with open("data.pkl", "rb") as f:
    data = pickle.load(f)
# WARNING: Never unpickle untrusted data — security risk!
```

### configparser — INI File Parsing
```python
import configparser

config = configparser.ConfigParser()
config.read("config.ini")
config["DEFAULT"]["server"]
config["database"]["port"]
config.getint("database", "port")
config.getboolean("settings", "debug")

# Write
config["newSection"] = {"key": "value"}
with open("config.ini", "w") as f:
    config.write(f)
```

### textwrap — Text Wrapping
```python
import textwrap
textwrap.wrap(text, width=70)       # list of wrapped lines
textwrap.fill(text, width=70)       # single wrapped string
textwrap.dedent(text)               # remove common indentation
textwrap.indent(text, "  ")         # add prefix to each line
textwrap.shorten(text, width=50)    # truncate with [...]
```

### pprint — Pretty Print
```python
from pprint import pprint, pformat
pprint(complex_data)                # pretty-print to console
s = pformat(complex_data)           # pretty-print to string
pprint(data, width=40, depth=2)     # custom width/depth
```

### typing — Type Hints
```python
from typing import (
    List, Dict, Tuple, Set, Optional, Union,
    Any, Callable, Iterator, Generator,
    TypeVar, Generic, Protocol, Literal
)

# Python 3.9+: use built-in types directly
def func(items: list[str]) -> dict[str, int]: ...

# Optional = Union[X, None]
def find(x: int) -> Optional[str]: ...
# Python 3.10+
def find(x: int) -> str | None: ...

# TypeVar
T = TypeVar('T')
def first(items: list[T]) -> T:
    return items[0]

# Literal
def set_mode(mode: Literal["read", "write"]): ...
```

---

## 33. SCRIPT TEMPLATE

```python
#!/usr/bin/env python3
"""
Script: myscript.py
Description: Brief description of what this script does.
Author: Your Name
Date: 2026-04-14
"""

import argparse
import logging
import sys
from pathlib import Path

# ─── Constants ────────────────────────────────────
VERSION = "1.0.0"
SCRIPT_DIR = Path(__file__).parent.resolve()

# ─── Logging Setup ────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
logger = logging.getLogger(__name__)


def parse_args():
    """Parse command-line arguments."""
    parser = argparse.ArgumentParser(
        description="What this script does.",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="Example:\n  %(prog)s -v input.txt",
    )
    parser.add_argument("input", help="Input file path")
    parser.add_argument("-o", "--output", default="output.txt", help="Output file")
    parser.add_argument("-v", "--verbose", action="store_true", help="Verbose output")
    parser.add_argument("-n", "--count", type=int, default=10, help="Number of items")
    parser.add_argument("--version", action="version", version=f"%(prog)s {VERSION}")
    return parser.parse_args()


def process(input_path: Path, output_path: Path, count: int) -> None:
    """Main processing logic."""
    if not input_path.exists():
        logger.error(f"Input file not found: {input_path}")
        sys.exit(1)

    logger.info(f"Processing {input_path} → {output_path}")

    # Your logic here
    content = input_path.read_text()
    output_path.write_text(content[:count])

    logger.info("Processing complete")


def main():
    """Entry point."""
    args = parse_args()

    if args.verbose:
        logging.getLogger().setLevel(logging.DEBUG)

    try:
        process(
            input_path=Path(args.input),
            output_path=Path(args.output),
            count=args.count,
        )
    except KeyboardInterrupt:
        logger.info("Interrupted by user")
        sys.exit(130)
    except Exception as e:
        logger.exception(f"Unexpected error: {e}")
        sys.exit(1)


if __name__ == "__main__":
    main()
```

---

## 34. COMMON RECIPES & PATTERNS

### Read/Write Files Quickly
```python
from pathlib import Path

# One-liners
text = Path("file.txt").read_text()
Path("out.txt").write_text("hello")
data = Path("file.bin").read_bytes()
```

### Run Shell Commands
```python
import subprocess
result = subprocess.run(["ls", "-la"], capture_output=True, text=True, check=True)
print(result.stdout)
```

### Walk Directory Tree
```python
from pathlib import Path
for f in Path(".").rglob("*.py"):
    print(f)
```

### Download File
```python
import urllib.request
urllib.request.urlretrieve("https://example.com/file.zip", "file.zip")

# Or with requests
import requests
r = requests.get("https://example.com/file.zip", stream=True)
with open("file.zip", "wb") as f:
    for chunk in r.iter_content(chunk_size=8192):
        f.write(chunk)
```

### Retry Pattern
```python
import time

def retry(func, max_attempts=3, delay=1, backoff=2):
    for attempt in range(1, max_attempts + 1):
        try:
            return func()
        except Exception as e:
            if attempt == max_attempts:
                raise
            print(f"Attempt {attempt} failed: {e}. Retrying in {delay}s...")
            time.sleep(delay)
            delay *= backoff
```

### Timer / Benchmark
```python
import time

# Context manager timer
from contextlib import contextmanager

@contextmanager
def timer(label=""):
    start = time.perf_counter()
    yield
    elapsed = time.perf_counter() - start
    print(f"{label}: {elapsed:.4f}s")

with timer("Processing"):
    heavy_computation()

# Decorator timer
def timed(func):
    from functools import wraps
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__}: {time.perf_counter()-start:.4f}s")
        return result
    return wrapper

@timed
def slow_function():
    time.sleep(1)
```

### Progress Bar (without libraries)
```python
import sys

def progress_bar(current, total, width=50):
    pct = current / total
    filled = int(width * pct)
    bar = "█" * filled + "░" * (width - filled)
    sys.stdout.write(f"\r|{bar}| {pct:.1%}")
    sys.stdout.flush()
    if current == total:
        print()

for i in range(1, 101):
    progress_bar(i, 100)
    import time; time.sleep(0.02)
```

### Config from Environment
```python
import os

config = {
    "DB_HOST": os.getenv("DB_HOST", "localhost"),
    "DB_PORT": int(os.getenv("DB_PORT", "5432")),
    "DEBUG": os.getenv("DEBUG", "false").lower() == "true",
    "API_KEY": os.environ["API_KEY"],   # required — raises KeyError
}
```

### Flatten Nested Structure
```python
def flatten(lst):
    for item in lst:
        if isinstance(item, (list, tuple)):
            yield from flatten(item)
        else:
            yield item

list(flatten([1, [2, [3, 4]], [5, 6]]))   # [1, 2, 3, 4, 5, 6]
```

### Memoization / Cache
```python
from functools import lru_cache

@lru_cache(maxsize=None)
def expensive(n):
    # computed only once per unique n
    return sum(range(n))
```

### Parallel File Processing
```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from pathlib import Path

def process_file(filepath):
    content = filepath.read_text()
    return filepath.name, len(content)

files = list(Path(".").glob("*.txt"))

with ThreadPoolExecutor(max_workers=4) as executor:
    futures = {executor.submit(process_file, f): f for f in files}
    for future in as_completed(futures):
        name, size = future.result()
        print(f"{name}: {size} chars")
```

### Validate IP Address
```python
import ipaddress

def is_valid_ip(addr):
    try:
        ipaddress.ip_address(addr)
        return True
    except ValueError:
        return False

is_valid_ip("192.168.1.1")    # True
is_valid_ip("999.999.999.999")  # False
```

### Color Output
```python
class Color:
    RED = "\033[91m"
    GREEN = "\033[92m"
    YELLOW = "\033[93m"
    BLUE = "\033[94m"
    BOLD = "\033[1m"
    END = "\033[0m"

print(f"{Color.RED}Error!{Color.END}")
print(f"{Color.GREEN}Success!{Color.END}")
print(f"{Color.BOLD}Bold text{Color.END}")
```

---

## 35. QUICK REFERENCE CHEAT SHEET

### Built-in Functions
```
abs()  all()  any()  ascii()  bin()  bool()  breakpoint()
bytearray()  bytes()  callable()  chr()  classmethod()
compile()  complex()  delattr()  dict()  dir()  divmod()
enumerate()  eval()  exec()  filter()  float()  format()
frozenset()  getattr()  globals()  hasattr()  hash()  help()
hex()  id()  input()  int()  isinstance()  issubclass()
iter()  len()  list()  locals()  map()  max()  memoryview()
min()  next()  object()  oct()  open()  ord()  pow()
print()  property()  range()  repr()  reversed()  round()
set()  setattr()  slice()  sorted()  staticmethod()  str()
sum()  super()  tuple()  type()  vars()  zip()  __import__()
```

### Operators Precedence (high → low)
```
()                    Parentheses
**                    Exponentiation
+x, -x, ~x           Unary
*, /, //, %           Multiplication, Division
+, -                  Addition, Subtraction
<<, >>                Bitwise shift
&                     Bitwise AND
^                     Bitwise XOR
|                     Bitwise OR
==, !=, <, >, <=, >=  Comparison
is, is not            Identity
in, not in            Membership
not                   Logical NOT
and                   Logical AND
or                    Logical OR
:=                    Walrus
```

### Comprehension Summary
```python
[expr for x in iterable]                # List
[expr for x in iterable if cond]        # List filtered
{expr for x in iterable}                # Set
{k: v for k, v in iterable}            # Dict
(expr for x in iterable)                # Generator
```

### Slice Notation
```python
seq[start:stop]         # items from start to stop-1
seq[start:]             # items from start to end
seq[:stop]              # items from beginning to stop-1
seq[:]                  # copy entire sequence
seq[::step]             # every step-th item
seq[::-1]               # reversed
seq[start:stop:step]    # full slice
```

### Truthiness Quick Reference
```
Falsy: False, None, 0, 0.0, "", [], (), {}, set(), range(0)
Everything else is Truthy.
```

### Common Magic Methods
```
__init__    Constructor
__str__     print(), str()
__repr__    repr(), debugger
__len__     len()
__getitem__ obj[key]
__setitem__ obj[key] = val
__iter__    for x in obj
__next__    next(obj)
__call__    obj()
__add__     obj + other
__eq__      obj == other
__lt__      obj < other
__hash__    hash(obj)
__enter__   with obj:
__exit__    end of with
__contains__ x in obj
__bool__    bool(obj)
```

---

*Complete Python Scripting Reference — All 35 Topics*
*Last updated: April 2026*
