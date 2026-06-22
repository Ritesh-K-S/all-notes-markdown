# Chapter 05: Information Theory for Machine Learning

## Overview
Information Theory is the mathematical study of communication, uncertainty, and data compression — invented by Claude Shannon in 1948. In ML, it gives us the language to measure "how surprised we are," "how different two distributions are," and "how much one variable tells us about another." It's the foundation of cross-entropy loss, KL divergence, decision trees, and generative models.

---

## Table of Contents
1. [Entropy — Measuring Uncertainty](#1-entropy--measuring-uncertainty)
2. [Cross-Entropy — Comparing Distributions](#2-cross-entropy--comparing-distributions)
3. [KL Divergence — Distance Between Distributions](#3-kl-divergence--distance-between-distributions)
4. [Mutual Information](#4-mutual-information)
5. [Information Gain and Decision Trees](#5-information-gain-and-decision-trees)
6. [Connections to ML Loss Functions](#6-connections-to-ml-loss-functions)
7. [Advanced Topics: ELBO, VAEs, and InfoNCE](#7-advanced-topics)

---

## 1. Entropy — Measuring Uncertainty

### What It Is
Imagine you're guessing the result of a coin flip:
- **Fair coin** (50/50): Maximum uncertainty — you truly don't know what's coming
- **Biased coin** (99/1): Low uncertainty — you're pretty sure it's heads
- **Always heads** (100/0): Zero uncertainty — no surprise at all

**Entropy** quantifies this uncertainty. High entropy = high unpredictability = more "information" in each observation.

### Why It Matters
- **Cross-entropy loss** (the standard classification loss) is built on entropy
- **Decision trees** split on the feature that reduces entropy the most
- **Compression** — entropy is the theoretical lower bound on how much you can compress data
- **Generative models** (VAEs, diffusion) use entropy and KL divergence extensively

### Mathematical Definition

For a discrete random variable $X$ with outcomes $x_1, x_2, ..., x_n$ and probabilities $p(x_i)$:

$$H(X) = -\sum_{i=1}^{n} p(x_i) \log_2 p(x_i)$$

Or using natural log (nats instead of bits):

$$H(X) = -\sum_{i=1}^{n} p(x_i) \ln p(x_i)$$

### Intuition: The "Surprise" Framework

Define **surprise** (or self-information) of event $x$:

$$I(x) = -\log p(x)$$

- If $p(x) = 1$ (certain): surprise = $-\log(1) = 0$ (no surprise)
- If $p(x) = 0.5$: surprise = $-\log(0.5) = 1$ bit (moderate surprise)
- If $p(x) = 0.01$ (rare): surprise = $-\log(0.01) = 6.64$ bits (very surprising!)

**Entropy = Average Surprise**:

$$H(X) = \mathbb{E}[I(X)] = \mathbb{E}[-\log p(X)]$$

### Visual Intuition

```
Entropy of a binary variable (coin flip) as a function of p:

H(X)
1.0 |          *****
    |       ***     ***
    |     **           **
    |    *               *
    |   *                 *
0.5 |  *                   *
    | *                     *
    |*                       *
0.0 *-------------------------*
    0    0.25   0.5   0.75   1.0
              p(heads)

Maximum entropy at p = 0.5 (most uncertain)
Zero entropy at p = 0 or p = 1 (completely certain)
```

### Properties of Entropy

| Property | Statement | Meaning |
|----------|-----------|---------|
| Non-negative | $H(X) \geq 0$ | Can't have negative uncertainty |
| Maximum | $H(X) \leq \log n$ | Uniform distribution has max entropy |
| Additivity | $H(X,Y) = H(X) + H(Y)$ if independent | Info from independent sources adds up |
| Concavity | $H$ is concave in $p$ | Mixing distributions increases entropy |

```python
import numpy as np
import matplotlib.pyplot as plt

def entropy(probs):
    """
    Compute Shannon entropy of a probability distribution.
    
    Args:
        probs: array of probabilities (must sum to 1)
    Returns:
        Entropy in bits (using log2) or nats (using ln)
    """
    # Filter out zero probabilities (0 * log(0) = 0 by convention)
    probs = np.array(probs)
    probs = probs[probs > 0]
    
    # Verify it's a valid distribution
    assert abs(sum(probs) - 1.0) < 1e-6, "Probabilities must sum to 1"
    
    return -np.sum(probs * np.log2(probs))


# Example 1: Fair coin vs biased coin
fair_coin = [0.5, 0.5]
biased_coin = [0.9, 0.1]
certain = [1.0, 0.0]

print("=== Entropy Examples ===")
print(f"Fair coin [0.5, 0.5]:   H = {entropy(fair_coin):.4f} bits")
print(f"Biased coin [0.9, 0.1]: H = {entropy(biased_coin):.4f} bits")
print(f"Certain [1.0, 0.0]:     H = {entropy(certain):.4f} bits")

# Example 2: Different number of outcomes
uniform_4 = [0.25, 0.25, 0.25, 0.25]  # 4-sided die
uniform_6 = [1/6]*6                     # 6-sided die
peaked = [0.7, 0.1, 0.1, 0.1]          # Peaked distribution

print(f"\nUniform 4-class: H = {entropy(uniform_4):.4f} bits (max = log2(4) = {np.log2(4):.4f})")
print(f"Uniform 6-class: H = {entropy(uniform_6):.4f} bits (max = log2(6) = {np.log2(6):.4f})")
print(f"Peaked [0.7,0.1,0.1,0.1]: H = {entropy(peaked):.4f} bits")

# Visualize: Binary entropy function
p_values = np.linspace(0.001, 0.999, 200)
H_values = [-p*np.log2(p) - (1-p)*np.log2(1-p) for p in p_values]

plt.figure(figsize=(8, 5))
plt.plot(p_values, H_values, 'b-', linewidth=2)
plt.xlabel('p (probability of class 1)')
plt.ylabel('H(X) in bits')
plt.title('Binary Entropy Function H(p) = -p·log₂(p) - (1-p)·log₂(1-p)')
plt.axvline(x=0.5, color='r', linestyle='--', alpha=0.5, label='Maximum at p=0.5')
plt.legend()
plt.grid(True)
plt.show()
```

### Continuous Entropy (Differential Entropy)

For continuous distributions:

$$h(X) = -\int_{-\infty}^{\infty} f(x) \ln f(x) \, dx$$

| Distribution | Differential Entropy |
|-------------|---------------------|
| Gaussian $\mathcal{N}(\mu, \sigma^2)$ | $\frac{1}{2} \ln(2\pi e \sigma^2)$ |
| Uniform $U(a, b)$ | $\ln(b - a)$ |
| Exponential $\text{Exp}(\lambda)$ | $1 + \ln(1/\lambda)$ |

> ⚠️ **Important**: Differential entropy can be **negative** (unlike discrete entropy). It's not directly comparable to discrete entropy.

---

## 2. Cross-Entropy — Comparing Distributions

### What It Is
Cross-entropy measures **how many bits you'd need to encode data from distribution $p$ using a code designed for distribution $q$**.

If $p$ is the true distribution and $q$ is your model's predicted distribution:
- If $q = p$: you use the minimum bits (= entropy of $p$)
- If $q \neq p$: you use more bits than necessary (cross-entropy > entropy)

### Why It Matters
- **THE standard loss function for classification** in deep learning
- It's what `nn.CrossEntropyLoss` in PyTorch computes
- Directly measures how well your model's predictions match reality
- Has nicer gradients than MSE for classification (avoids vanishing gradients in sigmoids)

### Mathematical Definition

$$H(p, q) = -\sum_{x} p(x) \log q(x)$$

For a single classification example with true label $y$ (one-hot) and predicted probabilities $\hat{y}$:

$$\mathcal{L} = -\sum_{c=1}^{C} y_c \log \hat{y}_c$$

Since $y$ is one-hot (only one class $k$ has $y_k = 1$):

$$\mathcal{L} = -\log \hat{y}_k \quad \text{(just the log of the predicted prob for the true class)}$$

### Relationship: Entropy, Cross-Entropy, KL Divergence

$$\underbrace{H(p, q)}_{\text{Cross-Entropy}} = \underbrace{H(p)}_{\text{Entropy}} + \underbrace{D_{KL}(p \| q)}_{\text{KL Divergence}}$$

Since $H(p)$ is constant (doesn't depend on model), minimizing cross-entropy = minimizing KL divergence.

```
Cross-Entropy = Entropy + KL Divergence

    H(p,q)    =   H(p)   +  D_KL(p||q)
      ↑            ↑           ↑
  Total bits    Minimum     Extra bits due
  needed        possible    to wrong model
```

### Visual Intuition

```
True distribution p = [0.7, 0.2, 0.1]  (cat is most likely)

Model q1 = [0.6, 0.3, 0.1]  →  H(p, q1) = -0.7·log(0.6) - 0.2·log(0.3) - 0.1·log(0.1)
                                           = 0.357 + 0.241 + 0.230 = 0.828

Model q2 = [0.1, 0.2, 0.7]  →  H(p, q2) = -0.7·log(0.1) - 0.2·log(0.2) - 0.1·log(0.7)
                                           = 1.612 + 0.322 + 0.036 = 1.970
                                           ↑ MUCH WORSE (model is "surprised" by true data)
```

```python
import numpy as np

def cross_entropy(p, q):
    """
    Compute cross-entropy H(p, q) = -sum(p * log(q)).
    
    Args:
        p: True distribution (array summing to 1)
        q: Predicted distribution (array summing to 1)
    Returns:
        Cross-entropy value (always >= entropy of p)
    """
    p = np.array(p, dtype=float)
    q = np.array(q, dtype=float)
    
    # Clip q to avoid log(0)
    q = np.clip(q, 1e-15, 1.0)
    
    return -np.sum(p * np.log(q))


def cross_entropy_loss_classification(y_true, y_pred):
    """
    Cross-entropy loss for classification.
    This is what PyTorch's nn.CrossEntropyLoss computes.
    
    Args:
        y_true: Integer labels (not one-hot), shape (N,)
        y_pred: Predicted probabilities, shape (N, C) where C = num classes
    Returns:
        Average cross-entropy loss
    """
    N = len(y_true)
    y_pred = np.clip(y_pred, 1e-15, 1.0)
    
    # For each sample, pick the predicted probability of the TRUE class
    correct_class_probs = y_pred[np.arange(N), y_true]
    
    # Loss = -log(probability of true class)
    losses = -np.log(correct_class_probs)
    
    return np.mean(losses)


# Example: Binary classification (logistic regression loss)
print("=== Binary Cross-Entropy ===")
# True label: class 1 (positive)
# Model predicts probability 0.9 for class 1
y_true = 1
y_pred = 0.9
bce = -(y_true * np.log(y_pred) + (1 - y_true) * np.log(1 - y_pred))
print(f"True=1, Predicted=0.9: Loss = {bce:.4f} (good prediction, low loss)")

y_pred = 0.1
bce = -(y_true * np.log(y_pred) + (1 - y_true) * np.log(1 - y_pred))
print(f"True=1, Predicted=0.1: Loss = {bce:.4f} (bad prediction, high loss)")

# Example: Multi-class classification
print("\n=== Multi-class Cross-Entropy ===")
# 3 samples, 4 classes
y_true_multi = np.array([0, 2, 1])  # True classes
y_pred_multi = np.array([
    [0.8, 0.1, 0.05, 0.05],  # Good prediction for class 0
    [0.1, 0.2, 0.6, 0.1],    # OK prediction for class 2
    [0.1, 0.1, 0.1, 0.7],    # Bad prediction for class 1 (puts 0.7 on class 3!)
])

loss = cross_entropy_loss_classification(y_true_multi, y_pred_multi)
print(f"Average cross-entropy loss: {loss:.4f}")

# Per-sample breakdown
for i in range(3):
    true_class = y_true_multi[i]
    pred_prob = y_pred_multi[i, true_class]
    sample_loss = -np.log(pred_prob)
    print(f"  Sample {i}: true class={true_class}, "
          f"predicted prob={pred_prob:.2f}, loss={sample_loss:.4f}")
```

### Why Cross-Entropy and Not MSE for Classification?

```python
import numpy as np
import matplotlib.pyplot as plt

# The gradient comparison — why CE works better
# For sigmoid output with true label y=1:
# MSE loss: L = (sigmoid(z) - 1)^2
# CE loss:  L = -log(sigmoid(z))

z = np.linspace(-6, 6, 100)
sigmoid = 1 / (1 + np.exp(-z))

# Gradients when true label = 1
mse_grad = 2 * (sigmoid - 1) * sigmoid * (1 - sigmoid)  # vanishes for extreme wrong predictions!
ce_grad = sigmoid - 1  # Nice, bounded gradient

plt.figure(figsize=(10, 5))
plt.plot(z, np.abs(mse_grad), 'r-', linewidth=2, label='|MSE gradient|')
plt.plot(z, np.abs(ce_grad), 'b-', linewidth=2, label='|Cross-Entropy gradient|')
plt.xlabel('z (logit)')
plt.ylabel('|Gradient| magnitude')
plt.title('Why Cross-Entropy > MSE for Classification\n'
          '(CE has strong gradient even when prediction is very wrong)')
plt.legend()
plt.grid(True)
plt.axvline(x=0, color='k', linestyle='--', alpha=0.3)
plt.show()
```

> 💡 **Key Insight**: When the model is **very wrong** (sigmoid near 0 but true label is 1), MSE gradient vanishes (sigmoid derivative → 0), so learning stops! Cross-entropy gradient stays strong, pushing the model to correct itself.

---

## 3. KL Divergence — Distance Between Distributions

### What It Is
KL Divergence measures **how different distribution $q$ is from distribution $p$**. Think of it as the "information penalty" you pay for using the wrong model.

### Why It Matters
- **VAE loss** has a KL divergence term (regularizes the latent space)
- **Knowledge distillation** minimizes KL between teacher and student
- **Policy gradient methods** (TRPO, PPO) constrain KL between policy updates
- **Bayesian inference** — posterior is the distribution minimizing KL from prior subject to data

### Mathematical Definition

$$D_{KL}(p \| q) = \sum_{x} p(x) \log \frac{p(x)}{q(x)} = \mathbb{E}_{x \sim p}\left[\log \frac{p(x)}{q(x)}\right]$$

Continuous version:

$$D_{KL}(p \| q) = \int p(x) \log \frac{p(x)}{q(x)} \, dx$$

### Critical Properties

| Property | Implication |
|----------|------------|
| $D_{KL}(p \| q) \geq 0$ | Always non-negative (Gibbs' inequality) |
| $D_{KL}(p \| q) = 0$ iff $p = q$ | Zero only when distributions are identical |
| $D_{KL}(p \| q) \neq D_{KL}(q \| p)$ | **NOT symmetric!** (not a true distance) |
| Not a metric | Doesn't satisfy triangle inequality |

### Asymmetry Explained

```
Forward KL: D_KL(p || q) — "mean-seeking" / "inclusive"
    → Penalizes q for having LOW probability where p has HIGH probability
    → q tries to COVER all of p → spreads out
    
Reverse KL: D_KL(q || p) — "mode-seeking" / "exclusive"  
    → Penalizes q for having HIGH probability where p has LOW probability
    → q focuses on ONE MODE of p → collapses

True distribution p (bimodal):     Forward KL fit:        Reverse KL fit:
        *       *                       ****                    *
       * *     * *                     *    *                  * *
      *   *   *   *                   *      *                *   *
     *     * *     *                 *        *              *     *
    *       *       *               *          *            *       *
___*_________________*___        __*____________*__      ___*_________*___
                                (covers both modes)     (picks one mode)
```

```python
import numpy as np
from scipy.stats import norm

def kl_divergence_discrete(p, q):
    """
    KL Divergence D_KL(p || q) for discrete distributions.
    
    Interpretation: Expected number of extra bits needed to encode 
    samples from p using a code optimized for q.
    """
    p = np.array(p, dtype=float)
    q = np.array(q, dtype=float)
    
    # Only consider where p > 0
    mask = p > 0
    q_safe = np.clip(q[mask], 1e-15, 1.0)
    
    return np.sum(p[mask] * np.log(p[mask] / q_safe))


def kl_divergence_gaussians(mu1, sigma1, mu2, sigma2):
    """
    Analytical KL divergence between two univariate Gaussians.
    D_KL(N(mu1, sigma1^2) || N(mu2, sigma2^2))
    
    This is used in VAEs to regularize the latent space.
    """
    return (np.log(sigma2/sigma1) + 
            (sigma1**2 + (mu1-mu2)**2) / (2*sigma2**2) - 0.5)


# Example 1: Discrete distributions
p = [0.4, 0.3, 0.2, 0.1]
q1 = [0.35, 0.3, 0.2, 0.15]  # Close to p
q2 = [0.1, 0.1, 0.1, 0.7]   # Very different from p

print("=== KL Divergence (Discrete) ===")
print(f"D_KL(p || q1) = {kl_divergence_discrete(p, q1):.4f} (similar distributions)")
print(f"D_KL(p || q2) = {kl_divergence_discrete(p, q2):.4f} (very different)")
print(f"D_KL(q2 || p) = {kl_divergence_discrete(q2, p):.4f} (asymmetric!)")

# Example 2: Gaussian KL divergence (used in VAEs)
print("\n=== KL Divergence (Gaussians) ===")
# KL from learned distribution to prior N(0,1)
print(f"D_KL(N(0,1) || N(0,1)) = {kl_divergence_gaussians(0, 1, 0, 1):.4f} (same = 0)")
print(f"D_KL(N(1,1) || N(0,1)) = {kl_divergence_gaussians(1, 1, 0, 1):.4f} (shifted mean)")
print(f"D_KL(N(0,2) || N(0,1)) = {kl_divergence_gaussians(0, 2, 0, 1):.4f} (wider variance)")
print(f"D_KL(N(3,0.5) || N(0,1)) = {kl_divergence_gaussians(3, 0.5, 0, 1):.4f} (very different)")

# Visualize asymmetry
import matplotlib.pyplot as plt

x = np.linspace(-5, 8, 1000)
p_dist = 0.5 * norm.pdf(x, -1, 0.8) + 0.5 * norm.pdf(x, 3, 0.8)  # Bimodal

# Forward KL fit (mean-seeking) → wider Gaussian
q_forward = norm.pdf(x, 1, 2.5)

# Reverse KL fit (mode-seeking) → narrow, on one mode  
q_reverse = norm.pdf(x, 3, 0.8)

fig, axes = plt.subplots(1, 2, figsize=(14, 5))

axes[0].plot(x, p_dist, 'b-', linewidth=2, label='p (true, bimodal)')
axes[0].plot(x, q_forward, 'r--', linewidth=2, label='q (forward KL fit)')
axes[0].set_title('Forward KL: D_KL(p||q)\n"Mean-seeking" — covers both modes')
axes[0].legend()
axes[0].grid(True)

axes[1].plot(x, p_dist, 'b-', linewidth=2, label='p (true, bimodal)')
axes[1].plot(x, q_reverse, 'r--', linewidth=2, label='q (reverse KL fit)')
axes[1].set_title('Reverse KL: D_KL(q||p)\n"Mode-seeking" — locks onto one mode')
axes[1].legend()
axes[1].grid(True)

plt.tight_layout()
plt.show()
```

### KL Divergence in VAEs

In a Variational Autoencoder, the loss is:

$$\mathcal{L} = \underbrace{-\mathbb{E}_{q(z|x)}[\log p(x|z)]}_{\text{Reconstruction loss}} + \underbrace{D_{KL}(q(z|x) \| p(z))}_{\text{Regularization}}$$

For Gaussian encoder $q(z|x) = \mathcal{N}(\mu, \sigma^2)$ and standard normal prior $p(z) = \mathcal{N}(0, 1)$:

$$D_{KL} = -\frac{1}{2} \sum_{j=1}^{d} \left(1 + \log \sigma_j^2 - \mu_j^2 - \sigma_j^2\right)$$

```python
import numpy as np

def vae_kl_loss(mu, log_var):
    """
    KL divergence between N(mu, sigma^2) and N(0, 1).
    Used as the regularization term in VAE loss.
    
    Args:
        mu: Mean of encoder output, shape (batch_size, latent_dim)
        log_var: Log variance of encoder output, shape (batch_size, latent_dim)
    
    Returns:
        KL divergence, scalar (averaged over batch)
    """
    # KL = -0.5 * sum(1 + log(sigma^2) - mu^2 - sigma^2)
    kl = -0.5 * np.sum(1 + log_var - mu**2 - np.exp(log_var), axis=1)
    return np.mean(kl)

# Example
batch_size, latent_dim = 32, 10
mu = np.random.randn(batch_size, latent_dim) * 0.5
log_var = np.random.randn(batch_size, latent_dim) * 0.3

kl_loss = vae_kl_loss(mu, log_var)
print(f"VAE KL Loss: {kl_loss:.4f}")
print(f"(Pushing encoder towards standard normal)")
```

---

## 4. Mutual Information

### What It Is
Mutual Information measures **how much knowing one variable tells you about another**. It's like asking: "If I know X, how much does my uncertainty about Y decrease?"

- $I(X;Y) = 0$: X and Y are independent (knowing one tells you nothing about the other)
- $I(X;Y)$ is high: X and Y are strongly dependent

### Why It Matters
- **Feature selection**: Pick features with highest MI with the target
- **Clustering evaluation**: Adjusted Mutual Information (AMI) score
- **Representation learning**: InfoNCE loss (used in contrastive learning like SimCLR)
- **Information bottleneck**: Framework for understanding deep learning

### Mathematical Definition

$$I(X; Y) = \sum_{x,y} p(x, y) \log \frac{p(x, y)}{p(x) p(y)}$$

Equivalently:

$$I(X; Y) = H(X) - H(X|Y) = H(Y) - H(Y|X) = H(X) + H(Y) - H(X, Y)$$

### Visual: Venn Diagram of Information

```
        ┌──────────────────────────────────┐
        │           H(X, Y)                │
        │                                  │
        │    ┌──────────┬──────────┐       │
        │    │          │          │       │
        │    │   H(X|Y) │  I(X;Y) │ H(Y|X)│
        │    │          │          │       │
        │    └──────────┴──────────┘       │
        │                                  │
        │    ├────H(X)───┤                 │
        │              ├────H(Y)───┤       │
        └──────────────────────────────────┘

H(X)   = H(X|Y) + I(X;Y)     ← Total uncertainty of X
H(Y)   = H(Y|X) + I(X;Y)     ← Total uncertainty of Y
I(X;Y) = H(X) - H(X|Y)       ← Info X gives about Y
        = H(Y) - H(Y|X)       ← Info Y gives about X
```

### MI = KL Divergence Between Joint and Product of Marginals

$$I(X; Y) = D_{KL}(p(x, y) \| p(x) p(y))$$

This measures how far the joint distribution is from independence.

```python
import numpy as np
from sklearn.metrics import mutual_info_score, normalized_mutual_info_score
from sklearn.feature_selection import mutual_info_classif

def mutual_information_discrete(joint_prob):
    """
    Compute mutual information from a joint probability table.
    
    Args:
        joint_prob: 2D array where joint_prob[i,j] = P(X=i, Y=j)
    Returns:
        Mutual information in nats
    """
    joint = np.array(joint_prob)
    assert abs(joint.sum() - 1.0) < 1e-6, "Joint probs must sum to 1"
    
    # Marginal distributions
    p_x = joint.sum(axis=1)  # Sum over y
    p_y = joint.sum(axis=0)  # Sum over x
    
    mi = 0.0
    for i in range(joint.shape[0]):
        for j in range(joint.shape[1]):
            if joint[i, j] > 0:
                mi += joint[i, j] * np.log(joint[i, j] / (p_x[i] * p_y[j]))
    
    return mi


# Example 1: Perfect dependence vs independence
print("=== Mutual Information Examples ===")

# Independent: X and Y are unrelated
joint_independent = np.array([
    [0.15, 0.10, 0.25],  # P(X=0, Y=j)
    [0.15, 0.10, 0.25],  # P(X=1, Y=j)
]) 
# Check: p(x=0) = 0.5, p(y=0) = 0.3, p(x=0,y=0) = 0.15 = 0.5*0.3 ✓

# Dependent: Knowing X tells you a lot about Y
joint_dependent = np.array([
    [0.45, 0.03, 0.02],  # If X=0, Y is almost certainly 0
    [0.02, 0.03, 0.45],  # If X=1, Y is almost certainly 2
])

print(f"Independent case: MI = {mutual_information_discrete(joint_independent):.4f} nats")
print(f"Dependent case:   MI = {mutual_information_discrete(joint_dependent):.4f} nats")

# Example 2: Feature selection using MI (sklearn)
from sklearn.datasets import make_classification

X, y = make_classification(n_samples=1000, n_features=10, 
                           n_informative=3,  # Only 3 features are useful
                           n_redundant=2, random_state=42)

# Compute MI between each feature and the target
mi_scores = mutual_info_classif(X, y, random_state=42)

print("\n=== Feature Selection by Mutual Information ===")
for i, score in enumerate(mi_scores):
    bar = '█' * int(score * 50)
    print(f"Feature {i:2d}: MI = {score:.4f} {bar}")

# Top features
top_features = np.argsort(mi_scores)[::-1][:3]
print(f"\nTop 3 features (most informative): {top_features}")
```

### Mutual Information vs Correlation

| Property | Correlation | Mutual Information |
|----------|------------|-------------------|
| Captures | Linear relationships only | Any relationship (linear + nonlinear) |
| Range | [-1, 1] | [0, ∞) |
| Zero means | No linear relationship | True independence |
| Example | Circular pattern → corr ≈ 0 | Circular pattern → MI > 0 |

```python
import numpy as np
from sklearn.feature_selection import mutual_info_regression

# Nonlinear relationship: Y = X^2
np.random.seed(42)
X = np.random.uniform(-3, 3, 1000).reshape(-1, 1)
Y_linear = 2 * X.ravel() + np.random.randn(1000) * 0.5
Y_quadratic = X.ravel()**2 + np.random.randn(1000) * 0.5
Y_sine = np.sin(3 * X.ravel()) + np.random.randn(1000) * 0.3

print("=== Correlation vs MI ===")
for name, Y in [("Linear", Y_linear), ("Quadratic", Y_quadratic), ("Sine", Y_sine)]:
    corr = np.corrcoef(X.ravel(), Y)[0, 1]
    mi = mutual_info_regression(X, Y, random_state=42)[0]
    print(f"{name:10s}: Correlation = {corr:+.3f}, MI = {mi:.3f}")

# Output shows MI captures nonlinear relationships that correlation misses!
```

---

## 5. Information Gain and Decision Trees

### What It Is
Information Gain is the reduction in entropy after splitting data on a feature. Decision trees greedily choose the split that maximizes Information Gain at each node.

### Why It Matters
- **ID3, C4.5, CART** all use information-theoretic criteria
- Understanding IG helps you understand why certain features are chosen
- Leads to understanding of **Gini impurity** (a simpler alternative)

### Mathematical Definition

$$IG(Y, X) = H(Y) - H(Y|X) = H(Y) - \sum_{v \in \text{values}(X)} \frac{|S_v|}{|S|} H(Y|X=v)$$

Where:
- $H(Y)$ = entropy of target before split
- $H(Y|X=v)$ = entropy of target in subset where $X = v$
- $|S_v|/|S|$ = proportion of data with $X = v$

### Step-by-Step Example

```
Dataset: Play tennis? (14 examples)
─────────────────────────────────────
Overall: 9 Yes, 5 No → H(Y) = -9/14·log(9/14) - 5/14·log(5/14) = 0.940 bits

Split on "Outlook":
├── Sunny:   2 Yes, 3 No → H = 0.971 bits (5 examples)
├── Overcast: 4 Yes, 0 No → H = 0.000 bits (4 examples)  ← Pure!
└── Rain:    3 Yes, 2 No → H = 0.971 bits (5 examples)

H(Y|Outlook) = (5/14)·0.971 + (4/14)·0.000 + (5/14)·0.971 = 0.694

Information Gain = H(Y) - H(Y|Outlook) = 0.940 - 0.694 = 0.246 bits
```

```python
import numpy as np
from collections import Counter

def entropy_from_labels(labels):
    """Compute entropy from a list of class labels."""
    counter = Counter(labels)
    total = len(labels)
    probs = [count/total for count in counter.values()]
    return -sum(p * np.log2(p) for p in probs if p > 0)

def information_gain(parent_labels, splits):
    """
    Compute information gain for a split.
    
    Args:
        parent_labels: Labels before split
        splits: List of label arrays after split
    Returns:
        Information gain in bits
    """
    parent_entropy = entropy_from_labels(parent_labels)
    total = len(parent_labels)
    
    # Weighted child entropy
    child_entropy = sum(
        (len(split) / total) * entropy_from_labels(split)
        for split in splits
    )
    
    return parent_entropy - child_entropy

def gini_impurity(labels):
    """
    Gini impurity — alternative to entropy (used by CART/sklearn).
    Gini = 1 - sum(p_i^2)
    Computationally simpler, very similar results.
    """
    counter = Counter(labels)
    total = len(labels)
    return 1 - sum((count/total)**2 for count in counter.values())


# Tennis example
labels_all = ['Yes']*9 + ['No']*5

# Split by Outlook
sunny = ['Yes', 'Yes', 'No', 'No', 'No']
overcast = ['Yes', 'Yes', 'Yes', 'Yes']
rain = ['Yes', 'Yes', 'Yes', 'No', 'No']

ig_outlook = information_gain(labels_all, [sunny, overcast, rain])
print(f"Information Gain (Outlook): {ig_outlook:.4f} bits")

# Split by Wind
weak = ['Yes']*6 + ['No']*2
strong = ['Yes']*3 + ['No']*3

ig_wind = information_gain(labels_all, [weak, strong])
print(f"Information Gain (Wind): {ig_wind:.4f} bits")

print(f"\nBest split: {'Outlook' if ig_outlook > ig_wind else 'Wind'}")

# Compare Entropy vs Gini
print(f"\n=== Entropy vs Gini ===")
test_labels = ['A']*7 + ['B']*3
print(f"Labels: 7A, 3B")
print(f"Entropy: {entropy_from_labels(test_labels):.4f}")
print(f"Gini:    {gini_impurity(test_labels):.4f}")
```

### Entropy vs Gini Impurity

| Criterion | Formula | Range | Behavior |
|-----------|---------|-------|----------|
| Entropy | $-\sum p_i \log_2 p_i$ | [0, log₂(C)] | Slightly favors balanced splits |
| Gini | $1 - \sum p_i^2$ | [0, 1-1/C] | Computationally faster |

> 💡 **Pro Tip**: In practice, entropy and Gini give nearly identical trees. sklearn uses Gini by default because it's faster (no logarithm). Use entropy when you want to compute information gain explicitly.

---

## 6. Connections to ML Loss Functions

### The Unified View

Almost every loss function in ML has an information-theoretic interpretation:

| Loss Function | Information Theory View |
|--------------|----------------------|
| Cross-entropy loss | $H(p_{true}, q_{model})$ |
| Binary cross-entropy | $H(p, q)$ for Bernoulli |
| KL divergence loss | Distance between distributions |
| MSE (for Gaussians) | Equivalent to cross-entropy under Gaussian assumption |
| Focal loss | Weighted cross-entropy (focuses on hard examples) |
| Contrastive loss (InfoNCE) | Lower bound on mutual information |

### Why MSE ≈ Cross-Entropy for Regression

If we assume the model outputs are Gaussian-distributed:

$$p(y|x) = \mathcal{N}(f(x), \sigma^2)$$

Then negative log-likelihood = cross-entropy = constant + MSE:

$$-\log p(y|x) = \frac{(y - f(x))^2}{2\sigma^2} + \frac{1}{2}\log(2\pi\sigma^2)$$

Minimizing this w.r.t. $f(x)$ is equivalent to minimizing MSE!

### Label Smoothing — An Entropy Trick

```python
import numpy as np

def label_smoothing(one_hot_labels, num_classes, smoothing=0.1):
    """
    Label smoothing: Replace hard targets with soft targets.
    Instead of [0, 0, 1, 0] → [0.025, 0.025, 0.925, 0.025]
    
    Why? Prevents the model from being overconfident.
    Information-theoretic view: increases entropy of the target distribution.
    """
    # Hard label has entropy = 0
    # Smoothed label has entropy > 0 (more "uncertain")
    smooth_labels = one_hot_labels * (1 - smoothing) + smoothing / num_classes
    return smooth_labels

# Example
hard_label = np.array([0, 0, 1, 0])
smooth_label = label_smoothing(hard_label, num_classes=4, smoothing=0.1)

print(f"Hard label: {hard_label}")
print(f"Smooth label: {smooth_label}")
print(f"Entropy (hard): {-sum(p*np.log(p) for p in hard_label if p > 0):.4f}")
print(f"Entropy (smooth): {-sum(p*np.log(p) for p in smooth_label if p > 0):.4f}")
```

### Focal Loss — Information Theory Meets Class Imbalance

$$FL(p_t) = -\alpha_t (1 - p_t)^{\gamma} \log(p_t)$$

Where $p_t$ is the predicted probability of the true class.

**Information-theoretic view**: Down-weights the "easy" examples (low surprise) and focuses on "hard" examples (high surprise).

```python
import numpy as np

def focal_loss(y_true, y_pred, gamma=2.0, alpha=0.25):
    """
    Focal Loss — used in object detection (RetinaNet).
    Modifies cross-entropy to focus on hard examples.
    
    gamma=0: Standard cross-entropy
    gamma=2: Default (works well in practice)
    gamma=5: Very aggressive focus on hard examples
    """
    y_pred = np.clip(y_pred, 1e-7, 1 - 1e-7)
    
    # p_t = predicted probability of the TRUE class
    p_t = np.where(y_true == 1, y_pred, 1 - y_pred)
    alpha_t = np.where(y_true == 1, alpha, 1 - alpha)
    
    # Focal modulating factor: (1 - p_t)^gamma
    # Easy examples (p_t close to 1) → factor ≈ 0 → loss reduced
    # Hard examples (p_t close to 0) → factor ≈ 1 → loss unchanged
    focal_weight = (1 - p_t) ** gamma
    
    loss = -alpha_t * focal_weight * np.log(p_t)
    return loss

# Compare CE vs Focal Loss
p_t = np.linspace(0.01, 0.99, 100)
ce_loss = -np.log(p_t)
focal_2 = -((1 - p_t)**2) * np.log(p_t)
focal_5 = -((1 - p_t)**5) * np.log(p_t)

import matplotlib.pyplot as plt
plt.figure(figsize=(8, 5))
plt.plot(p_t, ce_loss, 'b-', linewidth=2, label='Cross-Entropy (γ=0)')
plt.plot(p_t, focal_2, 'r-', linewidth=2, label='Focal Loss (γ=2)')
plt.plot(p_t, focal_5, 'g-', linewidth=2, label='Focal Loss (γ=5)')
plt.xlabel('Predicted probability of true class (p_t)')
plt.ylabel('Loss')
plt.title('Focal Loss vs Cross-Entropy\n(Focal loss down-weights easy examples)')
plt.legend()
plt.grid(True)
plt.ylim(0, 5)
plt.show()
```

---

## 7. Advanced Topics

### ELBO (Evidence Lower Bound)

In variational inference, we can't compute $p(z|x)$ directly. Instead, we approximate it with $q(z|x)$ by maximizing:

$$\text{ELBO} = \mathbb{E}_{q(z|x)}[\log p(x|z)] - D_{KL}(q(z|x) \| p(z))$$

$$\log p(x) \geq \text{ELBO} \quad \text{(hence "lower bound")}$$

The gap between $\log p(x)$ and ELBO is exactly $D_{KL}(q(z|x) \| p(z|x))$.

### InfoNCE Loss (Contrastive Learning)

Used in SimCLR, CLIP, and other contrastive methods:

$$\mathcal{L}_{\text{InfoNCE}} = -\log \frac{\exp(\text{sim}(z_i, z_j)/\tau)}{\sum_{k=1}^{2N} \mathbb{1}_{[k \neq i]} \exp(\text{sim}(z_i, z_k)/\tau)}$$

**Information-theoretic interpretation**: InfoNCE is a **lower bound on mutual information** between the two views of the data.

```python
import numpy as np

def info_nce_loss(anchor, positive, negatives, temperature=0.5):
    """
    InfoNCE loss for contrastive learning.
    
    Maximizes mutual information between anchor and positive pair
    while minimizing it with negative pairs.
    
    Args:
        anchor: Embedding of anchor sample (d,)
        positive: Embedding of positive sample (d,)
        negatives: Embeddings of negative samples (K, d)
        temperature: Scaling factor (lower = sharper distribution)
    """
    # Cosine similarity
    def sim(a, b):
        return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
    
    # Positive similarity
    pos_sim = sim(anchor, positive) / temperature
    
    # Negative similarities
    neg_sims = np.array([sim(anchor, neg) / temperature for neg in negatives])
    
    # InfoNCE: log softmax over positive
    all_sims = np.concatenate([[pos_sim], neg_sims])
    loss = -pos_sim + np.log(np.sum(np.exp(all_sims)))
    
    return loss

# Example
d = 128  # Embedding dimension
anchor = np.random.randn(d)
positive = anchor + 0.1 * np.random.randn(d)  # Similar to anchor
negatives = np.random.randn(10, d)  # Random negatives

loss = info_nce_loss(anchor, positive, negatives)
print(f"InfoNCE Loss: {loss:.4f}")
print(f"Lower bound on MI between anchor and positive view")
```

---

## Common Mistakes

1. **Confusing KL divergence direction**: $D_{KL}(p \| q) \neq D_{KL}(q \| p)$. The first argument is what you're measuring "from," the second is the reference. In VAEs, we use $D_{KL}(q_\phi(z|x) \| p(z))$.

2. **Forgetting log base**: Entropy in bits (log₂) vs nats (ln). ML typically uses nats (natural log). When comparing values, check which base was used.

3. **Applying entropy to continuous variables naively**: Differential entropy can be negative and isn't directly comparable to discrete entropy. Use KL divergence instead when comparing continuous distributions.

4. **Thinking MI = Correlation**: Correlation only captures linear relationships. MI captures any statistical dependency. Zero correlation doesn't mean zero MI.

5. **Using cross-entropy loss with un-normalized logits**: PyTorch's `nn.CrossEntropyLoss` applies softmax internally. Don't apply softmax twice!

6. **Ignoring numerical stability**: Always use `log_softmax` instead of `log(softmax(x))` to avoid underflow/overflow.

---

## Interview Questions

**Q1: What is cross-entropy loss and why is it preferred over MSE for classification?**
> Cross-entropy $H(p, q) = -\sum p(x) \log q(x)$ measures how well predicted distribution $q$ matches true distribution $p$. It's preferred because: (1) gradient doesn't vanish when predictions are very wrong (unlike MSE + sigmoid), (2) it has direct probabilistic interpretation (negative log-likelihood), and (3) it's the natural loss for categorical distributions.

**Q2: Explain KL divergence and its asymmetry. Why does it matter in practice?**
> $D_{KL}(p\|q) = \sum p(x) \log(p(x)/q(x))$. It's not symmetric: forward KL ($D_{KL}(p\|q)$) is "mean-seeking" — the approximation $q$ tries to cover all modes of $p$. Reverse KL ($D_{KL}(q\|p)$) is "mode-seeking" — $q$ focuses on one mode. This matters in VAEs (reverse KL in ELBO leads to mode collapse) and in policy optimization (PPO clips KL to prevent large policy changes).

**Q3: How is mutual information used in feature selection?**
> MI measures the statistical dependency between a feature $X$ and target $Y$: $I(X;Y) = H(Y) - H(Y|X)$. Features with high MI are most informative. Unlike correlation, MI captures nonlinear relationships. sklearn's `mutual_info_classif` estimates this non-parametrically.

**Q4: What is the ELBO in variational inference?**
> ELBO = $\mathbb{E}_q[\log p(x|z)] - D_{KL}(q(z|x) \| p(z))$. It's a lower bound on the log-evidence $\log p(x)$. Maximizing ELBO simultaneously trains the decoder (first term: reconstruction) and regularizes the encoder (second term: keep $q$ close to prior). The gap between ELBO and true evidence is $D_{KL}(q(z|x) \| p(z|x))$.

**Q5: Explain the relationship: Cross-Entropy = Entropy + KL Divergence.**
> $H(p, q) = H(p) + D_{KL}(p\|q)$. Since $H(p)$ is fixed (property of the true data), minimizing cross-entropy w.r.t. model $q$ is equivalent to minimizing KL divergence. This is why cross-entropy loss trains models to match the true distribution.

---

## Quick Reference

| Concept | Formula | Interpretation |
|---------|---------|---------------|
| Entropy | $H(X) = -\sum p(x) \log p(x)$ | Average surprise / uncertainty |
| Cross-Entropy | $H(p,q) = -\sum p(x) \log q(x)$ | Bits to encode $p$ using code for $q$ |
| KL Divergence | $D_{KL}(p\|q) = \sum p \log(p/q)$ | Extra bits due to using $q$ instead of $p$ |
| Mutual Info | $I(X;Y) = H(X) - H(X|Y)$ | Dependency between X and Y |
| Info Gain | $IG = H(Y) - H(Y|X)$ | Entropy reduction from knowing X |
| Relationship | $H(p,q) = H(p) + D_{KL}(p\|q)$ | Decomposition of cross-entropy |
| Binary CE | $-[y\log\hat{y} + (1-y)\log(1-\hat{y})]$ | Classification loss (2 classes) |
| Gaussian KL | $\frac{1}{2}(\mu^2 + \sigma^2 - \log\sigma^2 - 1)$ | VAE regularization |

---

*Previous: [Chapter 04 - Optimization Theory](04-Optimization-Theory.md) | Next: [Chapter 06 - Math Behind Key Algorithms](06-Math-Behind-Key-Algorithms.md)*
