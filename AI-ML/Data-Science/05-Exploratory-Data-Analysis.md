# Chapter 05: Exploratory Data Analysis (EDA)

## Table of Contents
- [5.1 What is EDA and Why It Matters](#51-what-is-eda-and-why-it-matters)
- [5.2 Univariate Analysis](#52-univariate-analysis)
- [5.3 Bivariate Analysis](#53-bivariate-analysis)
- [5.4 Multivariate Analysis](#54-multivariate-analysis)
- [5.5 Distribution Analysis](#55-distribution-analysis)
- [5.6 Correlation Analysis](#56-correlation-analysis)
- [5.7 Automated EDA Tools](#57-automated-eda-tools)
- [5.8 EDA for Different Data Types](#58-eda-for-different-data-types)
- [5.9 EDA Storytelling — From Data to Insights](#59-eda-storytelling--from-data-to-insights)

---

## 5.1 What is EDA and Why It Matters

### What It Is
Exploratory Data Analysis (EDA) is the process of investigating your data to discover patterns, spot anomalies, test hypotheses, and check assumptions — all before building any model. Think of it like being a detective at a crime scene: before you form a theory (model), you examine all the evidence (data), take notes (visualizations), and look for clues (patterns).

### Why It Matters
- **Understand your data** before making assumptions
- **Find data quality issues** — errors, outliers, missing patterns
- **Discover relationships** between variables that guide feature engineering
- **Select the right model** — is the relationship linear? Non-linear? Are there clusters?
- **Communicate findings** — visuals are the universal language of data
- **Prevent costly mistakes** — building a model on misunderstood data wastes weeks

### The EDA Mental Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                     EDA WORKFLOW                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Step 1: OVERVIEW        → Shape, types, head/tail, info         │
│           │                                                       │
│           ▼                                                       │
│  Step 2: UNIVARIATE      → Each variable alone                   │
│           │                  (distributions, counts, stats)       │
│           ▼                                                       │
│  Step 3: BIVARIATE       → Pairs of variables                    │
│           │                  (correlations, scatter, grouped)     │
│           ▼                                                       │
│  Step 4: MULTIVARIATE    → Multiple variables together           │
│           │                  (heatmaps, pair plots, PCA)         │
│           ▼                                                       │
│  Step 5: INSIGHTS        → Document findings, form hypotheses    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Code — Starting Every EDA

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

# Set visual style
sns.set_style("whitegrid")
plt.rcParams['figure.figsize'] = (10, 6)
plt.rcParams['font.size'] = 12

# Load dataset (using Titanic as example)
df = sns.load_dataset('titanic')

# ============================================
# THE FIRST 10 THINGS TO DO WITH ANY DATASET
# ============================================

# 1. Shape — how much data?
print(f"Shape: {df.shape}")  # (891, 15) → 891 rows, 15 columns

# 2. First few rows — what does it look like?
print("\nFirst 5 rows:")
print(df.head())

# 3. Data types and non-null counts
print("\nInfo:")
print(df.info())

# 4. Statistical summary for numeric columns
print("\nNumeric Summary:")
print(df.describe())

# 5. Statistical summary for categorical columns
print("\nCategorical Summary:")
print(df.describe(include='object'))

# 6. Missing values
print("\nMissing Values:")
missing = df.isnull().sum()
print(missing[missing > 0])

# 7. Unique values per column
print("\nUnique Values:")
print(df.nunique())

# 8. Target variable distribution (if classification)
print("\nTarget Distribution:")
print(df['survived'].value_counts(normalize=True))

# 9. Memory usage
print(f"\nMemory: {df.memory_usage(deep=True).sum() / 1024:.1f} KB")

# 10. Duplicate rows
print(f"Duplicate rows: {df.duplicated().sum()}")
```

---

## 5.2 Univariate Analysis

### What It Is
Analyzing each variable (column) independently to understand its distribution, central tendency, spread, and shape. Like examining each puzzle piece individually before trying to fit them together.

### Why It Matters
- Reveals the **nature** of each variable (normal? skewed? bimodal?)
- Identifies **outliers** and **data quality issues**
- Guides **preprocessing decisions** (do we need to transform? scale?)
- Helps decide **which statistical tests** are appropriate

### For Numeric Variables

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

df = sns.load_dataset('titanic')

# ============================================
# Numeric Variable Analysis: 'age'
# ============================================
col = 'age'
series = df[col].dropna()

# --- Summary Statistics ---
print(f"=== Analysis of '{col}' ===")
print(f"Count:    {series.count()}")
print(f"Mean:     {series.mean():.2f}")
print(f"Median:   {series.median():.2f}")
print(f"Mode:     {series.mode().values}")
print(f"Std Dev:  {series.std():.2f}")
print(f"Variance: {series.var():.2f}")
print(f"Min:      {series.min()}")
print(f"Max:      {series.max()}")
print(f"Range:    {series.max() - series.min()}")
print(f"IQR:      {series.quantile(0.75) - series.quantile(0.25):.2f}")
print(f"Skewness: {series.skew():.2f}")   # 0=symmetric, >0=right-skewed, <0=left-skewed
print(f"Kurtosis: {series.kurtosis():.2f}")  # 0=normal, >0=heavy tails, <0=light tails

# --- Percentiles ---
print(f"\nPercentiles:")
for p in [1, 5, 10, 25, 50, 75, 90, 95, 99]:
    print(f"  {p}th: {series.quantile(p/100):.1f}")

# --- Visual Analysis (4 plots in one figure) ---
fig, axes = plt.subplots(2, 2, figsize=(12, 8))
fig.suptitle(f"Univariate Analysis: {col}", fontsize=14)

# 1. Histogram with KDE (Kernel Density Estimation)
axes[0, 0].hist(series, bins=30, density=True, alpha=0.7, color='steelblue', edgecolor='black')
series.plot.kde(ax=axes[0, 0], color='red', linewidth=2)
axes[0, 0].set_title('Distribution (Histogram + KDE)')
axes[0, 0].axvline(series.mean(), color='green', linestyle='--', label=f'Mean: {series.mean():.1f}')
axes[0, 0].axvline(series.median(), color='orange', linestyle='--', label=f'Median: {series.median():.1f}')
axes[0, 0].legend()

# 2. Box Plot (shows quartiles and outliers)
axes[0, 1].boxplot(series, vert=True)
axes[0, 1].set_title('Box Plot')

# 3. Violin Plot (distribution shape + box plot)
axes[1, 0].violinplot(series, vert=True, showmeans=True, showmedians=True)
axes[1, 0].set_title('Violin Plot')

# 4. QQ Plot (check normality)
from scipy import stats
stats.probplot(series, dist="norm", plot=axes[1, 1])
axes[1, 1].set_title('Q-Q Plot (Normality Check)')

plt.tight_layout()
plt.savefig('univariate_numeric.png', dpi=100, bbox_inches='tight')
plt.show()

# --- Interpretation Guide ---
"""
INTERPRETING THE PLOTS:

Histogram + KDE:
  - Bell-shaped → Normal distribution
  - Long right tail → Right-skewed (income, house prices)
  - Long left tail → Left-skewed (exam scores in easy test)
  - Two humps → Bimodal (might be two groups mixed together!)

Box Plot:
  ─┬─  Upper whisker (Q3 + 1.5*IQR or max, whichever is smaller)
  │
  ┌┤   Q3 (75th percentile)
  ││   
  ├┤   Median (50th percentile)
  ││
  └┤   Q1 (25th percentile)
  │
  ─┴─  Lower whisker
  ○     Outliers (individual points beyond whiskers)

Q-Q Plot:
  - Points on the diagonal line → Normal distribution
  - S-shape → Heavy/light tails
  - Points curving up at ends → Outliers
"""
```

### For Categorical Variables

```python
# ============================================
# Categorical Variable Analysis: 'class'
# ============================================
col = 'class'

# --- Value Counts ---
print(f"\n=== Analysis of '{col}' ===")
value_counts = df[col].value_counts()
value_pcts = df[col].value_counts(normalize=True) * 100

print("\nValue Counts:")
for val, count in value_counts.items():
    pct = value_pcts[val]
    bar = '█' * int(pct / 2)  # ASCII bar chart
    print(f"  {val:>10}: {count:>4} ({pct:>5.1f}%) {bar}")

# --- Cardinality (number of unique values) ---
print(f"\nCardinality: {df[col].nunique()}")
print(f"Most common: {df[col].mode()[0]}")
print(f"Least common: {value_counts.index[-1]} ({value_counts.iloc[-1]} occurrences)")

# --- Visual Analysis ---
fig, axes = plt.subplots(1, 2, figsize=(12, 5))

# 1. Count Plot (bar chart)
sns.countplot(data=df, x=col, order=value_counts.index, ax=axes[0], palette='viridis')
axes[0].set_title(f'Count Plot: {col}')
# Add count labels on bars
for i, v in enumerate(value_counts.values):
    axes[0].text(i, v + 5, str(v), ha='center', fontweight='bold')

# 2. Pie Chart (for proportions)
axes[1].pie(value_counts.values, labels=value_counts.index, 
            autopct='%1.1f%%', startangle=90, colors=sns.color_palette('viridis', len(value_counts)))
axes[1].set_title(f'Proportion: {col}')

plt.tight_layout()
plt.savefig('univariate_categorical.png', dpi=100, bbox_inches='tight')
plt.show()

# ============================================
# High-Cardinality Categorical Variables
# ============================================
# When a column has many unique values (>20), show top N
if df[col].nunique() > 20:
    top_n = 15
    top_values = df[col].value_counts().head(top_n)
    others_count = len(df) - top_values.sum()
    
    print(f"\nTop {top_n} values (+ {others_count} 'Others'):")
    print(top_values)
```

### Skewness — Understanding and Handling

```python
# ============================================
# Detecting and Handling Skewness
# ============================================
# Skewness rules of thumb:
# |skew| < 0.5  → approximately symmetric
# 0.5 ≤ |skew| < 1  → moderately skewed
# |skew| ≥ 1  → highly skewed

numeric_cols = df.select_dtypes(include=[np.number]).columns
skewness = df[numeric_cols].skew().sort_values(ascending=False)

print("Skewness of numeric columns:")
for col, skew in skewness.items():
    indicator = "⚠️ HIGHLY SKEWED" if abs(skew) >= 1 else "✓ OK" if abs(skew) < 0.5 else "~ moderate"
    print(f"  {col:>15}: {skew:>6.2f}  {indicator}")

# Common transformations for right-skewed data:
# 1. Log transform: np.log1p(x)  — most common
# 2. Square root: np.sqrt(x)  — moderate skew
# 3. Box-Cox: scipy.stats.boxcox(x)  — finds optimal transform
# 4. Yeo-Johnson: works with negative values too
```

---

## 5.3 Bivariate Analysis

### What It Is
Analyzing the **relationship between two variables**. This is where things get interesting — you start discovering patterns like "older passengers were more likely to survive" or "price increases with size."

### Why It Matters
- Reveals **predictive relationships** (which features predict the target?)
- Finds **interactions** between variables
- Validates (or challenges) your assumptions
- Directly guides **feature selection** for models

### Analysis Matrix — Which Method for Which Combination?

| Variable 1 | Variable 2 | Method | Plot |
|------------|------------|--------|------|
| Numeric | Numeric | Correlation | Scatter plot |
| Numeric | Categorical | Group comparison | Box/Violin plot |
| Categorical | Categorical | Contingency table | Heatmap/Stacked bar |
| Numeric | Target (Binary) | Point-biserial corr | Distribution by group |

### Code Examples

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from scipy import stats

df = sns.load_dataset('titanic')

# ============================================
# 1. NUMERIC vs NUMERIC — Scatter & Correlation
# ============================================
# Using tips dataset for better numeric-numeric examples
tips = sns.load_dataset('tips')

fig, axes = plt.subplots(1, 3, figsize=(15, 5))

# Basic scatter plot
axes[0].scatter(tips['total_bill'], tips['tip'], alpha=0.6)
axes[0].set_xlabel('Total Bill ($)')
axes[0].set_ylabel('Tip ($)')
axes[0].set_title('Scatter: Bill vs Tip')

# Scatter with regression line
sns.regplot(data=tips, x='total_bill', y='tip', ax=axes[1], 
            scatter_kws={'alpha': 0.5}, line_kws={'color': 'red'})
axes[1].set_title('With Regression Line')

# Scatter with hue (third variable as color)
sns.scatterplot(data=tips, x='total_bill', y='tip', hue='time', 
                style='smoker', ax=axes[2], alpha=0.7)
axes[2].set_title('Colored by Time, Shaped by Smoker')

plt.tight_layout()
plt.show()

# Statistical correlation
corr, pvalue = stats.pearsonr(tips['total_bill'], tips['tip'])
print(f"Pearson correlation: {corr:.3f} (p-value: {pvalue:.2e})")
# Interpretation: 0.68 = moderate positive correlation

# Spearman (for non-linear relationships)
spearman_corr, _ = stats.spearmanr(tips['total_bill'], tips['tip'])
print(f"Spearman correlation: {spearman_corr:.3f}")

# ============================================
# 2. NUMERIC vs CATEGORICAL — Grouped Analysis
# ============================================
fig, axes = plt.subplots(2, 2, figsize=(14, 10))

# Box plot — compare distributions across groups
sns.boxplot(data=df, x='class', y='age', ax=axes[0, 0], palette='Set2')
axes[0, 0].set_title('Age Distribution by Class')

# Violin plot — shows full distribution shape
sns.violinplot(data=df, x='class', y='age', ax=axes[0, 1], 
               palette='Set2', inner='quartile')
axes[0, 1].set_title('Age Distribution (Violin) by Class')

# Strip/Swarm plot — shows individual points
sns.stripplot(data=df, x='class', y='age', ax=axes[1, 0], 
              alpha=0.4, jitter=True, palette='Set2')
axes[1, 0].set_title('Individual Points by Class')

# Bar plot with error bars (mean + confidence interval)
sns.barplot(data=df, x='class', y='survived', ax=axes[1, 1], 
            palette='Set2', ci=95)
axes[1, 1].set_title('Survival Rate by Class (with 95% CI)')

plt.tight_layout()
plt.show()

# --- Statistical Tests for Group Differences ---
# T-test: compare two groups
first_class_age = df[df['class'] == 'First']['age'].dropna()
third_class_age = df[df['class'] == 'Third']['age'].dropna()

t_stat, p_val = stats.ttest_ind(first_class_age, third_class_age)
print(f"\nT-test (First vs Third class age):")
print(f"  t-statistic: {t_stat:.3f}, p-value: {p_val:.4f}")
print(f"  Significant difference? {'Yes' if p_val < 0.05 else 'No'}")

# ANOVA: compare more than two groups
groups = [df[df['class'] == c]['age'].dropna() for c in df['class'].unique()]
f_stat, p_val = stats.f_oneway(*groups)
print(f"\nANOVA (age across all classes):")
print(f"  F-statistic: {f_stat:.3f}, p-value: {p_val:.4f}")

# ============================================
# 3. CATEGORICAL vs CATEGORICAL — Contingency
# ============================================
fig, axes = plt.subplots(1, 2, figsize=(14, 5))

# Contingency table (cross-tabulation)
contingency = pd.crosstab(df['class'], df['survived'], margins=True)
print("\nContingency Table:")
print(contingency)

# Proportional contingency (row percentages)
contingency_pct = pd.crosstab(df['class'], df['survived'], normalize='index') * 100
print("\nSurvival Rate by Class (%):")
print(contingency_pct.round(1))

# Stacked bar chart
contingency_pct_plot = pd.crosstab(df['class'], df['survived'], normalize='index')
contingency_pct_plot.plot(kind='bar', stacked=True, ax=axes[0], 
                           colormap='RdYlGn', edgecolor='black')
axes[0].set_title('Survival Proportion by Class')
axes[0].set_ylabel('Proportion')
axes[0].legend(['Died', 'Survived'])

# Heatmap of contingency
sns.heatmap(pd.crosstab(df['class'], df['sex']), annot=True, fmt='d', 
            cmap='Blues', ax=axes[1])
axes[1].set_title('Count: Class vs Sex')

plt.tight_layout()
plt.show()

# Chi-squared test for independence
chi2, p_val, dof, expected = stats.chi2_contingency(
    pd.crosstab(df['class'], df['survived'])
)
print(f"\nChi-squared test (Class vs Survived):")
print(f"  Chi2: {chi2:.3f}, p-value: {p_val:.4f}, dof: {dof}")
print(f"  Variables are {'dependent' if p_val < 0.05 else 'independent'}")

# Cramér's V (effect size for categorical association)
n = len(df)
min_dim = min(pd.crosstab(df['class'], df['survived']).shape) - 1
cramers_v = np.sqrt(chi2 / (n * min_dim))
print(f"  Cramér's V: {cramers_v:.3f}")  # 0=no association, 1=perfect
```

### Interpreting Correlations

```
Correlation Strength Guide:
──────────────────────────────────
|r| = 0.00 - 0.19  →  Very weak / negligible
|r| = 0.20 - 0.39  →  Weak
|r| = 0.40 - 0.59  →  Moderate
|r| = 0.60 - 0.79  →  Strong
|r| = 0.80 - 1.00  →  Very strong

⚠️ CORRELATION ≠ CAUSATION!
──────────────────────────────────
Ice cream sales ↑ and drownings ↑ are correlated.
But ice cream doesn't cause drowning.
(Confounding variable: hot weather causes both.)
```

---

## 5.4 Multivariate Analysis

### What It Is
Analyzing three or more variables simultaneously to find complex patterns that aren't visible when looking at variables in pairs. It's like watching a 3D movie instead of multiple 2D photos — you see depth and relationships that flat images can't show.

### Why It Matters
- Real-world phenomena are driven by **multiple factors** simultaneously
- Reveals **interaction effects** (e.g., "age matters only for females")
- Essential for understanding **confounders** (hidden variables)
- Guides **feature engineering** and model building

### Code Examples

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

df = sns.load_dataset('titanic')
tips = sns.load_dataset('tips')

# ============================================
# 1. PAIR PLOT — The Swiss Army Knife of EDA
# ============================================
# Shows all pairwise relationships at once
# Diagonal: distribution of each variable
# Off-diagonal: scatter plot between pairs

# For numeric columns, colored by target
numeric_cols = ['age', 'fare', 'survived', 'pclass']
sns.pairplot(
    df[numeric_cols].dropna(), 
    hue='survived',           # Color by survival
    diag_kind='kde',          # KDE on diagonal
    plot_kws={'alpha': 0.5},
    palette='husl'
)
plt.suptitle('Pair Plot: Titanic Numeric Features', y=1.02)
plt.savefig('pairplot.png', dpi=100, bbox_inches='tight')
plt.show()

# ============================================
# 2. CORRELATION HEATMAP
# ============================================
# Shows correlations between ALL numeric variables at once
fig, ax = plt.subplots(figsize=(10, 8))

# Calculate correlation matrix
corr_matrix = df.select_dtypes(include=[np.number]).corr()

# Create mask for upper triangle (avoid redundancy)
mask = np.triu(np.ones_like(corr_matrix, dtype=bool))

# Plot heatmap
sns.heatmap(
    corr_matrix, 
    mask=mask,               # Show only lower triangle
    annot=True,              # Show correlation values
    fmt='.2f',               # 2 decimal places
    cmap='RdBu_r',           # Red=negative, Blue=positive
    center=0,                # Center colormap at 0
    vmin=-1, vmax=1,         # Correlation range
    square=True,             # Square cells
    linewidths=0.5           # Grid lines
)
ax.set_title('Correlation Heatmap', fontsize=14, pad=20)
plt.tight_layout()
plt.show()

# Print top correlations
print("\nTop Correlations (absolute value):")
# Get upper triangle of correlation matrix
upper = corr_matrix.where(np.triu(np.ones_like(corr_matrix, dtype=bool), k=1))
# Find top correlations
top_corr = (upper.unstack()
    .dropna()
    .abs()
    .sort_values(ascending=False)
    .head(10))
for (col1, col2), corr in top_corr.items():
    actual = corr_matrix.loc[col1, col2]
    print(f"  {col1} ↔ {col2}: {actual:+.3f}")

# ============================================
# 3. FACETED PLOTS — Split by categories
# ============================================
# Same relationship, split across categories
g = sns.FacetGrid(df, col='class', row='sex', hue='survived',
                  height=3, aspect=1.2, palette='Set1')
g.map(plt.hist, 'age', alpha=0.7, bins=20)
g.add_legend()
g.fig.suptitle('Age Distribution by Class, Sex, and Survival', y=1.02)
plt.show()

# ============================================
# 4. GROUPED STATISTICS — Multi-dimensional aggregation
# ============================================
# Multi-level groupby
grouped_stats = df.groupby(['class', 'sex']).agg({
    'survived': ['mean', 'count'],
    'age': ['mean', 'median'],
    'fare': ['mean', 'median']
}).round(2)

print("\nGrouped Statistics (Class × Sex):")
print(grouped_stats)

# Pivot table — powerful multi-dimensional summary
pivot = df.pivot_table(
    values='survived', 
    index='class', 
    columns='sex', 
    aggfunc='mean'
)
print("\nSurvival Rate (Class × Sex):")
print((pivot * 100).round(1))
# Shows: First class females survived most (96.8%)

# ============================================
# 5. PARALLEL COORDINATES — High-dimensional visualization
# ============================================
from pandas.plotting import parallel_coordinates

# Prepare data (normalize numeric columns for visibility)
plot_df = df[['survived', 'pclass', 'age', 'fare']].dropna().copy()
for col in ['pclass', 'age', 'fare']:
    plot_df[col] = (plot_df[col] - plot_df[col].min()) / (plot_df[col].max() - plot_df[col].min())

plot_df['survived'] = plot_df['survived'].map({0: 'Died', 1: 'Survived'})

fig, ax = plt.subplots(figsize=(10, 5))
parallel_coordinates(plot_df, 'survived', colormap='coolwarm', alpha=0.3, ax=ax)
ax.set_title('Parallel Coordinates: Titanic Features by Survival')
plt.show()

# ============================================
# 6. INTERACTION EFFECTS — The "it depends" phenomenon
# ============================================
# Does the effect of one variable depend on another?

# Example: Does class affect survival DIFFERENTLY for males vs females?
print("\nInteraction: Class × Sex → Survival Rate")
interaction = df.groupby(['class', 'sex'])['survived'].mean().unstack()
print((interaction * 100).round(1))

"""
Output interpretation:
         female   male
First     96.8   36.9
Second    92.1   15.7
Third     50.0   13.5

Key insight: Class matters MORE for females than males.
For females: First=97% vs Third=50% (47 point difference)
For males:   First=37% vs Third=14% (23 point difference)
This IS an interaction effect!
"""
```

---

## 5.5 Distribution Analysis

### What It Is
Understanding the shape and characteristics of how your data values are spread out. Every variable tells a story through its distribution — is it symmetric? Does it cluster? Are there multiple peaks?

### Why It Matters
- Many statistical tests **assume normality** — you need to verify this
- Distribution shape determines which **summary statistics** are meaningful (mean vs median)
- Guides choice of **ML algorithms** (some assume normal features)
- Informs **preprocessing decisions** (log transforms, binning)

### Common Distributions in Data Science

```
Normal (Gaussian)          Right-Skewed              Bimodal
     ╭────╮               │╮                       ╭──╮    ╭──╮
    ╱      ╲              │ ╲                     ╱    ╲  ╱    ╲
   ╱        ╲             │  ╲____              ╱      ╲╱      ╲
──╱──────────╲──       ───│────────────     ───╱──────────────────╲──
  μ-2σ  μ  μ+2σ        Salary, House         Two distinct groups
                        prices, Claims        (e.g., male+female heights)


Uniform                  Exponential            Power Law
  ┌──────────┐             │╲                        │╲
  │          │             │ ╲                       │ ╲
  │          │             │  ╲___                   │  ╲_________
──┴──────────┴──        ───│───────────          ───│─────────────────
  Random numbers         Time between            Wealth, city sizes,
  Dice rolls             events (arrivals)       word frequencies
```

### Code Examples

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from scipy import stats

# ============================================
# Fitting Distributions to Data
# ============================================
np.random.seed(42)

# Generate sample data (let's say it's income data — right-skewed)
income = np.random.lognormal(mean=10.5, sigma=0.8, size=1000)

fig, axes = plt.subplots(2, 2, figsize=(12, 10))

# 1. Histogram with multiple distribution fits
axes[0, 0].hist(income, bins=50, density=True, alpha=0.7, color='steelblue', label='Data')

# Fit normal distribution
mu, std = stats.norm.fit(income)
x = np.linspace(income.min(), income.max(), 100)
axes[0, 0].plot(x, stats.norm.pdf(x, mu, std), 'r-', linewidth=2, label=f'Normal (μ={mu:.0f})')

# Fit lognormal distribution
shape, loc, scale = stats.lognorm.fit(income)
axes[0, 0].plot(x, stats.lognorm.pdf(x, shape, loc, scale), 'g-', linewidth=2, label='Lognormal')

axes[0, 0].legend()
axes[0, 0].set_title('Distribution Fitting')

# 2. Log-transformed distribution (should look normal)
log_income = np.log(income)
axes[0, 1].hist(log_income, bins=50, density=True, alpha=0.7, color='forestgreen')
axes[0, 1].set_title('Log-Transformed (Now Normal!)')

# 3. ECDF (Empirical Cumulative Distribution Function)
sorted_data = np.sort(income)
ecdf = np.arange(1, len(sorted_data) + 1) / len(sorted_data)
axes[1, 0].step(sorted_data, ecdf, where='post')
axes[1, 0].set_xlabel('Income')
axes[1, 0].set_ylabel('Cumulative Probability')
axes[1, 0].set_title('ECDF')
axes[1, 0].axhline(y=0.5, color='r', linestyle='--', alpha=0.5, label='Median')
axes[1, 0].legend()

# 4. Q-Q Plot (check normality)
stats.probplot(income, dist="norm", plot=axes[1, 1])
axes[1, 1].set_title('Q-Q Plot (vs Normal)')

plt.tight_layout()
plt.show()

# ============================================
# Normality Tests
# ============================================
print("=== Normality Tests ===")
print(f"Data: n={len(income)}, mean={income.mean():.2f}, median={np.median(income):.2f}")
print(f"Skewness: {stats.skew(income):.3f}")
print(f"Kurtosis: {stats.kurtosis(income):.3f}")

# Shapiro-Wilk test (best for n < 5000)
if len(income) <= 5000:
    stat, p_val = stats.shapiro(income)
    print(f"\nShapiro-Wilk test: statistic={stat:.4f}, p-value={p_val:.4e}")
    print(f"  Normal? {'Yes' if p_val > 0.05 else 'No'}")

# D'Agostino-Pearson test (good for n > 20)
stat, p_val = stats.normaltest(income)
print(f"\nD'Agostino-Pearson test: statistic={stat:.4f}, p-value={p_val:.4e}")
print(f"  Normal? {'Yes' if p_val > 0.05 else 'No'}")

# Kolmogorov-Smirnov test
stat, p_val = stats.kstest(income, 'norm', args=(income.mean(), income.std()))
print(f"\nKolmogorov-Smirnov test: statistic={stat:.4f}, p-value={p_val:.4e}")
print(f"  Normal? {'Yes' if p_val > 0.05 else 'No'}")

# ============================================
# Comparing Distributions Between Groups
# ============================================
df = sns.load_dataset('titanic')

# Compare age distributions: survivors vs non-survivors
survived_age = df[df['survived'] == 1]['age'].dropna()
died_age = df[df['survived'] == 0]['age'].dropna()

fig, ax = plt.subplots(figsize=(10, 5))
sns.kdeplot(survived_age, label='Survived', shade=True, alpha=0.5)
sns.kdeplot(died_age, label='Died', shade=True, alpha=0.5)
ax.set_title('Age Distribution: Survived vs Died')
ax.legend()
plt.show()

# Statistical test: are the distributions different?
# Mann-Whitney U test (non-parametric — doesn't assume normality)
u_stat, p_val = stats.mannwhitneyu(survived_age, died_age, alternative='two-sided')
print(f"\nMann-Whitney U test (Survived vs Died age):")
print(f"  U-statistic: {u_stat:.1f}, p-value: {p_val:.4f}")
print(f"  Different distributions? {'Yes' if p_val < 0.05 else 'No'}")
```

---

## 5.6 Correlation Analysis

### What It Is
Measuring the strength and direction of the relationship between variables. Correlation tells you "when X goes up, does Y tend to go up (positive), down (negative), or neither (no correlation)?"

### Types of Correlation

| Method | When to Use | Range | Assumptions |
|--------|-------------|-------|-------------|
| **Pearson** | Linear relationships, normal data | [-1, 1] | Both variables continuous and normal |
| **Spearman** | Monotonic (not necessarily linear) | [-1, 1] | Ordinal or continuous, any distribution |
| **Kendall** | Small samples, ordinal data | [-1, 1] | Ordinal, robust to outliers |
| **Point-biserial** | One binary + one continuous | [-1, 1] | Continuous variable is normal |
| **Cramér's V** | Two categorical variables | [0, 1] | Categorical with multiple levels |

### Code Examples

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from scipy import stats

df = sns.load_dataset('titanic')

# ============================================
# 1. Full Correlation Analysis
# ============================================
numeric_df = df.select_dtypes(include=[np.number])

# Pearson correlation
pearson_corr = numeric_df.corr(method='pearson')

# Spearman correlation (rank-based — handles non-linear)
spearman_corr = numeric_df.corr(method='spearman')

# Compare Pearson vs Spearman
fig, axes = plt.subplots(1, 2, figsize=(14, 6))
sns.heatmap(pearson_corr, annot=True, fmt='.2f', cmap='RdBu_r', 
            center=0, ax=axes[0], vmin=-1, vmax=1)
axes[0].set_title('Pearson Correlation')

sns.heatmap(spearman_corr, annot=True, fmt='.2f', cmap='RdBu_r', 
            center=0, ax=axes[1], vmin=-1, vmax=1)
axes[1].set_title('Spearman Correlation')
plt.tight_layout()
plt.show()

# ============================================
# 2. Correlation with Target Variable
# ============================================
# Most useful in supervised learning: which features correlate with target?
target = 'survived'
correlations_with_target = numeric_df.corr()[target].drop(target).sort_values(ascending=False)

print(f"\nCorrelation with '{target}':")
for feature, corr in correlations_with_target.items():
    strength = "Strong" if abs(corr) > 0.5 else "Moderate" if abs(corr) > 0.3 else "Weak"
    direction = "↑" if corr > 0 else "↓"
    print(f"  {feature:>15}: {corr:+.3f} ({strength} {direction})")

# Visual: bar chart of correlations with target
fig, ax = plt.subplots(figsize=(8, 5))
correlations_with_target.plot(kind='barh', ax=ax, color=['green' if x > 0 else 'red' 
                                                          for x in correlations_with_target])
ax.set_title(f'Feature Correlations with {target}')
ax.axvline(x=0, color='black', linewidth=0.5)
plt.tight_layout()
plt.show()

# ============================================
# 3. Statistical Significance of Correlations
# ============================================
print("\nCorrelation Significance Tests:")
print(f"{'Feature 1':<12} {'Feature 2':<12} {'Pearson r':<10} {'p-value':<10} {'Significant'}")
print("-" * 60)

cols = numeric_df.columns
for i in range(len(cols)):
    for j in range(i+1, len(cols)):
        col1, col2 = cols[i], cols[j]
        valid = numeric_df[[col1, col2]].dropna()
        if len(valid) > 2:
            r, p = stats.pearsonr(valid[col1], valid[col2])
            if abs(r) > 0.3:  # Only show moderate+ correlations
                sig = "Yes ✓" if p < 0.05 else "No"
                print(f"{col1:<12} {col2:<12} {r:<+10.3f} {p:<10.4f} {sig}")

# ============================================
# 4. Non-linear Correlation Detection
# ============================================
# Pearson misses non-linear relationships!
# Example: quadratic relationship has Pearson ≈ 0 but strong relationship

np.random.seed(42)
x = np.linspace(-3, 3, 200)
y_linear = 2 * x + np.random.normal(0, 0.5, 200)
y_quadratic = x**2 + np.random.normal(0, 0.5, 200)

fig, axes = plt.subplots(1, 2, figsize=(12, 5))

# Linear relationship
axes[0].scatter(x, y_linear, alpha=0.5)
pearson_r = stats.pearsonr(x, y_linear)[0]
axes[0].set_title(f'Linear: Pearson r = {pearson_r:.3f}')

# Quadratic relationship
axes[1].scatter(x, y_quadratic, alpha=0.5)
pearson_r = stats.pearsonr(x, y_quadratic)[0]
spearman_r = stats.spearmanr(x, y_quadratic)[0]
axes[1].set_title(f'Quadratic: Pearson r = {pearson_r:.3f}\n(Spearman = {spearman_r:.3f})')
# Pearson ≈ 0 but there's clearly a strong relationship!

plt.tight_layout()
plt.show()

# Solution: Use mutual information for non-linear relationships
from sklearn.feature_selection import mutual_info_regression, mutual_info_classif

# For regression target
# mi_scores = mutual_info_regression(X, y)
# For classification target
X_numeric = df[['pclass', 'age', 'fare']].dropna()
y_binary = df.loc[X_numeric.index, 'survived']
mi_scores = mutual_info_classif(X_numeric, y_binary, random_state=42)

print("\nMutual Information Scores (captures non-linear relationships):")
for col, mi in zip(X_numeric.columns, mi_scores):
    print(f"  {col}: {mi:.4f}")
```

### Common Correlation Pitfalls

```
⚠️ CORRELATION TRAPS — Don't Fall For These!

1. CORRELATION ≠ CAUSATION
   Ice cream sales ↑ → Drownings ↑
   (Both caused by summer heat, not each other)

2. SIMPSON'S PARADOX
   Overall correlation can REVERSE when you split by groups!
   Example: A treatment looks harmful overall, but is beneficial
   within every subgroup (because sicker patients got the treatment)

3. RESTRICTED RANGE
   If you only look at NBA players, height doesn't correlate
   with basketball skill (because they're ALL tall)

4. OUTLIER INFLUENCE
   A single outlier can create a false correlation
   Always plot before computing correlation!

5. SPURIOUS CORRELATIONS
   "Number of Nicolas Cage movies" correlates with
   "People who drowned in swimming pools"
   (Coincidence with large enough datasets)
```

---

## 5.7 Automated EDA Tools

### What It Is
Libraries that generate comprehensive EDA reports automatically with a single line of code. Like having a junior data scientist do the initial analysis for you.

### Why It Matters
- Saves hours of manual coding for standard EDA
- Ensures you don't miss basic checks
- Great for quick data profiling and presentations
- Standard reports for data documentation

### Popular Tools Comparison

| Tool | Speed | Detail | Report Type | Best For |
|------|-------|--------|-------------|----------|
| **ydata-profiling** (was pandas-profiling) | Slow | Very High | HTML | Full reports |
| **sweetviz** | Fast | High | HTML | Comparing datasets |
| **dtale** | Medium | High | Interactive web | Interactive exploration |
| **dataprep** | Fast | High | HTML | Quick profiling |

### Code Examples

```python
# ============================================
# 1. ydata-profiling (formerly pandas-profiling)
# ============================================
# pip install ydata-profiling
from ydata_profiling import ProfileReport
import pandas as pd

df = pd.read_csv('your_data.csv')

# Generate full report
profile = ProfileReport(
    df, 
    title="Dataset Profiling Report",
    explorative=True,           # Include correlations, interactions
    minimal=False               # Full analysis (set True for large datasets)
)

# Save as HTML
profile.to_file("eda_report.html")

# In Jupyter notebook — display inline
# profile.to_notebook_iframe()

# ============================================
# 2. Sweetviz — Great for comparing datasets
# ============================================
# pip install sweetviz
import sweetviz as sv

# Analyze single dataset
report = sv.analyze(df)
report.show_html("sweetviz_report.html")

# Compare two datasets (e.g., train vs test)
# report = sv.compare([train_df, "Training"], [test_df, "Testing"])
# report.show_html("comparison_report.html")

# Compare target variable
# report = sv.analyze(df, target_feat='survived')

# ============================================
# 3. Quick Manual EDA Function (lightweight)
# ============================================
def quick_eda(df, target=None):
    """
    Quick EDA summary when you don't want to install extra packages.
    Prints key statistics and flags potential issues.
    """
    print("=" * 60)
    print(f"DATASET OVERVIEW")
    print("=" * 60)
    print(f"Rows: {df.shape[0]:,} | Columns: {df.shape[1]}")
    print(f"Memory: {df.memory_usage(deep=True).sum() / 1024**2:.2f} MB")
    print(f"Duplicates: {df.duplicated().sum()}")
    
    print(f"\n{'─' * 60}")
    print("COLUMN ANALYSIS")
    print(f"{'─' * 60}")
    
    for col in df.columns:
        dtype = df[col].dtype
        null_pct = df[col].isnull().mean() * 100
        unique = df[col].nunique()
        
        print(f"\n  📊 {col} ({dtype})")
        print(f"     Missing: {null_pct:.1f}% | Unique: {unique}")
        
        if pd.api.types.is_numeric_dtype(df[col]):
            print(f"     Range: [{df[col].min():.2f}, {df[col].max():.2f}]")
            print(f"     Mean: {df[col].mean():.2f} | Median: {df[col].median():.2f}")
            skew = df[col].skew()
            if abs(skew) > 1:
                print(f"     ⚠️  Highly skewed ({skew:.2f}) — consider transform")
        else:
            top = df[col].value_counts().head(3)
            print(f"     Top values: {dict(top)}")
            if unique > 50:
                print(f"     ⚠️  High cardinality ({unique} unique) — consider encoding")
    
    if target and target in df.columns:
        print(f"\n{'─' * 60}")
        print(f"TARGET VARIABLE: {target}")
        print(f"{'─' * 60}")
        if pd.api.types.is_numeric_dtype(df[target]):
            if df[target].nunique() <= 10:
                print(df[target].value_counts(normalize=True).round(3))
            else:
                print(df[target].describe())

# Usage:
# quick_eda(df, target='survived')
```

---

## 5.8 EDA for Different Data Types

### Time Series EDA

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

# ============================================
# Time Series Specific EDA
# ============================================
# Generate sample time series
dates = pd.date_range('2020-01-01', periods=365*2, freq='D')
np.random.seed(42)
sales = (
    100 +                                           # Base level
    np.arange(len(dates)) * 0.1 +                  # Trend (growth)
    30 * np.sin(np.arange(len(dates)) * 2 * np.pi / 365) +  # Seasonality
    np.random.normal(0, 10, len(dates))            # Noise
)
ts = pd.Series(sales, index=dates, name='sales')

fig, axes = plt.subplots(4, 1, figsize=(14, 12))

# 1. Raw time series
axes[0].plot(ts)
axes[0].set_title('Raw Time Series')

# 2. Rolling statistics (detect trend and volatility changes)
rolling_mean = ts.rolling(window=30).mean()
rolling_std = ts.rolling(window=30).std()
axes[1].plot(ts, alpha=0.3, label='Original')
axes[1].plot(rolling_mean, label='30-day Rolling Mean', linewidth=2)
axes[1].fill_between(rolling_mean.index, 
                     rolling_mean - 2*rolling_std,
                     rolling_mean + 2*rolling_std, alpha=0.2)
axes[1].legend()
axes[1].set_title('Rolling Statistics (Trend Detection)')

# 3. Seasonal decomposition
from statsmodels.tsa.seasonal import seasonal_decompose
decomposition = seasonal_decompose(ts, model='additive', period=365)
decomposition.seasonal.plot(ax=axes[2])
axes[2].set_title('Seasonal Component')

# 4. Autocorrelation (how does today relate to yesterday, last week, etc.)
from statsmodels.graphics.tsaplots import plot_acf
plot_acf(ts, lags=60, ax=axes[3])
axes[3].set_title('Autocorrelation Function (ACF)')

plt.tight_layout()
plt.show()

# Key time series checks:
print("Time Series EDA Summary:")
print(f"  Date range: {ts.index[0]} to {ts.index[-1]}")
print(f"  Frequency: {pd.infer_freq(ts.index)}")
print(f"  Missing dates: {(ts.index[-1] - ts.index[0]).days - len(ts) + 1}")
print(f"  Mean: {ts.mean():.2f}")
print(f"  Trend: {'Upward' if ts.iloc[-30:].mean() > ts.iloc[:30].mean() else 'Downward'}")
```

### Text Data EDA

```python
import pandas as pd
import numpy as np
from collections import Counter

# ============================================
# Text Data EDA
# ============================================
texts = pd.Series([
    "The movie was absolutely fantastic! Best film of the year.",
    "Terrible waste of time. Would not recommend to anyone.",
    "It was okay, nothing special but not bad either.",
    "A masterpiece of cinema. The acting was brilliant!",
    "Boring and predictable. I fell asleep halfway through."
])

# Basic text statistics
text_stats = pd.DataFrame({
    'char_count': texts.str.len(),
    'word_count': texts.str.split().str.len(),
    'avg_word_length': texts.apply(lambda x: np.mean([len(w) for w in x.split()])),
    'sentence_count': texts.str.count(r'[.!?]'),
    'has_exclamation': texts.str.contains('!').astype(int),
    'capital_ratio': texts.apply(lambda x: sum(1 for c in x if c.isupper()) / len(x))
})

print("Text Statistics:")
print(text_stats)
print(f"\nAverage word count: {text_stats['word_count'].mean():.1f}")
print(f"Average char count: {text_stats['char_count'].mean():.1f}")

# Word frequency analysis
all_words = ' '.join(texts.str.lower()).split()
word_freq = Counter(all_words)
print(f"\nTop 10 words: {word_freq.most_common(10)}")
```

---

## 5.9 EDA Storytelling — From Data to Insights

### The EDA Report Structure

```
EDA FINDINGS TEMPLATE
━━━━━━━━━━━━━━━━━━━━━

1. EXECUTIVE SUMMARY
   - Dataset: what it contains, size, time period
   - Key finding 1
   - Key finding 2
   - Key finding 3

2. DATA QUALITY
   - Missing values: which columns, how much
   - Outliers: where found, likely cause
   - Duplicates: how many, resolution

3. DISTRIBUTION INSIGHTS
   - Skewed features → need transformation
   - Multimodal features → possible subgroups
   - Target balance → need for sampling techniques

4. RELATIONSHIPS
   - Strongest correlations with target
   - Surprising findings
   - Interaction effects

5. RECOMMENDATIONS
   - Features to engineer
   - Columns to drop
   - Preprocessing needed
   - Model type suggestions
```

### Code — Generating EDA Summary

```python
def eda_summary_report(df, target=None):
    """Generate a structured EDA summary with actionable insights."""
    
    report = []
    report.append("# EDA SUMMARY REPORT\n")
    
    # Dataset Overview
    report.append(f"## Dataset Overview")
    report.append(f"- **Shape**: {df.shape[0]:,} rows × {df.shape[1]} columns")
    report.append(f"- **Memory**: {df.memory_usage(deep=True).sum()/1024**2:.1f} MB")
    report.append(f"- **Duplicates**: {df.duplicated().sum()}")
    
    # Missing Values
    missing = df.isnull().sum()
    missing_pct = (missing / len(df) * 100).round(1)
    if missing.any():
        report.append(f"\n## Missing Values")
        for col in missing[missing > 0].index:
            report.append(f"- **{col}**: {missing[col]} ({missing_pct[col]}%)")
    
    # Numeric Column Insights
    numeric_cols = df.select_dtypes(include=[np.number]).columns
    report.append(f"\n## Numeric Features ({len(numeric_cols)})")
    for col in numeric_cols:
        skew = df[col].skew()
        if abs(skew) > 1:
            report.append(f"- **{col}**: Highly skewed ({skew:.2f}) — consider log transform")
    
    # Target Analysis
    if target and target in df.columns:
        report.append(f"\n## Target Variable: {target}")
        if df[target].nunique() <= 10:
            for val, pct in (df[target].value_counts(normalize=True) * 100).items():
                report.append(f"- Class {val}: {pct:.1f}%")
            
            # Class imbalance check
            min_class = df[target].value_counts(normalize=True).min()
            if min_class < 0.2:
                report.append(f"- ⚠️ **Class imbalance detected** — consider SMOTE/undersampling")
    
    return '\n'.join(report)

# Usage:
# print(eda_summary_report(df, target='survived'))
```

---

## Common Mistakes in EDA

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Skipping EDA, jumping to modeling | Models built on misunderstood data fail | Always EDA first, even 30 minutes |
| Only looking at summary statistics | Anscombe's Quartet — same stats, different data | Always visualize! |
| Not checking for data leakage | Future data in features → unrealistic accuracy | Check temporal ordering |
| Ignoring class imbalance | Model just predicts majority class | Check target distribution first |
| Using Pearson for non-linear data | Misses strong non-linear relationships | Use Spearman or mutual information |
| Not segmenting analysis | Overall patterns can hide group differences | Always look at subgroups |
| Over-interpreting small correlations | r=0.1 with n=10000 is "significant" but meaningless | Consider effect SIZE, not just p-value |

---

## Interview Questions

1. **Walk me through your EDA process for a new dataset.**
   - Overview (shape, types, head) → Univariate (distributions, missing, outliers) → Bivariate (correlations with target) → Multivariate (interactions, pair plots) → Document findings

2. **How do you decide which features are important from EDA?**
   - Correlation with target (Pearson/Spearman)
   - Mutual information (non-linear)
   - Group separation (box plots showing clear differences)
   - Domain knowledge

3. **What is Simpson's Paradox? Give an example.**
   - A trend that appears in overall data reverses when data is split by groups
   - Example: University admission rates appear to discriminate against women overall, but within each department, women are admitted at higher rates (because women applied to more competitive departments)

4. **What plots would you use for a numeric vs categorical comparison?**
   - Box plot (quartiles, outliers), Violin plot (full distribution), Bar plot with CI (means), Strip/swarm plot (individual points)

5. **How do you handle class imbalance discovered during EDA?**
   - Oversampling minority (SMOTE), undersampling majority, weighted loss functions, stratified sampling, collecting more minority data

---

## Quick Reference

| Task | Tool/Method | Code |
|------|-------------|------|
| Overview | Shape, info, describe | `df.shape`, `df.info()`, `df.describe()` |
| Distributions | Histogram + KDE | `sns.histplot(df['col'], kde=True)` |
| Categorical counts | Count plot | `sns.countplot(data=df, x='col')` |
| Numeric vs Numeric | Scatter + correlation | `sns.scatterplot()`, `df.corr()` |
| Numeric vs Categorical | Box/Violin plot | `sns.boxplot(data=df, x='cat', y='num')` |
| Cat vs Cat | Contingency table | `pd.crosstab(df['a'], df['b'])` |
| All pairs | Pair plot | `sns.pairplot(df, hue='target')` |
| Correlation matrix | Heatmap | `sns.heatmap(df.corr(), annot=True)` |
| Time series | Line + decompose | `ts.plot()`, `seasonal_decompose()` |
| Auto-EDA | Profiling report | `ProfileReport(df).to_file('report.html')` |
| Normality test | Shapiro-Wilk | `stats.shapiro(data)` |
| Group comparison | T-test / ANOVA | `stats.ttest_ind()`, `stats.f_oneway()` |

---

*Previous: [Chapter 04 - Data Collection and Cleaning](04-Data-Collection-and-Cleaning.md)*  
*Next: [Chapter 06 - Feature Engineering](06-Feature-Engineering.md)*
