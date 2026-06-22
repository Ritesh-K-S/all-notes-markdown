# Chapter 03: Training Loop and Optimization

## Table of Contents
- [1. The Training Loop — The Heartbeat of Deep Learning](#1-the-training-loop)
- [2. Loss Functions — Measuring How Wrong Your Model Is](#2-loss-functions)
- [3. Optimizers — Finding the Best Path Downhill](#3-optimizers)
- [4. Learning Rate Scheduling — Adaptive Step Sizes](#4-learning-rate-scheduling)
- [5. Validation and Evaluation — Checking Your Work](#5-validation-and-evaluation)
- [6. Checkpointing — Saving Your Progress](#6-checkpointing)
- [7. Gradient Clipping — Keeping Training Stable](#7-gradient-clipping)
- [8. Complete Training Pipeline — Putting It All Together](#8-complete-training-pipeline)
- [9. Common Mistakes](#9-common-mistakes)
- [10. Interview Questions](#10-interview-questions)
- [11. Quick Reference](#11-quick-reference)

---

## 1. The Training Loop — The Heartbeat of Deep Learning {#1-the-training-loop}

### What It Is
The training loop is the repetitive process where your model:
1. Looks at data (forward pass)
2. Checks how wrong it was (loss calculation)
3. Figures out how to improve (backward pass)
4. Updates itself to be less wrong (optimizer step)

Think of it like a student studying for an exam: read the material → take a practice test → see what you got wrong → study those topics more → repeat.

### Why It Matters
Unlike frameworks like Keras (which hides the loop in `model.fit()`), PyTorch gives you **full control** over every step. This means:
- You can debug exactly what's happening at each step
- You can implement custom training logic (GANs, reinforcement learning, multi-task learning)
- You can add logging, gradient inspection, or conditional logic mid-training

### How It Works

```
┌─────────────────────────────────────────────────────────┐
│                    TRAINING LOOP                         │
│                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐     │
│  │  Forward  │───▶│   Loss   │───▶│   Backward   │     │
│  │   Pass    │    │ Compute  │    │    Pass      │     │
│  └──────────┘    └──────────┘    └──────────────┘     │
│       ▲                                    │            │
│       │                                    ▼            │
│  ┌──────────┐                      ┌──────────────┐   │
│  │   Next   │◀─────────────────────│  Optimizer   │   │
│  │  Batch   │                      │    Step      │   │
│  └──────────┘                      └──────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### The Anatomy of a Training Step

```python
import torch
import torch.nn as nn
import torch.optim as optim

# === SETUP ===
model = nn.Linear(10, 1)          # Simple model: 10 inputs → 1 output
criterion = nn.MSELoss()           # Loss function: Mean Squared Error
optimizer = optim.SGD(model.parameters(), lr=0.01)  # Optimizer: SGD with lr=0.01

# === SINGLE TRAINING STEP ===
# Step 1: Get a batch of data
x = torch.randn(32, 10)           # 32 samples, 10 features each
y = torch.randn(32, 1)            # 32 target values

# Step 2: Forward pass — model makes predictions
predictions = model(x)             # Shape: (32, 1)

# Step 3: Compute loss — how wrong is the model?
loss = criterion(predictions, y)   # Single scalar value

# Step 4: Zero gradients — CRITICAL! (explained below)
optimizer.zero_grad()

# Step 5: Backward pass — compute gradients
loss.backward()                    # Fills .grad for all parameters

# Step 6: Update parameters — optimizer adjusts weights
optimizer.step()                   # weights -= lr * gradients

print(f"Loss: {loss.item():.4f}") # .item() converts tensor → Python number
```

### Why `optimizer.zero_grad()`?

> **Critical Concept**: PyTorch **accumulates** gradients by default. If you don't zero them, gradients from previous batches add up, leading to incorrect updates.

```python
# WITHOUT zero_grad() — WRONG!
# Batch 1: grad = 0.5
# Batch 2: grad = 0.5 + 0.3 = 0.8  ← ACCUMULATED! Should be just 0.3
# Batch 3: grad = 0.8 + 0.2 = 1.0  ← WRONG!

# WITH zero_grad() — CORRECT!
# Batch 1: grad = 0.5
# Batch 2: grad = 0.3  ← Fresh gradient
# Batch 3: grad = 0.2  ← Fresh gradient
```

**Why does PyTorch do this?** Gradient accumulation is actually useful when you want to simulate larger batch sizes on limited GPU memory (covered in Advanced section).

### Complete Training Loop

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset

# ==============================
# SETUP
# ==============================
# Create synthetic data
X_train = torch.randn(1000, 20)           # 1000 samples, 20 features
y_train = torch.randn(1000, 1)            # 1000 target values

# Create DataLoader (batches data automatically)
dataset = TensorDataset(X_train, y_train)
train_loader = DataLoader(dataset, batch_size=64, shuffle=True)

# Model, Loss, Optimizer
model = nn.Sequential(
    nn.Linear(20, 64),
    nn.ReLU(),
    nn.Linear(64, 32),
    nn.ReLU(),
    nn.Linear(32, 1)
)
criterion = nn.MSELoss()
optimizer = optim.Adam(model.parameters(), lr=0.001)

# ==============================
# TRAINING LOOP
# ==============================
num_epochs = 10

for epoch in range(num_epochs):
    model.train()  # Set model to training mode (enables dropout, batchnorm training behavior)
    running_loss = 0.0
    
    for batch_idx, (inputs, targets) in enumerate(train_loader):
        # Forward pass
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        
        # Backward pass and optimize
        optimizer.zero_grad()  # Clear previous gradients
        loss.backward()        # Compute new gradients
        optimizer.step()       # Update weights
        
        # Track loss
        running_loss += loss.item()
    
    # Print epoch summary
    avg_loss = running_loss / len(train_loader)
    print(f"Epoch [{epoch+1}/{num_epochs}], Loss: {avg_loss:.4f}")
```

**Output:**
```
Epoch [1/10], Loss: 1.0842
Epoch [2/10], Loss: 1.0156
Epoch [3/10], Loss: 0.9823
...
Epoch [10/10], Loss: 0.9412
```

### `model.train()` vs `model.eval()`

| Mode | `model.train()` | `model.eval()` |
|------|-----------------|----------------|
| Dropout | Active (randomly zeros neurons) | Disabled (uses all neurons) |
| BatchNorm | Uses batch statistics | Uses running statistics |
| Gradient computation | Typically enabled | Often wrapped in `torch.no_grad()` |
| When to use | During training | During validation/inference |

```python
# TRAINING phase
model.train()
for inputs, targets in train_loader:
    outputs = model(inputs)
    loss = criterion(outputs, targets)
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()

# EVALUATION phase
model.eval()
with torch.no_grad():  # Disables gradient computation (saves memory and speed)
    for inputs, targets in val_loader:
        outputs = model(inputs)
        val_loss = criterion(outputs, targets)
```

---

## 2. Loss Functions — Measuring How Wrong Your Model Is {#2-loss-functions}

### What It Is
A loss function (also called criterion or cost function) is a mathematical formula that quantifies **how far** your model's predictions are from the actual values. Lower loss = better model.

Think of it as a "disappointment score" — the higher it is, the more disappointed you are with the model's answers.

### Why It Matters
- The loss function defines **what your model optimizes for**
- Different tasks need different loss functions
- Choosing the wrong loss function can make your model learn the wrong thing entirely

### Core Loss Functions

#### For Regression (Predicting Numbers)

**MSELoss — Mean Squared Error**

$$\text{MSE} = \frac{1}{n}\sum_{i=1}^{n}(y_i - \hat{y}_i)^2$$

```python
import torch
import torch.nn as nn

# MSE Loss — penalizes large errors heavily (squared!)
criterion = nn.MSELoss()

predictions = torch.tensor([2.5, 0.0, 2.1, 7.8])
targets = torch.tensor([3.0, -0.5, 2.0, 8.0])

loss = criterion(predictions, targets)
print(f"MSE Loss: {loss.item():.4f}")  # Output: MSE Loss: 0.1025

# When to use: Standard regression, when outliers should be penalized
# When NOT to use: When data has many outliers (use MAE instead)
```

**L1Loss — Mean Absolute Error (MAE)**

$$\text{MAE} = \frac{1}{n}\sum_{i=1}^{n}|y_i - \hat{y}_i|$$

```python
# L1 Loss — robust to outliers (linear penalty)
criterion = nn.L1Loss()

loss = criterion(predictions, targets)
print(f"L1 Loss: {loss.item():.4f}")  # Output: L1 Loss: 0.2500

# When to use: Regression with outliers, when you want median-like predictions
```

**SmoothL1Loss (Huber Loss) — Best of Both Worlds**

$$\text{Huber}(x) = \begin{cases} 0.5x^2 & \text{if } |x| < 1 \\ |x| - 0.5 & \text{otherwise} \end{cases}$$

```python
# Huber Loss — smooth near zero, linear for large errors
criterion = nn.SmoothL1Loss()

loss = criterion(predictions, targets)
print(f"Huber Loss: {loss.item():.4f}")  # Output: Huber Loss: 0.0838

# When to use: Object detection (used in Faster R-CNN), robust regression
```

#### For Classification (Predicting Categories)

**CrossEntropyLoss — The King of Classification**

$$\text{CE} = -\sum_{c=1}^{C} y_c \log(\hat{y}_c)$$

```python
# CrossEntropyLoss — combines LogSoftmax + NLLLoss
# NOTE: Input should be RAW LOGITS (not softmax!)
criterion = nn.CrossEntropyLoss()

# 3 samples, 5 classes — raw model output (logits)
logits = torch.tensor([[2.0, 1.0, 0.1, -1.0, 0.5],
                       [0.5, 2.5, 0.3, 0.2, -0.1],
                       [1.0, 0.2, 3.0, 0.1, 0.8]])

# Target: class indices (NOT one-hot!)
targets = torch.tensor([0, 1, 2])  # Correct classes

loss = criterion(logits, targets)
print(f"Cross-Entropy Loss: {loss.item():.4f}")  # Output: Cross-Entropy Loss: 0.4019

# COMMON MISTAKE: Don't apply softmax before CrossEntropyLoss!
# PyTorch's CrossEntropyLoss does it internally (numerically stable)
```

> **Warning**: `nn.CrossEntropyLoss` expects **raw logits**, NOT probabilities. Applying softmax before this loss is a very common bug!

**BCELoss vs BCEWithLogitsLoss — Binary Classification**

```python
# BCELoss — requires sigmoid-activated output (probabilities)
criterion_bce = nn.BCELoss()
probs = torch.sigmoid(torch.tensor([0.8, -0.5, 1.2]))  # Apply sigmoid first
targets = torch.tensor([1.0, 0.0, 1.0])
loss = criterion_bce(probs, targets)

# BCEWithLogitsLoss — takes raw logits (PREFERRED — numerically stable)
criterion_bce_logits = nn.BCEWithLogitsLoss()
logits = torch.tensor([0.8, -0.5, 1.2])  # Raw output, no sigmoid
targets = torch.tensor([1.0, 0.0, 1.0])
loss = criterion_bce_logits(logits, targets)

# ALWAYS prefer BCEWithLogitsLoss over BCELoss
# It's numerically more stable (uses log-sum-exp trick internally)
```

**NLLLoss — Negative Log Likelihood**

```python
# NLLLoss — expects log-probabilities as input
# Usually paired with nn.LogSoftmax
criterion = nn.NLLLoss()

log_probs = torch.log_softmax(logits, dim=1)  # Apply log_softmax first
targets = torch.tensor([0, 1, 2])

loss = criterion(log_probs, targets)
# This is EQUIVALENT to nn.CrossEntropyLoss (which combines both steps)
```

#### Class Imbalance — Weighted Loss

```python
# When dataset is imbalanced (e.g., 90% class 0, 10% class 1)
# Give more weight to underrepresented classes

# Method 1: Class weights in CrossEntropyLoss
class_weights = torch.tensor([0.1, 0.9])  # More weight to class 1
criterion = nn.CrossEntropyLoss(weight=class_weights)

# Method 2: pos_weight in BCEWithLogitsLoss (binary case)
# If positive class is 10x less common, set pos_weight=10
criterion = nn.BCEWithLogitsLoss(pos_weight=torch.tensor([10.0]))
```

### Loss Functions Comparison Table

| Loss Function | Task | Input | Target | Key Notes |
|--------------|------|-------|--------|-----------|
| `MSELoss` | Regression | Any shape | Same shape | Sensitive to outliers |
| `L1Loss` | Regression | Any shape | Same shape | Robust to outliers |
| `SmoothL1Loss` | Regression | Any shape | Same shape | Best of MSE + L1 |
| `CrossEntropyLoss` | Multi-class | (N, C) logits | (N,) class indices | Don't softmax first! |
| `BCEWithLogitsLoss` | Binary | (N,) logits | (N,) float {0,1} | Don't sigmoid first! |
| `NLLLoss` | Multi-class | (N, C) log-probs | (N,) class indices | Pair with LogSoftmax |

### Custom Loss Functions

```python
# You can write custom loss functions as regular Python functions
def focal_loss(logits, targets, gamma=2.0, alpha=0.25):
    """
    Focal Loss — handles extreme class imbalance (used in RetinaNet)
    Down-weights easy examples, focuses on hard ones.
    
    Formula: FL(p_t) = -α_t (1 - p_t)^γ log(p_t)
    """
    bce_loss = nn.functional.binary_cross_entropy_with_logits(
        logits, targets, reduction='none'
    )
    probs = torch.sigmoid(logits)
    p_t = probs * targets + (1 - probs) * (1 - targets)
    focal_weight = (1 - p_t) ** gamma
    loss = alpha * focal_weight * bce_loss
    return loss.mean()

# Or as an nn.Module (preferred for complex losses with learnable params)
class FocalLoss(nn.Module):
    def __init__(self, gamma=2.0, alpha=0.25):
        super().__init__()
        self.gamma = gamma
        self.alpha = alpha
    
    def forward(self, logits, targets):
        bce_loss = nn.functional.binary_cross_entropy_with_logits(
            logits, targets, reduction='none'
        )
        probs = torch.sigmoid(logits)
        p_t = probs * targets + (1 - probs) * (1 - targets)
        focal_weight = (1 - p_t) ** self.gamma
        loss = self.alpha * focal_weight * bce_loss
        return loss.mean()
```

---

## 3. Optimizers — Finding the Best Path Downhill {#3-optimizers}

### What It Is
An optimizer is the algorithm that **updates model parameters** based on computed gradients to minimize the loss. If loss is a mountain landscape, the optimizer is your strategy for walking downhill to find the valley (minimum).

### Why It Matters
- The optimizer determines **how fast** and **how reliably** your model learns
- Wrong optimizer = stuck in local minima, oscillating, or diverging
- Adam is the default choice, but knowing alternatives makes you a better practitioner

### How Optimizers Work — Intuition

```
Loss Landscape (2D slice):

    Loss
     ▲
     │    ╭─╮
     │   ╱   ╲      ╭─╮
     │  ╱     ╲    ╱   ╲
     │ ╱       ╲  ╱     ╲
     │╱         ╲╱       ╲
     │          global    ╲
     │          minimum    ╲
     └──────────────────────▶ Parameters
     
     SGD: Takes steps proportional to gradient slope
     Momentum: Builds up velocity like a ball rolling downhill
     Adam: Adapts step size per parameter + momentum
```

### SGD — Stochastic Gradient Descent

$$\theta_{t+1} = \theta_t - \eta \cdot \nabla_\theta L(\theta_t)$$

```python
import torch.optim as optim

# Basic SGD — simplest optimizer
optimizer = optim.SGD(model.parameters(), lr=0.01)

# SGD with Momentum — accumulates velocity (much better!)
# v_t = μ * v_{t-1} + ∇L(θ)
# θ = θ - lr * v_t
optimizer = optim.SGD(model.parameters(), lr=0.01, momentum=0.9)

# SGD with Momentum + Weight Decay (L2 regularization)
optimizer = optim.SGD(
    model.parameters(), 
    lr=0.01, 
    momentum=0.9, 
    weight_decay=1e-4  # L2 penalty: adds λ*||w||² to loss
)

# SGD with Nesterov Momentum (looks ahead before computing gradient)
optimizer = optim.SGD(
    model.parameters(), 
    lr=0.01, 
    momentum=0.9, 
    nesterov=True  # Usually better than vanilla momentum
)
```

### Adam — Adaptive Moment Estimation (Default Choice)

$$m_t = \beta_1 m_{t-1} + (1 - \beta_1) g_t \quad \text{(1st moment: mean of gradients)}$$
$$v_t = \beta_2 v_{t-1} + (1 - \beta_2) g_t^2 \quad \text{(2nd moment: variance of gradients)}$$
$$\theta_{t+1} = \theta_t - \frac{\eta}{\sqrt{\hat{v}_t} + \epsilon} \hat{m}_t$$

```python
# Adam — works well out of the box for most problems
optimizer = optim.Adam(
    model.parameters(),
    lr=0.001,        # Default: 1e-3 (good starting point)
    betas=(0.9, 0.999),  # Momentum coefficients
    eps=1e-8,        # Numerical stability
    weight_decay=0   # L2 regularization (use AdamW instead!)
)
```

### AdamW — Adam with Decoupled Weight Decay (RECOMMENDED)

```python
# AdamW — fixes weight decay in Adam (better generalization)
# This is what you should use in 2024+ for most tasks
optimizer = optim.AdamW(
    model.parameters(),
    lr=0.001,
    weight_decay=0.01  # Proper weight decay (not L2 regularization!)
)

# Why AdamW over Adam?
# In Adam, weight_decay is applied to the gradient (coupled)
# In AdamW, weight_decay is applied directly to weights (decoupled)
# Result: AdamW generalizes better, especially with learning rate scheduling
```

### Other Important Optimizers

```python
# RMSprop — predecessor to Adam, good for RNNs
optimizer = optim.RMSprop(model.parameters(), lr=0.001, alpha=0.99)

# Adagrad — adapts lr per parameter (good for sparse data/NLP)
optimizer = optim.Adagrad(model.parameters(), lr=0.01)

# LBFGS — second-order optimizer (for small models, physics-informed NNs)
optimizer = optim.LBFGS(model.parameters(), lr=1.0)
# Note: LBFGS requires a closure function (special training loop)
```

### Per-Parameter Learning Rates (Fine-Tuning Pattern)

```python
# Different learning rates for different layers
# Common in transfer learning: smaller lr for pretrained, larger for new layers
model = torchvision.models.resnet18(pretrained=True)
model.fc = nn.Linear(512, 10)  # Replace classification head

optimizer = optim.AdamW([
    {'params': model.layer1.parameters(), 'lr': 1e-5},   # Pretrained: tiny lr
    {'params': model.layer2.parameters(), 'lr': 1e-5},   # Pretrained: tiny lr
    {'params': model.layer3.parameters(), 'lr': 1e-4},   # Middle layers: medium lr
    {'params': model.layer4.parameters(), 'lr': 1e-4},   # Middle layers: medium lr
    {'params': model.fc.parameters(), 'lr': 1e-3},       # New head: large lr
], weight_decay=0.01)
```

### Optimizer Comparison

| Optimizer | Best For | Pros | Cons |
|-----------|----------|------|------|
| SGD + Momentum | CV (ResNets) | Best generalization | Needs careful lr tuning |
| Adam | General / prototyping | Works without much tuning | May overfit, wrong weight decay |
| AdamW | Transformers, general | Best of Adam + proper decay | Slightly more memory |
| RMSprop | RNNs | Handles non-stationary objectives | Outdated by Adam |
| LBFGS | Small models, physics | Fast convergence | Memory-hungry, complex API |

> **Pro Tip**: For research/prototyping, start with AdamW (lr=1e-3). For final results in computer vision, SGD + momentum often gives better generalization with proper tuning.

---

## 4. Learning Rate Scheduling — Adaptive Step Sizes {#4-learning-rate-scheduling}

### What It Is
A learning rate scheduler adjusts the learning rate during training. You typically start with a larger learning rate (to learn fast) and reduce it over time (to fine-tune).

Analogy: When you're lost in a new city, you first take big steps to get to the right neighborhood, then smaller steps to find the exact house.

### Why It Matters
- A fixed learning rate is rarely optimal throughout training
- Too high → model oscillates and never converges
- Too low → training is painfully slow
- Scheduling gives you the best of both worlds

### Common Schedulers

```python
import torch.optim.lr_scheduler as lr_scheduler

optimizer = optim.AdamW(model.parameters(), lr=0.001)

# ──────────────────────────────────────────
# 1. StepLR — reduce lr every N epochs
# ──────────────────────────────────────────
scheduler = lr_scheduler.StepLR(
    optimizer, 
    step_size=30,   # Every 30 epochs
    gamma=0.1       # Multiply lr by 0.1
)
# lr: 0.001 → 0.0001 → 0.00001 (at epochs 30, 60)

# ──────────────────────────────────────────
# 2. MultiStepLR — reduce at specific epochs
# ──────────────────────────────────────────
scheduler = lr_scheduler.MultiStepLR(
    optimizer,
    milestones=[30, 60, 90],  # Reduce at these epochs
    gamma=0.1
)

# ──────────────────────────────────────────
# 3. ExponentialLR — decay every epoch
# ──────────────────────────────────────────
scheduler = lr_scheduler.ExponentialLR(optimizer, gamma=0.95)
# lr: 0.001 * 0.95^epoch

# ──────────────────────────────────────────
# 4. CosineAnnealingLR — smooth cosine decay (very popular!)
# ──────────────────────────────────────────
scheduler = lr_scheduler.CosineAnnealingLR(
    optimizer,
    T_max=100,      # Number of epochs for one cosine cycle
    eta_min=1e-6    # Minimum learning rate
)
# lr follows a cosine curve from max to min

# ──────────────────────────────────────────
# 5. OneCycleLR — ramp up then decay (best for fast training)
# ──────────────────────────────────────────
scheduler = lr_scheduler.OneCycleLR(
    optimizer,
    max_lr=0.01,           # Peak learning rate
    total_steps=1000,      # Total training steps
    pct_start=0.3,         # 30% warmup, 70% decay
    anneal_strategy='cos'  # Cosine annealing
)
# Note: OneCycleLR steps per BATCH, not per epoch!

# ──────────────────────────────────────────
# 6. ReduceLROnPlateau — reduce when metric stops improving
# ──────────────────────────────────────────
scheduler = lr_scheduler.ReduceLROnPlateau(
    optimizer,
    mode='min',       # 'min' for loss, 'max' for accuracy
    factor=0.1,       # Multiply lr by this
    patience=10,      # Wait 10 epochs before reducing
    min_lr=1e-7       # Don't go below this
)
# Note: This scheduler takes metric value: scheduler.step(val_loss)
```

### Using Schedulers in Training Loop

```python
# Most schedulers: step once per epoch
for epoch in range(num_epochs):
    model.train()
    for inputs, targets in train_loader:
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        loss.backward()
        optimizer.step()
    
    # Step scheduler AFTER optimizer (per epoch)
    scheduler.step()  # For StepLR, CosineAnnealing, etc.
    
    print(f"Epoch {epoch}, LR: {scheduler.get_last_lr()[0]:.6f}")

# ReduceLROnPlateau: pass the metric value
for epoch in range(num_epochs):
    train_loss = train_one_epoch(model, train_loader)
    val_loss = validate(model, val_loader)
    scheduler.step(val_loss)  # Pass the monitored metric!

# OneCycleLR: step per BATCH, not per epoch!
scheduler = lr_scheduler.OneCycleLR(optimizer, max_lr=0.01, 
                                      total_steps=len(train_loader) * num_epochs)
for epoch in range(num_epochs):
    for inputs, targets in train_loader:
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        loss.backward()
        optimizer.step()
        scheduler.step()  # ← Step every batch!
```

### Warmup + Cosine Decay (Modern Standard for Transformers)

```python
# Linear warmup for first N steps, then cosine decay
# This is what GPT, BERT, ViT all use

def get_cosine_schedule_with_warmup(optimizer, num_warmup_steps, num_training_steps):
    """Creates a cosine schedule with linear warmup."""
    def lr_lambda(current_step):
        if current_step < num_warmup_steps:
            # Linear warmup
            return float(current_step) / float(max(1, num_warmup_steps))
        # Cosine decay
        progress = float(current_step - num_warmup_steps) / float(
            max(1, num_training_steps - num_warmup_steps)
        )
        return max(0.0, 0.5 * (1.0 + math.cos(math.pi * progress)))
    
    return lr_scheduler.LambdaLR(optimizer, lr_lambda)

# Usage
import math
total_steps = len(train_loader) * num_epochs
warmup_steps = int(0.1 * total_steps)  # 10% warmup

scheduler = get_cosine_schedule_with_warmup(
    optimizer, 
    num_warmup_steps=warmup_steps,
    num_training_steps=total_steps
)
```

### Learning Rate Finder (Pro Technique)

```python
def find_lr(model, train_loader, criterion, optimizer, 
            start_lr=1e-7, end_lr=10, num_steps=100):
    """
    Sweep through learning rates exponentially.
    Plot loss vs lr to find the best starting lr.
    Best lr is where loss decreases fastest (steepest slope).
    """
    lrs = []
    losses = []
    lr_mult = (end_lr / start_lr) ** (1 / num_steps)
    lr = start_lr
    
    # Save model state
    original_state = {k: v.clone() for k, v in model.state_dict().items()}
    
    for param_group in optimizer.param_groups:
        param_group['lr'] = lr
    
    model.train()
    for i, (inputs, targets) in enumerate(train_loader):
        if i >= num_steps:
            break
        
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        loss.backward()
        optimizer.step()
        
        lrs.append(lr)
        losses.append(loss.item())
        
        lr *= lr_mult
        for param_group in optimizer.param_groups:
            param_group['lr'] = lr
    
    # Restore model state
    model.load_state_dict(original_state)
    
    return lrs, losses
    # Plot lrs vs losses — pick lr where loss drops fastest
```

---

## 5. Validation and Evaluation — Checking Your Work {#5-validation-and-evaluation}

### What It Is
Validation is the process of evaluating your model on data it **hasn't seen during training**. It tells you how well your model generalizes to new data.

### Why It Matters
- Training loss can be misleading (model might just memorize training data)
- Validation loss reveals overfitting early
- It's your early warning system: "Stop training, you're getting worse!"

### Train vs Validation Loop

```python
import torch
import torch.nn as nn
from torch.utils.data import DataLoader

def train_one_epoch(model, train_loader, criterion, optimizer, device):
    """One epoch of training."""
    model.train()  # Enable training mode
    total_loss = 0.0
    correct = 0
    total = 0
    
    for inputs, targets in train_loader:
        inputs, targets = inputs.to(device), targets.to(device)
        
        # Forward + backward + optimize
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Track metrics
        total_loss += loss.item() * inputs.size(0)
        _, predicted = outputs.max(1)
        total += targets.size(0)
        correct += predicted.eq(targets).sum().item()
    
    avg_loss = total_loss / total
    accuracy = 100. * correct / total
    return avg_loss, accuracy


@torch.no_grad()  # Decorator version of `with torch.no_grad():`
def validate(model, val_loader, criterion, device):
    """Evaluate model on validation set."""
    model.eval()  # Disable dropout, use running stats for batchnorm
    total_loss = 0.0
    correct = 0
    total = 0
    
    for inputs, targets in val_loader:
        inputs, targets = inputs.to(device), targets.to(device)
        
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        
        total_loss += loss.item() * inputs.size(0)
        _, predicted = outputs.max(1)
        total += targets.size(0)
        correct += predicted.eq(targets).sum().item()
    
    avg_loss = total_loss / total
    accuracy = 100. * correct / total
    return avg_loss, accuracy
```

### Early Stopping — Prevent Overfitting

```python
class EarlyStopping:
    """Stop training when validation loss stops improving."""
    
    def __init__(self, patience=7, min_delta=0.0, restore_best=True):
        self.patience = patience      # How many epochs to wait
        self.min_delta = min_delta    # Minimum improvement to count
        self.restore_best = restore_best
        self.counter = 0
        self.best_loss = None
        self.best_model_state = None
        self.should_stop = False
    
    def __call__(self, val_loss, model):
        if self.best_loss is None:
            self.best_loss = val_loss
            self.best_model_state = model.state_dict().copy()
        elif val_loss > self.best_loss - self.min_delta:
            self.counter += 1
            print(f"  EarlyStopping: {self.counter}/{self.patience}")
            if self.counter >= self.patience:
                self.should_stop = True
                if self.restore_best:
                    model.load_state_dict(self.best_model_state)
                    print("  Restored best model weights.")
        else:
            self.best_loss = val_loss
            self.best_model_state = model.state_dict().copy()
            self.counter = 0

# Usage
early_stopping = EarlyStopping(patience=10)

for epoch in range(100):
    train_loss, train_acc = train_one_epoch(model, train_loader, criterion, optimizer, device)
    val_loss, val_acc = validate(model, val_loader, criterion, device)
    
    early_stopping(val_loss, model)
    if early_stopping.should_stop:
        print(f"Early stopping at epoch {epoch}")
        break
    
    print(f"Epoch {epoch}: Train Loss={train_loss:.4f}, Val Loss={val_loss:.4f}")
```

### Detecting Overfitting

```
Loss
 ▲
 │  Train Loss ──────────────
 │                            ╲──────── keeps decreasing
 │
 │  Val Loss ────────╱╲──╱╲──── starts increasing!
 │                  ╱         ← OVERFITTING STARTS HERE
 │                 ╱
 │    ╲───────────╱
 └──────────────────────────────▶ Epochs
      ↑
   Both decrease = Good (underfitting/learning phase)
```

---

## 6. Checkpointing — Saving Your Progress {#6-checkpointing}

### What It Is
Checkpointing means saving your model's state during training so you can:
- Resume training if it crashes
- Keep the best model (not just the final one)
- Share models with others

### Saving and Loading

```python
# ═══════════════════════════════════════════
# METHOD 1: Save entire model (NOT recommended)
# ═══════════════════════════════════════════
torch.save(model, 'model_full.pth')
model = torch.load('model_full.pth')
# Problems: Tied to exact class definition, Python version, file structure

# ═══════════════════════════════════════════
# METHOD 2: Save state_dict only (RECOMMENDED)
# ═══════════════════════════════════════════
torch.save(model.state_dict(), 'model_weights.pth')

# Load (must create model first, then load weights)
model = MyModel()  # Create model architecture
model.load_state_dict(torch.load('model_weights.pth'))
model.eval()  # Set to evaluation mode

# ═══════════════════════════════════════════
# METHOD 3: Full checkpoint (for resuming training)
# ═══════════════════════════════════════════
checkpoint = {
    'epoch': epoch,
    'model_state_dict': model.state_dict(),
    'optimizer_state_dict': optimizer.state_dict(),
    'scheduler_state_dict': scheduler.state_dict(),
    'train_loss': train_loss,
    'val_loss': val_loss,
    'best_val_loss': best_val_loss,
}
torch.save(checkpoint, f'checkpoint_epoch_{epoch}.pth')

# Resume training from checkpoint
checkpoint = torch.load('checkpoint_epoch_50.pth')
model.load_state_dict(checkpoint['model_state_dict'])
optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
scheduler.load_state_dict(checkpoint['scheduler_state_dict'])
start_epoch = checkpoint['epoch'] + 1
best_val_loss = checkpoint['best_val_loss']
```

### Best Model Saving Pattern

```python
best_val_loss = float('inf')

for epoch in range(num_epochs):
    train_loss, _ = train_one_epoch(model, train_loader, criterion, optimizer, device)
    val_loss, val_acc = validate(model, val_loader, criterion, device)
    
    scheduler.step()
    
    # Save best model
    if val_loss < best_val_loss:
        best_val_loss = val_loss
        torch.save({
            'epoch': epoch,
            'model_state_dict': model.state_dict(),
            'val_loss': val_loss,
            'val_acc': val_acc,
        }, 'best_model.pth')
        print(f"  ✓ Saved best model (val_loss: {val_loss:.4f})")
    
    # Also save periodic checkpoints
    if (epoch + 1) % 10 == 0:
        torch.save({
            'epoch': epoch,
            'model_state_dict': model.state_dict(),
            'optimizer_state_dict': optimizer.state_dict(),
        }, f'checkpoint_epoch_{epoch+1}.pth')
```

### Loading with Device Mapping

```python
# Load model trained on GPU → CPU
model.load_state_dict(
    torch.load('model.pth', map_location=torch.device('cpu'))
)

# Load model trained on CPU → GPU
model.load_state_dict(
    torch.load('model.pth', map_location=torch.device('cuda:0'))
)

# Load to same device (auto-detect)
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model.load_state_dict(
    torch.load('model.pth', map_location=device)
)
model.to(device)
```

---

## 7. Gradient Clipping — Keeping Training Stable {#7-gradient-clipping}

### What It Is
Gradient clipping limits the magnitude of gradients during training to prevent **exploding gradients** — when gradients become extremely large and cause wild parameter updates.

### Why It Matters
- Essential for RNNs/LSTMs (long sequences amplify gradients)
- Useful when training is unstable (loss spikes or goes to NaN)
- Transformers also benefit from gradient clipping

### Two Types of Gradient Clipping

```python
# ═══════════════════════════════════════════
# Method 1: Clip by Norm (RECOMMENDED)
# Scales all gradients so their total norm ≤ max_norm
# ═══════════════════════════════════════════
optimizer.zero_grad()
loss.backward()

# Clip gradients: if total gradient norm > 1.0, scale down proportionally
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

optimizer.step()

# ═══════════════════════════════════════════
# Method 2: Clip by Value
# Clamps each gradient element to [-clip_value, clip_value]
# ═══════════════════════════════════════════
optimizer.zero_grad()
loss.backward()

torch.nn.utils.clip_grad_value_(model.parameters(), clip_value=0.5)

optimizer.step()
```

### Monitoring Gradient Norms

```python
def get_gradient_norm(model):
    """Calculate the total gradient norm across all parameters."""
    total_norm = 0.0
    for p in model.parameters():
        if p.grad is not None:
            param_norm = p.grad.data.norm(2)
            total_norm += param_norm.item() ** 2
    total_norm = total_norm ** 0.5
    return total_norm

# In training loop:
loss.backward()
grad_norm = get_gradient_norm(model)
print(f"Gradient Norm: {grad_norm:.4f}")

# If grad_norm >> 10, you likely need clipping
# If grad_norm is NaN or Inf, something is seriously wrong
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
optimizer.step()
```

---

## 8. Complete Training Pipeline — Putting It All Together {#8-complete-training-pipeline}

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset, random_split
import torch.optim.lr_scheduler as lr_scheduler
import time

# ══════════════════════════════════════════════════════════════
# CONFIGURATION
# ══════════════════════════════════════════════════════════════
class Config:
    # Data
    batch_size = 64
    num_workers = 4
    
    # Model
    input_dim = 20
    hidden_dim = 128
    output_dim = 5
    dropout = 0.3
    
    # Training
    num_epochs = 50
    learning_rate = 1e-3
    weight_decay = 1e-2
    max_grad_norm = 1.0
    
    # Scheduler
    warmup_epochs = 5
    
    # Early stopping
    patience = 10
    
    # Device
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

config = Config()

# ══════════════════════════════════════════════════════════════
# MODEL
# ══════════════════════════════════════════════════════════════
class MLP(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim, dropout=0.3):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.BatchNorm1d(hidden_dim),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(hidden_dim, hidden_dim // 2),
            nn.BatchNorm1d(hidden_dim // 2),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(hidden_dim // 2, output_dim)
        )
    
    def forward(self, x):
        return self.net(x)

# ══════════════════════════════════════════════════════════════
# DATA PREPARATION
# ══════════════════════════════════════════════════════════════
# Synthetic data for demonstration
X = torch.randn(5000, config.input_dim)
y = torch.randint(0, config.output_dim, (5000,))

dataset = TensorDataset(X, y)
train_size = int(0.8 * len(dataset))
val_size = len(dataset) - train_size
train_dataset, val_dataset = random_split(dataset, [train_size, val_size])

train_loader = DataLoader(train_dataset, batch_size=config.batch_size, 
                          shuffle=True, num_workers=config.num_workers,
                          pin_memory=True)
val_loader = DataLoader(val_dataset, batch_size=config.batch_size,
                        shuffle=False, num_workers=config.num_workers,
                        pin_memory=True)

# ══════════════════════════════════════════════════════════════
# INITIALIZE MODEL, OPTIMIZER, SCHEDULER, CRITERION
# ══════════════════════════════════════════════════════════════
model = MLP(config.input_dim, config.hidden_dim, config.output_dim, config.dropout)
model = model.to(config.device)

optimizer = optim.AdamW(model.parameters(), lr=config.learning_rate, 
                        weight_decay=config.weight_decay)

scheduler = lr_scheduler.CosineAnnealingLR(optimizer, T_max=config.num_epochs, 
                                            eta_min=1e-6)

criterion = nn.CrossEntropyLoss()

# ══════════════════════════════════════════════════════════════
# TRAINING LOOP
# ══════════════════════════════════════════════════════════════
best_val_loss = float('inf')
patience_counter = 0
history = {'train_loss': [], 'val_loss': [], 'train_acc': [], 'val_acc': [], 'lr': []}

print(f"Training on: {config.device}")
print(f"Model parameters: {sum(p.numel() for p in model.parameters()):,}")

for epoch in range(config.num_epochs):
    start_time = time.time()
    
    # ── TRAINING ──
    model.train()
    train_loss, train_correct, train_total = 0.0, 0, 0
    
    for inputs, targets in train_loader:
        inputs = inputs.to(config.device, non_blocking=True)
        targets = targets.to(config.device, non_blocking=True)
        
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        
        optimizer.zero_grad()
        loss.backward()
        
        # Gradient clipping
        torch.nn.utils.clip_grad_norm_(model.parameters(), config.max_grad_norm)
        
        optimizer.step()
        
        train_loss += loss.item() * inputs.size(0)
        _, predicted = outputs.max(1)
        train_total += targets.size(0)
        train_correct += predicted.eq(targets).sum().item()
    
    # ── VALIDATION ──
    model.eval()
    val_loss, val_correct, val_total = 0.0, 0, 0
    
    with torch.no_grad():
        for inputs, targets in val_loader:
            inputs = inputs.to(config.device, non_blocking=True)
            targets = targets.to(config.device, non_blocking=True)
            
            outputs = model(inputs)
            loss = criterion(outputs, targets)
            
            val_loss += loss.item() * inputs.size(0)
            _, predicted = outputs.max(1)
            val_total += targets.size(0)
            val_correct += predicted.eq(targets).sum().item()
    
    # ── METRICS ──
    train_loss /= train_total
    val_loss /= val_total
    train_acc = 100. * train_correct / train_total
    val_acc = 100. * val_correct / val_total
    current_lr = optimizer.param_groups[0]['lr']
    
    # ── SCHEDULER STEP ──
    scheduler.step()
    
    # ── LOGGING ──
    history['train_loss'].append(train_loss)
    history['val_loss'].append(val_loss)
    history['train_acc'].append(train_acc)
    history['val_acc'].append(val_acc)
    history['lr'].append(current_lr)
    
    elapsed = time.time() - start_time
    print(f"Epoch [{epoch+1:3d}/{config.num_epochs}] "
          f"Train Loss: {train_loss:.4f} | Train Acc: {train_acc:.1f}% | "
          f"Val Loss: {val_loss:.4f} | Val Acc: {val_acc:.1f}% | "
          f"LR: {current_lr:.2e} | Time: {elapsed:.1f}s")
    
    # ── CHECKPOINTING ──
    if val_loss < best_val_loss:
        best_val_loss = val_loss
        patience_counter = 0
        torch.save({
            'epoch': epoch,
            'model_state_dict': model.state_dict(),
            'optimizer_state_dict': optimizer.state_dict(),
            'val_loss': val_loss,
            'val_acc': val_acc,
        }, 'best_model.pth')
    else:
        patience_counter += 1
    
    # ── EARLY STOPPING ──
    if patience_counter >= config.patience:
        print(f"\nEarly stopping at epoch {epoch+1}!")
        # Load best model
        checkpoint = torch.load('best_model.pth')
        model.load_state_dict(checkpoint['model_state_dict'])
        print(f"Restored best model from epoch {checkpoint['epoch']+1} "
              f"(val_loss: {checkpoint['val_loss']:.4f})")
        break

print(f"\nTraining complete! Best Val Loss: {best_val_loss:.4f}")
```

---

## 9. Common Mistakes {#9-common-mistakes}

| Mistake | Problem | Fix |
|---------|---------|-----|
| Forgetting `optimizer.zero_grad()` | Gradients accumulate across batches | Always call before `loss.backward()` |
| Forgetting `model.eval()` | Dropout/BatchNorm behave wrong in validation | Always switch modes |
| Forgetting `torch.no_grad()` | Wastes memory during validation | Wrap evaluation in `with torch.no_grad():` |
| Applying softmax before `CrossEntropyLoss` | Double softmax → wrong gradients | CE expects raw logits |
| Not moving data to GPU | Model on GPU, data on CPU → crash | `inputs.to(device)` |
| Wrong learning rate | Too high → diverge, too low → no learning | Use lr finder or start with 1e-3 for Adam |
| Not shuffling training data | Model learns order, not patterns | `shuffle=True` in training DataLoader |
| Saving `torch.save(model)` | Breaks across environments | Use `model.state_dict()` |
| Calling `scheduler.step()` before `optimizer.step()` | Warning in PyTorch ≥1.1 | Always optimizer first, then scheduler |
| Not using `non_blocking=True` | Slower GPU transfer | Use with `pin_memory=True` in DataLoader |

---

## 10. Interview Questions {#10-interview-questions}

**Q1: Why does PyTorch accumulate gradients by default?**
> Gradient accumulation allows simulating larger batch sizes when GPU memory is limited. If your batch size is 16 but you want effective batch size 64, you can accumulate gradients for 4 steps before calling `optimizer.step()`.

**Q2: What's the difference between `model.eval()` and `torch.no_grad()`?**
> `model.eval()` changes layer behavior (disables dropout, switches BatchNorm to inference mode). `torch.no_grad()` disables gradient computation (saves memory). For proper evaluation, you need BOTH.

**Q3: Why use AdamW over Adam with weight_decay?**
> In Adam, weight decay is applied to the gradient before the adaptive scaling, effectively making it L2 regularization scaled by the adaptive rate. AdamW decouples weight decay from the gradient update, applying it directly to weights. This gives proper regularization regardless of adaptive scaling.

**Q4: When would you use SGD over Adam?**
> SGD with momentum often generalizes better in computer vision (ResNets, etc.) when properly tuned. Adam is better for NLP/Transformers and when you need fast prototyping without tuning lr.

**Q5: Explain the training loop order: zero_grad → forward → loss → backward → step**
> `zero_grad()` clears old gradients. Forward pass computes predictions. Loss quantifies error. `backward()` computes gradients via chain rule (backpropagation). `step()` uses those gradients to update weights. Order matters because each step depends on the previous.

**Q6: What is gradient clipping and when do you need it?**
> Gradient clipping caps gradient magnitudes to prevent exploding gradients. Essential for RNNs/LSTMs and helpful for Transformers. `clip_grad_norm_` scales all gradients proportionally (preserves direction), while `clip_grad_value_` clamps each element independently.

**Q7: How does learning rate warmup help training?**
> At initialization, model parameters are random, so gradients are unreliable. Large steps with bad gradients can push parameters to bad regions. Warmup starts with tiny lr, letting the model "get its bearings" before taking large steps. Critical for Transformers and large batch training.

---

## 11. Quick Reference {#11-quick-reference}

### Training Loop Template

```python
for epoch in range(num_epochs):
    model.train()
    for inputs, targets in train_loader:
        inputs, targets = inputs.to(device), targets.to(device)
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        optimizer.zero_grad()
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        optimizer.step()
    scheduler.step()
    
    model.eval()
    with torch.no_grad():
        # validate...
```

### Key Functions

| Function | Purpose |
|----------|---------|
| `optimizer.zero_grad()` | Clear gradients |
| `loss.backward()` | Compute gradients |
| `optimizer.step()` | Update parameters |
| `scheduler.step()` | Update learning rate |
| `model.train()` | Enable training mode |
| `model.eval()` | Enable evaluation mode |
| `torch.no_grad()` | Disable gradient tracking |
| `clip_grad_norm_()` | Clip gradient magnitude |
| `torch.save()` | Save checkpoint |
| `torch.load()` | Load checkpoint |

### Optimizer Cheat Sheet

```python
# Fast prototyping:
optim.AdamW(params, lr=1e-3, weight_decay=0.01)

# Computer Vision (best generalization):
optim.SGD(params, lr=0.1, momentum=0.9, weight_decay=1e-4, nesterov=True)

# Transformers/NLP:
optim.AdamW(params, lr=5e-5, weight_decay=0.01)  # + cosine schedule + warmup

# Fine-tuning pretrained models:
optim.AdamW(params, lr=2e-5, weight_decay=0.01)  # + linear decay
```

---

*Next Chapter: [04-Data-Loading-and-Transforms](04-Data-Loading-and-Transforms.md)*
