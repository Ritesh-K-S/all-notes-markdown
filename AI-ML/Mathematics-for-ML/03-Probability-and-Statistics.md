# Probability and Statistics for Machine Learning

## Overview
Probability and statistics form the **theoretical backbone** of machine learning. Every ML model is making probabilistic predictions, every evaluation metric is a statistical measure, and every training process is an optimization over probability distributions. Understanding this chapter means understanding WHY models work — not just HOW to use them.

---

## Table of Contents
1. [Probability Basics](#1-probability-basics)
2. [Conditional Probability and Bayes' Theorem](#2-conditional-probability-and-bayes-theorem)
3. [Random Variables and Distributions](#3-random-variables-and-distributions)
4. [Common Probability Distributions](#4-common-probability-distributions)
5. [Expectation, Variance, and Moments](#5-expectation-variance-and-moments)
6. [Joint, Marginal, and Conditional Distributions](#6-joint-marginal-and-conditional-distributions)
7. [Maximum Likelihood Estimation (MLE)](#7-maximum-likelihood-estimation-mle)
8. [Maximum A Posteriori (MAP)](#8-maximum-a-posteriori-map)
9. [Bayesian Thinking](#9-bayesian-thinking)
10. [Sampling and Monte Carlo Methods](#10-sampling-and-monte-carlo-methods)
11. [Hypothesis Testing and Confidence Intervals](#11-hypothesis-testing-and-confidence-intervals)
12. [Information Theory Essentials](#12-information-theory-essentials)
13. [Applications in ML and Interview Prep](#13-applications-in-ml-and-interview-prep)

---

## 1. Probability Basics

### What It Is
Probability measures **how likely** something is to happen, on a scale from 0 (impossible) to 1 (certain).

$$P(A) \in [0, 1]$$

### Why It Matters
- **Classification** = estimating $P(\text{class} | \text{features})$
- **Generative models** = learning $P(\text{data})$ and sampling from it
- **Uncertainty** = knowing HOW CONFIDENT a model is (not just its prediction)
- **Regularization** = imposing prior beliefs about parameters (Bayesian view)

### How It Works

Three interpretations of probability:

| View | Meaning | Example |
|------|---------|---------|
| **Frequentist** | Long-run frequency of events | "If I flip this coin 10,000 times, ~50% will be heads" |
| **Bayesian** | Degree of belief/uncertainty | "I'm 70% sure it will rain tomorrow" |
| **Axiomatic** | Mathematical rules (Kolmogorov) | Formal foundation for both views |

**Core Axioms:**
1. $P(A) \geq 0$ for any event A
2. $P(\Omega) = 1$ (something must happen)
3. For mutually exclusive events: $P(A \cup B) = P(A) + P(B)$

### Key Rules

```
ADDITION RULE (OR):
P(A or B) = P(A) + P(B) - P(A and B)

    ┌─────────────────┐
    │   A    ╔════╗   │
    │  ┌─────╢ A∩B╟───┤ B
    │  │     ╚════╝   │
    │  └──────────────┘
    
If mutually exclusive (no overlap):
P(A or B) = P(A) + P(B)

MULTIPLICATION RULE (AND):
P(A and B) = P(A) × P(B|A)

If independent (knowing A doesn't affect B):
P(A and B) = P(A) × P(B)

COMPLEMENT RULE (NOT):
P(not A) = 1 - P(A)
```

### Code Examples

```python
import numpy as np
from collections import Counter

# === SIMULATING PROBABILITY ===
np.random.seed(42)

# Rolling two dice: P(sum = 7)?
n_simulations = 100_000
die1 = np.random.randint(1, 7, n_simulations)
die2 = np.random.randint(1, 7, n_simulations)
sums = die1 + die2

p_sum_7 = np.mean(sums == 7)
print(f"P(sum=7) simulated: {p_sum_7:.4f}")  # ≈ 0.1667
print(f"P(sum=7) exact: {6/36:.4f}")           # = 0.1667 (6 ways out of 36)

# Distribution of all sums
sum_counts = Counter(sums)
for s in sorted(sum_counts):
    bar = '█' * int(sum_counts[s] / 1000)
    print(f"Sum {s:2d}: {sum_counts[s]/n_simulations:.4f} {bar}")

# === INDEPENDENCE TEST ===
# Are these events independent?
# A: first die is 6,  B: sum > 8

event_A = die1 == 6
event_B = sums > 8

P_A = np.mean(event_A)
P_B = np.mean(event_B)
P_A_and_B = np.mean(event_A & event_B)

print(f"\nP(A) = {P_A:.4f}")
print(f"P(B) = {P_B:.4f}")
print(f"P(A)×P(B) = {P_A * P_B:.4f}")
print(f"P(A∩B) = {P_A_and_B:.4f}")
print(f"Independent? {np.isclose(P_A * P_B, P_A_and_B, atol=0.01)}")
# NOT independent! Knowing die1=6 makes sum>8 more likely

# === BIRTHDAY PROBLEM ===
# P(at least 2 people share birthday) in a group of n
def birthday_probability(n):
    """Exact calculation"""
    p_no_match = 1.0
    for i in range(1, n):
        p_no_match *= (365 - i) / 365
    return 1 - p_no_match

for n in [10, 20, 23, 30, 50, 70]:
    print(f"n={n:2d}: P(shared birthday) = {birthday_probability(n):.4f}")
# n=23: P ≈ 0.507 → 50% chance with just 23 people!
# This is why hash collisions happen sooner than you'd expect

# === LAW OF LARGE NUMBERS ===
# As n grows, sample mean → true mean
coin_flips = np.random.randint(0, 2, 10000)  # 0=tails, 1=heads
running_mean = np.cumsum(coin_flips) / np.arange(1, 10001)
print(f"\nAfter 10 flips: mean = {running_mean[9]:.4f}")
print(f"After 100 flips: mean = {running_mean[99]:.4f}")
print(f"After 10000 flips: mean = {running_mean[9999]:.4f}")  # ≈ 0.5
```

### Common Mistakes
1. **Confusing independent with mutually exclusive** — Mutually exclusive events are NEVER independent (if A happens, B can't → knowing A tells you about B)
2. **Forgetting the complement rule** — P(at least one) = 1 - P(none) is often much easier to compute
3. **Base rate neglect** — Ignoring how common something is when interpreting test results (see Bayes section)

---

## 2. Conditional Probability and Bayes' Theorem

### What It Is
**Conditional probability** $P(A|B)$ is the probability of A happening GIVEN that B has already happened.

**Bayes' Theorem** flips the condition — it lets you compute $P(\text{cause}|\text{evidence})$ from $P(\text{evidence}|\text{cause})$.

$$P(A|B) = \frac{P(B|A) \cdot P(A)}{P(B)}$$

### Why It Matters
- **Naive Bayes classifier** — directly applies Bayes' theorem
- **Bayesian inference** — updating beliefs with new data
- **Medical diagnosis** — P(disease | positive test)
- **Spam filtering** — P(spam | contains "free money")
- **Every probabilistic model** implicitly uses conditional probability

### How It Works

**Analogy:** You hear a loud noise (evidence). What caused it? Thunder (80% of loud noises), car backfire (15%), or explosion (5%)? But right now you also see dark clouds (prior info). Bayes' theorem combines the evidence with your prior knowledge to update your belief.

```
Bayes' Theorem Breakdown:

P(Disease | Positive Test) = P(Positive | Disease) × P(Disease)
                              ────────────────────────────────────
                                        P(Positive)

                 Posterior    =  Likelihood   ×   Prior
                                 ─────────────────────
                                      Evidence

Where:
  Prior      = What you believed BEFORE seeing data
  Likelihood = How well the data fits each hypothesis
  Evidence   = Total probability of seeing this data
  Posterior  = Updated belief AFTER seeing data
```

### Code Examples

```python
import numpy as np

# === THE CLASSIC: Medical Test Problem ===
# Disease prevalence: 1 in 1000 (0.1%)
# Test sensitivity: 99% (P(positive | disease) = 0.99)
# Test specificity: 95% (P(negative | healthy) = 0.95)
# Question: If you test positive, what's P(disease)?

P_disease = 0.001
P_healthy = 1 - P_disease
P_positive_given_disease = 0.99   # Sensitivity
P_positive_given_healthy = 0.05   # False positive rate (1 - specificity)

# Bayes' theorem
P_positive = (P_positive_given_disease * P_disease + 
              P_positive_given_healthy * P_healthy)

P_disease_given_positive = (P_positive_given_disease * P_disease) / P_positive

print(f"P(disease | positive test) = {P_disease_given_positive:.4f}")
# ≈ 0.0194 → Only ~2%!!! 
# Even with a 99% accurate test, most positive results are FALSE POSITIVES
# because the disease is so rare (base rate fallacy)

# === SIMULATION VERIFICATION ===
np.random.seed(42)
population = 1_000_000

# Who has the disease?
has_disease = np.random.random(population) < P_disease

# Test results
test_positive = np.zeros(population, dtype=bool)
test_positive[has_disease] = np.random.random(has_disease.sum()) < P_positive_given_disease
test_positive[~has_disease] = np.random.random((~has_disease).sum()) < P_positive_given_healthy

# Among those who tested positive, how many actually have disease?
true_positives = (test_positive & has_disease).sum()
all_positives = test_positive.sum()
print(f"\nSimulated P(disease|positive) = {true_positives/all_positives:.4f}")

# === NAIVE BAYES CLASSIFIER FROM SCRATCH ===
class NaiveBayes:
    """
    Naive Bayes assumes features are conditionally independent given the class.
    P(class|features) ∝ P(class) × Π P(feature_i|class)
    
    "Naive" because features are rarely truly independent,
    but it works surprisingly well in practice!
    """
    
    def fit(self, X, y):
        self.classes = np.unique(y)
        self.n_classes = len(self.classes)
        self.n_features = X.shape[1]
        
        # Calculate P(class) — prior
        self.priors = np.array([np.mean(y == c) for c in self.classes])
        
        # Calculate P(feature|class) — likelihood (Gaussian assumption)
        self.means = np.zeros((self.n_classes, self.n_features))
        self.stds = np.zeros((self.n_classes, self.n_features))
        
        for i, c in enumerate(self.classes):
            X_c = X[y == c]
            self.means[i] = X_c.mean(axis=0)
            self.stds[i] = X_c.std(axis=0) + 1e-8  # Avoid division by zero
    
    def _gaussian_likelihood(self, x, mean, std):
        """P(x | class) assuming Gaussian distribution"""
        exponent = -0.5 * ((x - mean) / std) ** 2
        return np.exp(exponent) / (std * np.sqrt(2 * np.pi))
    
    def predict(self, X):
        predictions = []
        for x in X:
            posteriors = []
            for i, c in enumerate(self.classes):
                # log P(class) + Σ log P(feature_i | class)
                prior = np.log(self.priors[i])
                likelihood = np.sum(np.log(
                    self._gaussian_likelihood(x, self.means[i], self.stds[i])
                ))
                posteriors.append(prior + likelihood)
            predictions.append(self.classes[np.argmax(posteriors)])
        return np.array(predictions)

# Test on synthetic data
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split

X, y = make_classification(n_samples=500, n_features=4, n_classes=2, 
                           random_state=42)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, 
                                                     random_state=42)

nb = NaiveBayes()
nb.fit(X_train, y_train)
predictions = nb.predict(X_test)
accuracy = np.mean(predictions == y_test)
print(f"\nNaive Bayes accuracy: {accuracy:.4f}")

# Compare with sklearn
from sklearn.naive_bayes import GaussianNB
sklearn_nb = GaussianNB().fit(X_train, y_train)
print(f"Sklearn accuracy: {sklearn_nb.score(X_test, y_test):.4f}")

# === BAYESIAN UPDATING ===
def bayesian_update(prior, likelihood, evidence):
    """Apply Bayes' rule to update belief"""
    posterior = (likelihood * prior) / evidence
    return posterior

# Coin bias estimation: is the coin fair?
# Prior: we think P(heads) = 0.5 (uniform prior over [0,1])
# We flip and get: H, H, H, T, H (4 heads, 1 tail)

# Using Beta distribution as prior (conjugate prior for Bernoulli)
from scipy import stats

# Prior: Beta(1, 1) = Uniform
alpha_prior, beta_prior = 1, 1  

# After 4 heads, 1 tail: Beta(1+4, 1+1) = Beta(5, 2)
alpha_post = alpha_prior + 4  # Add successes
beta_post = beta_prior + 1    # Add failures

x = np.linspace(0, 1, 100)
prior_dist = stats.beta.pdf(x, alpha_prior, beta_prior)
posterior_dist = stats.beta.pdf(x, alpha_post, beta_post)

posterior_mean = alpha_post / (alpha_post + beta_post)
print(f"\nPrior belief: P(heads) = {alpha_prior/(alpha_prior+beta_prior):.2f}")
print(f"After 4H, 1T: P(heads) = {posterior_mean:.4f}")
# 0.714 — shifted toward heads, but not all the way to 0.8 (prior pulls it back)
```

### Common Mistakes
1. **Base rate neglect** — The #1 mistake. A 99% accurate test still gives mostly false positives for rare diseases
2. **Confusing P(A|B) with P(B|A)** — P(positive|disease) ≠ P(disease|positive). This is called the "prosecutor's fallacy"
3. **Forgetting to normalize** — Bayes gives unnormalized posteriors; you must divide by P(evidence)
4. **Assuming Naive Bayes needs independent features** — It works even when features are correlated (predictions are still good, probability estimates may be off)

---

## 3. Random Variables and Distributions

### What It Is
A **random variable** is a variable whose value is determined by a random process. A **probability distribution** describes the likelihood of each possible value.

- **Discrete** random variable: finite or countable values (coin flip, dice, word count)
- **Continuous** random variable: any value in a range (height, temperature, probability)

### Why It Matters
- **Every feature** in a dataset is a random variable
- **Model outputs** are random variables (prediction uncertainty)
- **Choosing the right distribution** for your data determines model quality
- **Loss functions** are derived from distributional assumptions

### How It Works

```
DISCRETE: Probability Mass Function (PMF)
P(X = x) = specific probability for each value

Example: Fair die
P(X=1) = P(X=2) = ... = P(X=6) = 1/6

    P(X)
    1/6 |  ▓  ▓  ▓  ▓  ▓  ▓
        |  ▓  ▓  ▓  ▓  ▓  ▓
        └──1──2──3──4──5──6── X

CONTINUOUS: Probability Density Function (PDF)
P(a ≤ X ≤ b) = ∫[a to b] f(x) dx
NOTE: P(X = exact value) = 0 for continuous variables!

Example: Normal distribution
         ╱╲
        ╱  ╲
       ╱    ╲
      ╱      ╲
    ─╱────────╲─
    μ-2σ  μ  μ+2σ
```

**Key property:** All probabilities must sum (discrete) or integrate (continuous) to 1.

### Code Examples

```python
import numpy as np
from scipy import stats

# === DISCRETE: PMF and CDF ===
# Binomial: number of successes in n trials
# Example: 10 coin flips, P(heads) = 0.6
n, p = 10, 0.6
binom = stats.binom(n, p)

# PMF: probability of getting exactly k heads
for k in range(11):
    prob = binom.pmf(k)
    bar = '█' * int(prob * 50)
    print(f"P(X={k:2d}) = {prob:.4f} {bar}")

# CDF: probability of getting AT MOST k heads
print(f"\nP(X ≤ 5) = {binom.cdf(5):.4f}")    # Cumulative
print(f"P(X > 5) = {1 - binom.cdf(5):.4f}")   # Complement

# === CONTINUOUS: PDF and CDF ===
# Normal distribution: μ=170, σ=10 (heights in cm)
mu, sigma = 170, 10
normal = stats.norm(mu, sigma)

# PDF at a specific point (NOT a probability!)
print(f"\nPDF at 170cm: {normal.pdf(170):.4f}")  # Density, not probability
print(f"PDF at 190cm: {normal.pdf(190):.4f}")

# CDF: probability of being BELOW a value
print(f"P(height < 180) = {normal.cdf(180):.4f}")    # ≈ 0.8413
print(f"P(height > 180) = {1 - normal.cdf(180):.4f}")  # ≈ 0.1587

# Probability of being BETWEEN two values
p_160_to_180 = normal.cdf(180) - normal.cdf(160)
print(f"P(160 < height < 180) = {p_160_to_180:.4f}")  # ≈ 0.6827 (1σ rule)

# === THE 68-95-99.7 RULE ===
for n_sigma in [1, 2, 3]:
    p = normal.cdf(mu + n_sigma*sigma) - normal.cdf(mu - n_sigma*sigma)
    print(f"Within {n_sigma}σ: {p:.4f} ({p*100:.1f}%)")
# 1σ: 68.27%, 2σ: 95.45%, 3σ: 99.73%

# === SAMPLING FROM DISTRIBUTIONS ===
np.random.seed(42)

# Generate samples and verify they match the distribution
samples = np.random.normal(mu, sigma, 10000)
print(f"\nSample mean: {samples.mean():.2f} (expected: {mu})")
print(f"Sample std:  {samples.std():.2f} (expected: {sigma})")

# === QUANTILES (Inverse CDF) ===
# "What height is at the 95th percentile?"
percentile_95 = normal.ppf(0.95)  # ppf = percent point function = inverse CDF
print(f"\n95th percentile height: {percentile_95:.1f} cm")

# "What range contains the middle 90% of data?"
lower = normal.ppf(0.05)
upper = normal.ppf(0.95)
print(f"Middle 90%: [{lower:.1f}, {upper:.1f}] cm")

# === KERNEL DENSITY ESTIMATION (Estimating PDF from data) ===
from scipy.stats import gaussian_kde

# Unknown distribution — estimate from samples
data = np.concatenate([
    np.random.normal(0, 1, 300),
    np.random.normal(5, 0.5, 200)
])  # Bimodal distribution

kde = gaussian_kde(data)
x = np.linspace(-4, 8, 200)
density = kde(x)
print(f"\nKDE peak locations: x = {x[np.argsort(density)[-2:]]}")
# Should find peaks near 0 and 5
```

### Common Mistakes
1. **Treating PDF values as probabilities** — PDF can be > 1! Only the integral over a range is a probability
2. **Forgetting P(X = exact value) = 0 for continuous variables** — Ask P(a < X < b) instead
3. **Assuming all data is normally distributed** — Real data often has heavy tails, skew, or multiple modes

---

## 4. Common Probability Distributions

### What It Is
A toolkit of distributions, each suited for different types of data and modeling scenarios.

### Why It Matters
Choosing the right distribution is critical:
- Wrong distribution → biased model, bad predictions
- Right distribution → proper uncertainty estimates, better generalization

### Distribution Catalog

#### Discrete Distributions

| Distribution | Parameters | Use Case | Example |
|-------------|-----------|----------|---------|
| Bernoulli | p | Single yes/no trial | Click or not |
| Binomial | n, p | Count of successes in n trials | # heads in 10 flips |
| Poisson | λ | Count of events in fixed interval | # website visits/hour |
| Categorical | p₁,...,pₖ | Single draw from k categories | Which word comes next |
| Multinomial | n, p₁,...,pₖ | Counts of k categories in n draws | Word frequencies |
| Geometric | p | # trials until first success | # attempts until sale |

#### Continuous Distributions

| Distribution | Parameters | Use Case | Example |
|-------------|-----------|----------|---------|
| Normal (Gaussian) | μ, σ | Naturally occurring measurements | Heights, errors |
| Uniform | a, b | Equal likelihood over range | Random initialization |
| Exponential | λ | Time between events | Time between failures |
| Beta | α, β | Probabilities/proportions | Click-through rates |
| Gamma | α, β | Positive continuous values | Insurance claims |
| Log-Normal | μ, σ | Multiplicative processes | Stock prices, incomes |
| Student's t | ν | Heavy-tailed data, small samples | Robust regression |
| Chi-squared | k | Sum of squared normals | Goodness-of-fit test |

### Code Examples

```python
import numpy as np
from scipy import stats

# === BERNOULLI — The Simplest Distribution ===
# Single coin flip: P(success) = p
p = 0.7
samples = np.random.binomial(1, p, 10000)
print(f"Bernoulli(p={p}): mean = {samples.mean():.4f} (expected: {p})")

# In ML: Binary classification output is Bernoulli
# sigmoid(logit) = P(y=1|x)

# === BINOMIAL — Counting Successes ===
# n trials, each with probability p
n, p = 20, 0.3
binom = stats.binom(n, p)
print(f"\nBinomial(n={n}, p={p}):")
print(f"  Mean: {binom.mean():.1f} (n×p = {n*p})")
print(f"  Std: {binom.std():.2f}")
print(f"  P(X=6) = {binom.pmf(6):.4f}")
print(f"  P(X≤6) = {binom.cdf(6):.4f}")

# === POISSON — Counting Rare Events ===
# Number of events per interval, when events are independent
# λ = average rate
lam = 5  # 5 events per hour on average
poisson = stats.poisson(lam)

print(f"\nPoisson(λ={lam}):")
for k in range(12):
    p = poisson.pmf(k)
    bar = '█' * int(p * 40)
    print(f"  P(X={k:2d}) = {p:.4f} {bar}")

# In ML: Count data (word frequencies, page views, defects)
# When λ is large, Poisson ≈ Normal(μ=λ, σ²=λ)

# === NORMAL (GAUSSIAN) — The King of Distributions ===
# Central Limit Theorem: sum of many independent variables → Normal
mu, sigma = 0, 1
normal = stats.norm(mu, sigma)

# Why it's everywhere: CLT demonstration
sample_means = []
for _ in range(10000):
    # Average of 30 uniform random numbers
    sample = np.random.uniform(0, 1, 30)
    sample_means.append(sample.mean())

sample_means = np.array(sample_means)
print(f"\nCLT Demo (avg of 30 uniform samples):")
print(f"  Mean: {sample_means.mean():.4f} (expected: 0.5)")
print(f"  Std: {sample_means.std():.4f} (expected: {1/np.sqrt(12*30):.4f})")
# The distribution is approximately Normal regardless of the original distribution!

# === EXPONENTIAL — Time Between Events ===
# If events happen at rate λ, time between events ~ Exp(λ)
lam = 2  # 2 events per minute
exponential = stats.expon(scale=1/lam)

print(f"\nExponential(λ={lam}):")
print(f"  Mean time between events: {exponential.mean():.2f} min")
print(f"  P(wait > 1 min) = {1 - exponential.cdf(1):.4f}")

# Memoryless property: P(X > s+t | X > s) = P(X > t)
# "The probability of waiting another t minutes doesn't depend on how long you've already waited"

# === BETA — Distribution of Probabilities ===
# Perfect for modeling probabilities/proportions (values between 0 and 1)
# Conjugate prior for Bernoulli/Binomial

print(f"\nBeta Distribution Examples:")
params = [(1, 1), (2, 5), (5, 2), (10, 10), (0.5, 0.5)]
for alpha, beta_param in params:
    dist = stats.beta(alpha, beta_param)
    print(f"  Beta({alpha},{beta_param}): mean={dist.mean():.3f}, "
          f"mode={(alpha-1)/(alpha+beta_param-2) if alpha>1 and beta_param>1 else 'N/A'}")

# In ML: 
# - Prior for probability parameters
# - Thompson Sampling for A/B testing
# - Beta(1,1) = Uniform (no prior knowledge)

# === MULTIVARIATE NORMAL — The ML Workhorse ===
# Multiple correlated features
mean = [0, 0]
cov = [[1, 0.8],    # Feature 1 variance=1, correlation=0.8
       [0.8, 1]]     # Feature 2 variance=1

mvn = stats.multivariate_normal(mean, cov)
samples = mvn.rvs(1000)
print(f"\nMultivariate Normal:")
print(f"  Sample mean: {samples.mean(axis=0)}")
print(f"  Sample covariance:\n{np.cov(samples.T)}")

# In ML: Gaussian Mixture Models, Gaussian Processes, 
# Variational Autoencoders (latent space)

# === CHOOSING THE RIGHT DISTRIBUTION ===
print("\n--- Distribution Selection Guide ---")
guide = {
    "Binary outcome (yes/no)": "Bernoulli",
    "Count of successes (fixed trials)": "Binomial",
    "Count of events (unbounded)": "Poisson",
    "Time until event": "Exponential",
    "Measurement with noise": "Normal",
    "Probability/proportion": "Beta",
    "Positive continuous, right-skewed": "Log-Normal or Gamma",
    "Heavy tails, small sample": "Student's t",
    "Multiple categories": "Categorical/Multinomial",
}
for scenario, dist in guide.items():
    print(f"  {scenario:45s} → {dist}")
```

### Common Mistakes
1. **Using Normal when data is bounded** — Heights can't be negative; use truncated Normal or Log-Normal
2. **Using Poisson for dependent events** — Poisson assumes independence between events
3. **Forgetting the Central Limit Theorem conditions** — CLT needs independent samples and finite variance
4. **Not checking distribution fit** — Always visualize data and run goodness-of-fit tests

---

## 5. Expectation, Variance, and Moments

### What It Is
Summary statistics that capture the "shape" of a distribution without listing every probability.

- **Expectation (Mean)** — the "center" or average value
- **Variance** — how spread out the values are
- **Higher moments** — skewness (asymmetry) and kurtosis (tail heaviness)

### Why It Matters
- **Bias-Variance tradeoff** — THE central concept in ML
- **Feature normalization** — requires mean and variance
- **Loss functions** — often derived from expectations
- **Batch Normalization** — normalizes using running mean and variance
- **Risk assessment** — variance = uncertainty in predictions

### How It Works

$$E[X] = \sum_x x \cdot P(X=x) \quad \text{(discrete)}$$
$$E[X] = \int_{-\infty}^{\infty} x \cdot f(x) \, dx \quad \text{(continuous)}$$

$$\text{Var}(X) = E[(X - E[X])^2] = E[X^2] - (E[X])^2$$

```
Low Variance (tight):        High Variance (spread):
       ╱╲                         ╱──────╲
      ╱  ╲                       ╱        ╲
     ╱    ╲                     ╱          ╲
    ╱      ╲                   ╱            ╲
───╱────────╲───           ───╱──────────────╲───
Predictions are              Predictions are
consistent but               diverse but
might be biased               might be noisy
```

### Key Properties

| Property | Formula | Why It Matters |
|----------|---------|---------------|
| Linearity of E | $E[aX + b] = aE[X] + b$ | Can simplify complex expectations |
| Variance scaling | $\text{Var}(aX) = a^2\text{Var}(X)$ | Explains why feature scaling matters |
| Sum of independent | $\text{Var}(X+Y) = \text{Var}(X) + \text{Var}(Y)$ | Foundation of CLT |
| Covariance | $\text{Cov}(X,Y) = E[XY] - E[X]E[Y]$ | Measures linear relationship |
| Correlation | $\rho = \frac{\text{Cov}(X,Y)}{\sigma_X \sigma_Y}$ | Normalized covariance ∈ [-1, 1] |

### Code Examples

```python
import numpy as np
from scipy import stats

# === EXPECTATION AND VARIANCE ===
# Example: Unfair die (loaded toward 6)
values = np.array([1, 2, 3, 4, 5, 6])
probs = np.array([0.1, 0.1, 0.1, 0.1, 0.1, 0.5])  # 50% chance of 6

expected_value = np.sum(values * probs)
variance = np.sum((values - expected_value)**2 * probs)
std_dev = np.sqrt(variance)

print(f"E[X] = {expected_value:.2f}")     # 4.50
print(f"Var(X) = {variance:.2f}")          # 3.25
print(f"Std(X) = {std_dev:.2f}")           # 1.80

# Verify with simulation
np.random.seed(42)
samples = np.random.choice(values, p=probs, size=100000)
print(f"Sample mean: {samples.mean():.4f}")
print(f"Sample var: {samples.var():.4f}")

# === BIAS-VARIANCE TRADEOFF ===
# The most important decomposition in ML:
# E[(y - ŷ)²] = Bias² + Variance + Irreducible Noise

np.random.seed(42)

# True function
f_true = lambda x: np.sin(x)

# Generate data
x_train = np.linspace(0, 2*np.pi, 20)
noise = 0.3

def fit_and_predict(degree, x_train, x_test, n_experiments=100):
    """Fit polynomial and track bias/variance"""
    predictions = []
    for _ in range(n_experiments):
        y_train = f_true(x_train) + np.random.randn(len(x_train)) * noise
        coeffs = np.polyfit(x_train, y_train, degree)
        y_pred = np.polyval(coeffs, x_test)
        predictions.append(y_pred)
    
    predictions = np.array(predictions)
    avg_prediction = predictions.mean(axis=0)
    
    bias_squared = ((avg_prediction - f_true(x_test))**2).mean()
    variance = predictions.var(axis=0).mean()
    
    return bias_squared, variance

x_test = np.linspace(0, 2*np.pi, 50)
print("\n--- Bias-Variance Tradeoff ---")
print(f"{'Degree':>6} {'Bias²':>10} {'Variance':>10} {'Total':>10}")
print("-" * 40)
for degree in [1, 3, 5, 10, 15]:
    bias2, var = fit_and_predict(degree, x_train, x_test)
    print(f"{degree:6d} {bias2:10.4f} {var:10.4f} {bias2+var:10.4f}")

# Low degree: high bias (underfitting), low variance
# High degree: low bias, high variance (overfitting)
# Sweet spot: somewhere in between

# === COVARIANCE AND CORRELATION ===
np.random.seed(42)
n = 1000

# Positively correlated features (e.g., height and weight)
height = np.random.normal(170, 10, n)
weight = 0.8 * height + np.random.normal(0, 10, n)  # Correlated

# Independent features (e.g., height and IQ)
iq = np.random.normal(100, 15, n)  # Independent of height

# Compute covariance and correlation
cov_hw = np.cov(height, weight)[0, 1]
corr_hw = np.corrcoef(height, weight)[0, 1]
corr_hi = np.corrcoef(height, iq)[0, 1]

print(f"\nHeight-Weight covariance: {cov_hw:.2f}")
print(f"Height-Weight correlation: {corr_hw:.4f}")    # ≈ 0.63 (strong positive)
print(f"Height-IQ correlation: {corr_hi:.4f}")          # ≈ 0.0 (independent)

# === COVARIANCE MATRIX ===
# Foundation of PCA!
data = np.column_stack([height, weight, iq])
cov_matrix = np.cov(data.T)
corr_matrix = np.corrcoef(data.T)

print(f"\nCorrelation Matrix:")
labels = ['Height', 'Weight', 'IQ']
print(f"{'':>8}", end='')
for l in labels:
    print(f"{l:>8}", end='')
print()
for i, l in enumerate(labels):
    print(f"{l:>8}", end='')
    for j in range(3):
        print(f"{corr_matrix[i,j]:8.3f}", end='')
    print()

# === LAW OF TOTAL EXPECTATION ===
# E[X] = E[E[X|Y]]  (average of conditional averages)
# Example: Overall average height = average of (avg male height, avg female height)
# weighted by proportion of males and females

# Male heights ~ N(175, 7²), Female heights ~ N(162, 6²), 50/50 split
male_heights = np.random.normal(175, 7, 5000)
female_heights = np.random.normal(162, 6, 5000)
all_heights = np.concatenate([male_heights, female_heights])

print(f"\nE[Height|Male] = {male_heights.mean():.1f}")
print(f"E[Height|Female] = {female_heights.mean():.1f}")
print(f"E[Height] = {all_heights.mean():.1f}")
print(f"0.5 × E[H|M] + 0.5 × E[H|F] = {0.5*male_heights.mean() + 0.5*female_heights.mean():.1f}")

# === HIGHER MOMENTS ===
data_normal = np.random.normal(0, 1, 10000)
data_skewed = np.random.exponential(1, 10000)

print(f"\nNormal: skewness={stats.skew(data_normal):.3f}, kurtosis={stats.kurtosis(data_normal):.3f}")
print(f"Exponential: skewness={stats.skew(data_skewed):.3f}, kurtosis={stats.kurtosis(data_skewed):.3f}")
# Skewness: 0 = symmetric, >0 = right-skewed, <0 = left-skewed
# Kurtosis: 0 = Normal-like tails, >0 = heavy tails, <0 = light tails
```

### Common Mistakes
1. **Using sample variance with n instead of n-1** — `np.var()` uses n by default; for unbiased estimate use `np.var(ddof=1)` or `np.std(ddof=1)`
2. **Confusing correlation with causation** — Correlated features may share a common cause
3. **Assuming zero correlation means independence** — Only true for Normal distributions. Variables can be dependent but uncorrelated ($X$ and $X^2$)

---

## 6. Joint, Marginal, and Conditional Distributions

### What It Is
When you have **multiple random variables**, you need to describe their combined behavior.

- **Joint** $P(X, Y)$: probability of X AND Y together
- **Marginal** $P(X)$: probability of X alone (summing out Y)
- **Conditional** $P(X|Y)$: probability of X given a specific Y

### Why It Matters
- **Bayesian networks** model joint distributions via conditionals
- **Marginalization** is used in latent variable models (VAE, EM algorithm)
- **Conditional distributions** are what classifiers learn: $P(Y|X)$

### How It Works

```
JOINT DISTRIBUTION P(X, Y):

           Y=sunny  Y=rainy
X=hot       0.30     0.05    → P(X=hot) = 0.35   (marginal)
X=mild      0.20     0.15    → P(X=mild) = 0.35
X=cold      0.05     0.25    → P(X=cold) = 0.30
             ↓        ↓
           0.55      0.45    ← marginals P(Y)

MARGINAL: Sum a row or column
P(X=hot) = P(hot,sunny) + P(hot,rainy) = 0.30 + 0.05 = 0.35

CONDITIONAL: Slice and normalize
P(Y=rainy | X=cold) = P(cold,rainy) / P(cold) = 0.25/0.30 = 0.833
```

### Code Examples

```python
import numpy as np

# === JOINT DISTRIBUTION TABLE ===
# Weather (X) vs Outdoor Activity (Y)
joint = np.array([
    [0.30, 0.05],   # hot: [sunny, rainy]
    [0.20, 0.15],   # mild: [sunny, rainy]
    [0.05, 0.25],   # cold: [sunny, rainy]
])

x_labels = ['hot', 'mild', 'cold']
y_labels = ['sunny', 'rainy']

print("Joint Distribution P(X, Y):")
print(f"{'':>8}", end='')
for y in y_labels:
    print(f"{y:>10}", end='')
print(f"{'P(X)':>10}")
for i, x in enumerate(x_labels):
    print(f"{x:>8}", end='')
    for j in range(len(y_labels)):
        print(f"{joint[i,j]:10.2f}", end='')
    print(f"{joint[i,:].sum():10.2f}")
print(f"{'P(Y)':>8}", end='')
for j in range(len(y_labels)):
    print(f"{joint[:,j].sum():10.2f}", end='')
print(f"{joint.sum():10.2f}")

# === MARGINAL DISTRIBUTIONS ===
P_X = joint.sum(axis=1)  # Sum over Y (columns)
P_Y = joint.sum(axis=0)  # Sum over X (rows)
print(f"\nMarginal P(X): {dict(zip(x_labels, P_X))}")
print(f"Marginal P(Y): {dict(zip(y_labels, P_Y))}")

# === CONDITIONAL DISTRIBUTIONS ===
# P(Y|X=cold) = P(X=cold, Y) / P(X=cold)
P_Y_given_cold = joint[2, :] / P_X[2]
print(f"\nP(Y | X=cold): {dict(zip(y_labels, np.round(P_Y_given_cold, 3)))}")
# P(rainy|cold) = 0.833 — when it's cold, it's usually rainy

# P(X|Y=sunny) = P(X, Y=sunny) / P(Y=sunny)
P_X_given_sunny = joint[:, 0] / P_Y[0]
print(f"P(X | Y=sunny): {dict(zip(x_labels, np.round(P_X_given_sunny, 3)))}")

# === INDEPENDENCE CHECK ===
# X and Y are independent iff P(X,Y) = P(X)P(Y) for all x,y
independent = True
for i in range(len(x_labels)):
    for j in range(len(y_labels)):
        product = P_X[i] * P_Y[j]
        if not np.isclose(joint[i,j], product, atol=0.01):
            independent = False
            print(f"P({x_labels[i]},{y_labels[j]})={joint[i,j]:.2f} ≠ "
                  f"P({x_labels[i]})×P({y_labels[j]})={product:.3f}")

print(f"Independent? {independent}")  # False

# === MARGINALIZATION IN ML ===
# Latent variable models: P(x) = Σ_z P(x|z)P(z)
# This is how VAEs and Gaussian Mixture Models work!

# GMM: P(x) = Σ_k π_k × N(x|μ_k, σ_k)
from scipy.stats import norm

def gmm_pdf(x, means, stds, weights):
    """Gaussian Mixture Model PDF via marginalization"""
    pdf = 0
    for mu, sigma, pi in zip(means, stds, weights):
        pdf += pi * norm.pdf(x, mu, sigma)  # Sum over latent variable (cluster)
    return pdf

x = np.linspace(-5, 10, 200)
means = [0, 5]       # Two cluster centers
stds = [1, 0.5]      # Cluster spreads
weights = [0.6, 0.4]  # Mixing proportions

density = gmm_pdf(x, means, stds, weights)
peak_idx = np.argsort(density)[-2:]
print(f"\nGMM peaks at x ≈ {x[peak_idx]}")

# === CHAIN RULE OF PROBABILITY ===
# P(A, B, C) = P(A) × P(B|A) × P(C|A,B)
# This is how autoregressive models (GPT) generate text:
# P(word₁, word₂, ..., wordₙ) = P(word₁) × P(word₂|word₁) × P(word₃|word₁,word₂) × ...
print("\nChain rule in language models:")
print("P('the cat sat') = P('the') × P('cat'|'the') × P('sat'|'the cat')")
```

### Common Mistakes
1. **Forgetting to normalize conditionals** — After slicing a joint distribution, probabilities must sum to 1
2. **Confusing joint with conditional** — P(X,Y) is a 2D table; P(X|Y=y) is a 1D distribution for each y
3. **Not understanding marginalization** — "Summing out" a variable is how you handle latent/hidden variables

---

## 7. Maximum Likelihood Estimation (MLE)

### What It Is
MLE finds the parameter values that make the **observed data most probable**. It asks: "Given this data, what parameters would have been most likely to produce it?"

$$\hat{\theta}_{MLE} = \arg\max_\theta P(\text{data} | \theta) = \arg\max_\theta \prod_{i=1}^n P(x_i | \theta)$$

In practice, we maximize the **log-likelihood** (products → sums):

$$\hat{\theta}_{MLE} = \arg\max_\theta \sum_{i=1}^n \log P(x_i | \theta)$$

### Why It Matters
- **Linear regression** with MSE = MLE assuming Gaussian noise
- **Logistic regression** with cross-entropy = MLE assuming Bernoulli
- **Neural network training** with cross-entropy loss = MLE
- It's the theoretical justification for most loss functions

### How It Works

**Analogy:** You find a loaded die and roll it 100 times: 50 sixes, 10 of each other number. MLE asks: "What P(six) would make these results most likely?" Answer: P(six) = 50/100 = 0.5.

```
MLE Process:

1. Assume a distribution family (e.g., Gaussian)
2. Write the likelihood: P(data | μ, σ)
3. Take the log (easier math)
4. Differentiate and set to zero
5. Solve for parameters

For Gaussian:
    log L = -n/2 log(2π) - n/2 log(σ²) - Σ(xᵢ - μ)²/(2σ²)
    
    ∂/∂μ → μ̂ = (1/n)Σxᵢ = sample mean  ← Makes intuitive sense!
    ∂/∂σ² → σ̂² = (1/n)Σ(xᵢ - μ̂)² = sample variance
```

### Code Examples

```python
import numpy as np
from scipy import stats
from scipy.optimize import minimize

# === MLE FOR NORMAL DISTRIBUTION ===
# Ground truth: N(5, 2²)
np.random.seed(42)
true_mu, true_sigma = 5.0, 2.0
data = np.random.normal(true_mu, true_sigma, 100)

# Analytical MLE
mu_mle = data.mean()
sigma_mle = data.std()  # Note: MLE uses n, not n-1
print(f"True: μ={true_mu}, σ={true_sigma}")
print(f"MLE:  μ={mu_mle:.4f}, σ={sigma_mle:.4f}")

# === MLE VIA OPTIMIZATION ===
def neg_log_likelihood_normal(params, data):
    """Negative log-likelihood for Normal distribution"""
    mu, log_sigma = params  # Use log_sigma to ensure σ > 0
    sigma = np.exp(log_sigma)
    n = len(data)
    
    # -log L = n/2 log(2π) + n*log(σ) + Σ(x-μ)²/(2σ²)
    nll = n/2 * np.log(2*np.pi) + n * np.log(sigma) + \
          np.sum((data - mu)**2) / (2 * sigma**2)
    return nll

# Optimize
result = minimize(neg_log_likelihood_normal, x0=[0, 0], args=(data,))
mu_opt, log_sigma_opt = result.x
print(f"\nOptimized MLE: μ={mu_opt:.4f}, σ={np.exp(log_sigma_opt):.4f}")

# === MLE FOR LINEAR REGRESSION ===
# y = Xw + ε, where ε ~ N(0, σ²)
# MLE for w is equivalent to minimizing MSE!

np.random.seed(42)
n_samples = 200
X = np.column_stack([np.ones(n_samples), np.random.randn(n_samples, 2)])
w_true = np.array([1.0, 3.0, -2.0])
y = X @ w_true + np.random.randn(n_samples) * 0.5

def neg_log_likelihood_linreg(params, X, y):
    """NLL for linear regression = scaled MSE"""
    w = params[:-1]
    log_sigma = params[-1]
    sigma = np.exp(log_sigma)
    n = len(y)
    
    residuals = y - X @ w
    nll = n/2 * np.log(2*np.pi) + n * np.log(sigma) + \
          np.sum(residuals**2) / (2 * sigma**2)
    return nll

# MLE solution
x0 = np.zeros(X.shape[1] + 1)  # w's + log_sigma
result = minimize(neg_log_likelihood_linreg, x0, args=(X, y))
w_mle = result.x[:-1]
sigma_mle = np.exp(result.x[-1])

print(f"\nLinear Regression MLE:")
print(f"True w: {w_true}, σ: 0.5")
print(f"MLE w:  {w_mle}, σ: {sigma_mle:.4f}")

# Compare with closed-form (Normal equation)
w_normal = np.linalg.lstsq(X, y, rcond=None)[0]
print(f"Normal eq: {w_normal}")
# They should match!

# === MLE FOR LOGISTIC REGRESSION ===
# P(y=1|x) = sigmoid(wᵀx)
# NLL = -Σ[yᵢ log(p) + (1-yᵢ) log(1-p)]  ← This IS cross-entropy!

def sigmoid(z):
    return 1 / (1 + np.exp(-np.clip(z, -500, 500)))

def neg_log_likelihood_logistic(w, X, y):
    """NLL for logistic regression = cross-entropy loss"""
    p = sigmoid(X @ w)
    # Clip to avoid log(0)
    p = np.clip(p, 1e-10, 1 - 1e-10)
    nll = -np.sum(y * np.log(p) + (1 - y) * np.log(1 - p))
    return nll

# Generate binary classification data
from sklearn.datasets import make_classification
X_cls, y_cls = make_classification(n_samples=300, n_features=3, 
                                    n_redundant=0, random_state=42)

# Add bias column
X_cls_bias = np.column_stack([np.ones(len(X_cls)), X_cls])

result = minimize(neg_log_likelihood_logistic, np.zeros(4), args=(X_cls_bias, y_cls))
w_logistic = result.x

# Verify against sklearn
from sklearn.linear_model import LogisticRegression
lr = LogisticRegression(fit_intercept=False, C=1e10)  # No regularization
lr.fit(X_cls_bias, y_cls)
print(f"\nLogistic Regression:")
print(f"Our MLE:  {w_logistic}")
print(f"Sklearn:  {lr.coef_[0]}")

# === WHY LOG-LIKELIHOOD? ===
# 1. Products → sums (numerically stable, gradient is sum not product)
# 2. log is monotonic → same argmax
# 3. For exponential family: log-likelihood is concave → guaranteed global max

print("\n--- Key Insight ---")
print("MSE Loss = MLE with Gaussian noise assumption")
print("Cross-Entropy Loss = MLE with Bernoulli/Categorical assumption")
print("This is why these are 'the right' loss functions!")
```

### Common Mistakes
1. **MLE can overfit** — With enough parameters, MLE can perfectly fit noise. Use regularization (= MAP estimation)
2. **Assuming Gaussian noise when it's not** — MLE with wrong distribution family gives biased estimates
3. **Numerical issues with products** — Always use log-likelihood (products of small numbers → underflow)
4. **MLE variance estimate is biased** — MLE uses $n$ in denominator; unbiased estimate uses $n-1$

---

## 8. Maximum A Posteriori (MAP)

### What It Is
MAP adds a **prior belief** about parameters on top of MLE. Instead of just maximizing likelihood, it maximizes the **posterior probability**.

$$\hat{\theta}_{MAP} = \arg\max_\theta P(\theta | \text{data}) = \arg\max_\theta P(\text{data} | \theta) \cdot P(\theta)$$

### Why It Matters
- **MAP with Gaussian prior = L2 regularization (Ridge)**
- **MAP with Laplace prior = L1 regularization (Lasso)**
- Prevents overfitting by encoding "simpler models are better"
- Bridges MLE and full Bayesian inference

### How It Works

```
MLE:  Find θ that maximizes P(data | θ)
      → "What parameters best explain the data?"
      
MAP:  Find θ that maximizes P(data | θ) × P(θ)
      → "What parameters best explain the data AND are plausible?"

Full Bayesian: Compute entire P(θ | data)
      → "What's the full distribution of plausible parameters?"

MLE ⊂ MAP ⊂ Full Bayesian
(increasing complexity, decreasing risk of overfitting)
```

### Code Examples

```python
import numpy as np
from scipy.optimize import minimize

# === MAP = MLE + REGULARIZATION ===
np.random.seed(42)

# Linear regression with more features than needed
n_samples = 50
n_features = 20
X = np.random.randn(n_samples, n_features)
w_true = np.zeros(n_features)
w_true[:5] = [3, -2, 1, 0.5, -1]  # Only 5 features matter
y = X @ w_true + np.random.randn(n_samples) * 0.5

# --- MLE (no prior = no regularization) ---
def nll_linear(w, X, y):
    return np.sum((y - X @ w)**2)

result_mle = minimize(nll_linear, np.zeros(n_features), args=(X, y))
w_mle = result_mle.x

# --- MAP with Gaussian prior (= Ridge/L2) ---
def neg_log_posterior_l2(w, X, y, lambda_reg):
    """NLL + L2 penalty = MAP with Gaussian prior N(0, 1/λ)"""
    nll = np.sum((y - X @ w)**2)
    prior = lambda_reg * np.sum(w**2)  # -log P(w) for Gaussian prior
    return nll + prior

result_map_l2 = minimize(neg_log_posterior_l2, np.zeros(n_features), 
                          args=(X, y, 1.0))
w_map_l2 = result_map_l2.x

# --- MAP with Laplace prior (= Lasso/L1) ---
def neg_log_posterior_l1(w, X, y, lambda_reg):
    """NLL + L1 penalty = MAP with Laplace prior"""
    nll = np.sum((y - X @ w)**2)
    prior = lambda_reg * np.sum(np.abs(w))  # -log P(w) for Laplace prior
    return nll + prior

result_map_l1 = minimize(neg_log_posterior_l1, np.zeros(n_features), 
                          args=(X, y, 1.0), method='L-BFGS-B')
w_map_l1 = result_map_l1.x

# Compare
print("True weights:    ", np.round(w_true, 2))
print("MLE (no prior):  ", np.round(w_mle, 2))
print("MAP L2 (Ridge):  ", np.round(w_map_l2, 2))
print("MAP L1 (Lasso):  ", np.round(w_map_l1, 2))

print(f"\nMSE from true weights:")
print(f"MLE:      {np.sum((w_mle - w_true)**2):.4f}")
print(f"MAP L2:   {np.sum((w_map_l2 - w_true)**2):.4f}")
print(f"MAP L1:   {np.sum((w_map_l1 - w_true)**2):.4f}")

# Non-zero weights (sparsity)
print(f"\nNon-zero weights (|w| > 0.1):")
print(f"MLE:    {np.sum(np.abs(w_mle) > 0.1)} / {n_features}")
print(f"MAP L2: {np.sum(np.abs(w_map_l2) > 0.1)} / {n_features}")
print(f"MAP L1: {np.sum(np.abs(w_map_l1) > 0.1)} / {n_features}")

# === PRIOR → REGULARIZATION MAPPING ===
print("\n--- Prior to Regularization ---")
print("Gaussian prior  N(0, σ²)   →  L2 penalty (Ridge),  λ = 1/(2σ²)")
print("Laplace prior   Laplace(0,b) → L1 penalty (Lasso), λ = 1/b")
print("Uniform prior   (flat)      →  No regularization (MLE)")
print("Spike-and-slab             →  L0 penalty (subset selection)")
```

> **Key Insight:** Every regularization technique in ML corresponds to a Bayesian prior. L2 regularization assumes weights are normally distributed around zero. L1 assumes they follow a Laplace distribution (which favors exactly-zero weights → sparsity).

### Common Mistakes
1. **MAP gives a point estimate, not full uncertainty** — For full uncertainty, you need full Bayesian inference
2. **Forgetting that MAP = MLE when prior is uniform** — Without regularization, MAP reduces to MLE
3. **Using the wrong prior/regularization** — L1 for sparsity, L2 for small weights, Elastic Net for both

---

## 9. Bayesian Thinking

### What It Is
Full Bayesian inference computes the **entire posterior distribution** $P(\theta|\text{data})$, not just a single point estimate. This gives you uncertainty quantification for free.

$$P(\theta | \mathcal{D}) = \frac{P(\mathcal{D} | \theta) \cdot P(\theta)}{P(\mathcal{D})} = \frac{P(\mathcal{D} | \theta) \cdot P(\theta)}{\int P(\mathcal{D} | \theta) P(\theta) d\theta}$$

### Why It Matters
- Know WHEN to trust your model (uncertainty estimates)
- Small data? Bayesian methods incorporate prior knowledge
- Automatic Occam's razor (naturally prefers simpler models)
- Active learning: sample where uncertainty is highest
- Safety-critical applications (medical, autonomous driving)

### How It Works

```
Frequentist vs Bayesian:

Frequentist:  "The parameter θ IS 5.3"  (point estimate)
              → Confidence interval: [4.8, 5.8] 
              (frequentist interpretation is confusing)

Bayesian:     "θ is probably between 4.5 and 6.0,
               most likely around 5.3"  (distribution)
              → Credible interval: [4.5, 6.0]
              (intuitive: 95% probability θ is in this range)
```

### Code Examples

```python
import numpy as np
from scipy import stats

# === BAYESIAN COIN FLIPPING ===
# Is the coin fair? Track our belief as we see more data

# Prior: Beta(1, 1) = Uniform (no initial bias)
alpha_prior = 1
beta_prior = 1

# Observed data: flip by flip
flips = [1, 1, 0, 1, 1, 1, 0, 1, 0, 1, 1, 1, 0, 1, 1, 
         1, 0, 1, 1, 1]  # 1=heads

print("Bayesian Updating (Coin Bias Estimation):")
print(f"{'Flips':>6} {'Heads':>6} {'Tails':>6} {'Mean':>8} {'95% CI':>20}")
print("-" * 55)

alpha, beta_p = alpha_prior, beta_prior
for i, flip in enumerate(flips, 1):
    alpha += flip       # Add heads
    beta_p += 1 - flip  # Add tails
    
    posterior = stats.beta(alpha, beta_p)
    mean = posterior.mean()
    ci_low, ci_high = posterior.ppf(0.025), posterior.ppf(0.975)
    
    if i in [1, 2, 5, 10, 20]:
        print(f"{i:6d} {alpha-1:6d} {beta_p-1:6d} {mean:8.4f} [{ci_low:.4f}, {ci_high:.4f}]")

# Notice: CI gets narrower with more data (more confident)
# With enough data, Bayesian → Frequentist (Bernstein-von Mises theorem)

# === BAYESIAN LINEAR REGRESSION ===
def bayesian_linear_regression(X, y, alpha=1.0, beta=25.0):
    """
    Bayesian linear regression with known noise variance.
    Prior: w ~ N(0, α⁻¹ I)
    Likelihood: y|X,w ~ N(Xw, β⁻¹ I)
    Posterior: w|X,y ~ N(m_N, S_N)
    
    alpha = prior precision (1/prior_variance)
    beta = noise precision (1/noise_variance)
    """
    n_features = X.shape[1]
    
    # Prior
    S_0_inv = alpha * np.eye(n_features)  # Prior precision
    m_0 = np.zeros(n_features)             # Prior mean
    
    # Posterior
    S_N_inv = S_0_inv + beta * X.T @ X               # Posterior precision
    S_N = np.linalg.inv(S_N_inv)                       # Posterior covariance
    m_N = S_N @ (S_0_inv @ m_0 + beta * X.T @ y)      # Posterior mean
    
    return m_N, S_N

np.random.seed(42)

# Generate data
n = 20
x = np.linspace(0, 1, n)
X = np.column_stack([np.ones(n), x])  # [1, x] for intercept + slope
w_true = np.array([0.5, 2.0])  # y = 0.5 + 2x
y = X @ w_true + np.random.randn(n) * 0.2

# Fit Bayesian regression
m_N, S_N = bayesian_linear_regression(X, y, alpha=1.0, beta=25.0)

print(f"\nBayesian Linear Regression:")
print(f"True weights: {w_true}")
print(f"Posterior mean: {m_N}")
print(f"Posterior std: {np.sqrt(np.diag(S_N))}")

# Prediction with uncertainty
x_test = np.array([0.5])  # Predict at x=0.5
X_test = np.array([[1, 0.5]])
y_pred = X_test @ m_N
y_uncertainty = np.sqrt(X_test @ S_N @ X_test.T + 1/25.0)  # Predictive variance

print(f"\nPrediction at x=0.5:")
print(f"Mean: {y_pred[0]:.4f}")
print(f"95% interval: [{y_pred[0]-2*y_uncertainty[0,0]:.4f}, {y_pred[0]+2*y_uncertainty[0,0]:.4f}]")

# === BAYESIAN MODEL COMPARISON ===
# Compute marginal likelihood P(data|model) to compare models
# Higher = better model (automatically penalizes complexity!)

print("\n--- Bayesian vs Frequentist Summary ---")
comparison = {
    'Parameters': ('Fixed but unknown', 'Random variables'),
    'Result': ('Point estimate + CI', 'Full posterior distribution'),
    'Uncertainty': ('Requires bootstrap/etc.', 'Built-in'),
    'Small data': ('Often overfits', 'Prior prevents overfitting'),
    'Computation': ('Usually fast', 'Can be expensive (MCMC)'),
    'Regularization': ('Ad hoc (cross-val λ)', 'Principled (prior choice)'),
}
print(f"{'Aspect':>15} {'Frequentist':>30} {'Bayesian':>30}")
for aspect, (freq, bayes) in comparison.items():
    print(f"{aspect:>15} {freq:>30} {bayes:>30}")
```

### Common Mistakes
1. **Choosing priors carelessly** — Informative priors should be based on domain knowledge, not convenience
2. **Thinking Bayesian = slow** — Conjugate priors give closed-form solutions. Only complex models need MCMC.
3. **Confusing credible intervals with confidence intervals** — Bayesian credible intervals have the intuitive interpretation that most people incorrectly attribute to confidence intervals

---

## 10. Sampling and Monte Carlo Methods

### What It Is
When you can't compute an integral or expectation analytically, you **approximate it using random samples**.

$$E[f(X)] = \int f(x) p(x) dx \approx \frac{1}{N} \sum_{i=1}^N f(x_i), \quad x_i \sim p(x)$$

### Why It Matters
- **MCMC** powers Bayesian inference when posteriors are intractable
- **Monte Carlo dropout** gives uncertainty estimates in neural networks
- **Variational inference** (VAEs) uses sampling for generative models
- **Reinforcement learning** uses Monte Carlo estimates of returns

### Code Examples

```python
import numpy as np

# === MONTE CARLO ESTIMATION ===
# Estimate π using random sampling
np.random.seed(42)
n = 1_000_000

# Sample random points in [0,1] × [0,1]
x = np.random.uniform(0, 1, n)
y = np.random.uniform(0, 1, n)

# Check if inside unit circle (quarter)
inside_circle = (x**2 + y**2) <= 1
pi_estimate = 4 * inside_circle.mean()
print(f"π ≈ {pi_estimate:.6f} (true: {np.pi:.6f})")

# === MONTE CARLO INTEGRATION ===
# Estimate ∫₀¹ sin(πx) dx = 2/π ≈ 0.6366
samples = np.random.uniform(0, 1, 100000)
integral_estimate = np.mean(np.sin(np.pi * samples))  # Average of f(x)
print(f"\n∫sin(πx)dx ≈ {integral_estimate:.4f} (true: {2/np.pi:.4f})")

# === REJECTION SAMPLING ===
def rejection_sampling(target_pdf, proposal_pdf, proposal_sampler, M, n_samples):
    """
    Sample from target distribution using a proposal distribution.
    M = upper bound on target_pdf / proposal_pdf
    """
    samples = []
    total_proposals = 0
    
    while len(samples) < n_samples:
        x = proposal_sampler()        # Sample from proposal
        u = np.random.uniform()       # Uniform [0,1]
        total_proposals += 1
        
        acceptance_prob = target_pdf(x) / (M * proposal_pdf(x))
        if u < acceptance_prob:
            samples.append(x)
    
    acceptance_rate = n_samples / total_proposals
    return np.array(samples), acceptance_rate

# Target: Beta(2, 5), Proposal: Uniform(0, 1)
from scipy.stats import beta as beta_dist

target = lambda x: beta_dist.pdf(x, 2, 5)
proposal = lambda x: 1.0  # Uniform PDF
proposal_sampler = lambda: np.random.uniform()
M = 3.0  # Upper bound on target PDF

samples, acc_rate = rejection_sampling(target, proposal, proposal_sampler, M, 5000)
print(f"\nRejection Sampling:")
print(f"Acceptance rate: {acc_rate:.2%}")
print(f"Sample mean: {samples.mean():.4f} (true: {2/7:.4f})")

# === MARKOV CHAIN MONTE CARLO (Metropolis-Hastings) ===
def metropolis_hastings(target_log_pdf, initial, n_samples, step_size=0.5):
    """
    MCMC sampling using Metropolis-Hastings algorithm.
    Can sample from ANY distribution (only need unnormalized PDF)!
    """
    current = initial
    samples = [current]
    accepted = 0
    
    for _ in range(n_samples - 1):
        # Propose new point (random walk)
        proposal = current + np.random.normal(0, step_size)
        
        # Acceptance ratio (in log space for stability)
        log_ratio = target_log_pdf(proposal) - target_log_pdf(current)
        
        # Accept or reject
        if np.log(np.random.uniform()) < log_ratio:
            current = proposal
            accepted += 1
        
        samples.append(current)
    
    return np.array(samples), accepted / (n_samples - 1)

# Sample from a mixture of Gaussians
def target_log_pdf(x):
    """Unnormalized log PDF of mixture of Gaussians"""
    return np.log(0.3 * np.exp(-0.5 * (x + 2)**2) + 
                  0.7 * np.exp(-0.5 * (x - 3)**2) + 1e-10)

samples, acc_rate = metropolis_hastings(target_log_pdf, 0.0, 50000, step_size=1.0)

# Discard burn-in (first 5000 samples)
samples = samples[5000:]
print(f"\nMCMC (Metropolis-Hastings):")
print(f"Acceptance rate: {acc_rate:.2%} (ideal: 20-50%)")
print(f"Sample mean: {samples.mean():.4f}")

# === IMPORTANCE SAMPLING ===
# Estimate E_p[f(x)] using samples from q(x) instead of p(x)
# E_p[f(x)] = E_q[f(x) × p(x)/q(x)]

def importance_sampling(f, target_pdf, proposal_dist, n_samples):
    """Estimate E_p[f(x)] using samples from proposal q"""
    samples = proposal_dist.rvs(n_samples)
    weights = target_pdf(samples) / proposal_dist.pdf(samples)
    weighted_estimate = np.mean(f(samples) * weights)
    return weighted_estimate

# Estimate E[X²] where X ~ Beta(2,5) using Normal proposal
from scipy.stats import norm

target_pdf = lambda x: beta_dist.pdf(x, 2, 5)
proposal = norm(loc=0.3, scale=0.3)  # Centered near Beta(2,5) mean
f = lambda x: x**2

estimate = importance_sampling(f, target_pdf, proposal, 50000)
true_value = beta_dist.moment(2, 2, 5)  # Exact E[X²]
print(f"\nImportance Sampling:")
print(f"Estimated E[X²]: {estimate:.4f}")
print(f"True E[X²]: {true_value:.4f}")
```

> **Pro Tip:** In modern deep learning, variational inference (used in VAEs) is preferred over MCMC because it's much faster. But MCMC gives exact samples given enough time; variational inference gives fast but approximate results.

---

## 11. Hypothesis Testing and Confidence Intervals

### What It Is
Statistical methods for making decisions from data with controlled error rates.

- **Hypothesis test** — "Is this effect real, or just random noise?"
- **Confidence interval** — "What's the plausible range for the true value?"

### Why It Matters
- **A/B testing** — "Did this new feature actually improve conversion?"
- **Model comparison** — "Is model A significantly better than model B?"
- **Feature importance** — "Is this feature's coefficient significantly non-zero?"

### How It Works

```
Hypothesis Testing Framework:

H₀ (Null):       "No effect" (boring hypothesis)
H₁ (Alternative): "There IS an effect" (what you want to prove)

p-value = P(data this extreme | H₀ is true)

If p-value < α (usually 0.05):
    → Reject H₀ → "Statistically significant!"
    
If p-value ≥ α:
    → Fail to reject H₀ → "Not enough evidence"
    (NOT the same as "H₀ is true"!)

Error Types:
┌──────────────────┬───────────────┬───────────────┐
│                  │ H₀ is True    │ H₀ is False   │
├──────────────────┼───────────────┼───────────────┤
│ Reject H₀       │ Type I (α)    │ Correct!      │
│                  │ False Positive│ (Power)        │
├──────────────────┼───────────────┼───────────────┤
│ Fail to reject   │ Correct!      │ Type II (β)   │
│                  │               │ False Negative │
└──────────────────┴───────────────┴───────────────┘
```

### Code Examples

```python
import numpy as np
from scipy import stats

# === A/B TEST (Two-Sample t-test) ===
np.random.seed(42)

# Control: current website (conversion rate ~5%)
# Treatment: new design (conversion rate ~5.5%)
n_control = 10000
n_treatment = 10000

control = np.random.binomial(1, 0.050, n_control)    # 5.0% conversion
treatment = np.random.binomial(1, 0.055, n_treatment) # 5.5% conversion

# Two-sample t-test
t_stat, p_value = stats.ttest_ind(treatment, control)
print(f"A/B Test Results:")
print(f"Control conversion: {control.mean():.4f}")
print(f"Treatment conversion: {treatment.mean():.4f}")
print(f"t-statistic: {t_stat:.4f}")
print(f"p-value: {p_value:.4f}")
print(f"Significant at α=0.05? {p_value < 0.05}")

# Effect size (practical significance ≠ statistical significance!)
cohens_d = (treatment.mean() - control.mean()) / np.sqrt(
    (control.var() + treatment.var()) / 2
)
print(f"Cohen's d (effect size): {cohens_d:.4f}")
# Small: 0.2, Medium: 0.5, Large: 0.8

# === CONFIDENCE INTERVAL ===
# 95% CI for the treatment effect (difference in proportions)
diff = treatment.mean() - control.mean()
se = np.sqrt(control.var()/n_control + treatment.var()/n_treatment)
ci_low = diff - 1.96 * se
ci_high = diff + 1.96 * se
print(f"\n95% CI for treatment effect: [{ci_low:.4f}, {ci_high:.4f}]")
# If CI doesn't contain 0 → significant

# === MULTIPLE TESTING CORRECTION ===
# Testing 20 features: some will be "significant" by chance!
np.random.seed(42)
n_tests = 20
p_values = []

for _ in range(n_tests):
    # All null (no real effect)
    group1 = np.random.normal(0, 1, 100)
    group2 = np.random.normal(0, 1, 100)  # Same distribution!
    _, p = stats.ttest_ind(group1, group2)
    p_values.append(p)

p_values = np.array(p_values)
print(f"\n--- Multiple Testing Problem ---")
print(f"Tests with p < 0.05: {np.sum(p_values < 0.05)} / {n_tests}")
print(f"Expected false positives: {n_tests * 0.05:.1f}")

# Bonferroni correction: divide α by number of tests
alpha_corrected = 0.05 / n_tests
print(f"Bonferroni-corrected significant: {np.sum(p_values < alpha_corrected)}")

# Benjamini-Hochberg (FDR control) — less conservative
from scipy.stats import false_discovery_control
# Use statsmodels or manual implementation for BH correction
sorted_p = np.sort(p_values)
bh_threshold = np.arange(1, n_tests+1) / n_tests * 0.05
significant_bh = np.sum(sorted_p < bh_threshold)
print(f"BH-corrected significant: {significant_bh}")

# === BOOTSTRAP CONFIDENCE INTERVAL ===
# No distributional assumptions needed!
def bootstrap_ci(data, statistic=np.mean, n_bootstrap=10000, ci=0.95):
    """Non-parametric confidence interval via bootstrap"""
    n = len(data)
    bootstrap_stats = []
    
    for _ in range(n_bootstrap):
        sample = np.random.choice(data, size=n, replace=True)
        bootstrap_stats.append(statistic(sample))
    
    lower = np.percentile(bootstrap_stats, (1-ci)/2 * 100)
    upper = np.percentile(bootstrap_stats, (1+ci)/2 * 100)
    return lower, upper

# Bootstrap CI for median income
incomes = np.random.lognormal(10, 1, 200)  # Skewed distribution
ci_low, ci_high = bootstrap_ci(incomes, statistic=np.median)
print(f"\nBootstrap 95% CI for median income: [{ci_low:.0f}, {ci_high:.0f}]")
print(f"Sample median: {np.median(incomes):.0f}")

# === POWER ANALYSIS ===
# How many samples do I need to detect a given effect?
from scipy.stats import norm as norm_dist

def required_sample_size(effect_size, alpha=0.05, power=0.8):
    """Minimum samples per group for two-sample t-test"""
    z_alpha = norm_dist.ppf(1 - alpha/2)
    z_beta = norm_dist.ppf(power)
    n = 2 * ((z_alpha + z_beta) / effect_size) ** 2
    return int(np.ceil(n))

for effect in [0.1, 0.2, 0.5, 0.8]:
    n = required_sample_size(effect)
    print(f"Effect size {effect}: need {n} samples per group")
```

### Common Mistakes
1. **p-value ≠ P(H₀ is true)** — It's P(data | H₀ is true). A subtle but crucial difference.
2. **Statistical significance ≠ practical significance** — With enough data, any tiny difference becomes "significant"
3. **Multiple testing** — If you test 20 features at α=0.05, expect 1 false positive. Use Bonferroni or BH correction.
4. **Peeking at results early** — Running a test, checking, then continuing until significant inflates false positive rate

---

## 12. Information Theory Essentials

### What It Is
Information theory quantifies **uncertainty** and **information content**. It connects to ML through loss functions.

### Why It Matters
- **Cross-entropy loss** = the standard classification loss
- **KL divergence** = measures how different two distributions are (used in VAEs)
- **Mutual information** = measures dependency between variables (feature selection)
- **Entropy** = measures randomness/uncertainty

### Key Concepts

$$\text{Entropy: } H(X) = -\sum_x P(x) \log P(x)$$

$$\text{Cross-Entropy: } H(P, Q) = -\sum_x P(x) \log Q(x)$$

$$\text{KL Divergence: } D_{KL}(P || Q) = \sum_x P(x) \log \frac{P(x)}{Q(x)}$$

$$\text{Relationship: } H(P, Q) = H(P) + D_{KL}(P || Q)$$

### Code Examples

```python
import numpy as np

# === ENTROPY ===
def entropy(probs):
    """Shannon entropy (bits if log2, nats if ln)"""
    probs = np.array(probs)
    probs = probs[probs > 0]  # Avoid log(0)
    return -np.sum(probs * np.log2(probs))

# Fair coin: maximum uncertainty
print(f"Fair coin entropy: {entropy([0.5, 0.5]):.4f} bits")  # 1.0

# Biased coin: less uncertainty
print(f"Biased (0.9/0.1) entropy: {entropy([0.9, 0.1]):.4f} bits")  # 0.469

# Certain outcome: zero uncertainty
print(f"Certain entropy: {entropy([1.0, 0.0]):.4f} bits")  # 0.0

# Die: 6 outcomes
print(f"Fair die entropy: {entropy([1/6]*6):.4f} bits")  # 2.585

# === CROSS-ENTROPY (The Classification Loss!) ===
def cross_entropy(p_true, q_predicted):
    """Cross-entropy: how well q approximates p"""
    p_true = np.array(p_true)
    q_predicted = np.array(q_predicted)
    q_predicted = np.clip(q_predicted, 1e-10, 1.0)
    return -np.sum(p_true * np.log(q_predicted))

# True: class 0 (one-hot: [1, 0, 0])
p = [1, 0, 0]

# Good prediction (confident and correct)
q_good = [0.9, 0.05, 0.05]
# Bad prediction (confident but wrong)
q_bad = [0.1, 0.8, 0.1]
# Uncertain prediction
q_uncertain = [0.33, 0.33, 0.34]

print(f"\nCross-Entropy (lower = better):")
print(f"Good prediction: {cross_entropy(p, q_good):.4f}")
print(f"Uncertain: {cross_entropy(p, q_uncertain):.4f}")
print(f"Bad prediction: {cross_entropy(p, q_bad):.4f}")

# === KL DIVERGENCE ===
def kl_divergence(p, q):
    """KL(P || Q): how much info is lost using Q to approximate P"""
    p, q = np.array(p), np.array(q)
    mask = p > 0
    return np.sum(p[mask] * np.log(p[mask] / q[mask]))

# KL is NOT symmetric!
p = [0.4, 0.6]
q = [0.5, 0.5]
print(f"\nKL(P||Q) = {kl_divergence(p, q):.4f}")
print(f"KL(Q||P) = {kl_divergence(q, p):.4f}")
print(f"KL(P||P) = {kl_divergence(p, p):.4f}")  # 0 (same distribution)

# === VAE LOSS = Reconstruction + KL ===
# VAE minimizes: E[log p(x|z)] - KL(q(z|x) || p(z))
# The KL term keeps the latent space close to N(0,1)

def vae_kl_loss(mu, log_var):
    """KL divergence between N(μ, σ²) and N(0, 1)
    Closed form: -0.5 * Σ(1 + log(σ²) - μ² - σ²)
    """
    return -0.5 * np.sum(1 + log_var - mu**2 - np.exp(log_var))

mu = np.array([0.5, -0.3, 0.1])      # Encoder mean
log_var = np.array([-0.5, -1.0, 0.2]) # Encoder log-variance
print(f"\nVAE KL Loss: {vae_kl_loss(mu, log_var):.4f}")
# This penalizes the encoder for deviating from standard normal

# === MUTUAL INFORMATION ===
# I(X;Y) = H(X) + H(Y) - H(X,Y)
# Measures how much knowing X tells you about Y
def mutual_information(joint):
    """MI from joint distribution table"""
    joint = np.array(joint)
    px = joint.sum(axis=1)  # Marginal of X
    py = joint.sum(axis=0)  # Marginal of Y
    
    mi = 0
    for i in range(joint.shape[0]):
        for j in range(joint.shape[1]):
            if joint[i,j] > 0:
                mi += joint[i,j] * np.log(joint[i,j] / (px[i] * py[j]))
    return mi

# Strongly dependent variables
joint_dependent = np.array([[0.4, 0.1], [0.1, 0.4]])
# Independent variables
joint_independent = np.array([[0.25, 0.25], [0.25, 0.25]])

print(f"\nMI (dependent): {mutual_information(joint_dependent):.4f}")    # > 0
print(f"MI (independent): {mutual_information(joint_independent):.4f}")  # ≈ 0
```

> **Key Insight:** Minimizing cross-entropy loss = minimizing KL divergence between true labels and predictions = maximum likelihood estimation. They're all the same thing viewed from different angles.

---

## 13. Applications in ML and Interview Prep

### Interview Questions

#### Conceptual

1. **Explain the difference between MLE and MAP.**
   → MLE maximizes $P(data|\theta)$. MAP maximizes $P(data|\theta) \cdot P(\theta)$. MAP = MLE + prior = regularized MLE. Gaussian prior → L2; Laplace prior → L1.

2. **Why do we use cross-entropy instead of MSE for classification?**
   → Cross-entropy is derived from MLE with Bernoulli/categorical likelihood. MSE assumes Gaussian noise. Cross-entropy gradients don't saturate (sigmoid + MSE has near-zero gradients far from decision boundary).

3. **What's the bias-variance tradeoff? Give a concrete example.**
   → Total error = Bias² + Variance + Noise. High bias = underfitting (linear model for nonlinear data). High variance = overfitting (degree-20 polynomial). Trade off by controlling model complexity.

4. **When would you use Bayesian methods over frequentist?**
   → Small datasets, when you have prior knowledge, when you need uncertainty estimates, when overfitting is a concern, safety-critical applications.

5. **Explain p-values in simple terms. What are their limitations?**
   → P-value = probability of seeing data this extreme if null hypothesis is true. Limitations: doesn't tell you effect size, affected by sample size, multiple testing inflates false positives, doesn't measure P(hypothesis is true).

6. **What is the reparameterization trick in VAEs?**
   → Instead of sampling $z \sim N(\mu, \sigma^2)$ (not differentiable), sample $\epsilon \sim N(0,1)$ and compute $z = \mu + \sigma \cdot \epsilon$. This makes the sampling step differentiable for backpropagation.

7. **Explain the Central Limit Theorem and why it matters for ML.**
   → Sum/average of many independent variables → Normal distribution, regardless of original distribution. Matters because: (1) justifies Gaussian assumptions, (2) explains why batch statistics are stable, (3) foundation of confidence intervals.

8. **What distribution would you use to model click-through rates?**
   → Beta distribution (continuous on [0,1], conjugate prior for Bernoulli). Can also use Bayesian updating: start with Beta(1,1), after seeing k clicks in n views, posterior is Beta(1+k, 1+n-k).

#### Coding

9. **Implement a Naive Bayes classifier from scratch.** → See Section 2
10. **Write a bootstrap function to estimate confidence intervals.** → See Section 11
11. **Implement MLE for a Gaussian distribution.** → See Section 7

---

## Quick Reference

### Distribution Cheat Sheet

| Distribution | PDF/PMF | Mean | Variance | ML Use |
|-------------|---------|------|----------|--------|
| Bernoulli(p) | $p^x(1-p)^{1-x}$ | $p$ | $p(1-p)$ | Binary classification |
| Binomial(n,p) | $\binom{n}{k}p^k(1-p)^{n-k}$ | $np$ | $np(1-p)$ | Counting successes |
| Poisson(λ) | $\frac{\lambda^k e^{-\lambda}}{k!}$ | $\lambda$ | $\lambda$ | Event counts |
| Normal(μ,σ²) | $\frac{1}{\sigma\sqrt{2\pi}}e^{-\frac{(x-\mu)^2}{2\sigma^2}}$ | $\mu$ | $\sigma^2$ | Everything |
| Exponential(λ) | $\lambda e^{-\lambda x}$ | $1/\lambda$ | $1/\lambda^2$ | Time between events |
| Beta(α,β) | $\frac{x^{\alpha-1}(1-x)^{\beta-1}}{B(\alpha,\beta)}$ | $\frac{\alpha}{\alpha+\beta}$ | $\frac{\alpha\beta}{(\alpha+\beta)^2(\alpha+\beta+1)}$ | Probabilities |

### Key Formulas

| Formula | Meaning |
|---------|---------|
| $P(A|B) = P(B|A)P(A)/P(B)$ | Bayes' theorem |
| $E[X] = \sum x P(x)$ | Expected value |
| $Var(X) = E[X^2] - (E[X])^2$ | Variance shortcut |
| $\hat{\theta}_{MLE} = \arg\max \sum \log P(x_i|\theta)$ | Maximum likelihood |
| $\hat{\theta}_{MAP} = \arg\max [\sum \log P(x_i|\theta) + \log P(\theta)]$ | MAP estimation |
| $H(P,Q) = -\sum P \log Q$ | Cross-entropy loss |
| $D_{KL}(P\|Q) = \sum P \log(P/Q)$ | KL divergence |
| Error = Bias² + Variance + Noise | Bias-variance decomposition |

### Decision Tree: Which Method to Use?

```
Need to make a prediction?
├── Have lots of data → MLE (standard training)
├── Have prior knowledge or small data → MAP / Bayesian
├── Need uncertainty → Full Bayesian / MC Dropout
└── Need to compare models → Hypothesis test / Bayesian model selection

Need to choose a loss function?
├── Regression → MSE (assumes Gaussian noise)
├── Binary classification → Binary cross-entropy (assumes Bernoulli)
├── Multi-class → Categorical cross-entropy (assumes Categorical)
└── Want robustness → Huber loss (less sensitive to outliers)
```
