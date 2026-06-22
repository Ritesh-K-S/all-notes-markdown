# Chapter 09: A/B Testing and Experimentation

## Table of Contents
- [What is A/B Testing?](#what-is-ab-testing)
- [Why It Matters](#why-it-matters)
- [How A/B Testing Works](#how-ab-testing-works)
- [Statistical Foundations](#statistical-foundations)
- [Designing an Experiment](#designing-an-experiment)
- [Sample Size Calculation](#sample-size-calculation)
- [Running the Test](#running-the-test)
- [Analyzing Results](#analyzing-results)
- [Advanced Topics](#advanced-topics)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is A/B Testing?

### Simple Explanation

Imagine you run a lemonade stand. You think a new sign design might attract more customers. Instead of guessing, you put the **old sign (A)** out on Monday and the **new sign (B)** out on Tuesday, then count customers. But wait — maybe Tuesday is just busier! So instead, you put **both signs at two identical stands side by side** and randomly direct people to one or the other. After enough customers, you compare. That's A/B testing.

**A/B Testing** = Showing two versions of something to random groups of users and measuring which version performs better, using statistics to determine if the difference is real or just luck.

```
┌─────────────────────────────────────────────────────────────┐
│                      ALL USERS                                │
│                         │                                     │
│              Random 50/50 Split                               │
│                    ┌────┴────┐                                │
│                    ▼         ▼                                │
│            ┌──────────┐ ┌──────────┐                         │
│            │ CONTROL  │ │TREATMENT │                         │
│            │ (Group A)│ │ (Group B)│                         │
│            │          │ │          │                         │
│            │ Old Page │ │ New Page │                         │
│            └────┬─────┘ └────┬─────┘                         │
│                 │            │                                │
│                 ▼            ▼                                │
│            Measure       Measure                             │
│            Metric        Metric                              │
│                 │            │                                │
│                 └─────┬──────┘                                │
│                       ▼                                       │
│              Statistical Test:                                │
│        "Is the difference significant?"                      │
└─────────────────────────────────────────────────────────────┘
```

---

## Why It Matters

### Real-World Impact

| Company | What They Tested | Result |
|---------|-----------------|--------|
| Google | 41 shades of blue for link color | Identified shade that earned extra $200M/year |
| Booking.com | Runs 1000+ experiments simultaneously | Data-driven culture, minimal HiPPO decisions |
| Netflix | Thumbnail images for shows | Personalized thumbnails increased engagement 20-30% |
| Amazon | "Add to Cart" button placement | Minor position change = millions in revenue |

### When to Use A/B Testing

| Use A/B Testing When | Don't Use A/B Testing When |
|---------------------|---------------------------|
| You have enough traffic/users | Sample size too small (<1000 per group) |
| Change is reversible | Change is permanent (new factory) |
| You can measure the outcome | Outcome takes years to observe |
| Randomization is possible | Can't randomly assign users |
| You want causal evidence | Correlation analysis is sufficient |

> **Key Insight:** A/B testing gives you **causation**, not just correlation. You can say "the new button **caused** a 5% increase in signups" — not just "users who saw the new button signed up more (maybe they were different people)."

---

## How A/B Testing Works

### The Complete Lifecycle

```
1. HYPOTHESIS          2. DESIGN             3. IMPLEMENT
   ─────────              ──────                ─────────
   "If we change X,      Power analysis        Build variants
   metric Y will         Sample size calc      Set up tracking
   improve by Z%"        Duration estimate     QA testing

4. RUN                 5. ANALYZE            6. DECIDE
   ───                    ───────               ──────
   Split traffic         Check significance    Ship winner OR
   Monitor health        Effect size           Iterate further
   Watch for bugs        Confidence interval   Document learnings
```

### Key Terminology

| Term | Definition | Analogy |
|------|-----------|---------|
| **Control (A)** | The existing version | The current medicine |
| **Treatment (B)** | The new version being tested | The experimental medicine |
| **Metric** | What you measure (conversion, revenue, etc.) | Patient's temperature |
| **Statistical Significance** | Confidence the result isn't due to chance | "Are we sure the medicine works?" |
| **Effect Size** | How big the difference is | "How much did the fever drop?" |
| **p-value** | Probability of seeing this result if there's no real difference | False alarm rate |
| **Power** | Ability to detect a real effect if it exists | Sensitivity of the thermometer |
| **Sample Size** | Number of observations needed | How many patients to test |

---

## Statistical Foundations

### Hypothesis Testing Framework

Every A/B test is a hypothesis test:

- **Null Hypothesis ($H_0$):** There is NO difference between A and B
  - "The new button doesn't change conversion rate"
- **Alternative Hypothesis ($H_1$):** There IS a difference
  - "The new button changes the conversion rate"

### Type I and Type II Errors

```
                        REALITY
                  ┌──────────┬──────────┐
                  │  H₀ True │ H₀ False │
                  │(No effect)│(Real effect)│
    ┌─────────────┼──────────┼──────────┤
    │ Reject H₀   │ Type I   │ Correct  │
D   │ (Ship it!)  │ Error α  │ (Power)  │
E   │             │ FALSE    │ TRUE     │
C   │             │ POSITIVE │ POSITIVE │
I   ├─────────────┼──────────┼──────────┤
S   │ Fail to     │ Correct  │ Type II  │
I   │ Reject H₀   │          │ Error β  │
O   │ (Don't ship)│ TRUE     │ FALSE    │
N   │             │ NEGATIVE │ NEGATIVE │
    └─────────────┴──────────┴──────────┘
```

| Error Type | What It Means | Consequence | Typical Threshold |
|-----------|---------------|-------------|-------------------|
| **Type I (α)** | Detected effect that doesn't exist | Ship a change that doesn't help (wasted effort) | 5% (0.05) |
| **Type II (β)** | Missed a real effect | Rejected a good idea | 20% (0.20) |
| **Power (1-β)** | Probability of detecting real effect | Ability to find winners | 80% |

### The p-value — Most Misunderstood Concept

**What p-value IS:**
> The probability of observing data this extreme (or more extreme) IF the null hypothesis were true.

**What p-value is NOT:**
- ❌ The probability that the null hypothesis is true
- ❌ The probability that the result happened by chance
- ❌ The probability that you'll get the same result again

**Analogy:** You flip a coin 10 times and get 9 heads. The p-value answers: "If the coin were fair, what's the probability of getting 9+ heads?" (Answer: ~1%). This makes you suspect the coin isn't fair — but it doesn't PROVE it.

### Confidence Intervals

A confidence interval gives you a **range** of plausible values for the true effect size.

$$CI = \hat{p} \pm z_{\alpha/2} \cdot \sqrt{\frac{\hat{p}(1-\hat{p})}{n}}$$

Where:
- $\hat{p}$ = observed proportion (e.g., conversion rate)
- $z_{\alpha/2}$ = z-score for desired confidence level (1.96 for 95%)
- $n$ = sample size

**Interpretation:** "We are 95% confident that the true conversion rate lies between 4.2% and 5.8%."

> **Pro Tip:** Always report confidence intervals, not just p-values. A p-value tells you IF there's an effect; a CI tells you HOW BIG it might be. A "significant" effect of 0.001% improvement is useless even if p < 0.05.

---

## Designing an Experiment

### Step 1: Define the Hypothesis

```
Template:
"If we [change], then [metric] will [direction] by [expected effect size]
because [reason/mechanism]."

Example:
"If we simplify the checkout form from 5 fields to 3 fields,
then checkout completion rate will increase by at least 5%
because fewer fields reduce friction and abandonment."
```

### Step 2: Choose Metrics

#### Primary Metric (One!)
The single metric that determines success/failure.
- Must directly relate to the hypothesis
- Example: "Checkout completion rate"

#### Secondary Metrics
Additional metrics you monitor but don't use for the decision.
- Revenue per user, time on page, return rate

#### Guardrail Metrics
Metrics that must NOT degrade.
- Page load time, error rate, customer support tickets

```
┌─────────────────────────────────────────────────────┐
│              METRIC HIERARCHY                         │
├─────────────────────────────────────────────────────┤
│                                                       │
│  Primary (1 metric):  Checkout Completion Rate       │
│       ↑ This decides if we ship                      │
│                                                       │
│  Secondary (2-3):     Revenue per user               │
│                       Average order value             │
│       ↑ These provide additional insight             │
│                                                       │
│  Guardrail (2-3):     Page load time < 3s            │
│                       Error rate < 1%                 │
│                       Support tickets not +10%        │
│       ↑ These must not get worse                     │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### Step 3: Determine the Randomization Unit

| Unit | When to Use | Example |
|------|-------------|---------|
| User | Most common. User sees consistent experience | UI changes, pricing |
| Session | When user-level isn't possible | Anonymous visitors |
| Page view | When each view is independent | Ad placement |
| Geographic region | When individual randomization isn't possible | Delivery speed changes |

> **Critical:** Whatever unit you randomize on, you must ANALYZE on the same unit. If you randomize by user, your analysis unit is the user.

---

## Sample Size Calculation

### Why It Matters

Running a test too short → You might miss a real effect (underpowered)
Running a test too long → Wasted time and resources, increased risk of external factors

### The Formula

For a two-proportion z-test (comparing two conversion rates):

$$n = \frac{(z_{\alpha/2} + z_{\beta})^2 \cdot [p_1(1-p_1) + p_2(1-p_2)]}{(p_1 - p_2)^2}$$

Where:
- $n$ = sample size per group
- $p_1$ = baseline conversion rate
- $p_2$ = expected conversion rate after change (= $p_1 \times (1 + \text{MDE})$)
- $z_{\alpha/2}$ = 1.96 for α=0.05 (two-tailed)
- $z_{\beta}$ = 0.84 for power=0.80

### Python: Sample Size Calculator

```python
import numpy as np
from scipy import stats
from statsmodels.stats.power import NormalIndPower
from statsmodels.stats.proportion import proportion_effectsize

def calculate_sample_size(
    baseline_rate: float,      # Current conversion rate (e.g., 0.10 for 10%)
    mde: float,                # Minimum Detectable Effect (e.g., 0.05 for 5% relative)
    alpha: float = 0.05,       # Significance level (Type I error rate)
    power: float = 0.80,       # Statistical power (1 - Type II error rate)
    two_tailed: bool = True    # Two-tailed test (detect increase OR decrease)
) -> dict:
    """
    Calculate required sample size for an A/B test.
    
    Parameters:
    -----------
    baseline_rate : float
        Current conversion rate (e.g., 0.10 means 10%)
    mde : float  
        Minimum detectable effect as RELATIVE change
        0.05 means detect 5% relative improvement (10% → 10.5%)
    alpha : float
        Probability of false positive (typically 0.05)
    power : float
        Probability of detecting true effect (typically 0.80)
    """
    # Calculate the new expected rate
    new_rate = baseline_rate * (1 + mde)
    
    # Calculate effect size (Cohen's h for proportions)
    effect_size = proportion_effectsize(new_rate, baseline_rate)
    
    # Calculate sample size using statsmodels
    analysis = NormalIndPower()
    sample_size_per_group = analysis.solve_power(
        effect_size=effect_size,
        alpha=alpha,
        power=power,
        alternative='two-sided' if two_tailed else 'larger'
    )
    
    sample_size_per_group = int(np.ceil(sample_size_per_group))
    total_sample = sample_size_per_group * 2
    
    return {
        'sample_size_per_group': sample_size_per_group,
        'total_sample_needed': total_sample,
        'baseline_rate': baseline_rate,
        'expected_new_rate': new_rate,
        'absolute_difference': new_rate - baseline_rate,
        'effect_size_cohens_h': round(effect_size, 4),
    }

# Example: Current conversion is 10%, want to detect 5% relative improvement
result = calculate_sample_size(
    baseline_rate=0.10,   # 10% current conversion
    mde=0.05,             # Detect 5% relative lift (10% → 10.5%)
    alpha=0.05,
    power=0.80
)

print("Sample Size Calculation:")
print(f"  Baseline rate: {result['baseline_rate']:.1%}")
print(f"  Expected new rate: {result['expected_new_rate']:.1%}")
print(f"  Absolute difference: {result['absolute_difference']:.2%}")
print(f"  Sample per group: {result['sample_size_per_group']:,}")
print(f"  Total needed: {result['total_sample_needed']:,}")
```

**Output:**
```
Sample Size Calculation:
  Baseline rate: 10.0%
  Expected new rate: 10.5%
  Absolute difference: 0.50%
  Sample per group: 31,234
  Total needed: 62,468
```

### Duration Estimation

```python
def estimate_test_duration(
    sample_size_per_group: int,
    daily_traffic: int,
    traffic_allocation: float = 1.0  # Fraction of traffic in test
) -> dict:
    """Estimate how many days the test needs to run."""
    
    # Each group gets half the allocated traffic
    daily_per_group = (daily_traffic * traffic_allocation) / 2
    
    days_needed = int(np.ceil(sample_size_per_group / daily_per_group))
    
    # Add buffer for weekday/weekend variation (run full weeks)
    weeks_needed = int(np.ceil(days_needed / 7))
    recommended_days = weeks_needed * 7
    
    return {
        'minimum_days': days_needed,
        'recommended_days': recommended_days,  # Full weeks
        'daily_users_per_group': int(daily_per_group),
    }

duration = estimate_test_duration(
    sample_size_per_group=31234,
    daily_traffic=10000,     # 10K visitors/day
    traffic_allocation=1.0   # 100% of traffic in test
)

print(f"Minimum duration: {duration['minimum_days']} days")
print(f"Recommended (full weeks): {duration['recommended_days']} days")
# Output: ~7 days minimum, 7 days recommended
```

### Sample Size Sensitivity

| Baseline Rate | MDE (Relative) | Sample Per Group | Intuition |
|--------------|----------------|-----------------|-----------|
| 10% | 5% (10%→10.5%) | ~31,000 | Small change on low base = hard to detect |
| 10% | 10% (10%→11%) | ~8,000 | Moderate change = reasonable sample |
| 10% | 20% (10%→12%) | ~2,100 | Large change = easy to detect |
| 50% | 5% (50%→52.5%) | ~3,000 | High base rate = easier to detect changes |
| 1% | 10% (1%→1.1%) | ~800,000 | Very low base = need massive sample |

> **Key Takeaway:** The smaller the effect you want to detect, the more data you need. The lower your baseline rate, the more data you need. This is why low-traffic sites can't detect small improvements.

---

## Running the Test

### Implementation in Python (Simulation)

```python
import numpy as np
import pandas as pd
from datetime import datetime, timedelta

def simulate_ab_test(
    baseline_rate: float = 0.10,
    treatment_effect: float = 0.02,  # Absolute increase (10% → 12%)
    n_per_group: int = 5000,
    seed: int = 42
) -> pd.DataFrame:
    """Simulate an A/B test with known truth (for learning)."""
    
    np.random.seed(seed)
    
    # Generate data for control group
    control = pd.DataFrame({
        'user_id': range(1, n_per_group + 1),
        'group': 'control',
        'converted': np.random.binomial(1, baseline_rate, n_per_group)
    })
    
    # Generate data for treatment group
    treatment = pd.DataFrame({
        'user_id': range(n_per_group + 1, 2 * n_per_group + 1),
        'group': 'treatment',
        'converted': np.random.binomial(1, baseline_rate + treatment_effect, n_per_group)
    })
    
    # Combine
    df = pd.concat([control, treatment], ignore_index=True)
    
    # Add some realistic noise: random timestamps
    start_date = datetime(2024, 1, 1)
    df['timestamp'] = [start_date + timedelta(hours=np.random.randint(0, 24*14)) 
                       for _ in range(len(df))]
    
    return df

# Simulate: true effect is 2 percentage points (10% → 12%)
data = simulate_ab_test(
    baseline_rate=0.10,
    treatment_effect=0.02,
    n_per_group=5000
)

# Quick summary
print(data.groupby('group')['converted'].agg(['count', 'sum', 'mean']))
```

### Health Checks During the Test

```python
def run_health_checks(df: pd.DataFrame) -> dict:
    """
    Sanity checks to run during the test.
    If any fail, the test may be invalid.
    """
    checks = {}
    
    # Check 1: Sample Ratio Mismatch (SRM)
    # Groups should be approximately equal in size
    control_n = len(df[df['group'] == 'control'])
    treatment_n = len(df[df['group'] == 'treatment'])
    total_n = control_n + treatment_n
    
    # Chi-squared test for 50/50 split
    from scipy.stats import chisquare
    chi2_stat, srm_pvalue = chisquare([control_n, treatment_n])
    
    checks['sample_ratio_mismatch'] = {
        'control_n': control_n,
        'treatment_n': treatment_n,
        'ratio': round(control_n / treatment_n, 3),
        'p_value': round(srm_pvalue, 4),
        'passed': srm_pvalue > 0.001  # Very strict threshold for SRM
    }
    
    # Check 2: Pre-experiment balance
    # (In real tests, check that groups are similar on pre-test metrics)
    
    # Check 3: Novelty/primacy effects
    # Check if the effect changes over time
    df_sorted = df.sort_values('timestamp')
    first_half = df_sorted.iloc[:len(df_sorted)//2]
    second_half = df_sorted.iloc[len(df_sorted)//2:]
    
    first_half_diff = (
        first_half[first_half['group']=='treatment']['converted'].mean() -
        first_half[first_half['group']=='control']['converted'].mean()
    )
    second_half_diff = (
        second_half[second_half['group']=='treatment']['converted'].mean() -
        second_half[second_half['group']=='control']['converted'].mean()
    )
    
    checks['novelty_check'] = {
        'first_half_effect': round(first_half_diff, 4),
        'second_half_effect': round(second_half_diff, 4),
        'consistent': abs(first_half_diff - second_half_diff) < 0.02
    }
    
    return checks

# Run health checks
health = run_health_checks(data)
for check_name, result in health.items():
    status = "✓ PASS" if result.get('passed', result.get('consistent')) else "✗ FAIL"
    print(f"{status}: {check_name}")
    for k, v in result.items():
        print(f"    {k}: {v}")
```

---

## Analyzing Results

### The Complete Analysis Pipeline

```python
import numpy as np
import pandas as pd
from scipy import stats
from statsmodels.stats.proportion import proportions_ztest, proportion_confint

def analyze_ab_test(
    df: pd.DataFrame,
    group_col: str = 'group',
    metric_col: str = 'converted',
    control_name: str = 'control',
    treatment_name: str = 'treatment',
    alpha: float = 0.05
) -> dict:
    """
    Complete A/B test analysis for binary metrics.
    
    Returns significance, effect size, confidence interval, and practical recommendations.
    """
    
    # Separate groups
    control = df[df[group_col] == control_name][metric_col]
    treatment = df[df[group_col] == treatment_name][metric_col]
    
    # Basic statistics
    n_control = len(control)
    n_treatment = len(treatment)
    conversions_control = control.sum()
    conversions_treatment = treatment.sum()
    rate_control = control.mean()
    rate_treatment = treatment.mean()
    
    # Absolute and relative difference
    absolute_diff = rate_treatment - rate_control
    relative_diff = absolute_diff / rate_control if rate_control > 0 else 0
    
    # ── Statistical Test: Two-proportion z-test ──
    # H0: p_treatment = p_control
    # H1: p_treatment ≠ p_control (two-tailed)
    
    count = np.array([conversions_treatment, conversions_control])
    nobs = np.array([n_treatment, n_control])
    
    z_stat, p_value = proportions_ztest(count, nobs, alternative='two-sided')
    
    # ── Confidence Interval for the difference ──
    # Standard error of the difference
    se_diff = np.sqrt(
        rate_control * (1 - rate_control) / n_control +
        rate_treatment * (1 - rate_treatment) / n_treatment
    )
    
    z_critical = stats.norm.ppf(1 - alpha / 2)
    ci_lower = absolute_diff - z_critical * se_diff
    ci_upper = absolute_diff + z_critical * se_diff
    
    # ── Decision ──
    is_significant = p_value < alpha
    
    results = {
        'control_rate': round(rate_control, 4),
        'treatment_rate': round(rate_treatment, 4),
        'absolute_difference': round(absolute_diff, 4),
        'relative_difference': round(relative_diff, 4),
        'n_control': n_control,
        'n_treatment': n_treatment,
        'z_statistic': round(z_stat, 4),
        'p_value': round(p_value, 6),
        'confidence_interval': (round(ci_lower, 4), round(ci_upper, 4)),
        'is_significant': is_significant,
        'alpha': alpha,
    }
    
    return results

# Run the analysis
results = analyze_ab_test(data)

print("\n" + "="*50)
print("A/B TEST RESULTS")
print("="*50)
print(f"\n  Control conversion rate:   {results['control_rate']:.2%}")
print(f"  Treatment conversion rate: {results['treatment_rate']:.2%}")
print(f"\n  Absolute difference:       {results['absolute_difference']:.2%}")
print(f"  Relative improvement:      {results['relative_difference']:.1%}")
print(f"\n  Z-statistic:              {results['z_statistic']}")
print(f"  P-value:                  {results['p_value']:.6f}")
print(f"  95% CI for difference:    [{results['confidence_interval'][0]:.2%}, {results['confidence_interval'][1]:.2%}]")
print(f"\n  Significant at α=0.05?    {'YES ✓' if results['is_significant'] else 'NO ✗'}")

if results['is_significant']:
    print(f"\n  → RECOMMENDATION: Ship the treatment. ")
    print(f"    Expected {results['relative_difference']:.1%} relative improvement.")
else:
    print(f"\n  → RECOMMENDATION: No significant difference detected.")
    print(f"    Consider running longer or testing a bigger change.")
```

### Analyzing Continuous Metrics (Revenue, Time on Page)

```python
from scipy import stats

def analyze_continuous_metric(
    control_values: np.ndarray,
    treatment_values: np.ndarray,
    alpha: float = 0.05
) -> dict:
    """
    Analyze A/B test for continuous metrics using Welch's t-test.
    Use this for revenue, session duration, pages viewed, etc.
    """
    
    # Welch's t-test (doesn't assume equal variances)
    t_stat, p_value = stats.ttest_ind(treatment_values, control_values, equal_var=False)
    
    # Effect size: Cohen's d
    pooled_std = np.sqrt(
        (np.std(control_values, ddof=1)**2 + np.std(treatment_values, ddof=1)**2) / 2
    )
    cohens_d = (treatment_values.mean() - control_values.mean()) / pooled_std
    
    # Confidence interval for the mean difference
    diff = treatment_values.mean() - control_values.mean()
    se = np.sqrt(
        np.var(control_values, ddof=1) / len(control_values) +
        np.var(treatment_values, ddof=1) / len(treatment_values)
    )
    ci_lower = diff - 1.96 * se
    ci_upper = diff + 1.96 * se
    
    return {
        'control_mean': round(control_values.mean(), 4),
        'treatment_mean': round(treatment_values.mean(), 4),
        'difference': round(diff, 4),
        'relative_change': round(diff / control_values.mean(), 4),
        't_statistic': round(t_stat, 4),
        'p_value': round(p_value, 6),
        'cohens_d': round(cohens_d, 4),
        'confidence_interval': (round(ci_lower, 4), round(ci_upper, 4)),
        'is_significant': p_value < alpha
    }

# Example: Revenue per user
np.random.seed(42)
control_revenue = np.random.lognormal(mean=3.0, sigma=1.0, size=5000)
treatment_revenue = np.random.lognormal(mean=3.05, sigma=1.0, size=5000)

revenue_results = analyze_continuous_metric(control_revenue, treatment_revenue)
print(f"Revenue Results:")
print(f"  Control mean: ${revenue_results['control_mean']:.2f}")
print(f"  Treatment mean: ${revenue_results['treatment_mean']:.2f}")
print(f"  Relative change: {revenue_results['relative_change']:.1%}")
print(f"  P-value: {revenue_results['p_value']}")
print(f"  Cohen's d: {revenue_results['cohens_d']} ({'small' if abs(revenue_results['cohens_d']) < 0.2 else 'medium' if abs(revenue_results['cohens_d']) < 0.5 else 'large'})")
```

### Bayesian A/B Testing

```python
import numpy as np
from scipy import stats

def bayesian_ab_test(
    conversions_a: int,
    total_a: int,
    conversions_b: int,
    total_b: int,
    n_simulations: int = 100000,
    prior_alpha: float = 1,  # Beta prior parameter (1,1 = uniform/uninformative)
    prior_beta: float = 1
) -> dict:
    """
    Bayesian A/B test using Beta-Binomial model.
    
    Instead of p-values, gives you:
    - Probability that B is better than A
    - Expected improvement
    - Risk of choosing B
    """
    
    # Posterior distributions (Beta distribution)
    # posterior = Beta(prior_alpha + successes, prior_beta + failures)
    posterior_a = stats.beta(
        prior_alpha + conversions_a,
        prior_beta + (total_a - conversions_a)
    )
    posterior_b = stats.beta(
        prior_alpha + conversions_b,
        prior_beta + (total_b - conversions_b)
    )
    
    # Monte Carlo simulation
    samples_a = posterior_a.rvs(n_simulations)
    samples_b = posterior_b.rvs(n_simulations)
    
    # Probability that B > A
    prob_b_better = (samples_b > samples_a).mean()
    
    # Expected lift (relative improvement)
    lift_samples = (samples_b - samples_a) / samples_a
    expected_lift = lift_samples.mean()
    
    # Risk: Expected loss if you choose B but A is actually better
    loss_if_choose_b = np.maximum(samples_a - samples_b, 0).mean()
    loss_if_choose_a = np.maximum(samples_b - samples_a, 0).mean()
    
    # Credible interval for the difference
    diff_samples = samples_b - samples_a
    ci_95 = np.percentile(diff_samples, [2.5, 97.5])
    
    return {
        'prob_b_better': round(prob_b_better, 4),
        'prob_a_better': round(1 - prob_b_better, 4),
        'expected_lift': round(expected_lift, 4),
        'credible_interval_95': (round(ci_95[0], 4), round(ci_95[1], 4)),
        'risk_choosing_b': round(loss_if_choose_b, 6),
        'risk_choosing_a': round(loss_if_choose_a, 6),
        'posterior_mean_a': round(posterior_a.mean(), 4),
        'posterior_mean_b': round(posterior_b.mean(), 4),
    }

# Example
results = bayesian_ab_test(
    conversions_a=500, total_a=5000,   # Control: 10% conversion
    conversions_b=560, total_b=5000    # Treatment: 11.2% conversion
)

print("Bayesian A/B Test Results:")
print(f"  P(Treatment > Control): {results['prob_b_better']:.1%}")
print(f"  Expected lift: {results['expected_lift']:.1%}")
print(f"  95% Credible Interval: [{results['credible_interval_95'][0]:.2%}, {results['credible_interval_95'][1]:.2%}]")
print(f"  Risk of choosing Treatment: {results['risk_choosing_b']:.4%}")
print(f"  Risk of choosing Control: {results['risk_choosing_a']:.4%}")
```

### Frequentist vs Bayesian: When to Use Which

| Aspect | Frequentist | Bayesian |
|--------|-------------|----------|
| Output | p-value, CI | Probability of winning, credible interval |
| Interpretability | Confusing for stakeholders | Intuitive ("95% chance B is better") |
| Early stopping | Not allowed (inflates false positives) | Built-in (update beliefs as data arrives) |
| Prior knowledge | Not used | Can incorporate prior beliefs |
| Fixed sample size | Required upfront | Can check continuously |
| Industry standard | Still dominant | Growing rapidly |
| Best for | Regulated industries, academic papers | Product decisions, speed |

---

## Advanced Topics

### Multiple Testing Correction

When you test multiple variants or metrics, the chance of at least one false positive increases dramatically.

```python
# Problem: Testing 20 metrics with α=0.05
# Expected false positives = 20 × 0.05 = 1 (you'll always find "something")

# Solution: Bonferroni Correction (simple but conservative)
def bonferroni_correction(p_values: list, alpha: float = 0.05) -> list:
    """Divide α by number of tests."""
    n_tests = len(p_values)
    corrected_alpha = alpha / n_tests
    return [{'p_value': p, 'significant': p < corrected_alpha} for p in p_values]

# Solution: Benjamini-Hochberg (FDR control — less conservative)
from statsmodels.stats.multitest import multipletests

p_values = [0.001, 0.008, 0.039, 0.041, 0.052, 0.10, 0.45, 0.88]

# BH method: controls False Discovery Rate
reject, corrected_pvals, _, _ = multipletests(p_values, method='fdr_bh', alpha=0.05)

print("Benjamini-Hochberg Correction:")
for i, (p, adj_p, sig) in enumerate(zip(p_values, corrected_pvals, reject)):
    print(f"  Test {i+1}: p={p:.4f}, adjusted_p={adj_p:.4f}, significant={sig}")
```

### Multi-Armed Bandit (Adaptive Allocation)

```python
import numpy as np

class ThompsonSampling:
    """
    Multi-armed bandit using Thompson Sampling.
    
    Unlike A/B testing (fixed 50/50 split), bandits dynamically
    allocate MORE traffic to the winning variant, reducing regret.
    
    Use when:
    - You want to minimize losses during the test
    - You're optimizing continuously (not a one-time decision)
    - You have many variants to test
    """
    
    def __init__(self, n_variants: int):
        # Beta distribution parameters for each variant
        # Start with uniform prior Beta(1,1)
        self.alphas = np.ones(n_variants)  # Successes + 1
        self.betas = np.ones(n_variants)   # Failures + 1
        self.n_variants = n_variants
    
    def select_variant(self) -> int:
        """Select which variant to show next (Thompson Sampling)."""
        # Sample from each variant's posterior
        samples = [
            np.random.beta(self.alphas[i], self.betas[i])
            for i in range(self.n_variants)
        ]
        # Choose the variant with the highest sample
        return int(np.argmax(samples))
    
    def update(self, variant: int, reward: int):
        """Update beliefs after observing a result."""
        if reward == 1:
            self.alphas[variant] += 1
        else:
            self.betas[variant] += 1
    
    def get_probabilities(self) -> list:
        """Estimated conversion rate for each variant."""
        return [
            self.alphas[i] / (self.alphas[i] + self.betas[i])
            for i in range(self.n_variants)
        ]

# Simulation: 3 variants with true rates [0.10, 0.12, 0.11]
true_rates = [0.10, 0.12, 0.11]
bandit = ThompsonSampling(n_variants=3)

selections = np.zeros(3)
for _ in range(10000):
    # Bandit picks which variant to show
    chosen = bandit.select_variant()
    selections[chosen] += 1
    
    # Simulate user response
    reward = np.random.binomial(1, true_rates[chosen])
    
    # Update bandit's beliefs
    bandit.update(chosen, reward)

print("Thompson Sampling Results (10K users):")
print(f"  Traffic allocation: {selections / selections.sum()}")
print(f"  Estimated rates: {bandit.get_probabilities()}")
print(f"  True rates: {true_rates}")
# The best variant (0.12) gets most traffic automatically!
```

### CUPED — Variance Reduction

```python
def cuped_analysis(
    df: pd.DataFrame,
    metric_col: str,          # Post-treatment metric
    covariate_col: str,       # Pre-treatment metric (same user, before test)
    group_col: str = 'group'
) -> dict:
    """
    CUPED (Controlled-experiment Using Pre-Experiment Data).
    
    Reduces variance by adjusting for pre-experiment behavior,
    allowing you to detect smaller effects with the same sample size.
    
    Intuition: If a user spent $100/week before the test, their
    spending during the test is partly explained by their baseline
    behavior. CUPED removes this "predictable" variation.
    """
    
    # Calculate theta (optimal coefficient)
    cov = df[metric_col].cov(df[covariate_col])
    var_covariate = df[covariate_col].var()
    theta = cov / var_covariate
    
    # Adjusted metric: remove predictable variation
    covariate_mean = df[covariate_col].mean()
    df['adjusted_metric'] = df[metric_col] - theta * (df[covariate_col] - covariate_mean)
    
    # Compare groups on adjusted metric
    control = df[df[group_col] == 'control']['adjusted_metric']
    treatment = df[df[group_col] == 'treatment']['adjusted_metric']
    
    # Standard analysis on adjusted metric
    from scipy.stats import ttest_ind
    t_stat, p_value = ttest_ind(treatment, control, equal_var=False)
    
    # Variance reduction achieved
    original_var = df[metric_col].var()
    adjusted_var = df['adjusted_metric'].var()
    variance_reduction = 1 - (adjusted_var / original_var)
    
    return {
        'theta': round(theta, 4),
        'original_variance': round(original_var, 4),
        'adjusted_variance': round(adjusted_var, 4),
        'variance_reduction': round(variance_reduction, 4),
        'adjusted_control_mean': round(control.mean(), 4),
        'adjusted_treatment_mean': round(treatment.mean(), 4),
        'p_value': round(p_value, 6),
        'effective_sample_multiplier': round(1 / (1 - variance_reduction), 2)
    }

# Example: 40% variance reduction = equivalent to 1.67x more sample
# This means you can detect effects faster or detect smaller effects!
```

---

## Common Mistakes

### 1. Peeking at Results Too Early

```python
# ❌ BAD: Checking p-value every day and stopping when significant
# This inflates your false positive rate from 5% to potentially 30%+!

# Day 1: p=0.23 (not significant, continue)
# Day 2: p=0.08 (not significant, continue)
# Day 3: p=0.04 (significant! ship it!)  ← WRONG!

# ✓ GOOD: Pre-determine sample size and ONLY analyze at the end
# OR use sequential testing methods designed for continuous monitoring
```

### 2. Not Accounting for Multiple Comparisons

```python
# ❌ BAD: Testing 10 metrics and reporting the one that's significant
# With 10 metrics at α=0.05, there's a 40% chance of at least one false positive!

# ✓ GOOD: Declare ONE primary metric upfront. Use Bonferroni or BH for others.
```

### 3. Sample Ratio Mismatch (SRM)

```
If your groups should be 50/50 but you see 52/48 or worse:
→ Something is BROKEN in your randomization
→ Results are NOT trustworthy
→ Stop the test, find the bug

Common causes:
• Bot traffic that only hits one variant
• Redirect-based tests where one variant loads slower (users leave)
• Caching issues serving wrong variant
```

### 4. Survivorship Bias

```python
# ❌ BAD: Only analyzing users who completed the funnel
# "Among people who made it to checkout, the new page had higher conversion"
# But maybe the new page PREVENTED more people from reaching checkout!

# ✓ GOOD: Analyze ALL users who were assigned to the test
# Use Intent-to-Treat (ITT) analysis
```

### 5. Running Too Short / Not Full Weeks

```
# ❌ BAD: 3-day test (Mon-Wed)
# Misses weekend behavior, which might be very different

# ✓ GOOD: Always run in multiples of 7 days
# Captures day-of-week effects, paydays, etc.
```

### 6. Testing During Anomalous Periods

```
# ❌ BAD: Running a test during Black Friday, product launch, site outage
# External events dominate the signal — you can't attribute changes to your variant

# ✓ GOOD: Run during "normal" periods, or account for the event in analysis
```

---

## Interview Questions

### Conceptual

**Q1: You run a test for 2 weeks. The p-value is 0.06. What do you do?**

> **Answer:** DO NOT conclude "almost significant" and ship it. Options:
> 1. If pre-planned sample size isn't reached, continue running (but don't peek again)
> 2. If sample size IS reached, the test is inconclusive — no significant effect detected
> 3. Consider if the MDE was set too ambitiously (the effect might be real but smaller than you powered for)
> 4. Report the confidence interval — stakeholders care about the range of possible effects
> 5. Consider running a follow-up test with a larger sample or bigger design change

**Q2: Your test shows a significant improvement in click-through rate but no change in revenue. What happened?**

> **Answer:** Several possible explanations:
> 1. **Metric mismatch:** More clicks ≠ more purchases. People might click more but not buy.
> 2. **Cannibalization:** Increased clicks on one feature took attention from a revenue-generating feature
> 3. **Segment effect:** Improvement only among low-value users who don't buy
> 4. **Short-term vs long-term:** Revenue impact may take longer to materialize
> 5. **Action:** Look at the full funnel, segment the data, extend the test duration

**Q3: How would you design an A/B test when you can't randomize at the user level?**

> **Answer:** Options for non-standard randomization:
> 1. **Geo-based testing:** Randomize by city/region, compare markets
> 2. **Time-based (switchback):** Alternate between A and B over time periods
> 3. **Quasi-experimental:** Use Difference-in-Differences, Regression Discontinuity, or Synthetic Control
> 4. **Cluster randomization:** Randomize at a group level (e.g., by store, by school)
> Important: These all require larger samples and more careful analysis

**Q4: What is the novelty effect and how do you handle it?**

> **Answer:** Users initially engage more with something just because it's NEW, not because it's better. This inflates early results. Solutions:
> 1. Exclude the first few days of data
> 2. Run the test longer (3-4 weeks minimum)
> 3. Analyze only returning users (who've habituated to the change)
> 4. Check if the effect decays over time (plot daily effect size)

**Q5: Explain when you'd use a one-tailed vs two-tailed test.**

> **Answer:** 
> - **Two-tailed (default):** Use when the change could EITHER help or hurt. "Does this change affect conversion?" — you'd want to know if it's harmful too.
> - **One-tailed:** Use ONLY when you genuinely don't care about one direction AND have a strong prior. Example: "We added a known-good feature — we just want to confirm it helps."
> - In practice, almost always use two-tailed. Regulators and reviewers prefer it, and one-tailed can be seen as "p-hacking lite."

### Practical

**Q6: You're a data scientist at an e-commerce company. Design a complete A/B test for changing the "Add to Cart" button from green to orange.**

> **Answer:**
> 1. **Hypothesis:** "Changing Add to Cart from green to orange will increase add-to-cart rate because orange creates urgency and higher contrast."
> 2. **Primary metric:** Add-to-cart rate (users who click / users who view product page)
> 3. **Secondary:** Purchase rate, revenue per visitor
> 4. **Guardrails:** Page load time, accessibility score, return rate
> 5. **Sample size:** Baseline ATC = 8%, MDE = 3% relative (8% → 8.24%), need ~150K per group
> 6. **Duration:** With 50K daily product page views, need ~6 days minimum → run 14 days (2 full weeks)
> 7. **Randomization:** User-level (cookie/user_id), ensure returning users always see same variant
> 8. **Segments to check:** Mobile vs desktop, new vs returning, by product category
> 9. **Risks:** Novelty effect, brand consistency concerns

---

## Quick Reference

### A/B Test Checklist

| Phase | Action | Done? |
|-------|--------|-------|
| Design | Define hypothesis with expected effect | ☐ |
| Design | Choose primary metric (ONE) | ☐ |
| Design | Set guardrail metrics | ☐ |
| Design | Calculate sample size | ☐ |
| Design | Estimate duration | ☐ |
| Design | Document test plan | ☐ |
| Implement | Build variants | ☐ |
| Implement | Set up tracking/logging | ☐ |
| Implement | QA both variants | ☐ |
| Run | Launch to small % first (burn-in) | ☐ |
| Run | Check for SRM | ☐ |
| Run | Monitor guardrail metrics | ☐ |
| Run | Run for full planned duration | ☐ |
| Analyze | Check sample ratio | ☐ |
| Analyze | Calculate statistical significance | ☐ |
| Analyze | Report confidence interval | ☐ |
| Analyze | Check segments | ☐ |
| Analyze | Check for novelty/primacy effects | ☐ |
| Decide | Ship, iterate, or kill | ☐ |
| Decide | Document learnings | ☐ |

### Statistical Test Selection

| Metric Type | Test | Python Function |
|-------------|------|-----------------|
| Binary (conversion rate) | Two-proportion z-test | `statsmodels.proportions_ztest()` |
| Continuous (revenue) | Welch's t-test | `scipy.stats.ttest_ind(equal_var=False)` |
| Count data | Chi-squared test | `scipy.stats.chi2_contingency()` |
| Non-normal continuous | Mann-Whitney U | `scipy.stats.mannwhitneyu()` |
| Multiple variants | ANOVA + post-hoc | `scipy.stats.f_oneway()` |

### Key Formulas

| Formula | When |
|---------|------|
| $n = \frac{(z_{\alpha/2} + z_\beta)^2 \cdot 2p(1-p)}{(\text{MDE})^2}$ | Sample size (proportions) |
| $z = \frac{\hat{p}_B - \hat{p}_A}{\sqrt{\hat{p}(1-\hat{p})(\frac{1}{n_A}+\frac{1}{n_B})}}$ | Z-statistic (proportions) |
| $\text{CI} = (\hat{p}_B - \hat{p}_A) \pm z_{\alpha/2} \cdot SE$ | Confidence interval |
| $d = \frac{\bar{x}_B - \bar{x}_A}{s_{pooled}}$ | Cohen's d (effect size) |

### Common Parameters

| Parameter | Typical Value | Meaning |
|-----------|---------------|---------|
| α (alpha) | 0.05 | 5% false positive rate |
| β (beta) | 0.20 | 20% false negative rate |
| Power (1-β) | 0.80 | 80% chance of detecting true effect |
| MDE | 2-10% relative | Smallest meaningful improvement |
| Test duration | 2-4 weeks | Capture weekly patterns |
| Confidence level | 95% | = 1 - α |

---

*Next: [Chapter 10 - Data Storytelling and Communication](10-Data-Storytelling-and-Communication.md)*
