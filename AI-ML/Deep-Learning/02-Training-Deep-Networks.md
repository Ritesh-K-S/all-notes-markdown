# Chapter 02: Training Deep Networks

> **Loss functions, optimizers, learning rate scheduling, and the art of making neural networks actually learn.**

---

## Table of Contents
1. [The Training Pipeline](#1-the-training-pipeline)
2. [Loss Functions In Depth](#2-loss-functions-in-depth)
3. [Optimizers — Beyond Basic SGD](#3-optimizers--beyond-basic-sgd)
4. [Learning Rate Scheduling](#4-learning-rate-scheduling)
5. [Weight Initialization](#5-weight-initialization)
6. [Training Diagnostics](#6-training-diagnostics)
7. [Practical Training Recipes](#7-practical-training-recipes)
8. [Common Mistakes](#8-common-mistakes)
9. [Interview Questions](#9-interview-questions)
10. [Quick Reference](#10-quick-reference)

---

## 1. The Training Pipeline

### What It Is
Training a neural network is an iterative process of showing the network data, measuring how wrong it is, and adjusting weights to be less wrong. One complete pass through the training data is called an **epoch**.

### The Big Picture

```
┌──────────────────────────────────────────────────────────────────┐
│                    TRAINING LOOP (per epoch)                      │
│                                                                  │
│  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌───────────┐   │
│  │  Load   │    │ Forward  │    │ Compute  │    │ Backward  │   │
│  │  Batch  │───→│  Pass    │───→│  Loss    │───→│  Pass     │   │
│  └─────────┘    └──────────┘    └──────────┘    └───────────┘   │
│                                                       │          │
│                                                       ▼          │
│                                               ┌───────────┐     │
│                                               │  Update   │     │
│                                               │  Weights  │     │
│                                               └───────────┘     │
│                                                                  │
│  Repeat for all batches → 1 epoch                                │
│  Repeat for N epochs → training complete                         │
└──────────────────────────────────────────────────────────────────┘
```

### Key Terminology

| Term | Definition |
|------|-----------|
| **Epoch** | One complete pass through the entire training dataset |
| **Batch** | A subset of training data processed together |
| **Iteration/Step** | One weight update (processing one batch) |
| **Training Loss** | Error on training data (should decrease) |
| **Validation Loss** | Error on held-out data (monitors overfitting) |
| **Convergence** | When loss stops meaningfully decreasing |

**Relationship:**
$$\text{Steps per epoch} = \lceil\frac{\text{Total samples}}{\text{Batch size}}\rceil$$

Example: 50,000 samples, batch_size=32 → 1,563 steps per epoch

---

## 2. Loss Functions In Depth

### Why the Choice of Loss Function Matters

The loss function is the **objective** your network optimizes. Choose wrong, and your network optimizes the wrong thing — it's like giving someone directions to the wrong city.

### Regression Losses

#### Mean Squared Error (MSE / L2 Loss)

$$\mathcal{L}_{MSE} = \frac{1}{n}\sum_{i=1}^{n}(y_i - \hat{y}_i)^2$$

```python
import torch
import torch.nn as nn
import numpy as np

# PyTorch
mse_loss = nn.MSELoss()
y_pred = torch.tensor([2.5, 0.0, 2.1, 7.8])
y_true = torch.tensor([3.0, -0.5, 2.0, 7.5])
loss = mse_loss(y_pred, y_true)
print(f"MSE Loss: {loss.item():.4f}")  # 0.1025

# Properties:
# - Penalizes large errors heavily (quadratic penalty)
# - Sensitive to outliers
# - Gradient: 2(ŷ - y)/n → larger error = larger gradient
```

**When to use:** Standard regression. When you want to penalize large errors more than small ones.

**Problems:** Very sensitive to outliers. One extreme value can dominate the loss.

#### Mean Absolute Error (MAE / L1 Loss)

$$\mathcal{L}_{MAE} = \frac{1}{n}\sum_{i=1}^{n}|y_i - \hat{y}_i|$$

```python
mae_loss = nn.L1Loss()
loss = mae_loss(y_pred, y_true)
print(f"MAE Loss: {loss.item():.4f}")  # 0.2750

# Properties:
# - Linear penalty — robust to outliers
# - Gradient is constant (±1) — doesn't slow down near optimum
# - Not differentiable at 0 (subgradient used)
```

**When to use:** When your data has outliers. Predicting medians rather than means.

#### Huber Loss (Smooth L1) — Best of Both Worlds

$$\mathcal{L}_{Huber} = \begin{cases} \frac{1}{2}(y - \hat{y})^2 & \text{if } |y - \hat{y}| \leq \delta \\ \delta|y - \hat{y}| - \frac{1}{2}\delta^2 & \text{otherwise} \end{cases}$$

```python
huber_loss = nn.HuberLoss(delta=1.0)
loss = huber_loss(y_pred, y_true)
print(f"Huber Loss: {loss.item():.4f}")

# Properties:
# - MSE for small errors (smooth, differentiable)
# - MAE for large errors (robust to outliers)
# - δ controls the transition point
```

```
Loss
  │    MSE /
  │       / 
  │      /  Huber
  │     / _/─────────── MAE (linear)
  │    //
  │   //
  │  //
  │ /
  └──────────────────── |error|
       δ
```

**When to use:** Object detection (bounding box regression), any regression with occasional outliers.

### Classification Losses

#### Binary Cross-Entropy (BCE)

$$\mathcal{L}_{BCE} = -\frac{1}{n}\sum_{i=1}^{n}\left[y_i\log(\hat{y}_i) + (1-y_i)\log(1-\hat{y}_i)\right]$$

```python
# Standard BCE (requires sigmoid output)
bce_loss = nn.BCELoss()
y_pred = torch.sigmoid(torch.tensor([0.8, -0.5, 1.2, -1.0]))  # Apply sigmoid first
y_true = torch.tensor([1.0, 0.0, 1.0, 0.0])
loss = bce_loss(y_pred, y_true)
print(f"BCE Loss: {loss.item():.4f}")

# BCEWithLogitsLoss = Sigmoid + BCE (numerically stable!)
bce_logits = nn.BCEWithLogitsLoss()
logits = torch.tensor([0.8, -0.5, 1.2, -1.0])  # Raw logits, no sigmoid needed
loss = bce_logits(logits, y_true)
print(f"BCE+Logits Loss: {loss.item():.4f}")
```

> **Pro Tip:** ALWAYS use `BCEWithLogitsLoss` over `BCELoss`. It's numerically stable because it combines the sigmoid and log operations using the log-sum-exp trick, avoiding log(0) issues.

#### Categorical Cross-Entropy (Multi-class)

$$\mathcal{L}_{CE} = -\sum_{c=1}^{C} y_c \log(\hat{y}_c)$$

For one-hot labels, this simplifies to:
$$\mathcal{L}_{CE} = -\log(\hat{y}_{true\_class})$$

```python
# PyTorch: CrossEntropyLoss = LogSoftmax + NLLLoss
# Input: raw logits (NOT softmax output!)
# Target: class indices (NOT one-hot!)
ce_loss = nn.CrossEntropyLoss()
logits = torch.tensor([[2.0, 1.0, 0.1],    # Prediction for sample 1
                       [0.5, 2.5, 0.3]])   # Prediction for sample 2
targets = torch.tensor([0, 1])              # True classes (indices)
loss = ce_loss(logits, targets)
print(f"CE Loss: {loss.item():.4f}")  # 0.4170

# With class weights (for imbalanced data)
weights = torch.tensor([1.0, 2.0, 1.0])  # Weight rare classes higher
ce_weighted = nn.CrossEntropyLoss(weight=weights)
```

> **Critical PyTorch gotcha:** `nn.CrossEntropyLoss` expects RAW LOGITS, not softmax probabilities. It applies log-softmax internally. Passing softmax output will give wrong results!

#### Focal Loss — For Extreme Class Imbalance

$$\mathcal{L}_{FL} = -\alpha_t (1 - p_t)^\gamma \log(p_t)$$

```python
class FocalLoss(nn.Module):
    """Focal Loss for handling class imbalance.
    Down-weights easy examples, focuses on hard ones.
    """
    def __init__(self, alpha=0.25, gamma=2.0):
        super().__init__()
        self.alpha = alpha   # Balancing factor
        self.gamma = gamma   # Focusing parameter (higher = more focus on hard examples)
    
    def forward(self, logits, targets):
        # Convert logits to probabilities
        probs = torch.sigmoid(logits)
        
        # Get probability of true class
        p_t = probs * targets + (1 - probs) * (1 - targets)
        
        # Focal weight: (1 - p_t)^gamma
        # Easy examples (high p_t) get very low weight
        # Hard examples (low p_t) get high weight
        focal_weight = (1 - p_t) ** self.gamma
        
        # Standard BCE
        bce = -(targets * torch.log(probs + 1e-8) + 
                (1 - targets) * torch.log(1 - probs + 1e-8))
        
        loss = self.alpha * focal_weight * bce
        return loss.mean()

# Usage: Great for object detection (many background vs few objects)
focal = FocalLoss(alpha=0.25, gamma=2.0)
```

**When to use:** Object detection, medical imaging, fraud detection — anywhere with 99%+ negative class.

#### Label Smoothing

Instead of hard targets [0, 0, 1, 0], use soft targets [0.033, 0.033, 0.9, 0.033]:

$$y_{smooth} = (1 - \epsilon) \cdot y_{hard} + \frac{\epsilon}{K}$$

```python
# PyTorch native support
ce_smooth = nn.CrossEntropyLoss(label_smoothing=0.1)
loss = ce_smooth(logits, targets)

# Why? Prevents overconfident predictions, improves generalization
# The network learns "I'm 90% sure" instead of "I'm 100% sure"
```

### Loss Function Comparison Table

| Loss | Task | Sensitive to Outliers | Gradient Behavior |
|------|------|----------------------|-------------------|
| MSE | Regression | Yes (quadratic) | Proportional to error |
| MAE | Regression | No (linear) | Constant magnitude |
| Huber | Regression | Moderate | Adaptive |
| BCE | Binary classification | N/A | Clean gradient (ŷ-y) |
| CE | Multi-class | N/A | Clean gradient |
| Focal | Imbalanced classification | N/A | Focuses on hard examples |

---

## 3. Optimizers — Beyond Basic SGD

### Why Optimizers Matter
The optimizer determines HOW weights are updated given the gradients. A bad optimizer can mean the difference between training converging in 10 epochs vs. 10,000 — or never converging at all.

### The Evolution of Optimizers

```
SGD → SGD+Momentum → Nesterov → AdaGrad → RMSProp → Adam → AdamW
 │         │              │          │          │        │       │
 │    Add velocity    Look ahead  Per-param   Decay   Combine  Fix weight
 │                                  lr        old      Momentum decay
 │                                            grads    + RMSProp
 ▼
Simple but slow ──────────────────────────────────────→ Fast and adaptive
```

### SGD (Stochastic Gradient Descent)

$$W_{t+1} = W_t - \alpha \cdot g_t$$

Where $g_t = \nabla_W \mathcal{L}(W_t)$

```python
optimizer = torch.optim.SGD(model.parameters(), lr=0.01)
```

- ✅ Simple, well-understood, great generalization
- ❌ Slow convergence, oscillates in narrow valleys
- ❌ Same learning rate for all parameters

### SGD with Momentum

**Analogy:** A ball rolling downhill. It accumulates velocity and can roll through small bumps (local minima).

$$v_{t+1} = \beta \cdot v_t + g_t$$
$$W_{t+1} = W_t - \alpha \cdot v_{t+1}$$

Where $\beta$ is the momentum coefficient (typically 0.9).

```python
optimizer = torch.optim.SGD(model.parameters(), lr=0.01, momentum=0.9)
```

```
WITHOUT MOMENTUM:              WITH MOMENTUM:
(oscillates in ravines)        (smooth path)

    ╲  /╲  /╲                     ╲
     ╲/  ╲/  ╲                     ╲
              ╲___                   ╲____
                  ╲___                    ╲________
                                                   ───→ minimum
```

- ✅ Faster convergence, dampens oscillations
- ✅ Can escape shallow local minima
- The momentum term acts like a moving average of gradients

### Nesterov Accelerated Gradient (NAG)

**Key idea:** Look ahead! Compute gradient at the position you're ABOUT to be at.

$$v_{t+1} = \beta \cdot v_t + \nabla_W \mathcal{L}(W_t - \alpha \beta v_t)$$
$$W_{t+1} = W_t - \alpha \cdot v_{t+1}$$

```python
optimizer = torch.optim.SGD(model.parameters(), lr=0.01, momentum=0.9, nesterov=True)
```

- ✅ Converges faster than standard momentum
- ✅ Better at slowing down before overshooting

### AdaGrad (Adaptive Gradient)

**Key idea:** Parameters that get frequent large gradients should have smaller learning rates (they've already learned a lot).

$$G_t = G_{t-1} + g_t^2 \quad \text{(accumulate squared gradients)}$$
$$W_{t+1} = W_t - \frac{\alpha}{\sqrt{G_t + \epsilon}} \cdot g_t$$

```python
optimizer = torch.optim.Adagrad(model.parameters(), lr=0.01)
```

- ✅ Per-parameter learning rate adaptation
- ✅ Great for sparse data (NLP embeddings)
- ❌ Learning rate always decreases → eventually stops learning (accumulated G grows forever)

### RMSProp (Root Mean Square Propagation)

**Key idea:** Fix AdaGrad's problem by using exponential moving average instead of sum.

$$G_t = \beta \cdot G_{t-1} + (1-\beta) \cdot g_t^2$$
$$W_{t+1} = W_t - \frac{\alpha}{\sqrt{G_t + \epsilon}} \cdot g_t$$

```python
optimizer = torch.optim.RMSprop(model.parameters(), lr=0.001, alpha=0.99)
```

- ✅ Doesn't kill learning rate like AdaGrad
- ✅ Good for RNNs and non-stationary problems
- Invented by Hinton in a Coursera lecture (never formally published!)

### Adam (Adaptive Moment Estimation)

**Key idea:** Combine Momentum (first moment) + RMSProp (second moment) + bias correction.

$$m_t = \beta_1 m_{t-1} + (1-\beta_1) g_t \quad \text{(first moment — mean of gradients)}$$
$$v_t = \beta_2 v_{t-1} + (1-\beta_2) g_t^2 \quad \text{(second moment — variance of gradients)}$$
$$\hat{m}_t = \frac{m_t}{1-\beta_1^t} \quad \text{(bias correction)}$$
$$\hat{v}_t = \frac{v_t}{1-\beta_2^t} \quad \text{(bias correction)}$$
$$W_{t+1} = W_t - \frac{\alpha \cdot \hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon}$$

Default hyperparameters: $\beta_1 = 0.9$, $\beta_2 = 0.999$, $\epsilon = 10^{-8}$

```python
optimizer = torch.optim.Adam(model.parameters(), lr=0.001, betas=(0.9, 0.999))
```

**Why bias correction?** At the start, $m_0 = 0$ and $v_0 = 0$, so early estimates are biased toward zero. The correction $\frac{m_t}{1-\beta_1^t}$ compensates for this (when t is small, $\beta_1^t$ is large, making the correction large).

- ✅ Fast convergence
- ✅ Works well out of the box
- ✅ Per-parameter adaptive learning rate
- ❌ Can generalize worse than SGD+momentum for some tasks
- ❌ Higher memory (stores m and v for each parameter)

### AdamW (Adam with Decoupled Weight Decay)

**Key insight:** L2 regularization and weight decay are NOT the same thing with adaptive optimizers! Adam's original L2 implementation is wrong.

$$W_{t+1} = W_t - \alpha\left(\frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon} + \lambda W_t\right)$$

```python
optimizer = torch.optim.AdamW(model.parameters(), lr=0.001, weight_decay=0.01)
```

- ✅ Properly implements weight decay for Adam
- ✅ Better generalization than Adam with L2
- **This is what you should use by default in 2024+**

### Code: Comparing Optimizers

```python
import torch
import torch.nn as nn
import torch.optim as optim
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
import numpy as np

# Generate data
X, y = make_classification(n_samples=5000, n_features=20, random_state=42)
X_train, X_val, y_train, y_val = train_test_split(X, y, test_size=0.2, random_state=42)
X_train_t = torch.FloatTensor(X_train)
y_train_t = torch.LongTensor(y_train)
X_val_t = torch.FloatTensor(X_val)
y_val_t = torch.LongTensor(y_val)

def create_model():
    """Fresh model for each optimizer test."""
    torch.manual_seed(42)
    return nn.Sequential(
        nn.Linear(20, 64), nn.ReLU(),
        nn.Linear(64, 32), nn.ReLU(),
        nn.Linear(32, 2)
    )

def train_with_optimizer(optimizer_name, optimizer_fn, n_epochs=50):
    """Train and record loss curve."""
    model = create_model()
    optimizer = optimizer_fn(model.parameters())
    criterion = nn.CrossEntropyLoss()
    
    losses = []
    for epoch in range(n_epochs):
        model.train()
        optimizer.zero_grad()
        output = model(X_train_t)
        loss = criterion(output, y_train_t)
        loss.backward()
        optimizer.step()
        losses.append(loss.item())
    
    # Final validation accuracy
    model.eval()
    with torch.no_grad():
        val_preds = model(X_val_t).argmax(dim=1)
        val_acc = (val_preds == y_val_t).float().mean().item()
    
    print(f"{optimizer_name:15s} | Final Loss: {losses[-1]:.4f} | Val Acc: {val_acc:.4f}")
    return losses

# Compare optimizers
results = {}
results['SGD'] = train_with_optimizer(
    'SGD', lambda p: optim.SGD(p, lr=0.01))
results['SGD+Momentum'] = train_with_optimizer(
    'SGD+Momentum', lambda p: optim.SGD(p, lr=0.01, momentum=0.9))
results['RMSProp'] = train_with_optimizer(
    'RMSProp', lambda p: optim.RMSprop(p, lr=0.001))
results['Adam'] = train_with_optimizer(
    'Adam', lambda p: optim.Adam(p, lr=0.001))
results['AdamW'] = train_with_optimizer(
    'AdamW', lambda p: optim.AdamW(p, lr=0.001, weight_decay=0.01))

# Expected output:
# SGD             | Final Loss: 0.4521 | Val Acc: 0.8320
# SGD+Momentum    | Final Loss: 0.2103 | Val Acc: 0.9040
# RMSProp         | Final Loss: 0.0812 | Val Acc: 0.9320
# Adam            | Final Loss: 0.0534 | Val Acc: 0.9400
# AdamW           | Final Loss: 0.0601 | Val Acc: 0.9380
```

### Optimizer Selection Guide

| Scenario | Recommended Optimizer |
|----------|----------------------|
| Default / quick experiments | AdamW (lr=0.001) |
| Computer Vision (CNNs) | SGD + Momentum + Cosine LR |
| NLP / Transformers | AdamW + Linear Warmup |
| Training from scratch | SGD + Momentum (better generalization) |
| Fine-tuning pretrained | AdamW (lower lr: 1e-5 to 3e-5) |
| Sparse gradients (embeddings) | Adam or AdaGrad |
| Research (reproducibility) | SGD + Momentum + Step LR |

> **Industry secret:** For state-of-the-art results in vision, top papers often use SGD with momentum and careful LR scheduling. Adam converges faster but SGD often finds flatter minima that generalize better. Many practitioners use Adam for prototyping, then switch to SGD for final training.

---

## 4. Learning Rate Scheduling

### What It Is
A learning rate schedule changes the learning rate during training. Start high (explore broadly) and decrease (fine-tune into minimum).

### Why It Matters
- Fixed LR is a compromise: too high = oscillate, too low = slow
- Scheduling gives you the best of both worlds
- Can mean the difference between 90% and 95% accuracy

### Common Schedules

#### Step Decay
Multiply LR by a factor every N epochs.

```python
# Reduce LR by 10x every 30 epochs
scheduler = torch.optim.lr_scheduler.StepLR(optimizer, step_size=30, gamma=0.1)

# Training loop:
for epoch in range(100):
    train_one_epoch()
    scheduler.step()  # Call AFTER optimizer.step()
```

```
LR
0.01 |████████████████████
0.001|                    ████████████████████
0.0001                                       ████████████████████
     └──────────────────────────────────────────────────────────── Epochs
          0           30          60          90
```

#### Cosine Annealing
Smoothly decreases LR following a cosine curve. Very popular.

$$\eta_t = \eta_{min} + \frac{1}{2}(\eta_{max} - \eta_{min})\left(1 + \cos\left(\frac{t}{T_{max}}\pi\right)\right)$$

```python
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(
    optimizer, T_max=100, eta_min=1e-6
)
```

```
LR
0.01 |╲
     |  ╲
     |    ╲
     |      ╲
     |         ╲
     |            ───╲
     |                  ───╲___
0.0  |                         ────────
     └──────────────────────────────── Epochs
```

#### Cosine Annealing with Warm Restarts
Reset LR periodically (restart cosine from high LR).

```python
scheduler = torch.optim.lr_scheduler.CosineAnnealingWarmRestarts(
    optimizer, T_0=10, T_mult=2  # First cycle: 10 epochs, then 20, 40, ...
)
```

```
LR
0.01 |╲   ╲       ╲
     | ╲   ╲       ╲
     |  ╲    ╲       ╲
     |   ╲     ╲       ╲
     |    ╲      ╲        ╲
     |     ╲       ╲         ───╲
0.0  |      ╲        ───╲         ────
     └──────────────────────────────── Epochs
      T=10    T=20         T=40
```

#### Linear Warmup + Decay (Transformer Standard)

```python
from torch.optim.lr_scheduler import LambdaLR

def get_linear_warmup_scheduler(optimizer, warmup_steps, total_steps):
    """Linear warmup followed by linear decay. Standard for transformers."""
    def lr_lambda(current_step):
        if current_step < warmup_steps:
            # Warmup: linearly increase from 0 to 1
            return float(current_step) / float(max(1, warmup_steps))
        else:
            # Decay: linearly decrease from 1 to 0
            return max(0.0, float(total_steps - current_step) / 
                       float(max(1, total_steps - warmup_steps)))
    
    return LambdaLR(optimizer, lr_lambda)

# Usage
optimizer = torch.optim.AdamW(model.parameters(), lr=5e-5)
scheduler = get_linear_warmup_scheduler(optimizer, warmup_steps=1000, total_steps=10000)

# In training loop:
for step in range(10000):
    loss = train_step()
    optimizer.step()
    scheduler.step()  # Step per iteration, not per epoch!
```

```
LR
5e-5 |     /─────╲
     |    /        ╲
     |   /           ╲
     |  /              ╲
     | /                 ╲
     |/                    ╲
0    |                       ╲___
     └──────────────────────────────── Steps
     0   1000              10000
      warmup     decay
```

#### ReduceLROnPlateau (Adaptive)
Reduce LR when metric stops improving. No schedule to tune!

```python
scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(
    optimizer, 
    mode='min',        # Minimize the metric (loss)
    factor=0.5,        # Multiply LR by 0.5
    patience=5,        # Wait 5 epochs before reducing
    min_lr=1e-7        # Don't go below this
)

# Usage: pass the metric to monitor
for epoch in range(100):
    train_loss = train_one_epoch()
    val_loss = validate()
    scheduler.step(val_loss)  # Pass the metric!
```

### Learning Rate Finder

The LR finder (by Leslie Smith) helps find the optimal starting LR:

```python
def lr_finder(model, train_loader, criterion, optimizer, start_lr=1e-7, end_lr=10, num_steps=100):
    """
    Train with exponentially increasing LR, plot loss vs LR.
    Best LR is where loss decreases fastest (steepest slope).
    """
    # Store original state
    original_state = model.state_dict()
    
    lrs = np.logspace(np.log10(start_lr), np.log10(end_lr), num_steps)
    losses = []
    
    model.train()
    data_iter = iter(train_loader)
    
    for lr in lrs:
        # Set learning rate
        for param_group in optimizer.param_groups:
            param_group['lr'] = lr
        
        # Get batch (cycle if needed)
        try:
            X_batch, y_batch = next(data_iter)
        except StopIteration:
            data_iter = iter(train_loader)
            X_batch, y_batch = next(data_iter)
        
        # Forward + backward
        optimizer.zero_grad()
        output = model(X_batch)
        loss = criterion(output, y_batch)
        loss.backward()
        optimizer.step()
        
        losses.append(loss.item())
        
        # Stop if loss explodes
        if loss.item() > 4 * losses[0]:
            break
    
    # Restore original weights
    model.load_state_dict(original_state)
    
    # Best LR: where loss drops fastest
    # Use ~1/10 of the LR where loss is minimum
    min_idx = np.argmin(losses[:len(lrs)])
    suggested_lr = lrs[min_idx] / 10
    
    print(f"Suggested LR: {suggested_lr:.2e}")
    return lrs[:len(losses)], losses

# Usage:
# lrs, losses = lr_finder(model, train_loader, criterion, optimizer)
# Then plot lrs vs losses on log scale
```

---

## 5. Weight Initialization

### What It Is
Weight initialization sets the starting values for all network parameters before training begins. It's like choosing your starting position in a maze — bad starts can make the maze unsolvable.

### Why It Matters Enormously

```
BAD INITIALIZATION:
Layer 1 output variance: 100
Layer 2 output variance: 10,000     → EXPLODING activations!
Layer 3 output variance: 1,000,000  → NaN loss

OR:
Layer 1 output variance: 0.01
Layer 2 output variance: 0.0001     → VANISHING activations!
Layer 3 output variance: 0.000001   → Network learns nothing
```

**Goal:** Keep variance of activations roughly constant across layers.

### Initialization Strategies

#### Zero Initialization — NEVER DO THIS

```python
# ❌ CATASTROPHIC: All neurons learn the same thing (symmetry problem)
nn.init.zeros_(layer.weight)
# Every neuron computes the same output → same gradient → same update
# The network effectively has 1 neuron per layer
```

#### Random Normal — Naive but Illustrative

```python
# ❌ BAD for deep networks: variance grows or shrinks per layer
nn.init.normal_(layer.weight, mean=0, std=0.01)
```

#### Xavier/Glorot Initialization (2010)

**For Sigmoid and Tanh activations.**

$$W \sim \mathcal{N}\left(0, \frac{2}{n_{in} + n_{out}}\right) \quad \text{or} \quad W \sim U\left(-\sqrt{\frac{6}{n_{in} + n_{out}}}, \sqrt{\frac{6}{n_{in} + n_{out}}}\right)$$

```python
# Xavier Normal
nn.init.xavier_normal_(layer.weight)

# Xavier Uniform
nn.init.xavier_uniform_(layer.weight)
```

**Intuition:** The variance is scaled by both input AND output size, maintaining activation variance in both forward and backward passes.

#### He/Kaiming Initialization (2015)

**For ReLU and its variants.**

$$W \sim \mathcal{N}\left(0, \frac{2}{n_{in}}\right)$$

```python
# He Normal (most common for ReLU networks)
nn.init.kaiming_normal_(layer.weight, mode='fan_in', nonlinearity='relu')

# He Uniform
nn.init.kaiming_uniform_(layer.weight, mode='fan_in', nonlinearity='relu')
```

**Why factor of 2?** ReLU kills half the activations (sets negative to 0), so we need twice the variance to compensate.

#### Initialization Selection Guide

| Activation | Initialization | Formula |
|-----------|---------------|---------|
| Sigmoid/Tanh | Xavier/Glorot | $\text{Var} = \frac{2}{n_{in} + n_{out}}$ |
| ReLU | He/Kaiming | $\text{Var} = \frac{2}{n_{in}}$ |
| Leaky ReLU | He (adjusted) | $\text{Var} = \frac{2}{(1+\alpha^2) \cdot n_{in}}$ |
| SELU | LeCun | $\text{Var} = \frac{1}{n_{in}}$ |

### Code: Custom Initialization

```python
import torch.nn as nn

def init_weights(model):
    """Apply proper initialization to all layers."""
    for module in model.modules():
        if isinstance(module, nn.Linear):
            # He initialization for ReLU layers
            nn.init.kaiming_normal_(module.weight, mode='fan_in', nonlinearity='relu')
            if module.bias is not None:
                nn.init.zeros_(module.bias)  # Biases always start at 0
        elif isinstance(module, nn.Conv2d):
            nn.init.kaiming_normal_(module.weight, mode='fan_out', nonlinearity='relu')
            if module.bias is not None:
                nn.init.zeros_(module.bias)
        elif isinstance(module, nn.BatchNorm2d):
            nn.init.ones_(module.weight)   # Scale (gamma) = 1
            nn.init.zeros_(module.bias)    # Shift (beta) = 0

# Apply to model
model = nn.Sequential(
    nn.Linear(784, 256), nn.ReLU(),
    nn.Linear(256, 128), nn.ReLU(),
    nn.Linear(128, 10)
)
model.apply(init_weights)

# Verify variance is maintained
with torch.no_grad():
    x = torch.randn(1000, 784)
    for i, layer in enumerate(model):
        x = layer(x)
        if isinstance(layer, nn.Linear):
            print(f"Layer {i} output — mean: {x.mean():.4f}, std: {x.std():.4f}")
```

---

## 6. Training Diagnostics

### What It Is
Training diagnostics help you understand whether your network is learning correctly, overfitting, underfitting, or has other issues.

### The Learning Curves

```
HEALTHY TRAINING:          OVERFITTING:              UNDERFITTING:
Loss                       Loss                      Loss
│╲                         │╲                        │╲_____
│ ╲                        │ ╲___ val loss ↑         │      ╲_____ both high
│  ╲___train               │     ╲___╱╲  ↗          │            ╲___
│      ╲___                │  ╲                      │
│          ╲___val         │   ╲_______train ↓       │
└───────────────Epochs     └───────────────Epochs    └───────────────Epochs
Gap is small               Gap is LARGE              Both losses are HIGH
```

### Diagnostic Checklist

```python
class TrainingMonitor:
    """Monitor training for common issues."""
    
    def __init__(self):
        self.train_losses = []
        self.val_losses = []
        self.train_accs = []
        self.val_accs = []
        self.learning_rates = []
        self.gradient_norms = []
    
    def log_epoch(self, train_loss, val_loss, train_acc, val_acc, lr, grad_norm):
        self.train_losses.append(train_loss)
        self.val_losses.append(val_loss)
        self.train_accs.append(train_acc)
        self.val_accs.append(val_acc)
        self.learning_rates.append(lr)
        self.gradient_norms.append(grad_norm)
    
    def diagnose(self):
        """Automatically diagnose training issues."""
        issues = []
        
        # Check for exploding loss
        if any(np.isnan(self.train_losses) | np.isinf(self.train_losses)):
            issues.append("🚨 EXPLODING LOSS: Reduce learning rate or check data")
        
        # Check for overfitting
        if len(self.val_losses) > 10:
            recent_gap = np.mean(self.val_losses[-5:]) - np.mean(self.train_losses[-5:])
            if recent_gap > 0.1 * np.mean(self.train_losses[-5:]):
                issues.append("⚠️ OVERFITTING: Val loss >> Train loss. Add regularization.")
        
        # Check for underfitting
        if len(self.train_losses) > 20:
            if self.train_losses[-1] > 0.9 * self.train_losses[10]:
                issues.append("⚠️ UNDERFITTING: Training loss not decreasing. Increase capacity or LR.")
        
        # Check for vanishing gradients
        if len(self.gradient_norms) > 5:
            if np.mean(self.gradient_norms[-5:]) < 1e-7:
                issues.append("🚨 VANISHING GRADIENTS: Use ReLU, better init, or skip connections.")
        
        # Check for exploding gradients
        if len(self.gradient_norms) > 5:
            if np.mean(self.gradient_norms[-5:]) > 1000:
                issues.append("🚨 EXPLODING GRADIENTS: Add gradient clipping.")
        
        if not issues:
            issues.append("✅ Training looks healthy!")
        
        return issues

def compute_gradient_norm(model):
    """Compute total gradient norm across all parameters."""
    total_norm = 0
    for p in model.parameters():
        if p.grad is not None:
            total_norm += p.grad.data.norm(2).item() ** 2
    return total_norm ** 0.5
```

### Gradient Clipping

When gradients explode, clip them:

```python
# Clip by norm (most common)
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

# Clip by value
torch.nn.utils.clip_grad_value_(model.parameters(), clip_value=0.5)

# Usage in training loop:
optimizer.zero_grad()
loss.backward()
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)  # AFTER backward, BEFORE step
optimizer.step()
```

### Dead Neurons Detection

```python
def check_dead_neurons(model, X_sample):
    """Check for dead ReLU neurons (always output 0)."""
    model.eval()
    hooks = []
    activations = {}
    
    def hook_fn(name):
        def hook(module, input, output):
            activations[name] = output.detach()
        return hook
    
    # Register hooks on ReLU layers
    for name, module in model.named_modules():
        if isinstance(module, nn.ReLU):
            hooks.append(module.register_forward_hook(hook_fn(name)))
    
    # Forward pass
    with torch.no_grad():
        model(X_sample)
    
    # Check each activation
    for name, act in activations.items():
        dead_fraction = (act == 0).float().mean().item()
        if dead_fraction > 0.9:
            print(f"⚠️ Layer '{name}': {dead_fraction*100:.1f}% neurons are dead!")
        else:
            print(f"✅ Layer '{name}': {dead_fraction*100:.1f}% dead neurons (normal)")
    
    # Remove hooks
    for h in hooks:
        h.remove()
```

---

## 7. Practical Training Recipes

### Recipe 1: Standard Image Classification

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.optim.lr_scheduler import CosineAnnealingLR

# Model
model = create_model()  # Your architecture

# Optimizer: SGD with momentum for vision tasks
optimizer = optim.SGD(
    model.parameters(),
    lr=0.1,           # High starting LR with cosine decay
    momentum=0.9,
    weight_decay=1e-4  # L2 regularization
)

# Scheduler: Cosine annealing (no restarts)
scheduler = CosineAnnealingLR(optimizer, T_max=200, eta_min=0)

# Training
for epoch in range(200):
    model.train()
    for batch_x, batch_y in train_loader:
        optimizer.zero_grad()
        output = model(batch_x)
        loss = nn.CrossEntropyLoss()(output, batch_y)
        loss.backward()
        optimizer.step()
    
    scheduler.step()
```

### Recipe 2: Fine-tuning a Pretrained Model

```python
# Different learning rates for different parts
pretrained_params = []
new_params = []

for name, param in model.named_parameters():
    if 'classifier' in name or 'fc' in name:
        new_params.append(param)      # New layers: higher LR
    else:
        pretrained_params.append(param)  # Pretrained: lower LR

optimizer = optim.AdamW([
    {'params': pretrained_params, 'lr': 1e-5},   # Pretrained backbone
    {'params': new_params, 'lr': 1e-3}            # New classifier head
], weight_decay=0.01)

# Warmup scheduler
scheduler = get_linear_warmup_scheduler(optimizer, warmup_steps=500, total_steps=10000)
```

### Recipe 3: NLP / Transformer Training

```python
# AdamW with linear warmup + decay (the Transformer recipe)
optimizer = optim.AdamW(
    model.parameters(),
    lr=5e-5,
    betas=(0.9, 0.999),
    eps=1e-8,
    weight_decay=0.01
)

# Total training steps
num_epochs = 3
steps_per_epoch = len(train_loader)
total_steps = num_epochs * steps_per_epoch
warmup_steps = int(0.1 * total_steps)  # 10% warmup

scheduler = get_linear_warmup_scheduler(optimizer, warmup_steps, total_steps)

# Gradient clipping is essential for transformers
for epoch in range(num_epochs):
    for batch in train_loader:
        optimizer.zero_grad()
        loss = model(**batch).loss
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        optimizer.step()
        scheduler.step()
```

### Recipe 4: Complete Production Training Loop

```python
import torch
import torch.nn as nn
from torch.utils.data import DataLoader
import time
import os

def train_model(model, train_loader, val_loader, config):
    """
    Production-ready training loop with all best practices.
    
    Args:
        config: dict with keys: epochs, lr, weight_decay, patience, save_dir
    """
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    model = model.to(device)
    
    # Optimizer
    optimizer = torch.optim.AdamW(
        model.parameters(), lr=config['lr'], weight_decay=config['weight_decay']
    )
    
    # Scheduler
    scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(
        optimizer, mode='min', factor=0.5, patience=config['patience']//2
    )
    
    # Loss
    criterion = nn.CrossEntropyLoss()
    
    # Early stopping
    best_val_loss = float('inf')
    patience_counter = 0
    
    for epoch in range(config['epochs']):
        # === TRAIN ===
        model.train()
        train_loss = 0
        train_correct = 0
        train_total = 0
        start_time = time.time()
        
        for batch_x, batch_y in train_loader:
            batch_x, batch_y = batch_x.to(device), batch_y.to(device)
            
            optimizer.zero_grad()
            output = model(batch_x)
            loss = criterion(output, batch_y)
            loss.backward()
            
            # Gradient clipping
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            
            optimizer.step()
            
            train_loss += loss.item() * batch_x.size(0)
            train_correct += (output.argmax(1) == batch_y).sum().item()
            train_total += batch_x.size(0)
        
        train_loss /= train_total
        train_acc = train_correct / train_total
        
        # === VALIDATE ===
        model.eval()
        val_loss = 0
        val_correct = 0
        val_total = 0
        
        with torch.no_grad():
            for batch_x, batch_y in val_loader:
                batch_x, batch_y = batch_x.to(device), batch_y.to(device)
                output = model(batch_x)
                loss = criterion(output, batch_y)
                
                val_loss += loss.item() * batch_x.size(0)
                val_correct += (output.argmax(1) == batch_y).sum().item()
                val_total += batch_x.size(0)
        
        val_loss /= val_total
        val_acc = val_correct / val_total
        
        # Learning rate scheduling
        scheduler.step(val_loss)
        current_lr = optimizer.param_groups[0]['lr']
        
        # Logging
        elapsed = time.time() - start_time
        print(f"Epoch {epoch+1}/{config['epochs']} ({elapsed:.1f}s) | "
              f"Train Loss: {train_loss:.4f} Acc: {train_acc:.4f} | "
              f"Val Loss: {val_loss:.4f} Acc: {val_acc:.4f} | "
              f"LR: {current_lr:.2e}")
        
        # Early stopping + checkpointing
        if val_loss < best_val_loss:
            best_val_loss = val_loss
            patience_counter = 0
            # Save best model
            torch.save({
                'epoch': epoch,
                'model_state_dict': model.state_dict(),
                'optimizer_state_dict': optimizer.state_dict(),
                'val_loss': val_loss,
                'val_acc': val_acc,
            }, os.path.join(config['save_dir'], 'best_model.pt'))
            print(f"  → Saved best model (val_loss: {val_loss:.4f})")
        else:
            patience_counter += 1
            if patience_counter >= config['patience']:
                print(f"Early stopping at epoch {epoch+1}")
                break
    
    # Load best model
    checkpoint = torch.load(os.path.join(config['save_dir'], 'best_model.pt'))
    model.load_state_dict(checkpoint['model_state_dict'])
    print(f"\nBest model: Epoch {checkpoint['epoch']+1}, "
          f"Val Loss: {checkpoint['val_loss']:.4f}, "
          f"Val Acc: {checkpoint['val_acc']:.4f}")
    
    return model

# Usage:
# config = {'epochs': 100, 'lr': 0.001, 'weight_decay': 0.01, 
#           'patience': 10, 'save_dir': './checkpoints'}
# trained_model = train_model(model, train_loader, val_loader, config)
```

---

## 8. Common Mistakes

### Mistake 1: Not Monitoring Validation Loss
```python
# ❌ BAD: Only tracking training loss
for epoch in range(100):
    train_loss = train()
    print(f"Loss: {train_loss}")  # Could be overfitting and you'd never know!

# ✅ GOOD: Always monitor validation
for epoch in range(100):
    train_loss = train()
    val_loss = validate()  # Held-out data
    print(f"Train: {train_loss:.4f} | Val: {val_loss:.4f}")
    # If val_loss starts increasing while train_loss decreases → overfitting!
```

### Mistake 2: Learning Rate Too High After Loading Checkpoint
```python
# ❌ BAD: Resume training with initial LR after loading checkpoint
model.load_state_dict(checkpoint['model_state_dict'])
optimizer = Adam(model.parameters(), lr=0.001)  # LR should be lower!

# ✅ GOOD: Also restore optimizer state (includes current LR)
model.load_state_dict(checkpoint['model_state_dict'])
optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
```

### Mistake 3: Data Leakage Through Preprocessing
```python
# ❌ BAD: Fit scaler on ALL data (including test!)
scaler = StandardScaler()
X_all_scaled = scaler.fit_transform(X_all)  # LEAKS test statistics!
X_train, X_test = split(X_all_scaled)

# ✅ GOOD: Fit only on training data
X_train, X_test = split(X_all)
scaler = StandardScaler()
X_train = scaler.fit_transform(X_train)  # Learn from train only
X_test = scaler.transform(X_test)        # Apply same transform to test
```

### Mistake 4: Wrong Batch Size Considerations
```python
# ❌ BAD: Batch size = 1 (too noisy, no GPU utilization)
# ❌ BAD: Batch size = 50000 (runs out of memory, poor generalization)

# ✅ GOOD: Powers of 2 that fit in GPU memory
# batch_size = 32 (good default for most)
# batch_size = 64-256 (if you have GPU memory)
# If you increase batch_size, scale LR proportionally (linear scaling rule)
```

### Mistake 5: Not Using Mixed Precision Training
```python
# ❌ BAD: Full FP32 everywhere (2x slower, 2x memory)
# ✅ GOOD: Use automatic mixed precision (free speedup!)
from torch.cuda.amp import autocast, GradScaler

scaler = GradScaler()

for batch_x, batch_y in train_loader:
    optimizer.zero_grad()
    
    with autocast():  # Automatically uses FP16 where safe
        output = model(batch_x)
        loss = criterion(output, batch_y)
    
    scaler.scale(loss).backward()  # Scale loss to prevent FP16 underflow
    scaler.step(optimizer)
    scaler.update()
```

---

## 9. Interview Questions

### Conceptual

**Q1: Compare Adam vs SGD with Momentum. When would you choose each?**
> Adam: Faster convergence, adaptive per-parameter LR, good for sparse gradients (NLP), requires less LR tuning. SGD+Momentum: Often finds flatter minima (better generalization), standard for computer vision SOTA, requires careful LR scheduling. Use Adam for prototyping and NLP; use SGD for final CV training when every 0.1% accuracy matters.

**Q2: What is the difference between L2 regularization and weight decay? Are they the same?**
> For vanilla SGD, L2 and weight decay are equivalent. For adaptive optimizers (Adam), they differ! L2 adds λw to the gradient (which then gets scaled by Adam's adaptive factor), while weight decay directly subtracts λw from weights. AdamW implements proper weight decay, while Adam's weight_decay parameter actually does L2. Use AdamW for correct behavior.

**Q3: Explain the learning rate warmup. Why is it needed?**
> At the start of training, the network has random weights, so gradients point in random directions. The Adam optimizer's moving averages (m, v) are initialized to zero and are unreliable. High LR + random gradients = instability. Warmup starts with a tiny LR, letting the model and optimizer statistics stabilize before ramping up. Essential for Transformers and large batch training.

**Q4: How does batch size affect training?**
> Larger batches: more stable gradient estimates, better GPU utilization, but may converge to sharper minima (worse generalization). Smaller batches: noisier gradients (acts as regularization), finds flatter minima, but slower wall-clock time. Linear scaling rule: when doubling batch size, double LR.

**Q5: What is gradient accumulation and when would you use it?**
> When your GPU can't fit the desired batch size, accumulate gradients over multiple forward passes before updating:
> ```python
> accumulation_steps = 4  # Effective batch = real_batch × 4
> for i, (x, y) in enumerate(loader):
>     loss = model(x, y) / accumulation_steps
>     loss.backward()
>     if (i + 1) % accumulation_steps == 0:
>         optimizer.step()
>         optimizer.zero_grad()
> ```

**Q6: What causes NaN loss and how do you debug it?**
> Common causes: (1) Learning rate too high → gradients explode, (2) Log of zero/negative → use log-softmax or clip, (3) Division by zero → add epsilon, (4) Bad data (inf/NaN in inputs). Debug: reduce LR to 1e-6, add gradient clipping, check data for NaN/inf, print intermediate values.

**Q7: Explain the difference between `model.train()` and `model.eval()` in PyTorch.**
> `model.train()`: Dropout layers are active (randomly drop neurons), BatchNorm uses batch statistics (mean/var of current batch). `model.eval()`: Dropout is disabled (all neurons active), BatchNorm uses running statistics (accumulated during training). Always use eval() for inference/validation to get deterministic outputs.

### Coding

**Q8: Implement a learning rate scheduler that does linear warmup for the first N steps, then cosine decay.**
```python
import math

class WarmupCosineScheduler:
    def __init__(self, optimizer, warmup_steps, total_steps, min_lr=0):
        self.optimizer = optimizer
        self.warmup_steps = warmup_steps
        self.total_steps = total_steps
        self.min_lr = min_lr
        self.base_lr = optimizer.param_groups[0]['lr']
        self.current_step = 0
    
    def step(self):
        self.current_step += 1
        if self.current_step <= self.warmup_steps:
            # Linear warmup
            lr = self.base_lr * (self.current_step / self.warmup_steps)
        else:
            # Cosine decay
            progress = (self.current_step - self.warmup_steps) / (self.total_steps - self.warmup_steps)
            lr = self.min_lr + 0.5 * (self.base_lr - self.min_lr) * (1 + math.cos(math.pi * progress))
        
        for param_group in self.optimizer.param_groups:
            param_group['lr'] = lr
        
        return lr
```

---

## 10. Quick Reference

### Optimizer Defaults

| Optimizer | LR | Other Defaults |
|-----------|-----|---------------|
| SGD | 0.01-0.1 | momentum=0.9, weight_decay=1e-4 |
| Adam | 0.001 | betas=(0.9, 0.999), eps=1e-8 |
| AdamW | 0.001 | betas=(0.9, 0.999), weight_decay=0.01 |
| RMSProp | 0.001 | alpha=0.99, eps=1e-8 |

### Loss Function Quick Reference

| Task | Loss | PyTorch | Notes |
|------|------|---------|-------|
| Regression | MSE | `nn.MSELoss()` | Sensitive to outliers |
| Regression | Huber | `nn.HuberLoss()` | Robust to outliers |
| Binary clf | BCE | `nn.BCEWithLogitsLoss()` | Raw logits input |
| Multi-class | CE | `nn.CrossEntropyLoss()` | Raw logits + class indices |
| Multi-label | BCE | `nn.BCEWithLogitsLoss()` | Per-label sigmoid |
| Imbalanced | Focal | Custom | Focus on hard examples |

### Training Hyperparameter Cheatsheet

| Hyperparameter | Default | Range to Search |
|----------------|---------|----------------|
| Learning Rate | 0.001 (Adam) | 1e-5 to 1e-1 |
| Batch Size | 32-64 | 16, 32, 64, 128, 256 |
| Epochs | Until convergence | Use early stopping |
| Weight Decay | 0.01 | 1e-5 to 0.1 |
| Momentum | 0.9 | 0.8 to 0.99 |
| Warmup Steps | 10% of total | 5%-20% of total |
| Gradient Clip | 1.0 | 0.5 to 5.0 |

### PyTorch Training Checklist

```python
# 1. Device setup
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = model.to(device)

# 2. Optimizer (pick one)
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=0.01)

# 3. Scheduler (pick one)
scheduler = CosineAnnealingLR(optimizer, T_max=epochs)

# 4. Training loop essentials
model.train()                    # Enable train mode
optimizer.zero_grad()            # Clear gradients
loss.backward()                  # Compute gradients
clip_grad_norm_(params, 1.0)    # Clip gradients
optimizer.step()                 # Update weights
scheduler.step()                 # Update LR

# 5. Evaluation essentials
model.eval()                     # Disable dropout/BN training
with torch.no_grad():           # Save memory, prevent gradient tracking
    predictions = model(x)
```

---

> **Next Chapter:** [03-Regularization-and-Optimization.md](03-Regularization-and-Optimization.md) — Dropout, Batch Normalization, Early Stopping, and all the techniques that prevent overfitting.
