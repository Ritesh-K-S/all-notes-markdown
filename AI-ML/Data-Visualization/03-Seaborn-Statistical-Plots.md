# Chapter 03: Seaborn — Statistical Data Visualization

## Table of Contents
- [3.1 Introduction to Seaborn](#31-introduction-to-seaborn)
- [3.2 Distribution Plots](#32-distribution-plots)
- [3.3 Categorical Plots](#33-categorical-plots)
- [3.4 Relational Plots](#34-relational-plots)
- [3.5 Matrix Plots (Heatmaps & Clustermaps)](#35-matrix-plots-heatmaps--clustermaps)
- [3.6 Regression Plots](#36-regression-plots)
- [3.7 Multi-Plot Grids (FacetGrid, PairGrid)](#37-multi-plot-grids-facetgrid-pairgrid)
- [3.8 Themes and Aesthetics](#38-themes-and-aesthetics)
- [3.9 Common Mistakes](#39-common-mistakes)
- [3.10 Interview Questions](#310-interview-questions)
- [3.11 Quick Reference](#311-quick-reference)

---

## 3.1 Introduction to Seaborn

### What It Is
Seaborn is a **statistical visualization library** built on top of Matplotlib. If Matplotlib is like building furniture from raw wood, Seaborn is like buying from IKEA — you get beautiful, well-designed results with much less effort. It's specifically designed for **exploring and understanding data** rather than creating custom artistic graphics.

### Why It Matters
- **Statistical intelligence** — automatically computes and shows confidence intervals, regression lines, distributions
- **DataFrame-native** — works directly with Pandas DataFrames (column names as parameters)
- **Beautiful defaults** — publication-ready plots without manual customization
- **Complex plots in one line** — what takes 20 lines in Matplotlib takes 1 in Seaborn

### Seaborn vs Matplotlib

| Feature | Matplotlib | Seaborn |
|---------|-----------|---------|
| Level | Low-level (full control) | High-level (quick insights) |
| Input | Arrays, lists | DataFrames (preferred) |
| Statistics | Manual calculation | Built-in (CI, regression, KDE) |
| Default style | Plain | Beautiful |
| Learning curve | Steeper | Gentler |
| Customization | Unlimited | Limited (falls back to Matplotlib) |
| Best for | Custom/publication figures | EDA and statistical plots |

### Setup

```python
import seaborn as sns
import matplotlib.pyplot as plt
import pandas as pd
import numpy as np

# Set default theme (do this once at the top)
sns.set_theme(style='whitegrid', palette='deep', font_scale=1.1)

# Built-in datasets for practice
tips = sns.load_dataset('tips')        # Restaurant tipping data
iris = sns.load_dataset('iris')        # Classic flower measurements
titanic = sns.load_dataset('titanic')  # Titanic survival data
penguins = sns.load_dataset('penguins') # Penguin measurements
diamonds = sns.load_dataset('diamonds') # Diamond prices

# Quick peek at the tips dataset
print(tips.head())
# Output:
#    total_bill   tip     sex smoker  day    time  size
# 0       16.99  1.01  Female     No  Sun  Dinner     2
# 1       10.34  1.66    Male     No  Sun  Dinner     3
# 2       21.01  3.50    Male     No  Sun  Dinner     3
```

### The Seaborn API Structure

```
Seaborn Functions
├── Figure-level (create own figure)     → return FacetGrid/PairGrid
│   ├── sns.relplot()      — relational (scatter, line)
│   ├── sns.displot()      — distributions (hist, kde, ecdf)
│   ├── sns.catplot()      — categorical (bar, box, violin, strip)
│   ├── sns.lmplot()       — regression
│   ├── sns.pairplot()     — pairwise relationships
│   └── sns.jointplot()    — joint + marginal distributions
│
└── Axes-level (plot on existing axes)   → return matplotlib Axes
    ├── sns.scatterplot(), sns.lineplot()
    ├── sns.histplot(), sns.kdeplot(), sns.ecdfplot()
    ├── sns.barplot(), sns.boxplot(), sns.violinplot()
    ├── sns.stripplot(), sns.swarmplot()
    ├── sns.regplot()
    └── sns.heatmap(), sns.clustermap()
```

> **Key Insight:** Figure-level functions (e.g., `displot`) create their own figure and can produce faceted plots (multiple panels). Axes-level functions (e.g., `histplot`) draw on a specific Matplotlib axes — use these when combining Seaborn with custom Matplotlib layouts.

---

## 3.2 Distribution Plots

### What It Is
Distribution plots show how a variable's values are spread out. They answer: "What values are common? What's rare? Is the data symmetric or skewed?" Think of it like a population survey — how are people distributed by height, income, or age?

### Why It Matters
- **First step in EDA** — understand each variable before modeling
- **Check assumptions** — many ML algorithms assume normal distribution
- **Detect anomalies** — unusual shapes reveal data quality issues
- **Feature engineering** — knowing the distribution guides transformations

### Types of Distribution Plots

```
HISTOGRAM         KDE (smooth)       ECDF (cumulative)    RUG (tick marks)
   ██                                    ___________
  ████            /\                    /                   ||||| | || ||| |
 ██████          /  \               ___/
████████        /    \            _/
              _/      \_         /
```

### Code Examples

```python
import seaborn as sns
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd

tips = sns.load_dataset('tips')

# === HISTPLOT: The modern histogram ===
fig, axes = plt.subplots(2, 2, figsize=(12, 10))

# Basic histogram
sns.histplot(data=tips, x='total_bill', ax=axes[0, 0], bins=20, color='steelblue')
axes[0, 0].set_title('Basic Histogram')

# Histogram with KDE overlay
sns.histplot(data=tips, x='total_bill', ax=axes[0, 1], 
             kde=True,           # Add smooth density curve
             bins=25, 
             color='coral')
axes[0, 1].set_title('Histogram + KDE')

# Histogram split by category (hue)
sns.histplot(data=tips, x='total_bill', ax=axes[1, 0],
             hue='sex',          # Color by gender
             multiple='stack',   # Options: 'layer', 'dodge', 'stack', 'fill'
             bins=20)
axes[1, 0].set_title('Stacked by Gender')

# 2D histogram (bivariate)
sns.histplot(data=tips, x='total_bill', y='tip', ax=axes[1, 1],
             cmap='YlOrRd',      # Color map for counts
             cbar=True)          # Show colorbar
axes[1, 1].set_title('2D Histogram')

plt.tight_layout()
plt.show()
```

```python
# === KDEPLOT: Kernel Density Estimation ===
# KDE smooths the histogram into a continuous probability curve
# Think of it like placing a tiny bell curve on each data point and summing them up

fig, axes = plt.subplots(1, 3, figsize=(15, 5))

# Basic KDE
sns.kdeplot(data=tips, x='total_bill', ax=axes[0], 
            fill=True,           # Fill under curve
            color='purple', alpha=0.5)
axes[0].set_title('KDE (Filled)')

# KDE comparing groups
sns.kdeplot(data=tips, x='total_bill', ax=axes[1],
            hue='time',          # Separate curves for Lunch vs Dinner
            fill=True, alpha=0.4,
            common_norm=False)   # Normalize each group independently
axes[1].set_title('KDE by Time of Day')

# 2D KDE (density contours)
sns.kdeplot(data=tips, x='total_bill', y='tip', ax=axes[2],
            cmap='Blues', fill=True, levels=10,  # Number of contour levels
            thresh=0.05)         # Don't show very low density regions
axes[2].set_title('2D KDE (Density Contours)')

plt.tight_layout()
plt.show()
```

```python
# === ECDF: Empirical Cumulative Distribution Function ===
# Shows the proportion of data ≤ each x value
# "What percentage of bills are under $20?" → read off the y-axis at x=20

fig, axes = plt.subplots(1, 2, figsize=(12, 5))

sns.ecdfplot(data=tips, x='total_bill', ax=axes[0])
axes[0].axhline(y=0.5, color='red', linestyle='--', alpha=0.5)
axes[0].axvline(x=tips['total_bill'].median(), color='red', linestyle='--', alpha=0.5)
axes[0].set_title('ECDF — Median visible at y=0.5')

# ECDF comparing groups
sns.ecdfplot(data=tips, x='total_bill', hue='day', ax=axes[1])
axes[1].set_title('ECDF by Day')

plt.tight_layout()
plt.show()
```

```python
# === DISPLOT: Figure-level distribution plot (creates own figure) ===
# displot can do hist, kde, or ecdf — and easily facets by category

# Faceted histogram: one panel per day
g = sns.displot(data=tips, x='total_bill', col='day', col_wrap=2,
                kde=True, height=4, aspect=1.3,
                bins=15, color='teal')
g.fig.suptitle('Bill Distribution by Day', y=1.02, fontsize=14)
plt.show()
```

### KDE Bandwidth: The Key Parameter

The bandwidth controls how smooth vs. jagged the KDE looks:

```python
fig, axes = plt.subplots(1, 3, figsize=(15, 4))

for ax, bw in zip(axes, [0.3, 1.0, 3.0]):
    sns.kdeplot(data=tips, x='total_bill', ax=ax, bw_adjust=bw, fill=True)
    ax.set_title(f'bw_adjust = {bw}')
    # bw_adjust < 1: more detail (jagged, may overfit)
    # bw_adjust > 1: more smooth (may hide real features)

plt.tight_layout()
plt.show()
```

- $bw_{adjust} = 0.3$ → Under-smoothed (noisy, shows too much detail)
- $bw_{adjust} = 1.0$ → Default (usually good)
- $bw_{adjust} = 3.0$ → Over-smoothed (hides real structure)

> **Pro Tip:** ECDF plots are underused but incredibly powerful. They never require binning/bandwidth choices, show exact percentiles, and work great for comparing distributions without overlap issues.

---

## 3.3 Categorical Plots

### What It Is
Categorical plots visualize the relationship between a **categorical variable** (like day, gender, product type) and a **numeric variable** (like price, score, count). They answer: "How does the number change across different groups?"

### Why It Matters
- Compare performance across groups (A/B testing results)
- Understand segment differences (customer segments, model variants)
- Fundamental for business analytics and reporting
- Essential for checking if group differences are meaningful

### The Categorical Plot Family

```
Shows EVERY point:        Shows SUMMARY stats:       Shows DISTRIBUTION:
  stripplot                  barplot (mean+CI)          boxplot
  swarmplot                  countplot (count)          violinplot
  (raw data visible)         pointplot (mean+CI)        boxenplot
```

### Code Examples

```python
import seaborn as sns
import matplotlib.pyplot as plt

tips = sns.load_dataset('tips')

# === BARPLOT: Shows mean + confidence interval ===
fig, axes = plt.subplots(1, 2, figsize=(12, 5))

# Basic barplot (automatically computes mean and 95% CI)
sns.barplot(data=tips, x='day', y='total_bill', ax=axes[0],
            palette='Set2',          # Color palette
            errorbar='ci',           # 'ci' (confidence interval), 'sd', 'se', or None
            capsize=0.1)             # Error bar cap width
axes[0].set_title('Mean Bill by Day (with 95% CI)')

# Grouped barplot (hue splits each category)
sns.barplot(data=tips, x='day', y='total_bill', hue='sex', ax=axes[1],
            palette='pastel', errorbar='sd')
axes[1].set_title('Mean Bill by Day & Gender')
axes[1].legend(title='Gender')

plt.tight_layout()
plt.show()
```

```python
# === COUNTPLOT: Bar chart of counts (like value_counts) ===
fig, axes = plt.subplots(1, 2, figsize=(12, 5))

# How many records per day?
sns.countplot(data=tips, x='day', ax=axes[0], 
              palette='viridis',
              order=tips['day'].value_counts().index)  # Order by count
axes[0].set_title('Number of Records per Day')

# Stacked counts with hue
sns.countplot(data=tips, x='day', hue='sex', ax=axes[1], palette='Set1')
axes[1].set_title('Count by Day and Gender')

plt.tight_layout()
plt.show()
```

```python
# === BOXPLOT: Shows median, quartiles, outliers ===
# Box = IQR (Q1 to Q3), Line = median, Whiskers = 1.5×IQR, Dots = outliers

fig, axes = plt.subplots(1, 2, figsize=(12, 5))

sns.boxplot(data=tips, x='day', y='total_bill', ax=axes[0],
            palette='Set3',
            width=0.6,               # Box width
            flierprops={'marker': 'o', 'markersize': 4})  # Outlier style
axes[0].set_title('Bill Distribution by Day')

# Horizontal box plot with hue
sns.boxplot(data=tips, y='day', x='total_bill', hue='sex', ax=axes[1],
            palette='pastel')
axes[1].set_title('Bill by Day & Gender (Horizontal)')

plt.tight_layout()
plt.show()
```

```python
# === VIOLINPLOT: Box plot + KDE (shows full distribution shape) ===
fig, axes = plt.subplots(1, 2, figsize=(12, 5))

sns.violinplot(data=tips, x='day', y='total_bill', ax=axes[0],
               palette='muted',
               inner='box',          # Show box inside: 'box', 'point', 'stick', None
               density_norm='width') # All violins same width
axes[0].set_title('Violin Plot (inner=box)')

# Split violin — great for comparing 2 groups
sns.violinplot(data=tips, x='day', y='total_bill', hue='sex', ax=axes[1],
               split=True,           # Left half = Female, Right half = Male
               palette='Set1', inner='quart')
axes[1].set_title('Split Violin (Male vs Female)')

plt.tight_layout()
plt.show()
```

```python
# === STRIPPLOT & SWARMPLOT: Show every individual point ===
fig, axes = plt.subplots(1, 2, figsize=(12, 5))

# Strip plot — random jitter to prevent overlap
sns.stripplot(data=tips, x='day', y='total_bill', ax=axes[0],
              jitter=0.25,           # Amount of horizontal spread
              alpha=0.5, size=4, palette='dark')
axes[0].set_title('Strip Plot (individual points)')

# Swarm plot — arranges points to prevent overlap (like a beeswarm)
sns.swarmplot(data=tips, x='day', y='total_bill', ax=axes[1],
              size=4, palette='Set2')
axes[1].set_title('Swarm Plot (no overlap)')
# Warning: swarmplot is SLOW with large datasets (>500 points per group)

plt.tight_layout()
plt.show()
```

```python
# === COMBINING: Violin/Box + Strip (show distribution AND raw points) ===
fig, ax = plt.subplots(figsize=(8, 6))

# Layer 1: Violin for shape
sns.violinplot(data=tips, x='day', y='total_bill', ax=ax,
               color='lightgray', inner=None, alpha=0.5)

# Layer 2: Strip for individual points
sns.stripplot(data=tips, x='day', y='total_bill', ax=ax,
              jitter=0.15, alpha=0.7, size=4, palette='dark')

ax.set_title('Violin + Strip (Best of Both Worlds)')
plt.tight_layout()
plt.show()
```

```python
# === POINTPLOT: Mean with confidence intervals connected by lines ===
# Great for showing TRENDS across categories (interaction effects)
fig, ax = plt.subplots(figsize=(8, 5))

sns.pointplot(data=tips, x='day', y='total_bill', hue='sex', ax=ax,
              palette='Set1',
              markers=['o', 's'],      # Different markers per group
              linestyles=['-', '--'],  # Different line styles
              dodge=True,              # Offset groups horizontally
              errorbar='ci', capsize=0.1)

ax.set_title('Mean Bill Trend by Day & Gender')
ax.legend(title='Gender')
plt.tight_layout()
plt.show()
```

### Choosing the Right Categorical Plot

| Scenario | Best Plot | Why |
|----------|-----------|-----|
| Compare means across groups | `barplot` | Clear, intuitive, shows CI |
| See full distribution per group | `violinplot` | Shows shape, bimodality |
| Detect outliers | `boxplot` | Outliers explicitly marked |
| Show raw data (< 500 pts) | `swarmplot` | Every point visible |
| Show raw data (> 500 pts) | `stripplot` | Handles more points |
| Show trend across ordered categories | `pointplot` | Connects means with lines |
| Count occurrences | `countplot` | No calculation needed |

---

## 3.4 Relational Plots

### What It Is
Relational plots show the relationship between two (or more) numeric variables. They're Seaborn's enhanced version of scatter and line plots, with built-in support for grouping by color (`hue`), size, and style.

### Why It Matters
- Core of exploratory analysis — "Does X relate to Y?"
- Can encode up to 5 dimensions in a single 2D plot (x, y, color, size, style)
- Built-in CI for line plots (great for time series with replicates)

### Code Examples

```python
import seaborn as sns
import matplotlib.pyplot as plt

tips = sns.load_dataset('tips')

# === SCATTERPLOT: Enhanced scatter with hue, size, style ===
fig, ax = plt.subplots(figsize=(9, 6))

sns.scatterplot(data=tips, x='total_bill', y='tip',
                hue='day',           # Color by day
                size='size',         # Point size by party size
                style='sex',         # Marker shape by gender
                palette='Set2',
                sizes=(20, 200),     # Min and max point sizes
                alpha=0.7,
                ax=ax)

ax.set_title('Tips vs Bill — Multi-dimensional View')
ax.legend(bbox_to_anchor=(1.05, 1), loc='upper left')  # Legend outside
plt.tight_layout()
plt.show()
```

```python
# === LINEPLOT: Line plot with confidence interval ===
# Perfect for time series with multiple measurements per time point

# Create sample time series data
np.random.seed(42)
n_subjects = 20
time_points = np.arange(0, 10)
data_list = []
for subject in range(n_subjects):
    values = np.cumsum(np.random.randn(10)) + subject * 0.1
    group = 'Treatment' if subject < 10 else 'Control'
    for t, v in zip(time_points, values):
        data_list.append({'time': t, 'value': v, 'group': group, 'subject': subject})

df = pd.DataFrame(data_list)

fig, ax = plt.subplots(figsize=(10, 6))
sns.lineplot(data=df, x='time', y='value', hue='group', ax=ax,
             errorbar='ci',          # Shows 95% CI band (default)
             palette='Set1')
# The shaded region = 95% confidence interval across subjects

ax.set_title('Treatment vs Control Over Time (mean ± 95% CI)')
ax.set_xlabel('Time Point')
ax.set_ylabel('Measurement Value')
plt.tight_layout()
plt.show()
```

```python
# === RELPLOT: Figure-level (faceting by columns/rows) ===
g = sns.relplot(data=tips, x='total_bill', y='tip',
                col='time',          # Separate panel per time
                hue='sex',           # Color by gender
                style='smoker',      # Marker by smoking status
                kind='scatter',      # 'scatter' or 'line'
                height=5, aspect=1.2)

g.fig.suptitle('Tips Analysis: Lunch vs Dinner', y=1.02)
plt.show()
```

### Encoding Multiple Dimensions

A single Seaborn scatter can encode **5 variables**:

| Dimension | Parameter | Visual Channel |
|-----------|-----------|---------------|
| X position | `x='total_bill'` | Horizontal position |
| Y position | `y='tip'` | Vertical position |
| Color | `hue='day'` | Point color |
| Size | `size='size'` | Point radius |
| Shape | `style='sex'` | Marker shape |

> **Warning:** Encoding more than 3 dimensions makes plots hard to read. Use faceting (multiple panels) instead of cramming everything into one plot.

---

## 3.5 Matrix Plots (Heatmaps & Clustermaps)

### What It Is
Matrix plots visualize entire matrices (2D arrays) as colored grids. Each cell's color represents its value. Think of it like a thermal camera image — hot spots are high values, cold spots are low.

### Why It Matters
- **Correlation matrices** — see which features are related
- **Confusion matrices** — evaluate classification models
- **Feature importance** — across multiple models/datasets
- **Missing data patterns** — visualize NaN locations

### Code Examples

```python
import seaborn as sns
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd

# === CORRELATION HEATMAP ===
tips = sns.load_dataset('tips')
# Only numeric columns for correlation
corr_matrix = tips.select_dtypes(include=[np.number]).corr()

fig, ax = plt.subplots(figsize=(8, 6))
sns.heatmap(corr_matrix, 
            annot=True,             # Show values in cells
            fmt='.2f',              # Format: 2 decimal places
            cmap='coolwarm',        # Diverging colormap (blue-white-red)
            center=0,               # Center colormap at 0
            vmin=-1, vmax=1,        # Fix scale to correlation range
            square=True,            # Square cells
            linewidths=0.5,         # Grid lines between cells
            cbar_kws={'shrink': 0.8},
            ax=ax)

ax.set_title('Feature Correlation Matrix', fontsize=14, pad=15)
plt.tight_layout()
plt.show()
```

```python
# === MASKING UPPER TRIANGLE (avoid redundancy) ===
fig, ax = plt.subplots(figsize=(8, 6))

# Create mask for upper triangle
mask = np.triu(np.ones_like(corr_matrix, dtype=bool))  # Upper triangle = True

sns.heatmap(corr_matrix, mask=mask, annot=True, fmt='.2f',
            cmap='RdYlBu_r', center=0, square=True,
            linewidths=1, ax=ax)
ax.set_title('Correlation (Lower Triangle Only)')
plt.tight_layout()
plt.show()
```

```python
# === CONFUSION MATRIX HEATMAP ===
# Simulated classification results
classes = ['Cat', 'Dog', 'Bird', 'Fish']
confusion = np.array([
    [45, 3, 1, 1],
    [2, 48, 0, 0],
    [1, 0, 42, 7],
    [0, 1, 5, 44]
])

fig, ax = plt.subplots(figsize=(7, 6))
sns.heatmap(confusion, annot=True, fmt='d',   # 'd' = integer format
            cmap='Blues',
            xticklabels=classes,
            yticklabels=classes,
            linewidths=1, linecolor='gray',
            ax=ax)

ax.set_xlabel('Predicted', fontsize=12)
ax.set_ylabel('Actual', fontsize=12)
ax.set_title('Confusion Matrix', fontsize=14)
plt.tight_layout()
plt.show()
```

```python
# === CLUSTERMAP: Heatmap with hierarchical clustering ===
# Automatically reorders rows/columns to group similar items together

iris = sns.load_dataset('iris')
iris_numeric = iris.drop('species', axis=1)

g = sns.clustermap(iris_numeric.sample(50, random_state=42),  # Subsample for clarity
                   cmap='viridis',
                   standard_scale=1,     # Normalize columns (0-1 scale)
                   method='ward',        # Clustering method
                   figsize=(8, 8),
                   dendrogram_ratio=0.15,  # Size of dendrogram
                   cbar_pos=(0.02, 0.8, 0.03, 0.15))  # Colorbar position

g.fig.suptitle('Clustered Heatmap of Iris Features', y=1.02)
plt.show()
```

```python
# === MISSING DATA VISUALIZATION ===
# Create dataset with missing values
np.random.seed(42)
df_missing = pd.DataFrame({
    'A': np.random.randn(100),
    'B': np.random.randn(100),
    'C': np.random.randn(100),
    'D': np.random.randn(100),
})
# Inject missing values
df_missing.loc[10:30, 'A'] = np.nan
df_missing.loc[50:70, 'B'] = np.nan
df_missing.loc[np.random.choice(100, 15), 'C'] = np.nan

fig, ax = plt.subplots(figsize=(10, 4))
sns.heatmap(df_missing.isna().T,        # Transpose: columns as rows
            cmap='YlOrRd',              # Yellow=present, Red=missing
            cbar_kws={'label': 'Missing (1=NaN)'},
            yticklabels=True,
            ax=ax)
ax.set_xlabel('Row Index')
ax.set_title('Missing Data Pattern')
plt.tight_layout()
plt.show()
```

### Heatmap Formatting Tips

| Parameter | What It Does | Tip |
|-----------|-------------|-----|
| `annot=True` | Show values in cells | Use with small matrices (< 15×15) |
| `fmt='.2f'` | Number format | `.2f`=float, `d`=integer, `.0%`=percent |
| `center=0` | Colormap center value | Essential for diverging data |
| `mask=` | Hide cells | Use for triangular correlation matrices |
| `square=True` | Force square cells | Better for correlation matrices |
| `linewidths=` | Cell borders | Makes cells distinct |
| `vmin`, `vmax` | Color scale range | Fix for comparability |

---

## 3.6 Regression Plots

### What It Is
Regression plots combine scatter plots with a fitted regression line (and confidence band). They help visualize: "Is there a linear relationship between X and Y? How strong? How uncertain?"

### Why It Matters
- Quick visual check before running regression models
- Shows if linearity assumption holds
- Reveals whether relationship is consistent across subgroups
- Detects non-linear patterns that simple correlation misses

### Code Examples

```python
import seaborn as sns
import matplotlib.pyplot as plt

tips = sns.load_dataset('tips')

# === REGPLOT: Axes-level regression plot ===
fig, axes = plt.subplots(1, 3, figsize=(15, 5))

# Linear regression (default)
sns.regplot(data=tips, x='total_bill', y='tip', ax=axes[0],
            scatter_kws={'alpha': 0.5, 's': 30},  # Scatter point style
            line_kws={'color': 'red', 'linewidth': 2})  # Regression line style
axes[0].set_title('Linear Regression')

# Polynomial regression (order=2)
sns.regplot(data=tips, x='total_bill', y='tip', ax=axes[1],
            order=2,             # Polynomial degree
            scatter_kws={'alpha': 0.5})
axes[1].set_title('Polynomial (degree=2)')

# Robust regression (less affected by outliers)
sns.regplot(data=tips, x='total_bill', y='tip', ax=axes[2],
            robust=True,         # Uses LOWESS-like robust estimation
            scatter_kws={'alpha': 0.5})
axes[2].set_title('Robust Regression')

plt.tight_layout()
plt.show()
```

```python
# === LMPLOT: Figure-level (supports faceting by category) ===
# "How does the tip~bill relationship differ by day and time?"

g = sns.lmplot(data=tips, x='total_bill', y='tip',
               col='time',          # Separate panels
               hue='sex',           # Color by gender
               height=5, aspect=1.2,
               scatter_kws={'alpha': 0.6},
               palette='Set1')

g.fig.suptitle('Regression: Tip vs Bill by Time & Gender', y=1.03)
plt.show()
```

```python
# === RESIDUAL PLOT (check if linear model is appropriate) ===
fig, axes = plt.subplots(1, 2, figsize=(12, 5))

# Good fit — residuals should be random (no pattern)
sns.residplot(data=tips, x='total_bill', y='tip', ax=axes[0],
              lowess=True,          # Add LOWESS curve to see patterns
              scatter_kws={'alpha': 0.5},
              line_kws={'color': 'red', 'linewidth': 2})
axes[0].set_title('Residual Plot (check for patterns)')
axes[0].axhline(y=0, color='gray', linestyle='--')

# If you see a curve → linear model is wrong, try polynomial/log

# Residual of a poor fit (simulated non-linear data)
x = np.linspace(0, 10, 100)
y = x**2 + np.random.randn(100) * 5  # Quadratic relationship
df_quad = pd.DataFrame({'x': x, 'y': y})

sns.residplot(data=df_quad, x='x', y='y', ax=axes[1],
              lowess=True, scatter_kws={'alpha': 0.5},
              line_kws={'color': 'red', 'linewidth': 2})
axes[1].set_title('Residual Plot (BAD — pattern visible)')
axes[1].axhline(y=0, color='gray', linestyle='--')

plt.tight_layout()
plt.show()
```

> **Pro Tip:** Always plot residuals after fitting a regression model. If you see a U-shape or any pattern, your model is missing something (try polynomial terms, log transform, or different model).

---

## 3.7 Multi-Plot Grids (FacetGrid, PairGrid)

### What It Is
Grid plots create **multiple panels** showing the same plot type across different subsets of data. Instead of cramming everything into one plot, you split by categories for clearer comparison. It's like looking at the same scene through different camera angles.

### Why It Matters
- Compare patterns across many subgroups simultaneously
- Avoid overplotting (too many groups on one axes)
- Essential for **exploratory data analysis** of multivariate data
- Publication-standard for showing interactions

### Code Examples

```python
import seaborn as sns
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd

tips = sns.load_dataset('tips')

# === PAIRPLOT: All pairwise relationships at once ===
# The SINGLE most useful EDA plot — shows relationships between ALL numeric variables

g = sns.pairplot(tips, hue='sex',         # Color by category
                 palette='Set1',
                 diag_kind='kde',          # Diagonal: 'kde' or 'hist'
                 plot_kws={'alpha': 0.6, 's': 30},
                 height=2.5)

g.fig.suptitle('Pairplot: Tips Dataset', y=1.02)
plt.show()
```

```python
# === PAIRPLOT with corner (avoid redundancy) ===
g = sns.pairplot(tips, corner=True,       # Only lower triangle
                 hue='time', palette='viridis',
                 diag_kind='hist',
                 height=2.5)
plt.show()
```

```python
# === FACETGRID: Custom faceted plots ===
# Create a grid where each cell shows a different subset

# Row = smoking status, Column = time of day
g = sns.FacetGrid(tips, row='smoker', col='time', 
                  height=4, aspect=1.3,
                  margin_titles=True)

# Map a plot function to each cell
g.map_dataframe(sns.scatterplot, x='total_bill', y='tip', 
                hue='sex', palette='Set1', alpha=0.7)
g.add_legend()
g.fig.suptitle('Bills & Tips Faceted by Smoking & Time', y=1.02)
plt.show()
```

```python
# === FACETGRID with custom function ===
g = sns.FacetGrid(tips, col='day', col_wrap=2,  # 2 columns, wraps to next row
                  height=4, aspect=1.3, 
                  sharex=True, sharey=True)

g.map_dataframe(sns.histplot, x='total_bill', kde=True, color='teal')
g.set_titles(col_template='{col_name}')   # Panel titles
g.set_axis_labels('Total Bill ($)', 'Count')
g.fig.suptitle('Bill Distribution by Day', y=1.02)
plt.show()
```

```python
# === JOINTPLOT: Bivariate + marginal distributions ===
# Shows scatter/hex/kde in center, with histograms on margins

# KDE joint plot
g = sns.jointplot(data=tips, x='total_bill', y='tip',
                  kind='kde',            # 'scatter', 'kde', 'hist', 'hex', 'reg'
                  fill=True,
                  cmap='YlOrRd',
                  height=7)
g.fig.suptitle('Joint KDE: Bill vs Tip', y=1.02)
plt.show()
```

```python
# Joint plot with regression + marginal histograms
g = sns.jointplot(data=tips, x='total_bill', y='tip',
                  kind='reg',
                  height=7,
                  marginal_kws={'bins': 20, 'kde': True})
plt.show()
```

```python
# === PAIRGRID: Full control over pairplot ===
g = sns.PairGrid(tips, hue='sex', palette='Set1',
                 vars=['total_bill', 'tip', 'size'])

g.map_upper(sns.scatterplot, alpha=0.5)   # Upper triangle: scatter
g.map_lower(sns.kdeplot, fill=True, alpha=0.4)  # Lower: KDE
g.map_diag(sns.histplot, kde=True)        # Diagonal: histogram
g.add_legend()
plt.show()
```

### Grid Types Comparison

| Type | Creates | Best For |
|------|---------|----------|
| `pairplot` | NxN grid of all variable pairs | Quick EDA of numeric columns |
| `jointplot` | Single bivariate + marginals | Deep dive into 2 variables |
| `FacetGrid` | Grid faceted by categories | Same plot across subgroups |
| `PairGrid` | NxN with custom plot per region | Full control over pair analysis |

---

## 3.8 Themes and Aesthetics

### What It Is
Seaborn provides a unified theming system that controls the overall look of all plots — background, grid, fonts, colors — all configurable with simple function calls.

### The Theme Functions

```python
import seaborn as sns

# === set_theme(): All-in-one configuration ===
sns.set_theme(
    style='whitegrid',      # 'white', 'dark', 'whitegrid', 'darkgrid', 'ticks'
    palette='deep',         # Color palette name
    font='sans-serif',      # Font family
    font_scale=1.2,         # Scale all text sizes
    rc={'figure.figsize': (10, 6)}  # Any rcParam override
)
```

### Styles Comparison

```python
import seaborn as sns
import matplotlib.pyplot as plt
import numpy as np

styles = ['white', 'dark', 'whitegrid', 'darkgrid', 'ticks']
fig, axes = plt.subplots(1, 5, figsize=(20, 4))

x = np.linspace(0, 10, 50)
for ax, style in zip(axes, styles):
    with sns.axes_style(style):
        ax.plot(x, np.sin(x), linewidth=2)
        ax.set_title(f"style='{style}'", fontsize=10)

plt.tight_layout()
plt.show()
```

| Style | Look | Best For |
|-------|------|----------|
| `white` | Clean, no grid | Final publication figures |
| `dark` | Dark background | Presentations on dark slides |
| `whitegrid` | White with horizontal grid | Reading exact values |
| `darkgrid` | Gray with white grid | Default, general use |
| `ticks` | Clean with tick marks | Minimal, elegant |

### Color Palettes

```python
import seaborn as sns
import matplotlib.pyplot as plt

# === BUILT-IN PALETTES ===
# Qualitative (categorical): 'deep', 'muted', 'pastel', 'dark', 'colorblind'
# Sequential: 'Blues', 'Greens', 'Reds', 'YlOrRd', 'viridis'
# Diverging: 'coolwarm', 'RdBu', 'vlag', 'icefire'

# View a palette
sns.palplot(sns.color_palette('Set2', 8))    # Show 8 colors
plt.title("Set2 Palette")
plt.show()

# Custom palette
custom_colors = ['#264653', '#2a9d8f', '#e9c46a', '#f4a261', '#e76f51']
sns.set_palette(custom_colors)
```

```python
# === CHOOSING PALETTES BASED ON DATA TYPE ===
fig, axes = plt.subplots(1, 3, figsize=(15, 4))
tips = sns.load_dataset('tips')

# Qualitative: distinct categories (no order)
sns.barplot(data=tips, x='day', y='total_bill', ax=axes[0],
            palette='Set2')
axes[0].set_title("Qualitative: 'Set2'")

# Sequential: ordered/continuous values
sns.barplot(data=tips, x='day', y='total_bill', ax=axes[1],
            palette='Blues_d')
axes[1].set_title("Sequential: 'Blues_d'")

# Diverging: values around a center point
data = pd.DataFrame({'x': ['A','B','C','D','E'], 
                     'y': [-2, -1, 0, 1, 2]})
sns.barplot(data=data, x='x', y='y', ax=axes[2],
            palette='coolwarm')
axes[2].set_title("Diverging: 'coolwarm'")

plt.tight_layout()
plt.show()
```

```python
# === DESPINE: Remove axis borders ===
fig, axes = plt.subplots(1, 2, figsize=(12, 4))

sns.histplot(tips['total_bill'], ax=axes[0])
axes[0].set_title('Before despine')

sns.histplot(tips['total_bill'], ax=axes[1])
sns.despine(ax=axes[1], left=True)   # Remove left and top/right spines
axes[1].set_title('After despine(left=True)')

plt.tight_layout()
plt.show()
```

### Context Scaling

```python
# set_context controls element sizing for different output media
# 'paper' < 'notebook' < 'talk' < 'poster'

fig, axes = plt.subplots(1, 4, figsize=(20, 4))
contexts = ['paper', 'notebook', 'talk', 'poster']

for ax, ctx in zip(axes, contexts):
    with sns.plotting_context(ctx):
        ax.plot([1, 2, 3], [1, 4, 9])
        ax.set_title(f"context='{ctx}'")

plt.tight_layout()
plt.show()
```

---

## 3.9 Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Using `pairplot` on 20+ columns | Creates 400+ plots, takes forever | Select relevant columns: `vars=['col1','col2','col3']` |
| `swarmplot` on large datasets | Extremely slow (> 1000 pts/group) | Use `stripplot` with jitter or `violinplot` |
| Ignoring `hue_order` | Categories appear in random order | Set `hue_order=['A', 'B', 'C']` explicitly |
| Not specifying `data=` parameter | Ambiguous; breaks with DataFrames | Always use `data=df, x='col'` syntax |
| Using figure-level in subplots | They create their own figure | Use axes-level functions (e.g., `histplot` not `displot`) |
| Not normalizing before heatmap | Features on different scales dominate | Normalize data or use `standard_scale` |
| Forgetting `plt.show()` after Seaborn | Plot might not display in scripts | Always end with `plt.show()` |
| Setting style after plotting | Style doesn't apply retroactively | Call `sns.set_theme()` at the TOP |
| Comparing groups with different N | Bars/boxes mislead without CI | Always show error bars or use `density=True` |
| Not using `order` parameter | Day order might be wrong | `order=['Mon','Tue','Wed',...]` |

---

## 3.10 Interview Questions

### Q1: What's the difference between `displot()` and `histplot()`?
**Answer:** 
- `histplot()` is **axes-level**: plots on a specific Matplotlib axes. Use when you're building custom figure layouts.
- `displot()` is **figure-level**: creates its own figure with a `FacetGrid`. Use when you want automatic faceting (`col=`, `row=` parameters).
- Same logic applies to `scatterplot` vs `relplot`, `boxplot` vs `catplot`.

### Q2: How does Seaborn compute the confidence interval in `barplot()`?
**Answer:** By default, Seaborn uses **bootstrapping** (resampling with replacement 1000 times) to estimate the 95% confidence interval of the mean. This is non-parametric — it doesn't assume normality. You can change it with `errorbar='sd'` (standard deviation), `errorbar='se'` (standard error), or `errorbar=None`.

### Q3: When would you use `clustermap()` vs `heatmap()`?
**Answer:**
- `heatmap()`: When the row/column order is meaningful (e.g., time on x-axis, or alphabetical categories)
- `clustermap()`: When you want to **discover** groups. It reorders rows/columns using hierarchical clustering to group similar items together. Great for gene expression data, customer segments, etc.

### Q4: How do you add a regression line to a scatter plot showing residual patterns?
**Answer:** Use `sns.residplot()` with `lowess=True`. This fits a linear model and plots the residuals (actual - predicted). If the LOWESS curve shows a pattern (U-shape, trend), the linear model is inappropriate. Then try `sns.regplot(order=2)` for polynomial or log-transform the data.

### Q5: How would you visualize the relationship between 5 numeric features and 1 categorical target?
**Answer:** 
1. `sns.pairplot(data, hue='target')` for overview of all pairwise relationships
2. `sns.boxplot()` or `sns.violinplot()` for each feature vs target
3. `sns.heatmap(df.corr())` for correlation structure
4. For deeper analysis: `sns.FacetGrid` with target as `col`/`hue`

### Q6: What's the difference between `palette` and `cmap` in Seaborn?
**Answer:**
- `palette`: Used for **categorical/discrete** colors (list of colors for different groups). Accepts palette names like `'Set2'`, `'viridis'`.
- `cmap`: Used for **continuous** color mapping (gradient). Used in heatmaps, 2D plots. Accepts Matplotlib colormap names.

---

## 3.11 Quick Reference

### Essential One-Liners

| Task | Code |
|------|------|
| Set theme | `sns.set_theme(style='whitegrid', palette='Set2')` |
| Histogram | `sns.histplot(data=df, x='col', kde=True, hue='group')` |
| KDE | `sns.kdeplot(data=df, x='col', fill=True, hue='group')` |
| ECDF | `sns.ecdfplot(data=df, x='col', hue='group')` |
| Box plot | `sns.boxplot(data=df, x='cat', y='num', hue='group')` |
| Violin | `sns.violinplot(data=df, x='cat', y='num', split=True, hue='g')` |
| Bar (mean) | `sns.barplot(data=df, x='cat', y='num', errorbar='ci')` |
| Count | `sns.countplot(data=df, x='cat', hue='group')` |
| Scatter | `sns.scatterplot(data=df, x='x', y='y', hue='g', size='s')` |
| Line + CI | `sns.lineplot(data=df, x='time', y='val', hue='group')` |
| Regression | `sns.regplot(data=df, x='x', y='y')` |
| Heatmap | `sns.heatmap(df.corr(), annot=True, cmap='coolwarm', center=0)` |
| Cluster map | `sns.clustermap(data, standard_scale=1)` |
| Pair plot | `sns.pairplot(df, hue='target', diag_kind='kde')` |
| Joint plot | `sns.jointplot(data=df, x='x', y='y', kind='kde')` |
| Faceted | `sns.catplot(data=df, x='x', y='y', col='group', kind='box')` |
| Despine | `sns.despine(left=True)` |
| Strip plot | `sns.stripplot(data=df, x='cat', y='num', jitter=0.2)` |
| Swarm plot | `sns.swarmplot(data=df, x='cat', y='num')` |

### Function Level Guide

| Need | Use Figure-Level | Or Axes-Level |
|------|-----------------|---------------|
| Distribution | `sns.displot(kind='hist')` | `sns.histplot()` |
| Categorical | `sns.catplot(kind='box')` | `sns.boxplot()` |
| Relational | `sns.relplot(kind='scatter')` | `sns.scatterplot()` |
| Regression | `sns.lmplot()` | `sns.regplot()` |

### When to Use Seaborn vs Matplotlib

| Scenario | Use |
|----------|-----|
| Quick EDA with DataFrames | **Seaborn** |
| Statistical summaries (CI, regression) | **Seaborn** |
| Highly custom/artistic plots | **Matplotlib** |
| Complex multi-panel layouts | **Both** (Seaborn on Matplotlib axes) |
| Animations | **Matplotlib** |
| 3D plots | **Matplotlib** |
| Heatmaps | **Seaborn** (easier) |
| Publication with specific formatting | **Matplotlib** (final touches) |

---

*Previous: [02-Matplotlib-Advanced.md](02-Matplotlib-Advanced.md)*
*Next: [04-Plotly-Interactive-Visualization.md](04-Plotly-Interactive-Visualization.md)*
