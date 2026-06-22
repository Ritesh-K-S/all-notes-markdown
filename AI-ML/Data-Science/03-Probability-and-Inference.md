# Chapter 03: Probability and Inference

## Table of Contents
- [3.1 What is Probability](#31-what-is-probability)
- [3.2 Probability Rules and Concepts](#32-probability-rules-and-concepts)
- [3.3 Conditional Probability](#33-conditional-probability)
- [3.4 Bayes' Theorem](#34-bayes-theorem)
- [3.5 Probability Distributions in Depth](#35-probability-distributions-in-depth)
- [3.6 Hypothesis Testing](#36-hypothesis-testing)
- [3.7 p-Values and Statistical Significance](#37-p-values-and-statistical-significance)
- [3.8 Confidence Intervals](#38-confidence-intervals)
- [3.9 Types of Errors (Type I and Type II)](#39-types-of-errors-type-i-and-type-ii)
- [3.10 Chi-Square, t-Tests, and ANOVA](#310-chi-square-t-tests-and-anova)
- [3.11 Common Mistakes](#311-common-mistakes)
- [3.12 Interview Questions](#312-interview-questions)
- [3.13 Quick Reference](#313-quick-reference)

---

## 3.1 What is Probability

### What It Is
Probability is the mathematics of uncertainty. It quantifies how likely something is to happen, on a scale from 0 (impossible) to 1 (certain).

**Analogy:** Imagine you're playing a video game with fog of war. Probability is your best estimate of what's behind the fog — it doesn't tell you for certain, but it tells you what to expect and how surprised you should be by different outcomes.

$$P(A) = \frac{\text{Number of favorable outcomes}}{\text{Total number of possible outcomes}}$$

### Why It Matters for Data Science
- **Machine Learning models output probabilities** (e.g., "90% chance this email is spam")
- **A/B testing** requires probability to determine if results are real
- **Risk assessment** in finance, healthcare, insurance
- **Recommendation systems** predict probability of user engagement
- **Bayesian methods** update beliefs as new data arrives

### Three Interpretations of Probability

| Interpretation | Definition | Example |
|---------------|-----------|---------|
| **Classical** (Frequentist) | Long-run frequency of events | P(heads) = 0.5 after infinite flips |
| **Subjective** (Bayesian) | Degree of belief | "I'm 70% sure it'll rain tomorrow" |
| **Axiomatic** | Mathematical framework (Kolmogorov) | Formal probability theory |

### Basic Probability Scale

```
0.0          0.25          0.5          0.75          1.0
 │─────────────│─────────────│─────────────│─────────────│
Impossible   Unlikely      Even chance   Likely       Certain

Examples:
• P(sun explodes tomorrow) ≈ 0
• P(rolling 6 on a die) = 1/6 ≈ 0.167
• P(coin heads) = 0.5
• P(surviving to next birthday, young adult) ≈ 0.999
• P(1+1=2) = 1
```

---

## 3.2 Probability Rules and Concepts

### Fundamental Rules

#### 1. Complement Rule
$$P(A') = 1 - P(A)$$

"The probability of something NOT happening = 1 - probability of it happening"

**Pro Tip:** Often easier to calculate P(not A) than P(A).
- P(at least one head in 10 flips) = 1 - P(all tails) = 1 - (0.5)^{10} = 0.999

#### 2. Addition Rule (OR)
$$P(A \cup B) = P(A) + P(B) - P(A \cap B)$$

For **mutually exclusive** events (can't happen together):
$$P(A \cup B) = P(A) + P(B)$$

```
    ┌─────────────────────────────────────────┐
    │              Sample Space (S)            │
    │                                         │
    │     ┌──────────┐    ┌──────────┐        │
    │     │    A     │    │    B     │        │
    │     │          ├────┤          │        │
    │     │     P(A) │A∩B │  P(B)   │        │
    │     │          ├────┤          │        │
    │     └──────────┘    └──────────┘        │
    │                                         │
    │  P(A∪B) = P(A) + P(B) - P(A∩B)         │
    │  (subtract overlap to avoid counting    │
    │   it twice!)                            │
    └─────────────────────────────────────────┘
```

#### 3. Multiplication Rule (AND)
$$P(A \cap B) = P(A) \times P(B|A)$$

For **independent** events:
$$P(A \cap B) = P(A) \times P(B)$$

#### 4. Independence
Two events are independent if knowing one tells you nothing about the other:
$$P(A|B) = P(A) \quad \text{and} \quad P(A \cap B) = P(A) \cdot P(B)$$

**Examples:**
- Independent: Flipping a coin and rolling a die
- NOT independent: Drawing cards without replacement, rain and carrying umbrella

### Code: Probability Fundamentals

```python
import numpy as np
from itertools import product

# ============================================
# Simulating Probability with Code
# ============================================
np.random.seed(42)
n_simulations = 100000

# --- Basic Probability: Rolling a Die ---
die_rolls = np.random.randint(1, 7, n_simulations)
print("=== Die Roll Probabilities ===")
for value in range(1, 7):
    prob = np.sum(die_rolls == value) / n_simulations
    print(f"  P(roll {value}) = {prob:.4f} (theoretical: {1/6:.4f})")

# --- Addition Rule ---
# P(roll even OR roll > 4)
# Even: {2, 4, 6}, > 4: {5, 6}, Overlap: {6}
even_or_gt4 = np.sum((die_rolls % 2 == 0) | (die_rolls > 4)) / n_simulations
theoretical = 4/6  # {2, 4, 5, 6}
print(f"\nP(even OR > 4) = {even_or_gt4:.4f} (theoretical: {theoretical:.4f})")

# --- Multiplication Rule (Independent Events) ---
# P(two dice both show 6)
die1 = np.random.randint(1, 7, n_simulations)
die2 = np.random.randint(1, 7, n_simulations)
both_six = np.sum((die1 == 6) & (die2 == 6)) / n_simulations
print(f"P(both dice = 6) = {both_six:.4f} (theoretical: {1/36:.4f})")

# --- Complement Rule ---
# P(at least one 6 in 4 rolls) = 1 - P(no 6 in 4 rolls)
rolls_4 = np.random.randint(1, 7, (n_simulations, 4))
at_least_one_6 = np.sum(np.any(rolls_4 == 6, axis=1)) / n_simulations
theoretical_complement = 1 - (5/6)**4
print(f"P(at least one 6 in 4 rolls) = {at_least_one_6:.4f} (theoretical: {theoretical_complement:.4f})")

# --- Birthday Problem ---
# P(at least 2 people share birthday in group of n)
def birthday_probability(group_size, n_sims=100000):
    """Simulate the birthday problem."""
    count = 0
    for _ in range(n_sims):
        birthdays = np.random.randint(0, 365, group_size)
        if len(birthdays) != len(set(birthdays)):  # Duplicate found
            count += 1
    return count / n_sims

print("\n=== Birthday Problem ===")
for n in [10, 23, 30, 50, 70]:
    prob = birthday_probability(n, 50000)
    print(f"  {n} people: P(shared birthday) = {prob:.3f}")
print("  (Note: Only 23 people needed for >50% chance!)")
```

---

## 3.3 Conditional Probability

### What It Is
The probability of event A happening **given that** event B has already happened.

$$P(A|B) = \frac{P(A \cap B)}{P(B)}$$

**Analogy:** What's the probability of getting an A in a class? Maybe 10%. But what's the probability of getting an A *given that you studied 20+ hours per week*? Much higher — maybe 60%. New information changes probabilities.

### Visual Intuition

```
Before knowing B:                After knowing B happened:
┌─────────────────────┐         We ONLY look at B:
│       S             │         ┌──────────────┐
│  ┌────┐   ┌──────┐ │         │ B            │
│  │ A  │   │  B   │ │    →    │  ┌────┐      │
│  │    ├───┤      │ │         │  │A∩B │      │
│  │    │A∩B│      │ │         │  └────┘      │
│  └────┘   └──────┘ │         └──────────────┘
│                     │         P(A|B) = P(A∩B) / P(B)
└─────────────────────┘         = shaded / total B area
```

### Real-World Examples

```python
import numpy as np

# ============================================
# Medical Test Example (Classic Interview Question)
# ============================================
# Disease prevalence: 1% of population has disease
# Test sensitivity: 99% (positive if you HAVE the disease)
# Test specificity: 95% (negative if you DON'T have the disease)

n_population = 1000000
has_disease = int(0.01 * n_population)      # 10,000 sick
no_disease = n_population - has_disease      # 990,000 healthy

# True positives: sick people who test positive
true_positives = int(0.99 * has_disease)     # 9,900
# False negatives: sick people who test negative
false_negatives = has_disease - true_positives  # 100

# False positives: healthy people who test positive
false_positives = int(0.05 * no_disease)     # 49,500
# True negatives: healthy people who test negative
true_negatives = no_disease - false_positives  # 940,500

print("=== Medical Test: Confusion Matrix ===")
print(f"{'':15} {'Has Disease':>15} {'No Disease':>15} {'Total':>10}")
print(f"{'Test Positive':<15} {true_positives:>15,} {false_positives:>15,} {true_positives+false_positives:>10,}")
print(f"{'Test Negative':<15} {false_negatives:>15,} {true_negatives:>15,} {false_negatives+true_negatives:>10,}")
print(f"{'Total':<15} {has_disease:>15,} {no_disease:>15,} {n_population:>10,}")

# The key question: If you test positive, what's the probability you're actually sick?
# P(Disease | Positive) = True Positives / All Positives
p_disease_given_positive = true_positives / (true_positives + false_positives)
print(f"\n🎯 P(Disease | Test Positive) = {p_disease_given_positive:.4f} = {p_disease_given_positive*100:.1f}%")
print(f"   Even with a 99% sensitive test, only ~{p_disease_given_positive*100:.0f}% of positive results are true!")
print(f"   This is because the disease is RARE (1%), so false positives dominate.")

# ============================================
# Conditional Probability: Cards
# ============================================
print("\n=== Card Probabilities ===")
# Standard deck: 52 cards, 4 suits, 13 ranks
# P(King | Face card)
# Face cards: J, Q, K = 12 total, Kings = 4
p_king_given_face = 4 / 12
print(f"P(King | Face card) = {p_king_given_face:.4f}")

# P(Heart | Red card)
# Red cards: 26, Hearts: 13
p_heart_given_red = 13 / 26
print(f"P(Heart | Red card) = {p_heart_given_red:.4f}")

# Dependent events: Drawing without replacement
# P(2nd card is King | 1st card was King)
p_king2_given_king1 = 3 / 51  # Only 3 kings left, 51 cards left
print(f"P(2nd King | 1st was King) = {p_king2_given_king1:.4f}")
print(f"P(2nd King | 1st was NOT King) = {4/51:.4f}")
```

---

## 3.4 Bayes' Theorem

### What It Is
Bayes' Theorem lets you **update your beliefs** when you get new evidence. It reverses conditional probability.

$$P(A|B) = \frac{P(B|A) \cdot P(A)}{P(B)}$$

Expanded form using Law of Total Probability:
$$P(A|B) = \frac{P(B|A) \cdot P(A)}{P(B|A) \cdot P(A) + P(B|A') \cdot P(A')}$$

### The Components

| Term | Name | Meaning |
|------|------|---------|
| $P(A\|B)$ | **Posterior** | Updated belief after seeing evidence |
| $P(A)$ | **Prior** | Initial belief before evidence |
| $P(B\|A)$ | **Likelihood** | How likely is the evidence if A is true? |
| $P(B)$ | **Evidence** (Marginal) | Overall probability of seeing this evidence |

### Intuition

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  PRIOR BELIEF ──── + NEW EVIDENCE ────→ UPDATED BELIEF       │
│  P(A)                  P(B|A)              P(A|B)            │
│                                                              │
│  "Before I look"    "What I observed"    "After I look"      │
│                                                              │
│  Example:                                                    │
│  "1% have disease"  "Test is positive"   "~17% chance       │
│                                            of disease"       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Analogy:** You hear a noise at night. Your **prior** belief: probably the cat (90%) or a burglar (10%). Then you see the front door is open (**evidence**). A burglar would definitely leave the door open (high likelihood). The cat wouldn't. Now your **posterior** belief shifts: maybe 60% burglar. Bayes' Theorem makes this intuition mathematically precise.

### Code: Bayes' Theorem Applications

```python
import numpy as np

# ============================================
# 1. Classic: Disease Testing (Bayes' way)
# ============================================
print("=== Bayes' Theorem: Medical Test ===")

# Prior: P(Disease) = 0.01
p_disease = 0.01
p_no_disease = 1 - p_disease

# Likelihood: P(Positive | Disease) = 0.99 (sensitivity)
p_pos_given_disease = 0.99

# P(Positive | No Disease) = 0.05 (false positive rate = 1 - specificity)
p_pos_given_no_disease = 0.05

# Evidence: P(Positive) using Law of Total Probability
p_positive = (p_pos_given_disease * p_disease + 
              p_pos_given_no_disease * p_no_disease)

# Posterior: P(Disease | Positive)
p_disease_given_pos = (p_pos_given_disease * p_disease) / p_positive

print(f"Prior P(Disease) = {p_disease}")
print(f"P(Positive | Disease) = {p_pos_given_disease}")
print(f"P(Positive | No Disease) = {p_pos_given_no_disease}")
print(f"P(Positive) = {p_positive:.4f}")
print(f"")
print(f"Posterior P(Disease | Positive) = {p_disease_given_pos:.4f} = {p_disease_given_pos*100:.1f}%")

# What if they test positive TWICE (independent tests)?
# Now prior becomes the posterior from first test
p_disease_after_1st = p_disease_given_pos  # New prior
p_positive_2nd = (p_pos_given_disease * p_disease_after_1st + 
                  p_pos_given_no_disease * (1 - p_disease_after_1st))
p_disease_after_2nd = (p_pos_given_disease * p_disease_after_1st) / p_positive_2nd

print(f"\nAfter 2nd positive test: P(Disease) = {p_disease_after_2nd:.4f} = {p_disease_after_2nd*100:.1f}%")
print("→ Each positive test dramatically increases confidence!")

# ============================================
# 2. Spam Filter (Naive Bayes concept)
# ============================================
print("\n=== Bayes' Theorem: Spam Filter ===")

# Prior: 40% of emails are spam
p_spam = 0.40
p_not_spam = 0.60

# Likelihood: P(word "free" | spam) = 0.70
p_free_given_spam = 0.70
# P(word "free" | not spam) = 0.05
p_free_given_not_spam = 0.05

# P("free" appears)
p_free = p_free_given_spam * p_spam + p_free_given_not_spam * p_not_spam

# Posterior: P(spam | contains "free")
p_spam_given_free = (p_free_given_spam * p_spam) / p_free
print(f"P(Spam | contains 'free') = {p_spam_given_free:.4f} = {p_spam_given_free*100:.1f}%")

# Now also contains "winner"
# P("winner" | spam) = 0.5, P("winner" | not spam) = 0.01
p_winner_given_spam = 0.50
p_winner_given_not_spam = 0.01

# Update using previous posterior as new prior (Naive Bayes assumption: words are independent)
new_prior_spam = p_spam_given_free
p_winner = p_winner_given_spam * new_prior_spam + p_winner_given_not_spam * (1 - new_prior_spam)
p_spam_given_both = (p_winner_given_spam * new_prior_spam) / p_winner

print(f"P(Spam | contains 'free' AND 'winner') = {p_spam_given_both:.4f} = {p_spam_given_both*100:.1f}%")

# ============================================
# 3. Bayesian Update Visualization (text-based)
# ============================================
print("\n=== Sequential Bayesian Updates ===")
print("Scenario: Is a coin fair (p=0.5) or biased (p=0.8)?")
print("We flip it multiple times and update our belief.\n")

# Prior: 50/50 between fair and biased
p_fair = 0.5
p_biased = 0.5

# Simulate flips from a biased coin (p=0.8)
np.random.seed(42)
flips = np.random.binomial(1, 0.8, 20)  # 1=heads, 0=tails

print(f"{'Flip':<6} {'Result':<8} {'P(Fair)':<12} {'P(Biased)':<12} {'Verdict'}")
print("-" * 50)

for i, flip in enumerate(flips):
    result = "Heads" if flip == 1 else "Tails"
    
    # Likelihood of this flip under each hypothesis
    p_flip_given_fair = 0.5  # Fair coin: always 0.5
    p_flip_given_biased = 0.8 if flip == 1 else 0.2
    
    # Update using Bayes
    p_evidence = p_flip_given_fair * p_fair + p_flip_given_biased * p_biased
    p_fair = (p_flip_given_fair * p_fair) / p_evidence
    p_biased = (p_flip_given_biased * p_biased) / p_evidence
    
    verdict = "Fair" if p_fair > 0.5 else "Biased"
    print(f"  {i+1:<4} {result:<8} {p_fair:<12.4f} {p_biased:<12.4f} {verdict}")

print(f"\nFinal belief: {p_biased*100:.1f}% confident coin is biased")
```

### Bayesian vs. Frequentist Thinking

| Aspect | Frequentist | Bayesian |
|--------|------------|----------|
| Probability means | Long-run frequency | Degree of belief |
| Parameters are | Fixed (unknown) constants | Random variables with distributions |
| Uses prior knowledge? | No (only data) | Yes (prior + data) |
| Result | Point estimate + CI | Full posterior distribution |
| Example output | "μ = 5.2, 95% CI: [4.8, 5.6]" | "P(μ > 5) = 0.92" |
| Strength | Objective, well-understood | Incorporates prior knowledge, intuitive |
| Weakness | Can't use prior info, CI misinterpreted | Prior choice is subjective |

---

## 3.5 Probability Distributions in Depth

### Discrete Distributions

#### Bernoulli Distribution
Single trial with two outcomes (success/failure).

$$P(X=x) = p^x (1-p)^{1-x}, \quad x \in \{0, 1\}$$

- Mean: $\mu = p$
- Variance: $\sigma^2 = p(1-p)$

#### Binomial Distribution
Number of successes in n independent Bernoulli trials.

$$P(X=k) = \binom{n}{k} p^k (1-p)^{n-k}$$

- Mean: $\mu = np$
- Variance: $\sigma^2 = np(1-p)$

#### Poisson Distribution
Number of events in a fixed interval (time/space) when events occur at constant rate.

$$P(X=k) = \frac{\lambda^k e^{-\lambda}}{k!}$$

- Mean: $\mu = \lambda$
- Variance: $\sigma^2 = \lambda$ (mean equals variance!)

**Use when:** Counting rare events — website visitors/min, typos/page, accidents/year.

### Continuous Distributions

#### Uniform Distribution
All values equally likely in range [a, b].

$$f(x) = \frac{1}{b-a}, \quad a \leq x \leq b$$

- Mean: $\mu = \frac{a+b}{2}$
- Variance: $\sigma^2 = \frac{(b-a)^2}{12}$

#### Exponential Distribution
Time between events in a Poisson process.

$$f(x) = \lambda e^{-\lambda x}, \quad x \geq 0$$

- Mean: $\mu = \frac{1}{\lambda}$
- Variance: $\sigma^2 = \frac{1}{\lambda^2}$
- **Memoryless property:** $P(X > s+t | X > s) = P(X > t)$

**Use when:** Modeling wait times, time between failures.

### Code: Distribution Applications

```python
import numpy as np
from scipy import stats

# ============================================
# Real-World Distribution Applications
# ============================================

# 1. Binomial: Quality Control
print("=== Binomial: Quality Control ===")
# A factory produces items with 2% defect rate
# In a batch of 100, what's the probability of finding:
n, p = 100, 0.02

print(f"Defect rate: {p*100}%, Batch size: {n}")
print(f"Expected defects: {n*p}")
print(f"P(0 defects) = {stats.binom.pmf(0, n, p):.4f}")
print(f"P(≤2 defects) = {stats.binom.cdf(2, n, p):.4f}")
print(f"P(>5 defects) = {1 - stats.binom.cdf(5, n, p):.4f}")
print(f"P(exactly 2) = {stats.binom.pmf(2, n, p):.4f}")

# 2. Poisson: Customer Service
print("\n=== Poisson: Call Center ===")
# Average 4 calls per minute
lam = 4
print(f"Average calls/min: {lam}")
print(f"P(0 calls in a minute) = {stats.poisson.pmf(0, lam):.4f}")
print(f"P(≤3 calls) = {stats.poisson.cdf(3, lam):.4f}")
print(f"P(>8 calls) = {1 - stats.poisson.cdf(8, lam):.4f}")
print(f"P(exactly 4) = {stats.poisson.pmf(4, lam):.4f}")

# Need to staff for 95th percentile
calls_95 = stats.poisson.ppf(0.95, lam)
print(f"95th percentile: need to handle {calls_95:.0f} calls/min")

# 3. Exponential: Reliability Engineering
print("\n=== Exponential: Server Uptime ===")
# Server fails on average every 500 hours (λ = 1/500)
mean_time_between_failures = 500  # hours
lam_exp = 1 / mean_time_between_failures

print(f"Mean time between failures: {mean_time_between_failures} hours")
print(f"P(failure within 100 hours) = {stats.expon.cdf(100, scale=mean_time_between_failures):.4f}")
print(f"P(survives 1000 hours) = {1 - stats.expon.cdf(1000, scale=mean_time_between_failures):.4f}")
print(f"Median time to failure: {stats.expon.ppf(0.5, scale=mean_time_between_failures):.0f} hours")

# 4. Normal: Process Control
print("\n=== Normal: Manufacturing Tolerance ===")
# Bolts should be 10mm diameter, std = 0.1mm
# Acceptable range: 9.8mm to 10.2mm
mu, sigma = 10, 0.1
lower, upper = 9.8, 10.2

p_acceptable = stats.norm.cdf(upper, mu, sigma) - stats.norm.cdf(lower, mu, sigma)
print(f"Target: {mu}mm ± 0.2mm")
print(f"P(within tolerance) = {p_acceptable:.4f} = {p_acceptable*100:.2f}%")
print(f"Defect rate: {(1-p_acceptable)*100:.2f}%")
print(f"In 1 million bolts: ~{int((1-p_acceptable)*1000000):,} defective")

# 5. Choosing the Right Distribution
print("\n=== Distribution Selection Guide ===")
print("""
┌─────────────────────────────────────────────────────────────┐
│ Question to ask:                        → Distribution      │
├─────────────────────────────────────────────────────────────┤
│ Yes/No outcome (single trial)?          → Bernoulli        │
│ Count of successes in n trials?         → Binomial         │
│ Count of events in fixed interval?      → Poisson          │
│ Time between events?                    → Exponential      │
│ Sum of many small random effects?       → Normal           │
│ Anything equally likely in range?       → Uniform          │
│ Product of many positive factors?       → Log-Normal       │
│ Count until first success?              → Geometric        │
│ Sampling without replacement?           → Hypergeometric   │
└─────────────────────────────────────────────────────────────┘
""")
```

---

## 3.6 Hypothesis Testing

### What It Is
Hypothesis testing is a formal procedure to decide whether observed data provides enough evidence to reject a claim about a population.

**Analogy:** It's like a criminal trial. The defendant (null hypothesis) is "innocent until proven guilty." The prosecution (your data) must provide evidence "beyond reasonable doubt" (statistical significance) to convict (reject null). If the evidence isn't strong enough, you don't convict — but that doesn't prove innocence.

### The Framework

```
Step 1: State Hypotheses
┌──────────────────────────────────┐
│ H₀ (Null): No effect/difference │  ← "Default assumption"
│ H₁ (Alt):  There IS an effect   │  ← "What we want to prove"
└──────────────────────────────────┘

Step 2: Choose significance level (α)
┌──────────────────────────────────┐
│ α = 0.05 (most common)          │  ← "How much evidence we need"
│ α = 0.01 (more stringent)       │
└──────────────────────────────────┘

Step 3: Collect data & compute test statistic

Step 4: Calculate p-value

Step 5: Make decision
┌──────────────────────────────────┐
│ If p-value < α → REJECT H₀      │  ← "Statistically significant"
│ If p-value ≥ α → FAIL TO REJECT │  ← "Not enough evidence"
└──────────────────────────────────┘
```

> **IMPORTANT:** "Fail to reject H₀" ≠ "Accept H₀". Absence of evidence is not evidence of absence.

### Types of Hypothesis Tests

| Test | Use When | Comparing |
|------|----------|-----------|
| **One-sample t-test** | Is the mean different from a known value? | Sample mean vs. hypothesized value |
| **Two-sample t-test** | Do two groups have different means? | Group A mean vs. Group B mean |
| **Paired t-test** | Do paired observations differ? | Before vs. After |
| **Chi-square test** | Are categorical variables independent? | Observed vs. Expected frequencies |
| **ANOVA (F-test)** | Do 3+ groups have different means? | Multiple group means |
| **Mann-Whitney U** | Non-parametric alternative to t-test | Distributions (not means) |
| **Z-test** | Large sample (n>30), known σ | Sample mean vs. hypothesized value |

### One-Tailed vs Two-Tailed

```
Two-Tailed (H₁: μ ≠ μ₀):          One-Tailed (H₁: μ > μ₀):
"Is it different?"                   "Is it greater?"

    Reject    │  Don't   │  Reject       Don't Reject   │  Reject
     ╱╲       │  Reject  │    ╱╲             ╱╲         │    ╱╲
    ╱  ╲      │  ╱╲      │   ╱  ╲           ╱  ╲        │   ╱  ╲
───╱────╲─────┼─╱──╲─────┼──╱────╲───   ───╱────╲───────┼──╱────╲───
   α/2        │         │    α/2                        │    α
```

### Code: Hypothesis Testing

```python
import numpy as np
from scipy import stats

np.random.seed(42)

# ============================================
# 1. One-Sample t-Test
# ============================================
print("=== One-Sample t-Test ===")
print("Question: Is the average delivery time different from the promised 30 minutes?\n")

# Simulated delivery times (minutes)
delivery_times = np.array([32, 28, 35, 30, 29, 33, 31, 34, 27, 36, 
                           30, 32, 28, 35, 33, 31, 29, 34, 30, 32])

# H₀: μ = 30 (delivers on time)
# H₁: μ ≠ 30 (delivery time is different)
t_stat, p_value = stats.ttest_1samp(delivery_times, popmean=30)

print(f"Sample mean: {np.mean(delivery_times):.2f} minutes")
print(f"Sample std:  {np.std(delivery_times, ddof=1):.2f} minutes")
print(f"t-statistic: {t_stat:.4f}")
print(f"p-value:     {p_value:.4f}")
print(f"Decision:    {'Reject H₀' if p_value < 0.05 else 'Fail to reject H₀'} (α=0.05)")
print(f"Conclusion:  {'Delivery time IS significantly different from 30 min' if p_value < 0.05 else 'No evidence delivery time differs from 30 min'}")

# ============================================
# 2. Two-Sample t-Test (Independent)
# ============================================
print("\n=== Two-Sample t-Test ===")
print("Question: Do users spend more time on Website A vs Website B?\n")

# Simulated time spent (seconds)
website_a = np.random.normal(120, 30, 50)   # Mean=120s, std=30s
website_b = np.random.normal(135, 35, 50)   # Mean=135s, std=35s

# H₀: μ_A = μ_B (no difference)
# H₁: μ_A ≠ μ_B (there is a difference)
t_stat, p_value = stats.ttest_ind(website_a, website_b)

print(f"Website A: mean = {np.mean(website_a):.1f}s (n={len(website_a)})")
print(f"Website B: mean = {np.mean(website_b):.1f}s (n={len(website_b)})")
print(f"Difference: {np.mean(website_b) - np.mean(website_a):.1f}s")
print(f"t-statistic: {t_stat:.4f}")
print(f"p-value:     {p_value:.4f}")
print(f"Decision:    {'Reject H₀ → Significant difference!' if p_value < 0.05 else 'Fail to reject H₀'}")

# Effect size (Cohen's d) — practical significance
cohens_d = (np.mean(website_b) - np.mean(website_a)) / np.sqrt(
    (np.std(website_a, ddof=1)**2 + np.std(website_b, ddof=1)**2) / 2)
print(f"Cohen's d:   {cohens_d:.3f} ({'Small' if abs(cohens_d) < 0.5 else 'Medium' if abs(cohens_d) < 0.8 else 'Large'} effect)")

# ============================================
# 3. Paired t-Test (Before/After)
# ============================================
print("\n=== Paired t-Test ===")
print("Question: Did the training program improve employee performance?\n")

# Performance scores before and after training
before = np.array([72, 68, 75, 80, 65, 70, 77, 73, 69, 74, 71, 76])
after  = np.array([78, 72, 80, 85, 70, 74, 82, 78, 73, 80, 75, 81])

# H₀: μ_diff = 0 (no improvement)
# H₁: μ_diff > 0 (improvement)
t_stat, p_value = stats.ttest_rel(after, before)
# For one-tailed test, divide p-value by 2
p_value_one_tailed = p_value / 2

differences = after - before
print(f"Mean before: {np.mean(before):.1f}")
print(f"Mean after:  {np.mean(after):.1f}")
print(f"Mean improvement: {np.mean(differences):.1f} points")
print(f"t-statistic: {t_stat:.4f}")
print(f"p-value (two-tailed): {p_value:.4f}")
print(f"p-value (one-tailed): {p_value_one_tailed:.4f}")
print(f"Decision: {'Reject H₀ → Training worked!' if p_value_one_tailed < 0.05 else 'Fail to reject H₀'}")

# ============================================
# 4. Welch's t-Test (Unequal Variances)
# ============================================
print("\n=== Welch's t-Test (Unequal Variances) ===")
# When two groups have very different spreads
group1 = np.random.normal(50, 5, 30)    # Low variance
group2 = np.random.normal(55, 15, 30)   # High variance

# Welch's test doesn't assume equal variances (equal_var=False)
t_stat, p_value = stats.ttest_ind(group1, group2, equal_var=False)
print(f"Group 1: mean={np.mean(group1):.1f}, std={np.std(group1, ddof=1):.1f}")
print(f"Group 2: mean={np.mean(group2):.1f}, std={np.std(group2, ddof=1):.1f}")
print(f"Welch's t-stat: {t_stat:.4f}, p-value: {p_value:.4f}")
print("Pro Tip: Always use Welch's test (equal_var=False) unless you have")
print("strong reason to believe variances are equal. It's safer.")
```

---

## 3.7 p-Values and Statistical Significance

### What a p-Value Is

> **The p-value is the probability of observing results as extreme (or more extreme) than what you got, IF the null hypothesis were true.**

$$\text{p-value} = P(\text{data this extreme} \mid H_0 \text{ is true})$$

**Analogy:** You flip a coin 10 times and get 9 heads. The p-value answers: "If the coin were fair, how likely is it to get 9+ heads?" Answer: p = 0.021. That's unlikely enough that we suspect the coin isn't fair.

### What a p-Value is NOT

| Common Misconception | Reality |
|---------------------|---------|
| "P-value = probability H₀ is true" | NO. It's P(data \| H₀), not P(H₀ \| data) |
| "p < 0.05 means the effect is large" | NO. Statistical significance ≠ practical significance |
| "p > 0.05 means there's no effect" | NO. It means not enough evidence to detect one |
| "p = 0.049 is significant but p = 0.051 is not" | Essentially the same! Don't treat α as a cliff |
| "Small p = important finding" | NO. With large n, even tiny differences are "significant" |

### Visualizing p-Value

```
Under H₀ (null is true), the test statistic has this distribution:

         ╭────────────────╮
        ╱                  ╲
       ╱                    ╲
      ╱                      ╲
     ╱                        ╲
────╱──────────────────────────╲────
                               ▲
                          Your observed
                          test statistic
                          
    ◄──── "Not surprising" ────►◄── p-value ──►
    If H₀ is true, most           This area = 
    data lands here              probability of
                                 getting result this
                                 extreme or more
```

### Code: Understanding p-Values

```python
import numpy as np
from scipy import stats

np.random.seed(42)

# ============================================
# Demonstration: What p-value means via simulation
# ============================================
print("=== Understanding p-Values Through Simulation ===\n")

# Scenario: A coin is flipped 100 times. We got 60 heads.
# Is the coin fair?
observed_heads = 60
n_flips = 100

# If coin IS fair (H₀: p = 0.5), what's the probability of 60+ heads?
# Method 1: Exact binomial
p_value_exact = 1 - stats.binom.cdf(59, n_flips, 0.5)  # P(X >= 60)
# For two-tailed: also count extreme in other direction
p_value_two_tailed = 2 * p_value_exact

print(f"Observed: {observed_heads} heads in {n_flips} flips")
print(f"Expected under H₀: {n_flips * 0.5} heads")
print(f"One-tailed p-value: {p_value_exact:.4f}")
print(f"Two-tailed p-value: {p_value_two_tailed:.4f}")

# Method 2: Simulation (Monte Carlo)
n_simulations = 100000
simulated_heads = np.random.binomial(n_flips, 0.5, n_simulations)
p_value_simulated = np.sum(simulated_heads >= observed_heads) / n_simulations
print(f"Simulated p-value:  {p_value_simulated:.4f}")

# ============================================
# The Multiple Testing Problem
# ============================================
print("\n=== The Multiple Testing Problem ===")
print("Testing 20 hypotheses where ALL are actually null (no real effect):\n")

n_tests = 20
alpha = 0.05
false_positives = 0

for i in range(n_tests):
    # Generate data under null (no difference between groups)
    group1 = np.random.normal(0, 1, 30)
    group2 = np.random.normal(0, 1, 30)  # Same distribution!
    _, p = stats.ttest_ind(group1, group2)
    
    if p < alpha:
        false_positives += 1
        print(f"  Test {i+1}: p = {p:.4f} ← FALSE POSITIVE! (significant by chance)")

print(f"\nFalse positives: {false_positives}/{n_tests}")
print(f"Expected: {n_tests * alpha:.0f} (= {n_tests} × {alpha})")
print(f"This is why multiple testing correction is critical!")

# Bonferroni correction: divide α by number of tests
alpha_corrected = alpha / n_tests
print(f"\nBonferroni corrected α: {alpha_corrected:.4f}")
print("Now much harder to get a false positive")

# ============================================
# Statistical vs Practical Significance
# ============================================
print("\n=== Statistical vs Practical Significance ===")

# With a HUGE sample, even tiny differences become "significant"
n_large = 100000
group_a = np.random.normal(100.0, 15, n_large)  # IQ = 100.0
group_b = np.random.normal(100.1, 15, n_large)  # IQ = 100.1 (trivial difference!)

t_stat, p_value = stats.ttest_ind(group_a, group_b)
cohens_d = (np.mean(group_b) - np.mean(group_a)) / np.sqrt(
    (np.std(group_a)**2 + np.std(group_b)**2) / 2)

print(f"Sample size: {n_large:,} per group")
print(f"Mean difference: {np.mean(group_b) - np.mean(group_a):.3f} IQ points")
print(f"p-value: {p_value:.6f} {'← Significant!' if p_value < 0.05 else ''}")
print(f"Cohen's d: {cohens_d:.4f} ← Tiny effect size!")
print(f"\n⚠️  Statistically significant but practically MEANINGLESS!")
print(f"   Always report effect size alongside p-value!")
```

### Effect Size Guidelines (Cohen's d)

| d value | Interpretation | Example |
|---------|---------------|---------|
| 0.2 | Small | Barely noticeable |
| 0.5 | Medium | Noticeable to careful observer |
| 0.8 | Large | Obvious difference |
| 1.2+ | Very large | Dramatic difference |

---

## 3.8 Confidence Intervals

### What They Are
A **confidence interval** gives a range of plausible values for a population parameter.

$$CI = \bar{x} \pm z_{\alpha/2} \times \frac{s}{\sqrt{n}}$$

For small samples (n < 30), use t-distribution:
$$CI = \bar{x} \pm t_{\alpha/2, n-1} \times \frac{s}{\sqrt{n}}$$

### What "95% Confidence" Actually Means

> **Correct interpretation:** If we repeated this sampling procedure many times, 95% of the confidence intervals would contain the true population parameter.

> **WRONG interpretation:** "There's a 95% probability the true value is in this interval." (The true value is fixed — it's either in or not in the interval.)

```
True population mean: μ = 50 (we don't know this in practice)

Sample 1:  ──────[    |  48.2  ──── 52.1  ]────── ✓ Contains μ
Sample 2:  ──────[  47.5  |──── 50.8  ]──────────── ✓ Contains μ  
Sample 3:  ──────────[  50.8  ────| 53.5  ]────── ✓ Contains μ
Sample 4:  ──[  44.2  ──── 47.8  ]──────────────── ✗ Misses μ!
Sample 5:  ──────[ 48.9  |──── 52.3  ]───────── ✓ Contains μ
                        ▲
                    μ = 50

With 95% CI: ~95 out of 100 intervals will capture the true mean
```

### Code: Confidence Intervals

```python
import numpy as np
from scipy import stats

np.random.seed(42)

# ============================================
# Building Confidence Intervals
# ============================================
print("=== Confidence Intervals ===\n")

# Scenario: Measuring average page load time
page_load_times = np.random.normal(2.5, 0.8, 40)  # True mean=2.5s

sample_mean = np.mean(page_load_times)
sample_std = np.std(page_load_times, ddof=1)
n = len(page_load_times)
se = sample_std / np.sqrt(n)

print(f"Sample size: {n}")
print(f"Sample mean: {sample_mean:.3f} seconds")
print(f"Sample std:  {sample_std:.3f} seconds")
print(f"Standard Error: {se:.4f}")

# CIs at different confidence levels
for confidence in [0.90, 0.95, 0.99]:
    ci = stats.t.interval(confidence, df=n-1, loc=sample_mean, scale=se)
    width = ci[1] - ci[0]
    print(f"\n{confidence*100:.0f}% CI: ({ci[0]:.3f}, {ci[1]:.3f})")
    print(f"  Width: {width:.3f}s")
    print(f"  Interpretation: We're {confidence*100:.0f}% confident the true")
    print(f"  average page load time is between {ci[0]:.3f}s and {ci[1]:.3f}s")

# ============================================
# CI for Proportions
# ============================================
print("\n\n=== Confidence Interval for Proportions ===")
# 100 users tried new feature, 73 liked it
n_users = 100
liked = 73
p_hat = liked / n_users

# Wald interval (normal approximation)
se_prop = np.sqrt(p_hat * (1 - p_hat) / n_users)
z_95 = stats.norm.ppf(0.975)  # 1.96
ci_lower = p_hat - z_95 * se_prop
ci_upper = p_hat + z_95 * se_prop

print(f"Sample proportion: {p_hat:.2f} ({liked}/{n_users})")
print(f"95% CI: ({ci_lower:.3f}, {ci_upper:.3f})")
print(f"Interpretation: Between {ci_lower*100:.1f}% and {ci_upper*100:.1f}%")
print(f"of ALL users would likely enjoy this feature")

# Wilson interval (better for extreme proportions)
ci_wilson = stats.proportion_confint(liked, n_users, method='wilson')
print(f"Wilson CI: ({ci_wilson[0]:.3f}, {ci_wilson[1]:.3f})")

# ============================================
# How Sample Size Affects CI
# ============================================
print("\n\n=== Sample Size vs CI Width ===")
print(f"{'n':<8} {'CI Width':<12} {'CI'}")
print("-" * 50)

true_std = 10  # Assume we know population std
for n in [10, 25, 50, 100, 500, 1000, 10000]:
    se = true_std / np.sqrt(n)
    margin = 1.96 * se
    print(f"{n:<8} {2*margin:<12.3f} ({50-margin:.2f}, {50+margin:.2f})")

print("\nKey insight: To halve CI width, you need 4x the sample size!")
print("(Because SE = σ/√n, so √n must double → n must quadruple)")

# ============================================
# Required Sample Size Calculation
# ============================================
print("\n\n=== Required Sample Size ===")
# "I want my CI width to be at most 2 units, with 95% confidence"
desired_width = 2
assumed_std = 10
z = 1.96

# Width = 2 × z × σ/√n → √n = 2 × z × σ / width → n = (2zσ/width)²
margin_of_error = desired_width / 2
n_required = (z * assumed_std / margin_of_error) ** 2
print(f"Desired CI width: ±{margin_of_error}")
print(f"Assumed std: {assumed_std}")
print(f"Required n: {int(np.ceil(n_required))}")
```

---

## 3.9 Types of Errors (Type I and Type II)

### What They Are

| | H₀ is TRUE (no effect) | H₀ is FALSE (real effect) |
|---|---|---|
| **Reject H₀** | Type I Error (α) — False Positive | Correct! (Power = 1-β) |
| **Fail to reject H₀** | Correct! (1-α) | Type II Error (β) — False Negative |

**Analogies:**
- **Type I (False Positive):** Fire alarm goes off but there's no fire → unnecessary evacuation
- **Type II (False Negative):** There's a fire but the alarm doesn't go off → danger!

```
                    Reality
                    ┌─────────────────────────────────────┐
                    │    H₀ True        │    H₀ False     │
                    │  (Innocent)       │   (Guilty)      │
    ┌───────────────┼──────────────────┼─────────────────┤
D   │ Reject H₀     │  TYPE I ERROR    │   CORRECT       │
e   │ (Convict)     │  α = P(reject    │   (Power)       │
c   │               │  when shouldn't) │   1 - β         │
i   ├───────────────┼──────────────────┼─────────────────┤
s   │ Don't Reject  │  CORRECT         │   TYPE II ERROR │
i   │ (Acquit)      │  (1 - α)         │   β = P(miss    │
o   │               │  (Specificity)   │   when should   │
n   │               │                  │   detect)       │
    └───────────────┴──────────────────┴─────────────────┘
```

### The Trade-Off

```
More Stringent (lower α):
├── Fewer false positives (good!)
├── More false negatives (bad!)
└── Harder to detect real effects

Less Stringent (higher α):
├── More false positives (bad!)
├── Fewer false negatives (good!)
└── Easier to detect real effects

    ← Conservative ─────────────────── Liberal →
    α = 0.001                          α = 0.10
    Medical drugs                      Exploratory research
    "Must be very sure"                "Don't want to miss anything"
```

### Statistical Power

$$\text{Power} = 1 - \beta = P(\text{Reject } H_0 | H_0 \text{ is false})$$

Power depends on:
1. **Effect size** — Bigger effects are easier to detect
2. **Sample size** — More data = more power
3. **Significance level (α)** — Higher α = more power (but more Type I errors)
4. **Variability** — Less noise = more power

**Rule of thumb:** Aim for power ≥ 0.80 (80% chance of detecting a real effect)

### Code: Power Analysis

```python
import numpy as np
from scipy import stats

# ============================================
# Demonstrating Type I and Type II Errors
# ============================================
np.random.seed(42)
n_experiments = 10000
alpha = 0.05

# Scenario 1: H₀ IS true (no real difference)
print("=== Type I Error Rate ===")
type_1_errors = 0
for _ in range(n_experiments):
    # Both groups from SAME distribution (H₀ true)
    group1 = np.random.normal(50, 10, 30)
    group2 = np.random.normal(50, 10, 30)
    _, p = stats.ttest_ind(group1, group2)
    if p < alpha:
        type_1_errors += 1

print(f"H₀ is TRUE, tested {n_experiments} times")
print(f"Type I errors (false rejections): {type_1_errors}")
print(f"Type I error rate: {type_1_errors/n_experiments:.4f} (expected: {alpha})")

# Scenario 2: H₀ IS false (real difference exists)
print("\n=== Type II Error Rate & Power ===")
true_effect = 5  # Real difference between groups

type_2_errors = 0
for _ in range(n_experiments):
    # Groups from DIFFERENT distributions (H₀ false)
    group1 = np.random.normal(50, 10, 30)
    group2 = np.random.normal(50 + true_effect, 10, 30)
    _, p = stats.ttest_ind(group1, group2)
    if p >= alpha:  # Failed to reject when should have
        type_2_errors += 1

power = 1 - type_2_errors / n_experiments
print(f"H₀ is FALSE (true effect = {true_effect})")
print(f"Type II errors (missed detections): {type_2_errors}")
print(f"Type II error rate (β): {type_2_errors/n_experiments:.4f}")
print(f"Power (1-β): {power:.4f}")

# ============================================
# Power vs Sample Size
# ============================================
print("\n=== Power vs Sample Size ===")
print(f"Effect size: {true_effect}, σ: 10, α: {alpha}")
print(f"{'n per group':<12} {'Power':<10} {'Sufficient?'}")
print("-" * 35)

for n in [10, 20, 30, 50, 75, 100, 150]:
    # Calculate power analytically
    # Cohen's d for this scenario
    d = true_effect / 10  # effect / std
    # Non-centrality parameter
    ncp = d * np.sqrt(n / 2)
    # Critical value
    t_crit = stats.t.ppf(1 - alpha/2, df=2*n-2)
    # Power
    power_calc = 1 - stats.nct.cdf(t_crit, df=2*n-2, nc=ncp)
    power_calc += stats.nct.cdf(-t_crit, df=2*n-2, nc=ncp)
    
    sufficient = "✓" if power_calc >= 0.80 else "✗"
    print(f"{n:<12} {power_calc:<10.3f} {sufficient}")

# ============================================
# Minimum Sample Size for Desired Power
# ============================================
print("\n=== Required Sample Size Calculator ===")
# Using the formula: n ≈ (z_α + z_β)² × 2σ² / δ²
# Where δ = effect size (difference in means)

def required_sample_size(effect_size, std, alpha=0.05, power=0.80):
    """Calculate minimum n per group for two-sample t-test."""
    z_alpha = stats.norm.ppf(1 - alpha/2)
    z_beta = stats.norm.ppf(power)
    n = 2 * ((z_alpha + z_beta) * std / effect_size) ** 2
    return int(np.ceil(n))

scenarios = [
    ("Large effect (d=0.8)", 0.8*10, 10),
    ("Medium effect (d=0.5)", 0.5*10, 10),
    ("Small effect (d=0.2)", 0.2*10, 10),
]

for name, effect, std in scenarios:
    n = required_sample_size(effect, std)
    print(f"{name}: need n = {n} per group")
```

---

## 3.10 Chi-Square, t-Tests, and ANOVA

### Chi-Square Test

**Use for:** Testing relationships between categorical variables.

$$\chi^2 = \sum \frac{(O_i - E_i)^2}{E_i}$$

Where $O_i$ = observed frequency, $E_i$ = expected frequency.

| Type | Tests | Example |
|------|-------|---------|
| Goodness of fit | Does data match expected distribution? | Are die rolls fair? |
| Independence | Are two variables independent? | Is gender related to product preference? |

### ANOVA (Analysis of Variance)

**Use for:** Comparing means of 3+ groups simultaneously.

$$F = \frac{\text{Between-group variance}}{\text{Within-group variance}} = \frac{MS_{between}}{MS_{within}}$$

**Why not just do multiple t-tests?**
- 3 groups → 3 comparisons → inflated Type I error
- 10 groups → 45 comparisons → P(at least one false positive) = 1 - (0.95)^45 = 90%!
- ANOVA controls this by testing all at once

### Code: Chi-Square and ANOVA

```python
import numpy as np
from scipy import stats
import pandas as pd

np.random.seed(42)

# ============================================
# 1. Chi-Square Goodness of Fit
# ============================================
print("=== Chi-Square: Goodness of Fit ===")
print("Is this die fair?\n")

# Observed die rolls (600 rolls)
observed = np.array([95, 108, 92, 104, 100, 101])  # Counts for each face
expected = np.array([100, 100, 100, 100, 100, 100])  # Expected if fair

chi2, p_value = stats.chisquare(observed, expected)
print(f"Observed: {observed}")
print(f"Expected: {expected}")
print(f"χ² statistic: {chi2:.4f}")
print(f"p-value: {p_value:.4f}")
print(f"Conclusion: {'Die appears unfair' if p_value < 0.05 else 'No evidence die is unfair'}")

# ============================================
# 2. Chi-Square Test of Independence
# ============================================
print("\n=== Chi-Square: Test of Independence ===")
print("Is product preference related to age group?\n")

# Contingency table
# Rows: Age groups, Columns: Product preference
#            Product A  Product B  Product C
# Young        150        80         70
# Middle       120       100         80
# Old           80       120        100

contingency_table = np.array([
    [150, 80, 70],    # Young
    [120, 100, 80],   # Middle
    [80, 120, 100]    # Old
])

chi2, p_value, dof, expected_freq = stats.chi2_contingency(contingency_table)

print("Contingency Table:")
print(pd.DataFrame(contingency_table, 
                   index=['Young', 'Middle', 'Old'],
                   columns=['Product A', 'Product B', 'Product C']))
print(f"\nχ² statistic: {chi2:.4f}")
print(f"p-value: {p_value:.6f}")
print(f"Degrees of freedom: {dof}")
print(f"Conclusion: {'Variables ARE related' if p_value < 0.05 else 'Variables are independent'}")

# Cramér's V (effect size for chi-square)
n_total = contingency_table.sum()
min_dim = min(contingency_table.shape) - 1
cramers_v = np.sqrt(chi2 / (n_total * min_dim))
print(f"Cramér's V: {cramers_v:.4f} (effect size)")

# ============================================
# 3. One-Way ANOVA
# ============================================
print("\n=== One-Way ANOVA ===")
print("Do three teaching methods produce different test scores?\n")

# Three groups of students
method_a = np.random.normal(75, 10, 30)   # Traditional
method_b = np.random.normal(80, 12, 30)   # Interactive  
method_c = np.random.normal(82, 11, 30)   # Online + projects

# H₀: μ_A = μ_B = μ_C (all methods equal)
# H₁: At least one mean is different
f_stat, p_value = stats.f_oneway(method_a, method_b, method_c)

print(f"Method A mean: {np.mean(method_a):.1f} (Traditional)")
print(f"Method B mean: {np.mean(method_b):.1f} (Interactive)")
print(f"Method C mean: {np.mean(method_c):.1f} (Online + Projects)")
print(f"\nF-statistic: {f_stat:.4f}")
print(f"p-value: {p_value:.4f}")
print(f"Conclusion: {'At least one method differs' if p_value < 0.05 else 'No significant difference'}")

# Post-hoc test: WHICH groups differ? (Tukey HSD)
from scipy.stats import tukey_hsd
result = tukey_hsd(method_a, method_b, method_c)
print(f"\nPost-hoc Tukey HSD (pairwise comparisons):")
groups = ['Method A', 'Method B', 'Method C']
for i in range(3):
    for j in range(i+1, 3):
        # Access the confidence interval to determine significance
        ci = result.confidence_interval(confidence_level=0.95)
        p_ij = result.pvalue[i][j]
        sig = "***" if p_ij < 0.001 else "**" if p_ij < 0.01 else "*" if p_ij < 0.05 else "ns"
        print(f"  {groups[i]} vs {groups[j]}: p = {p_ij:.4f} {sig}")

# ============================================
# 4. Kruskal-Wallis (Non-parametric ANOVA)
# ============================================
print("\n=== Kruskal-Wallis (Non-parametric Alternative) ===")
print("Use when assumptions of ANOVA are violated (non-normal, unequal variances)\n")

# Same data but using non-parametric test
h_stat, p_value = stats.kruskal(method_a, method_b, method_c)
print(f"H-statistic: {h_stat:.4f}")
print(f"p-value: {p_value:.4f}")

# ============================================
# 5. Choosing the Right Test: Decision Tree
# ============================================
print("\n=== Test Selection Guide ===")
print("""
┌─── What are you comparing? ────────────────────────────────────┐
│                                                                 │
├── One sample vs known value?                                    │
│   ├── Numerical, normal → One-sample t-test                     │
│   └── Numerical, non-normal → Wilcoxon signed-rank              │
│                                                                 │
├── Two independent groups?                                       │
│   ├── Numerical, normal → Independent t-test (Welch's)          │
│   ├── Numerical, non-normal → Mann-Whitney U                    │
│   └── Categorical → Chi-square test                             │
│                                                                 │
├── Two paired/matched groups?                                    │
│   ├── Numerical, normal → Paired t-test                         │
│   └── Numerical, non-normal → Wilcoxon signed-rank              │
│                                                                 │
├── Three or more independent groups?                             │
│   ├── Numerical, normal → One-way ANOVA                         │
│   ├── Numerical, non-normal → Kruskal-Wallis                    │
│   └── Categorical → Chi-square test                             │
│                                                                 │
└── Relationship between two variables?                           │
    ├── Both numerical → Pearson/Spearman correlation             │
    └── Both categorical → Chi-square independence test           │
└─────────────────────────────────────────────────────────────────┘
""")
```

---

## 3.11 Common Mistakes

| # | Mistake | Why It's Wrong | Correct Approach |
|---|---------|---------------|-----------------|
| 1 | "p=0.04 means 4% chance H₀ is true" | p-value is P(data\|H₀), not P(H₀\|data) | State: "If H₀ were true, this extreme result has 4% probability" |
| 2 | Treating p=0.05 as a hard cutoff | p=0.049 and p=0.051 are essentially the same | Report exact p-values, interpret in context |
| 3 | Not correcting for multiple comparisons | 20 tests at α=0.05 → expect 1 false positive | Use Bonferroni, Holm, or FDR correction |
| 4 | Ignoring effect size | Large n makes tiny differences "significant" | Always report and interpret effect size (Cohen's d, r²) |
| 5 | Confusing confidence interval meaning | "95% chance the parameter is in this interval" | "95% of intervals constructed this way contain the parameter" |
| 6 | Base rate neglect (ignoring priors) | A 99% accurate test can still have 80% false positives | Apply Bayes' theorem; consider base rates |
| 7 | Assuming independence without checking | Many real-world events are dependent | Test for independence; use appropriate methods |
| 8 | Using parametric tests on non-normal data | Results may be invalid | Check assumptions; use non-parametric alternatives |
| 9 | p-hacking (testing until p < 0.05) | Inflates false positive rate enormously | Pre-register hypotheses, use adjusted alphas |
| 10 | Survivorship bias | Only looking at "survivors" | Consider what data is missing from your sample |

---

## 3.12 Interview Questions

**Q1: Explain Bayes' Theorem with a real example.**
> Bayes' Theorem: P(A|B) = P(B|A)×P(A) / P(B). Example: Email spam filter. Prior: 30% of emails are spam. Likelihood: P("free money"|spam) = 0.8. P("free money"|not spam) = 0.01. P(spam|"free money") = (0.8×0.3)/(0.8×0.3 + 0.01×0.7) = 0.24/0.247 = 97.2%. The word "free money" makes us 97% confident it's spam.

**Q2: What is a p-value and what are its limitations?**
> A p-value is the probability of observing results as extreme as the data, assuming H₀ is true. Limitations: (1) It's not P(H₀ is true). (2) Statistically significant ≠ practically significant. (3) Affected by sample size — large n can make trivial effects significant. (4) Binary thresholds (p<0.05) are arbitrary. Always complement with effect sizes and confidence intervals.

**Q3: Type I vs Type II error — which is worse?**
> It depends on context. Medical test for cancer: Type II (missing cancer) is worse → use sensitive tests. Criminal justice: Type I (convicting innocent) is worse → require "beyond reasonable doubt." Spam filter: Type I (blocking legit email) might be worse than Type II (letting spam through). The cost of each error type should guide your choice of α.

**Q4: How would you determine sample size for an A/B test?**
> I'd need: (1) Minimum detectable effect (MDE) — smallest meaningful difference, (2) Baseline metric and its variance, (3) Desired power (typically 80%), (4) Significance level (typically 5%). Formula: n ≈ 2×(z_α + z_β)²×σ²/δ². For proportions: n ≈ 2×p(1-p)×(z_α + z_β)²/δ². Typical A/B tests need thousands of users per variant.

**Q5: Explain the difference between parametric and non-parametric tests.**
> Parametric tests (t-test, ANOVA) assume specific data distributions (usually normal) and work with population parameters (mean, variance). Non-parametric tests (Mann-Whitney, Kruskal-Wallis) make fewer assumptions and work with ranks. Use non-parametric when: data is ordinal, clearly non-normal, has small sample size, or has outliers. Trade-off: parametric tests have more power when assumptions hold.

**Q6: A model has 99.5% accuracy on fraud detection. Is this good?**
> Not necessarily. If only 0.1% of transactions are fraudulent, a model that always predicts "not fraud" achieves 99.9% accuracy. I'd look at: (1) Precision: Of flagged transactions, how many are actual fraud? (2) Recall: Of all frauds, how many did we catch? (3) F1-score. (4) AUC-ROC. For fraud, high recall is usually priority (catch all frauds), even at cost of some false positives.

**Q7: What is the Central Limit Theorem and when does it break down?**
> CLT: Sample means approach normal distribution as n→∞, regardless of population shape. Mean of sample means = population mean, SE = σ/√n. It breaks down when: (1) Sample size is too small (n<30 for skewed data). (2) Data has infinite variance (heavy-tailed distributions like Cauchy). (3) Observations aren't independent. (4) The distribution has no finite mean. Rule of thumb: n≥30 works for most distributions.

---

## 3.13 Quick Reference

### Probability Formulas

| Rule | Formula | When to Use |
|------|---------|------------|
| Complement | $P(A') = 1 - P(A)$ | "At least one" problems |
| Addition (OR) | $P(A \cup B) = P(A) + P(B) - P(A \cap B)$ | Either event happening |
| Multiplication (AND) | $P(A \cap B) = P(A) \cdot P(B\|A)$ | Both events happening |
| Independence | $P(A \cap B) = P(A) \cdot P(B)$ | When events don't affect each other |
| Conditional | $P(A\|B) = \frac{P(A \cap B)}{P(B)}$ | Probability given information |
| Bayes' Theorem | $P(A\|B) = \frac{P(B\|A) \cdot P(A)}{P(B)}$ | Updating beliefs with evidence |
| Total Probability | $P(B) = \sum_i P(B\|A_i) \cdot P(A_i)$ | Computing marginal probability |

### Hypothesis Testing Cheat Sheet

| Step | Action | Key Point |
|------|--------|-----------|
| 1 | State H₀ and H₁ | H₀ always has "=" |
| 2 | Choose α (usually 0.05) | Lower α = fewer false positives |
| 3 | Select test | Based on data type, groups, assumptions |
| 4 | Compute test statistic | t, z, χ², F depending on test |
| 5 | Get p-value | Compare to α |
| 6 | Decide | p < α → Reject H₀ |
| 7 | Report | Include effect size, CI, n |

### Distribution Quick Guide

| Distribution | Key Formula | Python |
|-------------|-------------|--------|
| Normal PDF | $\frac{1}{\sigma\sqrt{2\pi}}e^{-(x-\mu)^2/2\sigma^2}$ | `stats.norm.pdf(x, mu, sigma)` |
| Normal CDF | P(X ≤ x) | `stats.norm.cdf(x, mu, sigma)` |
| Binomial PMF | $\binom{n}{k}p^k(1-p)^{n-k}$ | `stats.binom.pmf(k, n, p)` |
| Poisson PMF | $\frac{\lambda^k e^{-\lambda}}{k!}$ | `stats.poisson.pmf(k, lam)` |
| t-distribution | Used for small samples | `stats.t.ppf(q, df)` |
| χ² distribution | Sum of squared normals | `stats.chi2.cdf(x, df)` |

### Critical Values (Most Common)

| Test | α = 0.05 (two-tailed) | α = 0.01 (two-tailed) |
|------|----------------------|----------------------|
| Z-test | ±1.96 | ±2.576 |
| t-test (df=30) | ±2.042 | ±2.750 |
| t-test (df=∞) | ±1.96 | ±2.576 |
| χ² (df=1) | 3.841 | 6.635 |
| χ² (df=5) | 11.070 | 15.086 |

---

**← Previous:** [02-Statistics-Fundamentals](./02-Statistics-Fundamentals.md) | **Next:** [04-Data-Collection-and-Cleaning](./04-Data-Collection-and-Cleaning.md) →
