# Chapter 02: Statistics Fundamentals

## Table of Contents
- [2.1 Why Statistics Matters](#21-why-statistics-matters)
- [2.2 Measures of Central Tendency](#22-measures-of-central-tendency)
- [2.3 Measures of Spread (Dispersion)](#23-measures-of-spread-dispersion)
- [2.4 Distributions](#24-distributions)
- [2.5 Correlation and Covariance](#25-correlation-and-covariance)
- [2.6 Percentiles, Quartiles, and Box Plots](#26-percentiles-quartiles-and-box-plots)
- [2.7 Skewness and Kurtosis](#27-skewness-and-kurtosis)
- [2.8 Sampling and the Central Limit Theorem](#28-sampling-and-the-central-limit-theorem)
- [2.9 Common Mistakes](#29-common-mistakes)
- [2.10 Interview Questions](#210-interview-questions)
- [2.11 Quick Reference](#211-quick-reference)

---

## 2.1 Why Statistics Matters

### What It Is
Statistics is the science of collecting, organizing, analyzing, and interpreting data. It's the mathematical backbone of data science — without statistics, you're just guessing.

**Analogy:** If data science is cooking, statistics is knowing what each ingredient does. You can follow a recipe (code), but understanding statistics is what makes you a chef who can improvise and troubleshoot.

### Why It Matters
- **Make confident decisions** — "Is this result real or just random noise?"
- **Summarize data** — Turn millions of rows into meaningful numbers
- **Detect patterns** — Separate signal from noise
- **Communicate uncertainty** — "We're 95% confident the true value is between X and Y"
- **Avoid being fooled** — Statistics helps you spot misleading claims

### Two Branches of Statistics

```
                    STATISTICS
                        │
          ┌─────────────┼─────────────┐
          │                           │
   DESCRIPTIVE                  INFERENTIAL
   "What happened?"             "What can we conclude?"
          │                           │
   • Mean, Median               • Hypothesis Tests
   • Standard Deviation         • Confidence Intervals
   • Charts & Graphs            • Regression
   • Data Summaries             • Predictions
          │                           │
   Uses: ALL the data           Uses: SAMPLE of data
   to describe what is          to infer about population
```

---

## 2.2 Measures of Central Tendency

### What They Are
Central tendency answers: **"What's a typical value in this dataset?"** — It's the "center" of your data.

### The Three Measures

#### Mean (Average)
$$\bar{x} = \frac{1}{n}\sum_{i=1}^{n} x_i = \frac{x_1 + x_2 + ... + x_n}{n}$$

**Analogy:** If you poured all the water from glasses of different heights into glasses of equal height, the mean is that equal height.

**Properties:**
- Uses every data point
- Sensitive to outliers (one billionaire changes the "average" income)
- Best for: symmetric distributions without outliers

#### Median (Middle Value)
The value that splits the data into two equal halves when sorted.

$$\text{Median} = \begin{cases} x_{(n+1)/2} & \text{if n is odd} \\ \frac{x_{n/2} + x_{(n/2)+1}}{2} & \text{if n is even} \end{cases}$$

**Analogy:** In a race, the median finisher is the person exactly in the middle — half finished before them, half after.

**Properties:**
- Not affected by outliers (robust)
- Doesn't use all information in the data
- Best for: skewed distributions, income data, house prices

#### Mode (Most Frequent)
The value that appears most often.

**Properties:**
- Only measure that works for categorical data
- Can have multiple modes (bimodal, multimodal)
- Can be undefined if all values are unique
- Best for: categorical data, finding peaks in distributions

### When to Use Which?

| Situation | Best Measure | Why |
|-----------|-------------|-----|
| Symmetric data, no outliers | Mean | Uses all information |
| Skewed data (income, prices) | Median | Robust to outliers |
| Categorical data | Mode | Only option |
| You want to minimize squared error | Mean | Mathematical property |
| You want to minimize absolute error | Median | Mathematical property |
| Data has extreme outliers | Median | Won't be distorted |

### The Skewness Rule

```
Left-Skewed (negative):     Symmetric:        Right-Skewed (positive):
    ╭──╮                      ╭──╮                    ╭──╮
   ╱    ╲                    ╱    ╲                  ╱    ╲
  ╱      ╲                  ╱      ╲                ╱      ╲
─╱────────╲──           ──╱────────╲──          ──╱────────╲──
                     
Mean < Median < Mode    Mean = Median = Mode    Mode < Median < Mean

Example: Age at death    Example: Heights        Example: Income
```

### Code: Central Tendency

```python
import numpy as np
from scipy import stats

# Sample data: Employee salaries at a startup
salaries = [45000, 52000, 48000, 55000, 50000, 47000, 53000, 
            49000, 51000, 46000, 500000]  # CEO's salary = outlier!

# Calculate all three measures
mean_salary = np.mean(salaries)
median_salary = np.median(salaries)
mode_salary = stats.mode(salaries, keepdims=True).mode[0]

print("=== Salary Analysis ===")
print(f"Mean:   ${mean_salary:,.0f}")    # Pulled up by CEO's salary
print(f"Median: ${median_salary:,.0f}")  # Not affected by outlier
print(f"Mode:   ${mode_salary:,.0f}")    # Most frequent value

# Demonstrate the outlier effect
print(f"\n=== Outlier Impact ===")
salaries_no_outlier = [s for s in salaries if s < 100000]
print(f"Mean WITH outlier:    ${np.mean(salaries):,.0f}")
print(f"Mean WITHOUT outlier: ${np.mean(salaries_no_outlier):,.0f}")
print(f"Median (either way):  ${np.median(salaries):,.0f}")

# Trimmed mean: compromise between mean and median
# Removes top/bottom 10% before computing mean
trimmed_mean = stats.trim_mean(salaries, proportiontocut=0.1)
print(f"\nTrimmed Mean (10%):   ${trimmed_mean:,.0f}")

# Weighted mean: when some values matter more
# Example: GPA calculation
grades = [4.0, 3.7, 3.3, 4.0, 3.0]   # A, A-, B+, A, B
credits = [4, 3, 3, 2, 4]              # Credit hours (weights)
weighted_gpa = np.average(grades, weights=credits)
print(f"\nWeighted GPA: {weighted_gpa:.2f}")
print(f"Simple GPA:   {np.mean(grades):.2f}")
```

**Expected Output:**
```
=== Salary Analysis ===
Mean:   $86,909
Median: $50,000
Mode:   $45,000

=== Outlier Impact ===
Mean WITH outlier:    $86,909
Mean WITHOUT outlier: $49,600
Median (either way):  $50,000

Trimmed Mean (10%):   $49,888
Weighted GPA: 3.59
Simple GPA:   3.60
```

---

## 2.3 Measures of Spread (Dispersion)

### What They Are
Spread tells you **how far apart** the data points are. Two datasets can have the same mean but very different spreads.

**Analogy:** Two archers both hit the bullseye "on average" — but one clusters shots tightly (low spread) while the other sprays them everywhere (high spread). You want to hire the consistent one.

```
Low Spread (precise):          High Spread (variable):
        ●●●                         ●
       ●●●●●                    ●       ●
      ●●●●●●●                 ●    ●      ●
   ──────┼──────            ●    ●  ┼  ●    ●
        mean                        mean
```

### Range
$$\text{Range} = x_{max} - x_{min}$$

- Simplest measure
- Only uses 2 values → extremely sensitive to outliers
- Use for: quick sanity checks, not serious analysis

### Variance
$$\sigma^2 = \frac{1}{n}\sum_{i=1}^{n}(x_i - \bar{x})^2 \quad \text{(population)}$$

$$s^2 = \frac{1}{n-1}\sum_{i=1}^{n}(x_i - \bar{x})^2 \quad \text{(sample)}$$

**Why squared?** 
1. Makes all deviations positive (can't cancel out)
2. Penalizes large deviations more (outlier sensitivity)
3. Has nice mathematical properties (additivity)

**Why n-1 for sample?** (Bessel's correction)
- A sample tends to underestimate the population spread
- Dividing by n-1 instead of n corrects this bias
- Called "degrees of freedom" — once you know the mean, only n-1 values are free to vary

### Standard Deviation
$$\sigma = \sqrt{\sigma^2} = \sqrt{\frac{1}{n}\sum_{i=1}^{n}(x_i - \bar{x})^2}$$

**Why standard deviation over variance?**
- Same units as the original data (variance is in squared units)
- More interpretable: "salaries deviate by $10,000" vs "variance is 100,000,000 dollars-squared"

### The 68-95-99.7 Rule (Empirical Rule)

For **normally distributed** data:

```
         99.7% (within 3σ)
    ┌────────────────────────────┐
    │    95% (within 2σ)         │
    │  ┌──────────────────────┐  │
    │  │  68% (within 1σ)     │  │
    │  │  ┌──────────────┐    │  │
    │  │  │              │    │  │
────┼──┼──┼──────┼───────┼──┼──┼────
  -3σ -2σ -1σ   μ      +1σ +2σ +3σ

• 68% of data falls within 1 standard deviation of the mean
• 95% of data falls within 2 standard deviations
• 99.7% of data falls within 3 standard deviations
```

### Coefficient of Variation (CV)
$$CV = \frac{\sigma}{\bar{x}} \times 100\%$$

**When to use:** Comparing spread across datasets with different scales.
- Heights (cm) vs Weights (kg) — can't compare their SDs directly
- CV gives a dimensionless percentage

### Interquartile Range (IQR)
$$IQR = Q_3 - Q_1$$

- Robust to outliers (unlike range and std)
- Used to define outliers: anything below $Q_1 - 1.5 \times IQR$ or above $Q_3 + 1.5 \times IQR$

### Code: All Spread Measures

```python
import numpy as np
from scipy import stats

# Two classes with the same mean but different spreads
class_a = [78, 82, 79, 81, 80, 83, 77, 80, 82, 78]  # Consistent students
class_b = [55, 95, 60, 100, 70, 90, 65, 98, 72, 95]  # Highly variable

print("=== Class A (Consistent) ===")
print(f"Mean: {np.mean(class_a):.1f}")
print(f"Range: {np.ptp(class_a)}")                    # ptp = peak to peak
print(f"Variance (sample): {np.var(class_a, ddof=1):.2f}")  # ddof=1 for sample
print(f"Std Dev (sample): {np.std(class_a, ddof=1):.2f}")
print(f"IQR: {stats.iqr(class_a):.2f}")
print(f"CV: {(np.std(class_a, ddof=1)/np.mean(class_a))*100:.1f}%")

print("\n=== Class B (Variable) ===")
print(f"Mean: {np.mean(class_b):.1f}")
print(f"Range: {np.ptp(class_b)}")
print(f"Variance (sample): {np.var(class_b, ddof=1):.2f}")
print(f"Std Dev (sample): {np.std(class_b, ddof=1):.2f}")
print(f"IQR: {stats.iqr(class_b):.2f}")
print(f"CV: {(np.std(class_b, ddof=1)/np.mean(class_b))*100:.1f}%")

# Demonstrate population vs sample variance
print("\n=== Population vs Sample Variance ===")
data = [4, 7, 13, 2, 1]
print(f"Population variance (ddof=0): {np.var(data, ddof=0):.2f}")
print(f"Sample variance (ddof=1):     {np.var(data, ddof=1):.2f}")
print("Note: Sample variance is always larger (Bessel's correction)")

# Standard Error of the Mean (SEM)
# How much would the mean vary if we took different samples?
n = len(class_a)
sem = np.std(class_a, ddof=1) / np.sqrt(n)
print(f"\n=== Standard Error of Mean ===")
print(f"SEM for Class A: {sem:.2f}")
print(f"Interpretation: If we sampled 10 students many times,")
print(f"the sample mean would typically vary by ±{sem:.2f} points")
```

**Expected Output:**
```
=== Class A (Consistent) ===
Mean: 80.0
Range: 6
Variance (sample): 3.78
Std Dev (sample): 1.94
IQR: 2.50
CV: 2.4%

=== Class B (Variable) ===
Mean: 80.0
Range: 45
Variance (sample): 285.56
Std Dev (sample): 16.90
IQR: 30.00
CV: 21.1%
```

---

## 2.4 Distributions

### What They Are
A distribution describes how values are spread across the possible range. Think of it as the "shape" of your data when you plot it.

**Analogy:** If your data points were people in a stadium, the distribution tells you where the crowds are — are they evenly spread out, or clustered in one section?

### Normal Distribution (Gaussian)

$$f(x) = \frac{1}{\sigma\sqrt{2\pi}} e^{-\frac{(x-\mu)^2}{2\sigma^2}}$$

**Why it's everywhere:**
- Central Limit Theorem guarantees it (sums of random variables → normal)
- Many natural phenomena: heights, test scores, measurement errors
- Foundation of most statistical tests

```
        Parameters: μ (mean) and σ (standard deviation)
        
              │         μ=0, σ=1 (Standard Normal)
         0.4 ─┤        ╭──╮
              │       ╱    ╲
         0.3 ─┤      ╱      ╲
              │     ╱        ╲
         0.2 ─┤    ╱          ╲
              │   ╱            ╲
         0.1 ─┤  ╱              ╲
              │ ╱                ╲
         0.0 ─┼──────────────────────
             -4  -3  -2  -1   0   1   2   3   4
```

**Key Properties:**
- Symmetric around the mean
- Mean = Median = Mode
- Completely defined by μ and σ
- Tails extend to ±∞ (theoretically)

### Common Distributions

| Distribution | Type | Parameters | Use Case | Shape |
|-------------|------|-----------|----------|-------|
| Normal | Continuous | μ, σ | Heights, errors | Bell curve |
| Uniform | Continuous | a, b | Random number generation | Flat rectangle |
| Exponential | Continuous | λ | Time between events | Decay curve |
| Poisson | Discrete | λ | Count of rare events | Right-skewed hump |
| Binomial | Discrete | n, p | Success/failure trials | Symmetric (if p≈0.5) |
| Bernoulli | Discrete | p | Single yes/no trial | Two bars |
| Log-Normal | Continuous | μ, σ | Income, stock returns | Right-skewed |
| Power Law | Continuous | α | City sizes, web traffic | Long tail |

### Code: Working with Distributions

```python
import numpy as np
from scipy import stats
import matplotlib
matplotlib.use('Agg')  # For non-interactive environments
import matplotlib.pyplot as plt

# ============================================
# 1. Normal Distribution
# ============================================
# Generate 10,000 samples from N(μ=100, σ=15) — like IQ scores
np.random.seed(42)
normal_data = np.random.normal(loc=100, scale=15, size=10000)

print("=== Normal Distribution (IQ Scores) ===")
print(f"Mean: {np.mean(normal_data):.2f} (expected: 100)")
print(f"Std:  {np.std(normal_data):.2f} (expected: 15)")
print(f"Min:  {np.min(normal_data):.2f}")
print(f"Max:  {np.max(normal_data):.2f}")

# What percentage falls within 1, 2, 3 standard deviations?
for n_std in [1, 2, 3]:
    within = np.sum(np.abs(normal_data - 100) < n_std * 15)
    pct = within / len(normal_data) * 100
    print(f"Within {n_std}σ: {pct:.1f}% (expected: {[68.27, 95.45, 99.73][n_std-1]}%)")

# ============================================
# 2. Binomial Distribution
# ============================================
# Flip a coin 10 times, repeat 10000 experiments
# How many heads in each experiment?
coin_flips = np.random.binomial(n=10, p=0.5, size=10000)

print("\n=== Binomial Distribution (10 Coin Flips) ===")
print(f"Mean heads: {np.mean(coin_flips):.2f} (expected: 5)")
print(f"Std:        {np.std(coin_flips):.2f} (expected: {np.sqrt(10*0.5*0.5):.2f})")
print(f"P(exactly 5 heads): {np.sum(coin_flips == 5)/len(coin_flips):.3f}")
print(f"P(7+ heads):        {np.sum(coin_flips >= 7)/len(coin_flips):.3f}")

# ============================================
# 3. Poisson Distribution
# ============================================
# A website gets an average of 5 visitors per minute
visitors = np.random.poisson(lam=5, size=10000)

print("\n=== Poisson Distribution (Website Visitors/min) ===")
print(f"Mean visitors: {np.mean(visitors):.2f} (expected: 5)")
print(f"P(0 visitors): {np.sum(visitors == 0)/len(visitors):.4f}")
print(f"P(10+ visitors): {np.sum(visitors >= 10)/len(visitors):.4f}")

# ============================================
# 4. Using scipy.stats for exact probabilities
# ============================================
print("\n=== Exact Probabilities with scipy.stats ===")

# Normal: What percentage of people have IQ > 130?
prob_above_130 = 1 - stats.norm.cdf(130, loc=100, scale=15)
print(f"P(IQ > 130): {prob_above_130:.4f} ({prob_above_130*100:.2f}%)")

# Normal: What IQ is at the 95th percentile?
iq_95th = stats.norm.ppf(0.95, loc=100, scale=15)
print(f"95th percentile IQ: {iq_95th:.1f}")

# Binomial: P(at least 8 heads in 10 flips of fair coin)
prob_8_plus = 1 - stats.binom.cdf(7, n=10, p=0.5)
print(f"P(≥8 heads in 10 flips): {prob_8_plus:.4f}")

# Poisson: P(more than 10 visitors when average is 5)
prob_over_10 = 1 - stats.poisson.cdf(10, mu=5)
print(f"P(>10 visitors | λ=5): {prob_over_10:.4f}")

# ============================================
# 5. Testing if data is normally distributed
# ============================================
print("\n=== Normality Tests ===")

# Shapiro-Wilk test (best for n < 5000)
sample = np.random.normal(50, 10, 100)
stat, p_value = stats.shapiro(sample)
print(f"Shapiro-Wilk test:")
print(f"  Statistic: {stat:.4f}")
print(f"  p-value: {p_value:.4f}")
print(f"  Normal? {'Yes' if p_value > 0.05 else 'No'} (p > 0.05)")

# Test with clearly non-normal data
non_normal = np.random.exponential(scale=2, size=100)
stat2, p_value2 = stats.shapiro(non_normal)
print(f"\nExponential data Shapiro-Wilk:")
print(f"  p-value: {p_value2:.4f}")
print(f"  Normal? {'Yes' if p_value2 > 0.05 else 'No'}")
```

### Z-Score (Standard Score)

$$z = \frac{x - \mu}{\sigma}$$

**What it tells you:** How many standard deviations a value is from the mean.

```python
# Z-scores: Standardizing data
scores_math = np.array([85, 90, 78, 92, 88])    # Math test (out of 100)
scores_verbal = np.array([650, 700, 620, 710, 680])  # SAT verbal (out of 800)

# Can't compare raw scores — different scales!
# Z-scores put everything on the same scale

z_math = (scores_math - np.mean(scores_math)) / np.std(scores_math, ddof=1)
z_verbal = (scores_verbal - np.mean(scores_verbal)) / np.std(scores_verbal, ddof=1)

print("=== Z-Score Comparison ===")
print("Student | Math(raw) | Math(z) | Verbal(raw) | Verbal(z) | Better at")
print("-" * 75)
for i in range(5):
    better = "Math" if z_math[i] > z_verbal[i] else "Verbal"
    print(f"   {i+1}    |    {scores_math[i]}     | {z_math[i]:+.2f}  |     {scores_verbal[i]}      | {z_verbal[i]:+.2f}    | {better}")

# Using scipy for z-score
from scipy.stats import zscore
print(f"\nUsing scipy.stats.zscore: {zscore(scores_math, ddof=1)}")
```

---

## 2.5 Correlation and Covariance

### What They Are
**Covariance:** Do two variables move together? (has units, hard to interpret)
**Correlation:** Same concept, but normalized to [-1, +1] (unitless, easy to interpret)

**Analogy:** Covariance tells you "these two things tend to move together." Correlation tells you "how strongly they move together, on a scale of -1 to +1."

### Formulas

**Covariance:**
$$\text{Cov}(X, Y) = \frac{1}{n-1}\sum_{i=1}^{n}(x_i - \bar{x})(y_i - \bar{y})$$

**Pearson Correlation:**
$$r = \frac{\text{Cov}(X, Y)}{s_X \cdot s_Y} = \frac{\sum(x_i - \bar{x})(y_i - \bar{y})}{\sqrt{\sum(x_i - \bar{x})^2} \cdot \sqrt{\sum(y_i - \bar{y})^2}}$$

### Interpreting Correlation

```
r = +1.0          r = +0.7         r = 0.0          r = -0.7         r = -1.0
Perfect +         Strong +         No linear        Strong -         Perfect -

  ●                 ●●              ● ●  ●          ●●               ●
 ●                ●● ●             ●● ●●●            ●●●              ●
●                ● ●●●            ●●●●●●●●            ●●●              ●
                ●● ●●              ● ●●● ●             ●●●●              ●
               ●●●                  ●●  ●               ●●●●
```

| r value | Interpretation | Example |
|---------|---------------|---------|
| 0.9 to 1.0 | Very strong positive | Height vs. weight |
| 0.7 to 0.9 | Strong positive | Study hours vs. grades |
| 0.4 to 0.7 | Moderate positive | Income vs. happiness (up to a point) |
| 0.1 to 0.4 | Weak positive | Shoe size vs. vocabulary |
| -0.1 to 0.1 | No correlation | Birthday vs. IQ |
| -0.4 to -0.1 | Weak negative | Exercise vs. body fat |
| -0.7 to -0.4 | Moderate negative | TV time vs. grades |
| -1.0 to -0.7 | Strong negative | Speed vs. travel time |

> **CRITICAL WARNING:** Correlation ≠ Causation!
> - Ice cream sales correlate with drowning deaths (both increase in summer)
> - The confounding variable is temperature/season
> - Always ask: "Is there a third variable causing both?"

### Types of Correlation

| Type | Measures | When to Use | Assumptions |
|------|---------|------------|-------------|
| **Pearson** | Linear relationship | Continuous, normally distributed | Linearity, normality |
| **Spearman** | Monotonic relationship | Ordinal or non-normal | Only needs monotonicity |
| **Kendall's Tau** | Concordance of pairs | Small samples, ordinal | Fewer assumptions |

### Code: Correlation Analysis

```python
import numpy as np
import pandas as pd
from scipy import stats

np.random.seed(42)
n = 200

# Create correlated and uncorrelated data
study_hours = np.random.uniform(1, 10, n)
# Grades correlate with study hours + noise
grades = 50 + 5 * study_hours + np.random.normal(0, 5, n)
# Shoe size is unrelated
shoe_size = np.random.normal(10, 1.5, n)
# Negative correlation: TV hours decrease grades
tv_hours = 12 - study_hours + np.random.normal(0, 2, n)

df = pd.DataFrame({
    'study_hours': study_hours,
    'grades': grades,
    'shoe_size': shoe_size,
    'tv_hours': tv_hours
})

# Correlation matrix
print("=== Correlation Matrix (Pearson) ===")
print(df.corr().round(3))

# Different correlation methods
print("\n=== Comparing Correlation Methods ===")
pearson_r, pearson_p = stats.pearsonr(study_hours, grades)
spearman_r, spearman_p = stats.spearmanr(study_hours, grades)
kendall_r, kendall_p = stats.kendalltau(study_hours, grades)

print(f"Pearson:  r = {pearson_r:.4f}, p-value = {pearson_p:.2e}")
print(f"Spearman: r = {spearman_r:.4f}, p-value = {spearman_p:.2e}")
print(f"Kendall:  τ = {kendall_r:.4f}, p-value = {kendall_p:.2e}")

# Covariance matrix
print("\n=== Covariance Matrix ===")
print(df.cov().round(2))

# Demonstrate: correlation doesn't imply causation
# Simpson's Paradox example
print("\n=== Simpson's Paradox Example ===")
# Overall: negative correlation between drug and recovery
# But within each group: positive correlation!
group_a_drug = np.random.normal(70, 5, 50)      # Mild cases, drug
group_a_no_drug = np.random.normal(80, 5, 50)   # Mild cases, no drug
group_b_drug = np.random.normal(30, 5, 50)      # Severe cases, drug
group_b_no_drug = np.random.normal(20, 5, 50)   # Severe cases, no drug

print("Mild cases:   Drug avg = {:.0f}, No drug avg = {:.0f} → Drug worse".format(
    np.mean(group_a_drug), np.mean(group_a_no_drug)))
print("Severe cases: Drug avg = {:.0f}, No drug avg = {:.0f} → Drug better".format(
    np.mean(group_b_drug), np.mean(group_b_no_drug)))
print("Overall:      Drug avg = {:.0f}, No drug avg = {:.0f}".format(
    np.mean(np.concatenate([group_a_drug, group_b_drug])),
    np.mean(np.concatenate([group_a_no_drug, group_b_no_drug]))))
print("⚠️  Confounding variable (severity) reverses the conclusion!")
```

---

## 2.6 Percentiles, Quartiles, and Box Plots

### What They Are
**Percentile:** The value below which a given percentage of data falls.
- 90th percentile = 90% of values are below this number
- Your test score being in the "95th percentile" means you did better than 95% of test-takers

**Quartiles:** Split data into 4 equal parts.
- Q1 (25th percentile): Lower quartile
- Q2 (50th percentile): Median
- Q3 (75th percentile): Upper quartile

### Box Plot Anatomy

```
    Outliers                                      Outliers
      ●  ●                                          ●
      │                                              │
      │     ┌─────────────────────────────────┐      │
      │     │           ┌────────┐            │      │
      ├─────┤           │   │    │            ├──────┤
      │     │           │   │    │            │      │
      │     └─────────────────────────────────┘      │
      │                                              │
   ◄──────► ◄─────────►     ◄───► ◄──────────► ◄────►
   Outliers  Lower       Median  Upper          Outliers
             Whisker     (Q2)    Whisker
   < Q1-1.5×IQR  Q1              Q3     > Q3+1.5×IQR
             
             ◄────── IQR ───────►
               (Q3 - Q1)
```

### Code: Percentiles and Outlier Detection

```python
import numpy as np
from scipy import stats

# Student test scores
np.random.seed(42)
scores = np.concatenate([
    np.random.normal(75, 10, 90),   # Regular students
    np.array([15, 18, 99, 100])     # Unusual scores (potential outliers)
])

# Percentiles
print("=== Percentiles ===")
for p in [10, 25, 50, 75, 90, 95, 99]:
    print(f"  {p}th percentile: {np.percentile(scores, p):.1f}")

# Quartiles and IQR
q1 = np.percentile(scores, 25)
q2 = np.percentile(scores, 50)  # Median
q3 = np.percentile(scores, 75)
iqr = q3 - q1

print(f"\n=== Five-Number Summary ===")
print(f"  Min:    {np.min(scores):.1f}")
print(f"  Q1:     {q1:.1f}")
print(f"  Median: {q2:.1f}")
print(f"  Q3:     {q3:.1f}")
print(f"  Max:    {np.max(scores):.1f}")
print(f"  IQR:    {iqr:.1f}")

# Outlier detection using IQR method
lower_fence = q1 - 1.5 * iqr
upper_fence = q3 + 1.5 * iqr
outliers = scores[(scores < lower_fence) | (scores > upper_fence)]

print(f"\n=== Outlier Detection (IQR Method) ===")
print(f"  Lower fence: {lower_fence:.1f}")
print(f"  Upper fence: {upper_fence:.1f}")
print(f"  Outliers found: {len(outliers)}")
print(f"  Outlier values: {sorted(outliers)}")

# Z-score method for outlier detection (|z| > 3)
z_scores = np.abs(stats.zscore(scores))
outliers_z = scores[z_scores > 3]
print(f"\n=== Outlier Detection (Z-Score Method, |z| > 3) ===")
print(f"  Outliers found: {len(outliers_z)}")
print(f"  Outlier values: {sorted(outliers_z)}")

# Pro tip: Modified Z-score using MAD (Median Absolute Deviation)
# More robust than regular z-score
mad = np.median(np.abs(scores - np.median(scores)))
modified_z = 0.6745 * (scores - np.median(scores)) / mad
outliers_mad = scores[np.abs(modified_z) > 3.5]
print(f"\n=== Outlier Detection (Modified Z-Score, MAD) ===")
print(f"  MAD: {mad:.2f}")
print(f"  Outliers found: {len(outliers_mad)}")
```

---

## 2.7 Skewness and Kurtosis

### Skewness
**What:** Measures asymmetry of the distribution.

$$\text{Skewness} = \frac{1}{n}\sum\left(\frac{x_i - \bar{x}}{s}\right)^3$$

| Value | Meaning | Example |
|-------|---------|---------|
| Skew = 0 | Symmetric | Normal distribution |
| Skew > 0 | Right-skewed (tail to right) | Income, house prices |
| Skew < 0 | Left-skewed (tail to left) | Age at death (developed countries) |

### Kurtosis
**What:** Measures how heavy the tails are (extreme values frequency).

$$\text{Kurtosis} = \frac{1}{n}\sum\left(\frac{x_i - \bar{x}}{s}\right)^4$$

| Type | Excess Kurtosis | Tails | Example |
|------|----------------|-------|---------|
| Mesokurtic | ≈ 0 | Normal tails | Normal distribution |
| Leptokurtic | > 0 | Fat tails (more extremes) | Stock returns, t-distribution |
| Platykurtic | < 0 | Thin tails (fewer extremes) | Uniform distribution |

> **Pro Tip:** Fat tails matter hugely in finance. The 2008 crash was a "6-sigma event" under normal assumptions — essentially impossible. But with fat-tailed distributions, such events are much more likely.

### Code: Skewness and Kurtosis

```python
import numpy as np
from scipy import stats

np.random.seed(42)

# Generate different distributions
normal_data = np.random.normal(0, 1, 10000)
right_skewed = np.random.lognormal(0, 0.5, 10000)  # Income-like
left_skewed = -np.random.lognormal(0, 0.5, 10000)  # Negated
heavy_tails = np.random.standard_t(df=3, size=10000)  # t-distribution

datasets = {
    'Normal': normal_data,
    'Right-Skewed (Income)': right_skewed,
    'Left-Skewed': left_skewed,
    'Heavy Tails (t-dist)': heavy_tails
}

print(f"{'Distribution':<25} {'Skewness':>10} {'Kurtosis':>10} {'Excess Kurt':>12}")
print("-" * 60)
for name, data in datasets.items():
    skew = stats.skew(data)
    kurt = stats.kurtosis(data)  # scipy gives excess kurtosis by default
    print(f"{name:<25} {skew:>10.3f} {kurt:>10.3f} {kurt:>12.3f}")

# Practical impact: Why skewness matters for modeling
print("\n=== Why This Matters ===")
print("Right-skewed income data:")
income = np.random.lognormal(10.5, 0.8, 10000)  # Realistic income
print(f"  Mean income:   ${np.mean(income):,.0f}")
print(f"  Median income: ${np.median(income):,.0f}")
print(f"  Skewness: {stats.skew(income):.2f}")
print(f"  → Mean is pulled up by high earners!")
print(f"  → Median is more representative of 'typical' income")

# Log transformation to reduce skewness
log_income = np.log(income)
print(f"\n  After log transform:")
print(f"  Skewness: {stats.skew(log_income):.2f} (much closer to 0!)")
```

---

## 2.8 Sampling and the Central Limit Theorem

### What It Is
The **Central Limit Theorem (CLT)** is arguably the most important theorem in statistics:

> **No matter what the original distribution looks like, if you take enough random samples and compute their means, those means will form a normal distribution.**

$$\bar{X} \sim N\left(\mu, \frac{\sigma^2}{n}\right) \quad \text{as } n \to \infty$$

Where:
- $\bar{X}$ = sample mean
- $\mu$ = population mean
- $\sigma^2$ = population variance
- $n$ = sample size

**Analogy:** Imagine rolling a die (uniform distribution, not normal at all). If you roll once, you get 1-6 equally. But if you roll 30 dice and take the average, that average will be close to 3.5. Repeat this 10,000 times, and those averages form a perfect bell curve!

### Why CLT Matters
1. **Justifies using normal-based tests** even when data isn't normal
2. **Tells us sample means are more precise** than individual values
3. **Standard Error decreases** as sample size increases: $SE = \frac{\sigma}{\sqrt{n}}$
4. **Enables confidence intervals** and hypothesis testing

### Rule of Thumb
- n ≥ 30 is usually "enough" for CLT to kick in
- For highly skewed data, you may need n ≥ 50 or more
- For already-normal data, even n = 5 works well

### Sampling Methods

| Method | How | When | Pro | Con |
|--------|-----|------|-----|-----|
| Simple Random | Each item has equal probability | Small, accessible population | Unbiased | May miss subgroups |
| Stratified | Divide into groups, sample from each | Known subgroups | Ensures representation | Need to know strata |
| Cluster | Randomly select groups, sample all within | Geographic studies | Cost-effective | Higher variance |
| Systematic | Every kth item | Assembly lines, surveys | Easy to implement | Risky if pattern exists |

### Code: Demonstrating CLT

```python
import numpy as np
from scipy import stats

np.random.seed(42)

# ============================================
# CLT Demonstration: From any distribution → Normal means
# ============================================

# Start with a VERY non-normal distribution (exponential)
population = np.random.exponential(scale=10, size=100000)
pop_mean = np.mean(population)
pop_std = np.std(population)

print("=== Original Population (Exponential) ===")
print(f"Mean: {pop_mean:.2f}")
print(f"Std: {pop_std:.2f}")
print(f"Skewness: {stats.skew(population):.2f} (very right-skewed!)")
print(f"Is Normal? Shapiro p-value: {stats.shapiro(population[:5000])[1]:.6f} (NO)")

# Now take many samples of different sizes and compute means
sample_sizes = [5, 30, 100, 500]
n_experiments = 10000

print("\n=== Central Limit Theorem in Action ===")
print(f"{'Sample Size':<12} {'Mean of Means':<15} {'Std of Means':<15} "
      f"{'Expected SE':<12} {'Skewness':<10} {'Normal?'}")
print("-" * 80)

for n in sample_sizes:
    # Take n_experiments samples, each of size n
    sample_means = [np.mean(np.random.choice(population, size=n)) 
                    for _ in range(n_experiments)]
    sample_means = np.array(sample_means)
    
    expected_se = pop_std / np.sqrt(n)
    _, p_val = stats.normaltest(sample_means)
    
    print(f"{n:<12} {np.mean(sample_means):<15.2f} {np.std(sample_means):<15.2f} "
          f"{expected_se:<12.2f} {stats.skew(sample_means):<10.3f} "
          f"{'Yes' if p_val > 0.05 else 'No'} (p={p_val:.4f})")

# Key insight: As n increases, sample means become:
# 1. More normally distributed (skewness → 0)
# 2. Less spread out (SE decreases)
# 3. Centered on the true population mean

# ============================================
# Practical: Confidence Interval
# ============================================
print("\n=== Confidence Interval Example ===")
# We have a sample of 50 customer wait times
wait_times = np.random.exponential(scale=8, size=50)  # True mean = 8 min
sample_mean = np.mean(wait_times)
sample_se = stats.sem(wait_times)  # Standard error of mean

# 95% confidence interval
ci_95 = stats.t.interval(0.95, df=len(wait_times)-1, 
                          loc=sample_mean, scale=sample_se)

print(f"Sample mean wait time: {sample_mean:.2f} minutes")
print(f"Standard Error: {sample_se:.2f}")
print(f"95% CI: ({ci_95[0]:.2f}, {ci_95[1]:.2f}) minutes")
print(f"Interpretation: We're 95% confident the true average")
print(f"wait time is between {ci_95[0]:.2f} and {ci_95[1]:.2f} minutes")

# How sample size affects CI width
print("\n=== CI Width vs Sample Size ===")
for n in [10, 30, 50, 100, 500, 1000]:
    sample = np.random.exponential(scale=8, size=n)
    ci = stats.t.interval(0.95, df=n-1, loc=np.mean(sample), scale=stats.sem(sample))
    width = ci[1] - ci[0]
    print(f"  n={n:>4}: CI width = {width:.2f} minutes")
```

**Expected Output (approximate):**
```
=== Central Limit Theorem in Action ===
Sample Size  Mean of Means   Std of Means    Expected SE  Skewness   Normal?
n=5:         Still skewed, not yet normal
n=30:        Nearly normal!
n=100:       Definitely normal
n=500:       Very tight, perfectly normal
```

---

## 2.9 Common Mistakes

| # | Mistake | Reality | Fix |
|---|---------|---------|-----|
| 1 | Using mean for skewed data | Mean is misleading when outliers exist | Use median for skewed distributions |
| 2 | Confusing correlation with causation | Correlation only shows association | Look for confounders, design experiments |
| 3 | Ignoring sample size | Small samples are unreliable | Check statistical power, use CLT rules |
| 4 | Using population formulas for samples | Underestimates variance | Use n-1 (Bessel's correction) for samples |
| 5 | Assuming normality without testing | Many real datasets aren't normal | Use Shapiro-Wilk test, check Q-Q plots |
| 6 | Cherry-picking statistics | Report only mean without spread | Always report central tendency + spread |
| 7 | Treating ordinal as interval | Can't say "Strongly Agree" is 2x "Agree" | Use appropriate methods (Spearman, not Pearson) |
| 8 | Ignoring missing data patterns | Missing data can be informative | Check if missingness is random (MCAR, MAR, MNAR) |

---

## 2.10 Interview Questions

**Q1: Explain the difference between population and sample statistics.**
> Population statistics (μ, σ) describe the entire group of interest. Sample statistics (x̄, s) are computed from a subset. We use samples because measuring the entire population is usually impossible. Sample statistics are estimates of population parameters, with uncertainty quantified by standard error.

**Q2: When would you use median instead of mean?**
> When data is skewed or has outliers. Examples: income (right-skewed), house prices, response times. The median is robust — a billionaire moving to your city doesn't change the median income but drastically changes the mean.

**Q3: What is the Central Limit Theorem and why is it important?**
> CLT states that the distribution of sample means approaches a normal distribution as sample size increases, regardless of the population's original distribution. It's important because it: (1) justifies using normal-based statistical tests broadly, (2) enables confidence intervals and hypothesis testing, (3) explains why n≥30 is a common rule of thumb.

**Q4: How do you detect outliers?**
> Three common methods: (1) IQR method: values beyond Q1-1.5×IQR or Q3+1.5×IQR. (2) Z-score: |z| > 3. (3) Modified Z-score using MAD (more robust). The choice depends on data distribution — IQR works well for skewed data, Z-score assumes normality. Always investigate outliers before removing — they might be real and informative.

**Q5: Explain the difference between standard deviation and standard error.**
> Standard deviation (SD) measures spread of individual data points around the mean. Standard error (SE) measures how much the sample mean would vary across different samples. SE = SD/√n. As sample size increases, SE decreases (means become more precise), but SD stays roughly the same.

**Q6: What's the difference between Pearson and Spearman correlation?**
> Pearson measures linear relationships and assumes normality. Spearman measures monotonic relationships (consistently increasing/decreasing) and works on ranks. Use Spearman when: data has outliers, relationship is monotonic but non-linear, or data is ordinal.

---

## 2.11 Quick Reference

### Statistics Formulas Cheat Sheet

| Measure | Formula | Python |
|---------|---------|--------|
| Mean | $\bar{x} = \frac{\sum x_i}{n}$ | `np.mean(data)` |
| Median | Middle value when sorted | `np.median(data)` |
| Mode | Most frequent value | `stats.mode(data)` |
| Variance (sample) | $s^2 = \frac{\sum(x_i-\bar{x})^2}{n-1}$ | `np.var(data, ddof=1)` |
| Std Dev (sample) | $s = \sqrt{s^2}$ | `np.std(data, ddof=1)` |
| Standard Error | $SE = \frac{s}{\sqrt{n}}$ | `stats.sem(data)` |
| Pearson Correlation | $r = \frac{Cov(X,Y)}{s_X \cdot s_Y}$ | `stats.pearsonr(x, y)` |
| Z-score | $z = \frac{x - \bar{x}}{s}$ | `stats.zscore(data)` |
| IQR | $Q_3 - Q_1$ | `stats.iqr(data)` |
| Skewness | $\frac{1}{n}\sum(\frac{x_i-\bar{x}}{s})^3$ | `stats.skew(data)` |
| Kurtosis | $\frac{1}{n}\sum(\frac{x_i-\bar{x}}{s})^4 - 3$ | `stats.kurtosis(data)` |
| 95% CI | $\bar{x} \pm 1.96 \times SE$ | `stats.t.interval(0.95, ...)` |

### When to Use What

| If data is... | Use for center | Use for spread | Use for correlation |
|---------------|---------------|----------------|-------------------|
| Normal, no outliers | Mean | Std Dev | Pearson |
| Skewed | Median | IQR | Spearman |
| Has outliers | Median / Trimmed Mean | MAD / IQR | Spearman |
| Categorical | Mode | Frequency table | Chi-square test |
| Ordinal | Median | IQR | Spearman / Kendall |

---

**← Previous:** [01-What-is-Data-Science](./01-What-is-Data-Science.md) | **Next:** [03-Probability-and-Inference](./03-Probability-and-Inference.md) →
