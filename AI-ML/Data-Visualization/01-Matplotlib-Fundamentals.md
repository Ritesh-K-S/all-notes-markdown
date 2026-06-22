# Chapter 01: Matplotlib Fundamentals

## Table of Contents
- [1.1 Introduction to Matplotlib](#11-introduction-to-matplotlib)
- [1.2 The Architecture: Figure and Axes](#12-the-architecture-figure-and-axes)
- [1.3 Line Plots](#13-line-plots)
- [1.4 Bar Plots](#14-bar-plots)
- [1.5 Scatter Plots](#15-scatter-plots)
- [1.6 Histogram Plots](#16-histogram-plots)
- [1.7 Subplots and Layouts](#17-subplots-and-layouts)
- [1.8 Basic Customization](#18-basic-customization)
- [1.9 Common Mistakes](#19-common-mistakes)
- [1.10 Interview Questions](#110-interview-questions)
- [1.11 Quick Reference](#111-quick-reference)

---

## 1.1 Introduction to Matplotlib

### What It Is
Matplotlib is Python's foundational plotting library — think of it as the "Photoshop of data charts." Just like Photoshop gives you pixel-level control over images, Matplotlib gives you element-level control over every piece of a chart: the lines, colors, labels, ticks, legends, and even the whitespace.

### Why It Matters
- **Industry standard** — Nearly every Python data visualization library (Seaborn, Pandas plotting, even some Plotly features) is built on top of Matplotlib
- **Publication-quality figures** — Used in scientific papers, reports, and presentations
- **Complete control** — Unlike high-level libraries, you can customize literally everything
- **Huge ecosystem** — Works with NumPy, Pandas, SciPy, and all major ML libraries

### How It Works

Matplotlib has two interfaces:

```
┌─────────────────────────────────────────────┐
│           Matplotlib Interfaces              │
├─────────────────────────────────────────────┤
│                                             │
│  1. pyplot (State-based)                    │
│     - Quick & dirty plotting                │
│     - Like a TV remote (press buttons)      │
│     - plt.plot(), plt.show()                │
│                                             │
│  2. Object-Oriented (OO)                    │
│     - Full control                          │
│     - Like building with LEGO (construct)   │
│     - fig, ax = plt.subplots()              │
│                                             │
└─────────────────────────────────────────────┘
```

> **Important:** Always prefer the Object-Oriented interface for anything beyond quick exploration. It's more explicit, reproducible, and less bug-prone.

### Installation & Setup

```python
# Install (if not already installed)
# pip install matplotlib

# Standard import convention
import matplotlib.pyplot as plt
import numpy as np

# For Jupyter notebooks — display plots inline
# %matplotlib inline

# For high-DPI displays (retina)
# %config InlineBackend.figure_format = 'retina'
```

---

## 1.2 The Architecture: Figure and Axes

### What It Is
Every Matplotlib plot has a hierarchy: **Figure → Axes → Elements**. Think of it like a painting:
- **Figure** = The canvas (the entire window/image)
- **Axes** = A single plot area on that canvas (NOT the x/y axis lines!)
- **Axis** = The actual x-axis or y-axis number lines
- **Elements** = Lines, bars, text, legends, etc.

### The Hierarchy (ASCII Diagram)

```
┌──────────────────────────────────────────────────────┐
│  FIGURE (the entire canvas/window)                    │
│                                                      │
│  ┌────────────────────┐  ┌────────────────────┐     │
│  │  AXES 1 (a plot)   │  │  AXES 2 (a plot)   │     │
│  │                    │  │                    │     │
│  │  ┌──────────────┐  │  │  ┌──────────────┐  │     │
│  │  │ Plot Area    │  │  │  │ Plot Area    │  │     │
│  │  │  (data drawn │  │  │  │  (data drawn │  │     │
│  │  │   here)      │  │  │  │   here)      │  │     │
│  │  └──────────────┘  │  │  └──────────────┘  │     │
│  │  x-axis, y-axis    │  │  x-axis, y-axis    │     │
│  │  title, legend     │  │  title, legend     │     │
│  └────────────────────┘  └────────────────────┘     │
│                                                      │
│  suptitle (super title for entire figure)            │
└──────────────────────────────────────────────────────┘
```

### Code: Creating Figures and Axes

```python
import matplotlib.pyplot as plt
import numpy as np

# Method 1: Simple — one axes
fig, ax = plt.subplots()  # Creates a figure with a single axes
ax.plot([1, 2, 3, 4], [1, 4, 9, 16])  # Plot on that axes
ax.set_title("My First Plot")
plt.show()

# Method 2: Multiple axes (2 rows, 2 columns)
fig, axes = plt.subplots(nrows=2, ncols=2, figsize=(10, 8))
# axes is now a 2x2 numpy array of Axes objects
axes[0, 0].plot([1, 2, 3], [1, 2, 3])    # Top-left
axes[0, 1].plot([1, 2, 3], [3, 2, 1])    # Top-right
axes[1, 0].plot([1, 2, 3], [1, 3, 2])    # Bottom-left
axes[1, 1].plot([1, 2, 3], [2, 1, 3])    # Bottom-right
plt.tight_layout()  # Prevent overlapping
plt.show()

# Method 3: Figure with custom size and DPI
fig = plt.figure(figsize=(12, 6), dpi=100)  # 12 inches wide, 6 tall
ax1 = fig.add_subplot(1, 2, 1)  # 1 row, 2 cols, position 1
ax2 = fig.add_subplot(1, 2, 2)  # 1 row, 2 cols, position 2
plt.show()
```

### Key Parameters

| Parameter | What It Does | Example |
|-----------|-------------|---------|
| `figsize` | Width × Height in inches | `figsize=(10, 6)` |
| `dpi` | Dots per inch (resolution) | `dpi=150` |
| `facecolor` | Background color of figure | `facecolor='white'` |
| `tight_layout` | Auto-adjust spacing | `tight_layout=True` |
| `constrained_layout` | Better auto-spacing (newer) | `constrained_layout=True` |

> **Pro Tip:** Use `figsize=(width, height)` where width:height ratio matches your output medium. For presentations use 16:9 (e.g., `figsize=(16, 9)`), for papers use 4:3 or square.

---

## 1.3 Line Plots

### What It Is
A line plot connects data points with straight lines. It's the most basic plot type, ideal for showing **trends over time** or **continuous relationships**.

### Why It Matters
- Best for **time series** data (stock prices, temperatures, sales over months)
- Shows **trends, patterns, and anomalies** clearly
- Most commonly used plot in dashboards and reports

### When to Use vs. Not Use

| Use Line Plot When | Don't Use When |
|-------------------|----------------|
| Data is continuous (time, measurements) | Data is categorical (countries, products) |
| Showing trends/changes over time | Comparing discrete categories |
| Few lines (< 7) to compare | Too many lines (becomes spaghetti) |
| X-axis has natural ordering | No natural order in x-axis |

### Code Examples

```python
import matplotlib.pyplot as plt
import numpy as np

# === BASIC LINE PLOT ===
x = np.linspace(0, 10, 100)  # 100 points from 0 to 10
y = np.sin(x)                 # Sine wave

fig, ax = plt.subplots(figsize=(10, 6))
ax.plot(x, y)                 # Simple line plot
ax.set_xlabel('X values')
ax.set_ylabel('sin(x)')
ax.set_title('Basic Sine Wave')
plt.show()
```

```python
# === MULTIPLE LINES WITH STYLING ===
x = np.linspace(0, 2 * np.pi, 100)

fig, ax = plt.subplots(figsize=(10, 6))

# Each plot() call adds a new line
ax.plot(x, np.sin(x), 
        color='blue',           # Line color
        linewidth=2,            # Line thickness
        linestyle='-',          # Solid line
        label='sin(x)')         # Legend label

ax.plot(x, np.cos(x), 
        color='red', 
        linewidth=2, 
        linestyle='--',         # Dashed line
        label='cos(x)')

ax.plot(x, np.sin(x) + np.cos(x), 
        color='green', 
        linewidth=1.5, 
        linestyle='-.',         # Dash-dot line
        label='sin(x) + cos(x)')

# Customization
ax.set_xlabel('Angle (radians)', fontsize=12)
ax.set_ylabel('Value', fontsize=12)
ax.set_title('Trigonometric Functions', fontsize=14, fontweight='bold')
ax.legend(loc='upper right', fontsize=10)  # Show legend
ax.grid(True, alpha=0.3)                    # Add grid with transparency
ax.set_xlim(0, 2 * np.pi)                  # Set x-axis limits
ax.set_ylim(-2, 2)                          # Set y-axis limits

plt.tight_layout()
plt.show()
```

```python
# === LINE PLOT WITH MARKERS ===
months = np.arange(1, 13)
sales_2024 = [45, 52, 48, 61, 55, 67, 72, 68, 75, 80, 85, 92]
sales_2025 = [50, 58, 55, 68, 62, 74, 79, 76, 82, 88, 93, 99]

fig, ax = plt.subplots(figsize=(10, 6))

ax.plot(months, sales_2024, 
        marker='o',             # Circle markers at data points
        markersize=6,           # Marker size
        markerfacecolor='white',# Marker fill color
        markeredgecolor='blue', # Marker border color
        linewidth=2, 
        color='blue',
        label='2024 Sales')

ax.plot(months, sales_2025, 
        marker='s',             # Square markers
        markersize=6,
        markerfacecolor='white',
        markeredgecolor='red',
        linewidth=2, 
        color='red',
        label='2025 Sales')

ax.set_xticks(months)
ax.set_xticklabels(['Jan','Feb','Mar','Apr','May','Jun',
                    'Jul','Aug','Sep','Oct','Nov','Dec'])
ax.set_xlabel('Month')
ax.set_ylabel('Sales (in thousands $)')
ax.set_title('Monthly Sales Comparison')
ax.legend()
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

### Line Styles & Markers Reference

| Line Style | Code | Marker | Code |
|-----------|------|--------|------|
| Solid | `'-'` | Circle | `'o'` |
| Dashed | `'--'` | Square | `'s'` |
| Dash-dot | `'-.'` | Triangle up | `'^'` |
| Dotted | `':'` | Diamond | `'D'` |
| None | `''` | Plus | `'+'` |
| | | Cross | `'x'` |
| | | Star | `'*'` |

### Pro Tips

```python
# Format string shorthand: 'color+marker+linestyle'
ax.plot(x, y, 'ro--')  # Red circles with dashed line
ax.plot(x, y, 'g^-')   # Green triangles with solid line
ax.plot(x, y, 'b*:')   # Blue stars with dotted line
```

---

## 1.4 Bar Plots

### What It Is
Bar plots use rectangular bars to represent **categorical data**. The length/height of each bar is proportional to the value it represents. Think of it like a "race" — taller bar = bigger value.

### Why It Matters
- Best for **comparing quantities across categories**
- Most intuitive chart type for non-technical audiences
- Used in almost every business report and dashboard

### Types of Bar Plots

```
VERTICAL BAR        HORIZONTAL BAR       GROUPED BAR         STACKED BAR
    │ ██                                  │ ██ ░░            │ ██████
    │ ██ ██          ████████ │           │ ██ ░░ ██ ░░     │ ███░░░
    │ ██ ██ ██       ██████   │           │ ██ ░░ ██ ░░     │ ██░░░░
    └─────────       ████     │           └─────────────     └────────
     A  B  C         C B A                 A    B    C        A  B  C
```

### Code Examples

```python
import matplotlib.pyplot as plt
import numpy as np

# === BASIC VERTICAL BAR PLOT ===
categories = ['Python', 'JavaScript', 'Java', 'C++', 'Go']
popularity = [30, 25, 20, 15, 10]

fig, ax = plt.subplots(figsize=(8, 5))
bars = ax.bar(categories, popularity,    # x positions, heights
              color='steelblue',          # Bar color
              edgecolor='navy',           # Bar border color
              width=0.6)                  # Bar width (0-1)

ax.set_xlabel('Programming Language')
ax.set_ylabel('Popularity (%)')
ax.set_title('Programming Language Popularity 2025')
ax.set_ylim(0, 35)  # Leave space above tallest bar

# Add value labels on top of each bar
for bar in bars:
    height = bar.get_height()
    ax.text(bar.get_x() + bar.get_width()/2., height + 0.5,
            f'{height}%', ha='center', va='bottom', fontsize=10)

plt.tight_layout()
plt.show()
```

```python
# === HORIZONTAL BAR PLOT (better for long category names) ===
departments = ['Engineering', 'Marketing', 'Sales', 'Human Resources', 'Finance']
budget = [500, 300, 450, 150, 350]

fig, ax = plt.subplots(figsize=(8, 5))
bars = ax.barh(departments, budget,       # barh = horizontal
               color='coral', 
               edgecolor='darkred',
               height=0.5)

ax.set_xlabel('Budget (in thousands $)')
ax.set_title('Department Budgets')

# Add value labels at end of bars
for bar in bars:
    width = bar.get_width()
    ax.text(width + 5, bar.get_y() + bar.get_height()/2.,
            f'${width}K', ha='left', va='center')

plt.tight_layout()
plt.show()
```

```python
# === GROUPED BAR PLOT (comparing multiple series) ===
categories = ['Q1', 'Q2', 'Q3', 'Q4']
product_a = [20, 35, 30, 35]
product_b = [25, 32, 34, 20]
product_c = [15, 28, 25, 30]

x = np.arange(len(categories))  # Label positions: [0, 1, 2, 3]
width = 0.25                     # Width of each bar

fig, ax = plt.subplots(figsize=(10, 6))

# Offset each group by 'width' to prevent overlap
bars1 = ax.bar(x - width, product_a, width, label='Product A', color='#2196F3')
bars2 = ax.bar(x, product_b, width, label='Product B', color='#FF9800')
bars3 = ax.bar(x + width, product_c, width, label='Product C', color='#4CAF50')

ax.set_xlabel('Quarter')
ax.set_ylabel('Revenue (millions $)')
ax.set_title('Quarterly Revenue by Product')
ax.set_xticks(x)
ax.set_xticklabels(categories)
ax.legend()
ax.grid(axis='y', alpha=0.3)  # Only horizontal grid lines

plt.tight_layout()
plt.show()
```

```python
# === STACKED BAR PLOT ===
categories = ['2021', '2022', '2023', '2024', '2025']
online = [30, 35, 42, 50, 58]
offline = [50, 45, 40, 35, 30]
hybrid = [20, 20, 18, 15, 12]

fig, ax = plt.subplots(figsize=(8, 6))

ax.bar(categories, online, label='Online', color='#42A5F5')
ax.bar(categories, offline, bottom=online, label='Offline', color='#66BB6A')
# For stacking: bottom = sum of all bars below
ax.bar(categories, hybrid, 
       bottom=np.array(online) + np.array(offline),  # Stack on top of both
       label='Hybrid', color='#FFA726')

ax.set_xlabel('Year')
ax.set_ylabel('Sales Channel (%)')
ax.set_title('Sales Channel Distribution Over Time')
ax.legend(loc='upper right')
ax.set_ylim(0, 110)

plt.tight_layout()
plt.show()
```

> **Pro Tip:** Use horizontal bars (`barh`) when category names are long. Use grouped bars for comparison, stacked bars for composition (parts of a whole).

---

## 1.5 Scatter Plots

### What It Is
Scatter plots show individual data points as dots on a 2D plane. Each point represents one observation with an x-value and y-value. It's like plotting stars in the sky — each star (point) has its own position.

### Why It Matters
- Reveals **relationships/correlations** between two variables
- Shows **clusters, outliers, and patterns** that tables hide
- Essential for **exploratory data analysis (EDA)**
- Foundation for understanding regression, clustering, and classification

### When to Use

| Use Scatter Plot When | Don't Use When |
|----------------------|----------------|
| Exploring relationship between 2 numeric variables | Showing trends over time (use line) |
| Looking for correlations | Comparing categories (use bar) |
| Identifying outliers | Showing distribution of 1 variable (use histogram) |
| Visualizing clusters | Too many overlapping points (use density/hex) |

### Correlation Patterns

```
POSITIVE            NEGATIVE            NO CORRELATION      NON-LINEAR
    •  •                •                 •   •  •              •  •
   • •              •  •                •    •               •      •
  •  •             •  •              •  •      •           •        •
 • •              • •                  •  •  •              •      •
• •              ••                •     •                    •  •
                                    •
r ≈ +0.9         r ≈ -0.9          r ≈ 0                r ≈ 0 (but pattern exists!)
```

### Code Examples

```python
import matplotlib.pyplot as plt
import numpy as np

# === BASIC SCATTER PLOT ===
np.random.seed(42)
x = np.random.randn(100)        # 100 random points
y = 2 * x + np.random.randn(100) * 0.5  # Linear relationship + noise

fig, ax = plt.subplots(figsize=(8, 6))
ax.scatter(x, y, 
           color='steelblue',    # Point color
           alpha=0.7,            # Transparency (0=invisible, 1=solid)
           s=50,                 # Point size
           edgecolors='navy',    # Point border color
           linewidths=0.5)       # Border width

ax.set_xlabel('Study Hours')
ax.set_ylabel('Exam Score')
ax.set_title('Study Hours vs Exam Score')
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

```python
# === SCATTER WITH COLOR MAPPING (3rd dimension via color) ===
np.random.seed(42)
n = 200
x = np.random.randn(n)
y = np.random.randn(n)
colors = np.sqrt(x**2 + y**2)    # Distance from origin = color intensity
sizes = np.abs(x * y) * 100 + 20  # Size encodes another variable

fig, ax = plt.subplots(figsize=(8, 8))
scatter = ax.scatter(x, y, 
                     c=colors,        # Color based on values
                     cmap='viridis',  # Colormap (color scheme)
                     s=sizes,         # Size varies per point
                     alpha=0.6,
                     edgecolors='black',
                     linewidths=0.3)

# Add colorbar to show what colors mean
cbar = plt.colorbar(scatter, ax=ax)
cbar.set_label('Distance from Origin', fontsize=11)

ax.set_xlabel('X')
ax.set_ylabel('Y')
ax.set_title('Scatter Plot with Color & Size Encoding')
ax.axhline(y=0, color='gray', linestyle='--', linewidth=0.5)  # Horizontal line at y=0
ax.axvline(x=0, color='gray', linestyle='--', linewidth=0.5)  # Vertical line at x=0
plt.tight_layout()
plt.show()
```

```python
# === SCATTER WITH CATEGORIES (different colors per group) ===
np.random.seed(42)

# Simulate 3 clusters
cluster1_x = np.random.normal(2, 0.5, 50)
cluster1_y = np.random.normal(2, 0.5, 50)
cluster2_x = np.random.normal(-1, 0.7, 50)
cluster2_y = np.random.normal(-1, 0.7, 50)
cluster3_x = np.random.normal(3, 0.4, 50)
cluster3_y = np.random.normal(-2, 0.6, 50)

fig, ax = plt.subplots(figsize=(8, 6))

ax.scatter(cluster1_x, cluster1_y, c='red', label='Class A', alpha=0.7, s=60)
ax.scatter(cluster2_x, cluster2_y, c='blue', label='Class B', alpha=0.7, s=60)
ax.scatter(cluster3_x, cluster3_y, c='green', label='Class C', alpha=0.7, s=60)

ax.set_xlabel('Feature 1')
ax.set_ylabel('Feature 2')
ax.set_title('Cluster Visualization')
ax.legend(fontsize=11)
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```

> **Pro Tip:** When you have thousands of points that overlap, use `alpha=0.1` to 0.3, or switch to a hexbin plot (`ax.hexbin()`) or 2D density plot to avoid overplotting.

---

## 1.6 Histogram Plots

### What It Is
A histogram shows the **distribution** (frequency/shape) of a single numeric variable. It divides data into "bins" (ranges) and counts how many values fall in each bin. Think of it like sorting students into grade ranges: how many scored 60-70, 70-80, 80-90, etc.

### Why It Matters
- **First thing to plot** when exploring any numeric variable
- Reveals: shape (normal, skewed, bimodal), spread, center, outliers
- Critical for checking assumptions in statistical tests and ML models
- Helps decide which transformations to apply (log, sqrt, etc.)

### Distribution Shapes

```
NORMAL (Bell)     RIGHT-SKEWED      LEFT-SKEWED       BIMODAL
     ██                ██                        ██       ██    ██
    ████             ████              ████       ████  ████
   ██████          ██████            ██████      ████████████
  ████████        ████████          ████████    ██████████████
 ██████████      ██████████        ██████████  ████████████████
───────────     ───────────       ───────────  ─────────────────
mean=median     mean > median     mean < median   two peaks
```

### The Bin Count Problem

Choosing the right number of bins is critical:
- **Too few bins** → Hides the true shape (over-smoothed)
- **Too many bins** → Too noisy, hard to see pattern
- **Rule of thumb**: $\sqrt{n}$ where $n$ = number of data points
- **Sturges' rule**: $k = 1 + \log_2(n)$
- **Freedman-Diaconis rule**: bin width = $2 \times \frac{IQR}{n^{1/3}}$ (best for skewed data)

### Code Examples

```python
import matplotlib.pyplot as plt
import numpy as np

# === BASIC HISTOGRAM ===
np.random.seed(42)
data = np.random.normal(loc=170, scale=10, size=1000)  # Heights: mean=170cm, std=10cm

fig, ax = plt.subplots(figsize=(8, 5))
ax.hist(data, 
        bins=30,              # Number of bins
        color='steelblue',    # Bar fill color
        edgecolor='white',    # Bar border color (makes bars distinct)
        alpha=0.8)            # Slight transparency

ax.set_xlabel('Height (cm)')
ax.set_ylabel('Frequency (count)')
ax.set_title('Distribution of Heights (n=1000)')
ax.axvline(np.mean(data), color='red', linestyle='--', label=f'Mean: {np.mean(data):.1f}')
ax.axvline(np.median(data), color='green', linestyle='-', label=f'Median: {np.median(data):.1f}')
ax.legend()
plt.tight_layout()
plt.show()
```

```python
# === COMPARING DISTRIBUTIONS (overlapping histograms) ===
np.random.seed(42)
men_heights = np.random.normal(175, 8, 500)    # Mean 175, std 8
women_heights = np.random.normal(162, 7, 500)  # Mean 162, std 7

fig, ax = plt.subplots(figsize=(10, 6))

ax.hist(men_heights, bins=30, alpha=0.5, color='blue', 
        label=f'Men (μ={np.mean(men_heights):.1f})', edgecolor='darkblue')
ax.hist(women_heights, bins=30, alpha=0.5, color='red', 
        label=f'Women (μ={np.mean(women_heights):.1f})', edgecolor='darkred')

ax.set_xlabel('Height (cm)')
ax.set_ylabel('Frequency')
ax.set_title('Height Distribution: Men vs Women')
ax.legend(fontsize=11)
ax.grid(axis='y', alpha=0.3)
plt.tight_layout()
plt.show()
```

```python
# === DENSITY HISTOGRAM (normalized to probability) ===
np.random.seed(42)
data = np.random.exponential(scale=2, size=2000)  # Skewed distribution

fig, ax = plt.subplots(figsize=(8, 5))

# density=True normalizes so area under curve = 1
counts, bins, patches = ax.hist(data, bins=50, density=True, 
                                 color='lightcoral', edgecolor='white', alpha=0.7)

# Overlay the theoretical PDF
x = np.linspace(0, 15, 100)
pdf = (1/2) * np.exp(-x/2)  # Exponential PDF with scale=2
ax.plot(x, pdf, 'k-', linewidth=2, label='Theoretical PDF')

ax.set_xlabel('Value')
ax.set_ylabel('Probability Density')
ax.set_title('Exponential Distribution (λ=0.5)')
ax.legend()
plt.tight_layout()
plt.show()
```

```python
# === HISTOGRAM WITH STATISTICS BOX ===
np.random.seed(42)
data = np.random.lognormal(mean=3, sigma=0.5, size=1000)

fig, ax = plt.subplots(figsize=(9, 6))
ax.hist(data, bins='auto', color='mediumpurple', edgecolor='white', alpha=0.8)
# bins='auto' lets matplotlib choose using the best algorithm

# Add statistics textbox
stats_text = (f'n = {len(data)}\n'
              f'Mean = {np.mean(data):.2f}\n'
              f'Median = {np.median(data):.2f}\n'
              f'Std = {np.std(data):.2f}\n'
              f'Skew = {((data - np.mean(data))**3).mean() / np.std(data)**3:.2f}')

ax.text(0.95, 0.95, stats_text, transform=ax.transAxes,
        fontsize=10, verticalalignment='top', horizontalalignment='right',
        bbox=dict(boxstyle='round', facecolor='wheat', alpha=0.5))

ax.set_xlabel('Value')
ax.set_ylabel('Frequency')
ax.set_title('Log-Normal Distribution')
plt.tight_layout()
plt.show()
```

> **Pro Tip:** Use `bins='auto'` to let Matplotlib choose the optimal bin count. For comparing distributions, always use `density=True` so different sample sizes don't mislead you.

---

## 1.7 Subplots and Layouts

### What It Is
Subplots let you place multiple plots side-by-side or in a grid within one figure. Think of it like a photo collage — multiple pictures arranged in a frame.

### Why It Matters
- Compare related plots without switching windows
- Create comprehensive dashboards
- Tell a data story with multiple perspectives
- Required skill for publication-quality figures

### Code Examples

```python
import matplotlib.pyplot as plt
import numpy as np

# === BASIC GRID OF SUBPLOTS ===
fig, axes = plt.subplots(2, 2, figsize=(12, 8))

# Access each axes by row, column index
x = np.linspace(0, 10, 100)

axes[0, 0].plot(x, np.sin(x), 'b-')
axes[0, 0].set_title('sin(x)')

axes[0, 1].plot(x, np.cos(x), 'r-')
axes[0, 1].set_title('cos(x)')

axes[1, 0].plot(x, np.exp(-x/5), 'g-')
axes[1, 0].set_title('exp(-x/5)')

axes[1, 1].plot(x, np.log(x + 1), 'm-')
axes[1, 1].set_title('log(x+1)')

# Add a super title for the entire figure
fig.suptitle('Mathematical Functions', fontsize=14, fontweight='bold')
plt.tight_layout()
plt.show()
```

```python
# === DIFFERENT SIZED SUBPLOTS using GridSpec ===
from matplotlib.gridspec import GridSpec

fig = plt.figure(figsize=(12, 8))
gs = GridSpec(2, 3, figure=fig)  # 2 rows, 3 columns grid

# Span multiple grid cells
ax1 = fig.add_subplot(gs[0, :])    # Top row, all columns (wide plot)
ax2 = fig.add_subplot(gs[1, 0])    # Bottom-left
ax3 = fig.add_subplot(gs[1, 1])    # Bottom-middle
ax4 = fig.add_subplot(gs[1, 2])    # Bottom-right

# Plot on each
x = np.linspace(0, 10, 100)
ax1.plot(x, np.sin(x) * np.exp(-x/10), 'b-', linewidth=2)
ax1.set_title('Damped Sine Wave (Full Width)')

ax2.hist(np.random.normal(0, 1, 500), bins=20, color='coral')
ax2.set_title('Normal Dist')

ax3.scatter(np.random.rand(50), np.random.rand(50), c='green', alpha=0.6)
ax3.set_title('Random Scatter')

ax4.bar(['A', 'B', 'C'], [3, 7, 5], color='purple')
ax4.set_title('Bar Chart')

plt.tight_layout()
plt.show()
```

```python
# === SHARING AXES (same scale across subplots) ===
np.random.seed(42)
data1 = np.random.normal(50, 10, 500)
data2 = np.random.normal(70, 15, 500)

# sharey=True means both plots use the same y-axis scale
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 5), sharey=True)

ax1.hist(data1, bins=25, color='skyblue', edgecolor='white')
ax1.set_title('Group A')
ax1.set_xlabel('Value')
ax1.set_ylabel('Frequency')

ax2.hist(data2, bins=25, color='salmon', edgecolor='white')
ax2.set_title('Group B')
ax2.set_xlabel('Value')

plt.suptitle('Comparison with Shared Y-Axis', fontsize=13)
plt.tight_layout()
plt.show()
```

---

## 1.8 Basic Customization

### What It Is
Customization transforms a default plot into a professional, publication-ready visualization. It's the difference between a rough sketch and a polished illustration.

### Essential Customization Elements

```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0, 10, 50)
y1 = np.sin(x)
y2 = np.cos(x)

fig, ax = plt.subplots(figsize=(10, 6))

# Plot with full customization
ax.plot(x, y1, color='#E74C3C', linewidth=2.5, linestyle='-', 
        marker='o', markersize=4, label='sin(x)')
ax.plot(x, y2, color='#3498DB', linewidth=2.5, linestyle='--',
        marker='s', markersize=4, label='cos(x)')

# Title and labels
ax.set_title('Customized Plot Example', fontsize=16, fontweight='bold', pad=15)
ax.set_xlabel('X Axis Label', fontsize=12, labelpad=10)
ax.set_ylabel('Y Axis Label', fontsize=12, labelpad=10)

# Legend
ax.legend(loc='upper right', fontsize=11, framealpha=0.9, 
          edgecolor='gray', fancybox=True, shadow=True)

# Grid
ax.grid(True, linestyle='--', alpha=0.4, which='both')

# Tick customization
ax.tick_params(axis='both', labelsize=10, direction='in', length=5)
ax.set_xticks(np.arange(0, 11, 2))

# Spine customization (the box borders)
ax.spines['top'].set_visible(False)     # Remove top border
ax.spines['right'].set_visible(False)   # Remove right border
ax.spines['left'].set_linewidth(1.2)
ax.spines['bottom'].set_linewidth(1.2)

# Background color
ax.set_facecolor('#FAFAFA')

plt.tight_layout()
plt.show()
```

```python
# === SAVING FIGURES ===
fig.savefig('my_plot.png', dpi=300, bbox_inches='tight', 
            facecolor='white', transparent=False)
# dpi=300 for print quality
# bbox_inches='tight' removes extra whitespace
# Also supports: .pdf, .svg, .jpg, .eps
```

### Color Options

| Method | Example | Use When |
|--------|---------|----------|
| Named colors | `'red'`, `'steelblue'` | Quick plots |
| Hex codes | `'#FF5733'` | Brand colors, precise matching |
| RGB tuples | `(0.1, 0.2, 0.5)` | Programmatic color generation |
| Colormaps | `cmap='viridis'` | Continuous color scales |

### Top Colormaps

| Colormap | Best For |
|----------|----------|
| `viridis` | Sequential data (default, perceptually uniform) |
| `plasma` | Sequential data (vibrant) |
| `coolwarm` | Diverging data (positive/negative) |
| `RdYlGn` | Good/bad scale (red=bad, green=good) |
| `Set2`, `tab10` | Categorical data (discrete groups) |

---

## 1.9 Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Using `plt.plot()` instead of `ax.plot()` | State-based interface is harder to control with multiple plots | Always use `fig, ax = plt.subplots()` |
| Not calling `plt.tight_layout()` | Labels and titles overlap | Add `plt.tight_layout()` before `plt.show()` |
| Too many colors | Confuses the reader | Limit to 5-7 colors max |
| Missing axis labels | Uninterpretable plot | Always label axes with units |
| Using default figsize | Too small/cramped | Set `figsize=(10, 6)` or appropriate |
| Forgetting `plt.show()` in scripts | Plot doesn't display | Always end with `plt.show()` |
| Not setting `alpha` for overlapping data | Can't see density/overlap | Use `alpha=0.3` to `0.7` |
| Using pie charts | Hard to compare angles | Use bar charts instead (almost always) |
| Rainbow colormaps (`jet`) | Not perceptually uniform, misleading | Use `viridis`, `plasma`, or `cividis` |
| Not saving in vector format for papers | Pixelated when zoomed | Save as `.pdf` or `.svg` for publications |

---

## 1.10 Interview Questions

### Q1: What is the difference between `plt.plot()` and `ax.plot()`?
**Answer:** `plt.plot()` uses the state-based (pyplot) interface which implicitly manages figures and axes. `ax.plot()` uses the object-oriented interface where you explicitly reference the axes object. The OO approach is preferred because:
- It's explicit about which plot you're modifying
- It works correctly with multiple subplots
- It's less prone to bugs when creating complex figures

### Q2: How do you create a figure with subplots of different sizes?
**Answer:** Use `GridSpec` from `matplotlib.gridspec`. It creates a grid and allows axes to span multiple rows/columns using array slicing syntax.

### Q3: What's the difference between `fig.savefig()` and `plt.savefig()`?
**Answer:** They do the same thing, but `fig.savefig()` is explicit about which figure to save. `plt.savefig()` saves the current active figure, which can be ambiguous if multiple figures exist.

### Q4: How do you handle overlapping data points in a scatter plot?
**Answer:** Use one or more of:
- `alpha` transparency (0.1–0.5)
- Smaller point sizes
- Hexbin plots (`ax.hexbin()`)
- 2D KDE/density plots
- Jittering (adding small random noise to positions)

### Q5: When would you use `density=True` in a histogram?
**Answer:** When comparing distributions with different sample sizes. `density=True` normalizes the histogram so the total area = 1, making it a probability density. Without it, a larger sample will always have taller bars regardless of shape.

---

## 1.11 Quick Reference

| Task | Code |
|------|------|
| Create figure + axes | `fig, ax = plt.subplots(figsize=(10, 6))` |
| Line plot | `ax.plot(x, y, color='blue', linewidth=2, label='data')` |
| Bar plot | `ax.bar(categories, values, color='steelblue')` |
| Horizontal bar | `ax.barh(categories, values)` |
| Scatter plot | `ax.scatter(x, y, c=colors, s=sizes, cmap='viridis')` |
| Histogram | `ax.hist(data, bins=30, edgecolor='white')` |
| Set title | `ax.set_title('Title', fontsize=14, fontweight='bold')` |
| Set labels | `ax.set_xlabel('X')` / `ax.set_ylabel('Y')` |
| Add legend | `ax.legend(loc='upper right')` |
| Add grid | `ax.grid(True, alpha=0.3)` |
| Set limits | `ax.set_xlim(0, 10)` / `ax.set_ylim(0, 100)` |
| Add text | `ax.text(x, y, 'text', fontsize=10)` |
| Vertical line | `ax.axvline(x=5, color='red', linestyle='--')` |
| Horizontal line | `ax.axhline(y=0, color='gray', linestyle='--')` |
| Remove spines | `ax.spines['top'].set_visible(False)` |
| Save figure | `fig.savefig('plot.png', dpi=300, bbox_inches='tight')` |
| Tight layout | `plt.tight_layout()` |
| Show plot | `plt.show()` |

---

*Next Chapter: [02-Matplotlib-Advanced.md](02-Matplotlib-Advanced.md) — Customization, Annotations, 3D Plots, Animations*
