# Chapter 11: Hyperparameter Tuning for Deep Learning

## Architecture Search, Optuna, Ray Tune, and Best Practices

---

## Table of Contents
- [11.1 What Are Hyperparameters?](#111-what-are-hyperparameters)
- [11.2 Manual Tuning — The Practitioner's Intuition](#112-manual-tuning--the-practitioners-intuition)
- [11.3 Grid Search and Random Search](#113-grid-search-and-random-search)
- [11.4 Bayesian Optimization](#114-bayesian-optimization)
- [11.5 Optuna — Modern Hyperparameter Optimization](#115-optuna--modern-hyperparameter-optimization)
- [11.6 Ray Tune — Distributed Tuning at Scale](#116-ray-tune--distributed-tuning-at-scale)
- [11.7 Neural Architecture Search (NAS)](#117-neural-architecture-search-nas)
- [11.8 Learning Rate Strategies Deep Dive](#118-learning-rate-strategies-deep-dive)
- [11.9 Common Mistakes](#119-common-mistakes)
- [11.10 Interview Questions](#1110-interview-questions)
- [11.11 Quick Reference](#1111-quick-reference)

---

## 11.1 What Are Hyperparameters?

### What It Is
Hyperparameters are settings you choose **before** training begins — they control **how** the model learns, not **what** it learns. Think of them like the knobs on an oven: temperature (learning rate), cooking time (epochs), rack position (architecture). The model's parameters (weights) are learned automatically during training, but hyperparameters must be set by you.

### Why It Matters
- The **same architecture** can perform amazingly or terribly depending on hyperparameters
- A well-tuned small model often beats a poorly-tuned large model
- In production, proper tuning can save 2-10× in compute costs
- It's the difference between 85% and 95% accuracy on the same task

### Categories of Hyperparameters

| Category | Examples | Impact |
|----------|----------|--------|
| **Optimization** | Learning rate, optimizer, momentum, weight decay | How fast/well the model converges |
| **Architecture** | Layers, units, kernel size, attention heads | Model capacity and inductive bias |
| **Regularization** | Dropout, L2 penalty, data augmentation | Prevents overfitting |
| **Training** | Batch size, epochs, gradient clipping | Training stability and speed |
| **Data** | Image size, augmentation strength, sampling strategy | What the model "sees" |

### Parameters vs Hyperparameters

```
PARAMETERS (learned during training):
├── Weights in conv/linear layers
├── Biases
├── BatchNorm running statistics
└── Embedding vectors

HYPERPARAMETERS (set before training):
├── Learning rate = 0.001
├── Batch size = 32
├── Number of layers = 12
├── Dropout rate = 0.1
├── Optimizer = Adam
└── Weight decay = 1e-4
```

---

## 11.2 Manual Tuning — The Practitioner's Intuition

### The Most Important Hyperparameters (Ranked)

Based on empirical evidence from thousands of experiments:

| Rank | Hyperparameter | Typical Range | Impact |
|------|---------------|---------------|--------|
| 1 | **Learning rate** | 1e-5 to 1e-1 | Highest — wrong LR = training failure |
| 2 | **Batch size** | 8 to 512 | Affects generalization and speed |
| 3 | **Architecture depth/width** | Task-dependent | Model capacity |
| 4 | **Weight decay** | 1e-6 to 1e-2 | Regularization strength |
| 5 | **Dropout** | 0.0 to 0.5 | Prevents overfitting |
| 6 | **Learning rate schedule** | Step/Cosine/OneCycle | Training dynamics |
| 7 | **Data augmentation** | Mild to aggressive | Data efficiency |

### Rules of Thumb (From Experienced Practitioners)

```
Learning Rate:
├── Start with 3e-4 for Adam (the "Karpathy constant")
├── Start with 0.1 for SGD with momentum
├── If loss explodes → reduce LR by 10x
├── If loss plateaus early → increase LR by 3x
└── Fine-tuning pretrained models → use 10-100x smaller LR

Batch Size:
├── Larger batch → need larger LR (linear scaling rule)
├── Start with 32 or 64
├── If GPU memory allows, try 128-256
├── Very small batch (<8) → BatchNorm breaks, use GroupNorm
└── Gradient accumulation for effective larger batch

Architecture:
├── Start with a known good architecture (don't reinvent)
├── Deeper before wider (depth helps more usually)
├── Double channels when halving spatial dimensions
└── If overfitting → smaller model or more regularization
```

### The Learning Rate Finder

The most important single technique for manual tuning:

```python
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np
import matplotlib.pyplot as plt
from torch.utils.data import DataLoader

class LRFinder:
    """
    Learning Rate Finder: sweep LR from very small to very large,
    plot loss vs LR. The optimal LR is ~1 order of magnitude before the minimum.
    
    Based on Leslie Smith's paper "Cyclical Learning Rates for Training Neural Networks"
    """
    def __init__(self, model, optimizer, criterion, device='cuda'):
        self.model = model
        self.optimizer = optimizer
        self.criterion = criterion
        self.device = device
        
        # Save initial state to restore later
        self.model_state = model.state_dict()
        self.optimizer_state = optimizer.state_dict()
    
    def find(self, train_loader, start_lr=1e-7, end_lr=10, num_steps=100, smooth_factor=0.05):
        """Run the LR range test."""
        lrs = []
        losses = []
        best_loss = float('inf')
        
        # Exponentially increase LR from start to end
        lr_mult = (end_lr / start_lr) ** (1 / num_steps)
        lr = start_lr
        
        # Set initial LR
        for param_group in self.optimizer.param_groups:
            param_group['lr'] = lr
        
        self.model.train()
        iterator = iter(train_loader)
        
        for step in range(num_steps):
            # Get next batch (cycle if needed)
            try:
                inputs, targets = next(iterator)
            except StopIteration:
                iterator = iter(train_loader)
                inputs, targets = next(iterator)
            
            inputs, targets = inputs.to(self.device), targets.to(self.device)
            
            # Forward pass
            self.optimizer.zero_grad()
            outputs = self.model(inputs)
            loss = self.criterion(outputs, targets)
            
            # Smooth the loss
            if step == 0:
                smoothed_loss = loss.item()
            else:
                smoothed_loss = smooth_factor * loss.item() + (1 - smooth_factor) * losses[-1]
            
            # Stop if loss explodes (4x the minimum)
            if smoothed_loss > 4 * best_loss and step > 10:
                break
            
            if smoothed_loss < best_loss:
                best_loss = smoothed_loss
            
            lrs.append(lr)
            losses.append(smoothed_loss)
            
            # Backward pass
            loss.backward()
            self.optimizer.step()
            
            # Increase LR
            lr *= lr_mult
            for param_group in self.optimizer.param_groups:
                param_group['lr'] = lr
        
        # Restore model and optimizer state
        self.model.load_state_dict(self.model_state)
        self.optimizer.load_state_dict(self.optimizer_state)
        
        return lrs, losses
    
    def plot(self, lrs, losses):
        """Plot the LR vs Loss curve."""
        plt.figure(figsize=(10, 6))
        plt.semilogx(lrs, losses)
        plt.xlabel('Learning Rate (log scale)')
        plt.ylabel('Loss')
        plt.title('Learning Rate Finder')
        plt.grid(True)
        
        # Mark suggested LR (steepest descent point)
        min_idx = np.argmin(losses)
        suggested_lr = lrs[min_idx] / 10  # one order of magnitude before minimum
        plt.axvline(x=suggested_lr, color='r', linestyle='--', label=f'Suggested LR: {suggested_lr:.2e}')
        plt.legend()
        plt.savefig('lr_finder.png', dpi=150, bbox_inches='tight')
        plt.show()
        
        return suggested_lr


# Usage
model = nn.Sequential(nn.Linear(784, 256), nn.ReLU(), nn.Linear(256, 10))
optimizer = optim.Adam(model.parameters(), lr=1e-7)
criterion = nn.CrossEntropyLoss()

finder = LRFinder(model, optimizer, criterion, device='cpu')
# lrs, losses = finder.find(train_loader)
# suggested_lr = finder.plot(lrs, losses)
# print(f"Suggested learning rate: {suggested_lr:.2e}")
```

---

## 11.3 Grid Search and Random Search

### What They Are

**Grid Search**: Try every combination of hyperparameter values on a predefined grid.
**Random Search**: Sample hyperparameter combinations randomly from defined distributions.

### Why Random Search Beats Grid Search

```
Grid Search (9 trials):            Random Search (9 trials):

LR: [0.001, 0.01, 0.1]           LR: sampled randomly
BS: [16, 32, 64]                  BS: sampled randomly

┌─────┬─────┬─────┐              ┌─────────────────────┐
│  x  │  x  │  x  │              │    x    x        x  │
│     │     │     │              │         x           │
├─────┼─────┼─────┤              │  x           x      │
│  x  │  x  │  x  │              │       x             │
│     │     │     │              │   x         x       │
├─────┼─────┼─────┤              └─────────────────────┘
│  x  │  x  │  x  │              
│     │     │     │              Random explores more of
└─────┴─────┴─────┘              the space! 9 unique LR
Only 3 unique LR values!          values tested.
If optimal LR is between 
grid points, you miss it.
```

**Key insight**: When some hyperparameters matter more than others (which is always the case), random search explores more values of the important ones.

### Code Example

```python
import itertools
import random
import numpy as np
from typing import Dict, List, Tuple

# --- Grid Search ---
def grid_search(param_grid: Dict[str, List], train_fn) -> Tuple[dict, float]:
    """
    Exhaustive grid search over all combinations.
    
    Args:
        param_grid: dict mapping param names to lists of values
        train_fn: function that takes params dict and returns validation score
    
    Returns:
        best_params, best_score
    """
    keys = param_grid.keys()
    values = param_grid.values()
    
    best_score = -float('inf')
    best_params = None
    
    # Generate all combinations
    for combination in itertools.product(*values):
        params = dict(zip(keys, combination))
        score = train_fn(params)
        
        if score > best_score:
            best_score = score
            best_params = params
            print(f"  New best: {params} → Score: {score:.4f}")
    
    return best_params, best_score


# --- Random Search ---
def random_search(param_distributions: Dict, n_trials: int, train_fn) -> Tuple[dict, float]:
    """
    Random search: sample n_trials random combinations.
    
    param_distributions: dict mapping param names to:
        - list: sample uniformly from list
        - tuple (low, high): uniform float
        - tuple (low, high, 'log'): log-uniform (for learning rates)
        - tuple (low, high, 'int'): uniform integer
    """
    best_score = -float('inf')
    best_params = None
    
    for trial in range(n_trials):
        params = {}
        for name, dist in param_distributions.items():
            if isinstance(dist, list):
                params[name] = random.choice(dist)
            elif isinstance(dist, tuple):
                if len(dist) == 3 and dist[2] == 'log':
                    # Log-uniform sampling (crucial for learning rate!)
                    log_low, log_high = np.log10(dist[0]), np.log10(dist[1])
                    params[name] = 10 ** random.uniform(log_low, log_high)
                elif len(dist) == 3 and dist[2] == 'int':
                    params[name] = random.randint(dist[0], dist[1])
                else:
                    params[name] = random.uniform(dist[0], dist[1])
        
        score = train_fn(params)
        
        if score > best_score:
            best_score = score
            best_params = params
            print(f"  Trial {trial+1}/{n_trials} — New best: Score={score:.4f}")
            print(f"    Params: {params}")
    
    return best_params, best_score


# --- Example Usage ---
def dummy_train(params):
    """Simulate training — replace with actual training loop."""
    # Simulated: optimal is lr=0.003, bs=64, dropout=0.2
    lr_score = -((np.log10(params['lr']) - np.log10(0.003)) ** 2)
    bs_score = -((params['batch_size'] - 64) / 100) ** 2
    drop_score = -((params['dropout'] - 0.2) ** 2)
    return lr_score + bs_score + drop_score + random.gauss(0, 0.01)


# Grid Search example
param_grid = {
    'lr': [0.0001, 0.001, 0.01, 0.1],
    'batch_size': [16, 32, 64, 128],
    'dropout': [0.0, 0.1, 0.2, 0.3, 0.5]
}
# Total trials: 4 × 4 × 5 = 80
# best_params, best_score = grid_search(param_grid, dummy_train)

# Random Search example — usually better with same budget
param_distributions = {
    'lr': (1e-5, 1e-1, 'log'),        # log-uniform: crucial for LR
    'batch_size': [16, 32, 64, 128, 256],
    'dropout': (0.0, 0.5),             # uniform
    'weight_decay': (1e-6, 1e-2, 'log'),
    'num_layers': (2, 8, 'int'),
}
# best_params, best_score = random_search(param_distributions, n_trials=50, train_fn=dummy_train)
```

---

## 11.4 Bayesian Optimization

### What It Is
Bayesian Optimization is a **smart** search strategy that builds a **model of the objective function** (a surrogate model) and uses it to decide where to search next. Instead of blindly trying random points, it picks points that are most likely to improve results. Think of it like a chef who, after tasting 5 variations of a recipe, can intelligently guess which ingredient adjustment will taste best next — rather than trying random combinations.

### Why It Matters
- Finds good hyperparameters in **10-20× fewer trials** than random search
- Particularly valuable when each trial is expensive (hours of GPU time)
- Balances **exploration** (trying new regions) vs **exploitation** (refining promising regions)

### How It Works

```
Bayesian Optimization Loop:
━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Initialize: try a few random hyperparameter combinations
2. Build surrogate model (GP or TPE) of: hyperparams → performance
3. Use acquisition function to pick next point to try
4. Evaluate that point (actually train the model)
5. Update surrogate model with new observation
6. Repeat from step 3

┌────────────────────────────────────────────────────────┐
│  Surrogate Model (what we think the landscape looks like)│
│                                                          │
│  Performance                                             │
│  ▲         ╭──╮                                         │
│  │     ╭──╯  │    ╭─??─╮  (uncertainty = wide band)    │
│  │  ╭─╯      ╰──╮│     │╭──╮                           │
│  │ ╯             ╰╯     ╰╯  ╰───                       │
│  │●               ●          ●    ← observed points    │
│  │                    ↑                                  │
│  │              Next point to try                        │
│  │              (high uncertainty OR                     │
│  │               predicted to be good)                   │
│  └────────────────────────────────────────── HP value →  │
└────────────────────────────────────────────────────────┘
```

#### Key Concepts:

1. **Surrogate Model**: Approximates the true objective. Common choices:
   - **Gaussian Process (GP)**: Models uncertainty explicitly, good for <20 dimensions
   - **Tree-Parzen Estimator (TPE)**: Used in Optuna/Hyperopt, handles categorical/conditional params well

2. **Acquisition Function**: Decides where to sample next:
   - **Expected Improvement (EI)**: How much improvement over current best?
   - **Upper Confidence Bound (UCB)**: Mean + β × uncertainty
   - **Probability of Improvement (PI)**: Likelihood of beating current best

$$\text{EI}(x) = \mathbb{E}[\max(f(x) - f(x^+), 0)]$$

Where $f(x^+)$ is the best observed value so far.

---

## 11.5 Optuna — Modern Hyperparameter Optimization

### What It Is
Optuna is a Python library for **automated hyperparameter optimization**. It uses **TPE (Tree-Parzen Estimator)** by default, supports pruning (early stopping of bad trials), and has a beautiful dashboard for visualization.

### Why It Matters
- Most popular HPO library in industry (used by Toyota, Sony, Preferred Networks)
- **Define-by-run** API — define search space within the training function (very Pythonic)
- **Pruning** — automatically kills unpromising trials early, saving massive compute
- Built-in visualization, distributed training, and integration with PyTorch/TF

### Code Example — Complete Optuna Workflow

```python
import optuna
from optuna.trial import TrialState
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, random_split
from torchvision import datasets, transforms

# --- Define the Model ---
def create_model(trial):
    """
    Optuna suggests architecture hyperparameters.
    The search space is defined within this function (define-by-run).
    """
    # Suggest number of layers (2 to 5)
    n_layers = trial.suggest_int('n_layers', 2, 5)
    
    layers = []
    in_features = 784  # MNIST flattened
    
    for i in range(n_layers):
        # Suggest number of units for each layer
        out_features = trial.suggest_int(f'n_units_l{i}', 32, 512, step=32)
        
        layers.append(nn.Linear(in_features, out_features))
        layers.append(nn.ReLU())
        
        # Suggest dropout rate
        dropout = trial.suggest_float(f'dropout_l{i}', 0.0, 0.5)
        layers.append(nn.Dropout(dropout))
        
        in_features = out_features
    
    layers.append(nn.Linear(in_features, 10))  # output layer
    return nn.Sequential(*layers)


def objective(trial):
    """
    Objective function that Optuna will minimize/maximize.
    Returns the metric to optimize (e.g., validation accuracy).
    """
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    
    # --- Suggest Hyperparameters ---
    lr = trial.suggest_float('lr', 1e-5, 1e-1, log=True)  # log-uniform!
    optimizer_name = trial.suggest_categorical('optimizer', ['Adam', 'SGD', 'AdamW'])
    batch_size = trial.suggest_categorical('batch_size', [32, 64, 128, 256])
    weight_decay = trial.suggest_float('weight_decay', 1e-6, 1e-2, log=True)
    
    # --- Data ---
    transform = transforms.Compose([
        transforms.ToTensor(),
        transforms.Normalize((0.1307,), (0.3081,))
    ])
    dataset = datasets.MNIST('./data', train=True, download=True, transform=transform)
    
    # Split into train/val
    train_size = int(0.8 * len(dataset))
    val_size = len(dataset) - train_size
    train_dataset, val_dataset = random_split(dataset, [train_size, val_size])
    
    train_loader = DataLoader(train_dataset, batch_size=batch_size, shuffle=True)
    val_loader = DataLoader(val_dataset, batch_size=batch_size)
    
    # --- Model ---
    model = create_model(trial).to(device)
    
    # --- Optimizer ---
    if optimizer_name == 'Adam':
        optimizer = optim.Adam(model.parameters(), lr=lr, weight_decay=weight_decay)
    elif optimizer_name == 'SGD':
        optimizer = optim.SGD(model.parameters(), lr=lr, momentum=0.9, weight_decay=weight_decay)
    else:
        optimizer = optim.AdamW(model.parameters(), lr=lr, weight_decay=weight_decay)
    
    criterion = nn.CrossEntropyLoss()
    
    # --- Training Loop ---
    epochs = 20
    for epoch in range(epochs):
        model.train()
        for images, labels in train_loader:
            images = images.view(-1, 784).to(device)
            labels = labels.to(device)
            
            optimizer.zero_grad()
            outputs = model(images)
            loss = criterion(outputs, labels)
            loss.backward()
            optimizer.step()
        
        # --- Validation ---
        model.eval()
        correct = 0
        total = 0
        with torch.no_grad():
            for images, labels in val_loader:
                images = images.view(-1, 784).to(device)
                labels = labels.to(device)
                outputs = model(images)
                _, predicted = outputs.max(1)
                total += labels.size(0)
                correct += predicted.eq(labels).sum().item()
        
        accuracy = correct / total
        
        # --- PRUNING: Report intermediate value ---
        # Optuna can kill this trial early if it's not promising
        trial.report(accuracy, epoch)
        
        if trial.should_prune():
            raise optuna.exceptions.TrialPruned()
    
    return accuracy  # we want to maximize this


# --- Run Optimization ---
def run_optuna_study():
    # Create study (direction='maximize' for accuracy)
    study = optuna.create_study(
        study_name='mnist_optimization',
        direction='maximize',
        pruner=optuna.pruners.MedianPruner(  # prune if below median of completed trials
            n_startup_trials=5,    # don't prune first 5 trials
            n_warmup_steps=5,      # don't prune first 5 epochs
        ),
        sampler=optuna.samplers.TPESampler(seed=42)  # reproducibility
    )
    
    # Run 100 trials
    study.optimize(objective, n_trials=100, timeout=3600)  # 1 hour timeout
    
    # --- Results ---
    print(f"\nBest trial:")
    trial = study.best_trial
    print(f"  Value (accuracy): {trial.value:.4f}")
    print(f"  Params:")
    for key, value in trial.params.items():
        print(f"    {key}: {value}")
    
    # --- Visualization ---
    # Optuna has built-in visualization
    from optuna.visualization import (
        plot_optimization_history,
        plot_param_importances,
        plot_parallel_coordinate,
        plot_slice,
    )
    
    # Which hyperparameters matter most?
    fig = plot_param_importances(study)
    fig.show()
    
    # How did optimization progress over time?
    fig = plot_optimization_history(study)
    fig.show()
    
    # Parallel coordinate plot — see relationships
    fig = plot_parallel_coordinate(study)
    fig.show()
    
    return study


# --- Advanced: Multi-Objective Optimization ---
def multi_objective(trial):
    """Optimize both accuracy AND model size simultaneously."""
    # ... (same model/training code as above)
    
    accuracy = 0.95  # placeholder
    n_params = sum(p.numel() for p in model.parameters())
    
    return accuracy, n_params  # maximize accuracy, minimize params


def run_multi_objective():
    study = optuna.create_study(
        directions=['maximize', 'minimize'],  # accuracy up, params down
        study_name='pareto_optimization'
    )
    study.optimize(multi_objective, n_trials=50)
    
    # Get Pareto front (best trade-offs)
    pareto_trials = study.best_trials
    print(f"\nPareto-optimal solutions: {len(pareto_trials)}")
    for t in pareto_trials:
        print(f"  Accuracy: {t.values[0]:.4f}, Params: {t.values[1]:,}")


if __name__ == "__main__":
    run_optuna_study()
```

#### Optuna with PyTorch Lightning Integration

```python
import optuna
from optuna.integration import PyTorchLightningPruningCallback
import pytorch_lightning as pl

class LitModel(pl.LightningModule):
    def __init__(self, trial):
        super().__init__()
        self.trial = trial
        
        # Suggest architecture
        hidden_size = trial.suggest_int('hidden_size', 64, 512, step=64)
        self.lr = trial.suggest_float('lr', 1e-5, 1e-2, log=True)
        
        self.model = nn.Sequential(
            nn.Linear(784, hidden_size),
            nn.ReLU(),
            nn.Dropout(trial.suggest_float('dropout', 0.1, 0.5)),
            nn.Linear(hidden_size, 10)
        )
    
    def training_step(self, batch, batch_idx):
        x, y = batch
        loss = nn.functional.cross_entropy(self.model(x.view(-1, 784)), y)
        return loss
    
    def validation_step(self, batch, batch_idx):
        x, y = batch
        logits = self.model(x.view(-1, 784))
        acc = (logits.argmax(1) == y).float().mean()
        self.log('val_acc', acc)
    
    def configure_optimizers(self):
        return optim.Adam(self.parameters(), lr=self.lr)


def objective_lightning(trial):
    model = LitModel(trial)
    
    trainer = pl.Trainer(
        max_epochs=20,
        callbacks=[PyTorchLightningPruningCallback(trial, monitor='val_acc')],
        enable_progress_bar=False,
    )
    trainer.fit(model, train_dataloaders=train_loader, val_dataloaders=val_loader)
    
    return trainer.callback_metrics['val_acc'].item()
```

---

## 11.6 Ray Tune — Distributed Tuning at Scale

### What It Is
Ray Tune is a **distributed** hyperparameter tuning library that can run trials across **multiple GPUs and machines**. It integrates with Optuna, HyperOpt, and other search algorithms, adding the ability to scale across a cluster.

### Why It Matters
- Scale from 1 GPU to 100+ GPUs seamlessly
- **Population-Based Training (PBT)** — evolves hyperparameters DURING training
- **ASHA scheduler** — aggressively prunes bad trials
- Integrates with all major DL frameworks
- Used by OpenAI, Uber, Ant Financial for large-scale tuning

### Code Example

```python
import ray
from ray import tune
from ray.tune import CLIReporter
from ray.tune.schedulers import ASHAScheduler, PopulationBasedTraining
from ray.tune.search.optuna import OptunaSearch
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader
from torchvision import datasets, transforms

def train_model(config):
    """
    Training function for Ray Tune.
    `config` dict contains the hyperparameters for this trial.
    """
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    
    # Build model based on config
    model = nn.Sequential(
        nn.Linear(784, config['hidden_size']),
        nn.ReLU(),
        nn.Dropout(config['dropout']),
        nn.Linear(config['hidden_size'], config['hidden_size'] // 2),
        nn.ReLU(),
        nn.Dropout(config['dropout']),
        nn.Linear(config['hidden_size'] // 2, 10)
    ).to(device)
    
    optimizer = optim.Adam(
        model.parameters(), 
        lr=config['lr'],
        weight_decay=config['weight_decay']
    )
    criterion = nn.CrossEntropyLoss()
    
    # Data
    transform = transforms.Compose([transforms.ToTensor(), transforms.Normalize((0.1307,), (0.3081,))])
    train_dataset = datasets.MNIST('./data', train=True, download=True, transform=transform)
    val_dataset = datasets.MNIST('./data', train=False, transform=transform)
    
    train_loader = DataLoader(train_dataset, batch_size=config['batch_size'], shuffle=True)
    val_loader = DataLoader(val_dataset, batch_size=256)
    
    for epoch in range(20):
        # Train
        model.train()
        for images, labels in train_loader:
            images = images.view(-1, 784).to(device)
            labels = labels.to(device)
            
            optimizer.zero_grad()
            loss = criterion(model(images), labels)
            loss.backward()
            optimizer.step()
        
        # Validate
        model.eval()
        correct = total = 0
        val_loss = 0
        with torch.no_grad():
            for images, labels in val_loader:
                images = images.view(-1, 784).to(device)
                labels = labels.to(device)
                outputs = model(images)
                val_loss += criterion(outputs, labels).item()
                _, predicted = outputs.max(1)
                total += labels.size(0)
                correct += predicted.eq(labels).sum().item()
        
        accuracy = correct / total
        avg_val_loss = val_loss / len(val_loader)
        
        # Report metrics to Ray Tune (required for scheduling/pruning)
        tune.report({"val_accuracy": accuracy, "val_loss": avg_val_loss})


def run_ray_tune():
    """Run distributed hyperparameter optimization with Ray Tune."""
    
    # Define search space
    search_space = {
        'lr': tune.loguniform(1e-5, 1e-1),           # log-uniform
        'hidden_size': tune.choice([128, 256, 512]),   # categorical
        'batch_size': tune.choice([32, 64, 128]),
        'dropout': tune.uniform(0.1, 0.5),             # uniform
        'weight_decay': tune.loguniform(1e-6, 1e-2),
    }
    
    # ASHA Scheduler: aggressively prunes bad trials
    scheduler = ASHAScheduler(
        metric='val_accuracy',
        mode='max',
        max_t=20,          # maximum epochs
        grace_period=5,    # minimum epochs before pruning
        reduction_factor=3  # keep top 1/3 at each rung
    )
    
    # Use Optuna's TPE as the search algorithm
    search_alg = OptunaSearch(metric='val_accuracy', mode='max')
    
    # Reporter for terminal output
    reporter = CLIReporter(
        metric_columns=['val_accuracy', 'val_loss', 'training_iteration']
    )
    
    # Run trials
    result = tune.run(
        train_model,
        config=search_space,
        num_samples=50,           # total number of trials
        scheduler=scheduler,
        search_alg=search_alg,
        progress_reporter=reporter,
        resources_per_trial={'cpu': 2, 'gpu': 0.5},  # fractional GPU!
        local_dir='./ray_results',
        name='mnist_tune',
    )
    
    # Best result
    best_trial = result.get_best_trial('val_accuracy', 'max', 'last')
    print(f"\nBest trial config: {best_trial.config}")
    print(f"Best validation accuracy: {best_trial.last_result['val_accuracy']:.4f}")
    
    return result


# --- Population-Based Training (PBT) ---
def run_pbt():
    """
    PBT: trains a population of models in parallel.
    Periodically, poor performers copy weights from good performers
    and mutate their hyperparameters.
    
    Unlike other methods, PBT changes hyperparameters DURING training!
    """
    pbt_scheduler = PopulationBasedTraining(
        time_attr='training_iteration',
        perturbation_interval=5,  # every 5 epochs
        metric='val_accuracy',
        mode='max',
        hyperparam_mutations={
            # These can be mutated during training
            'lr': tune.loguniform(1e-5, 1e-1),
            'weight_decay': tune.loguniform(1e-6, 1e-2),
        }
    )
    
    result = tune.run(
        train_model,
        config={
            'lr': tune.loguniform(1e-4, 1e-2),
            'hidden_size': 256,  # fixed — PBT only mutates continuous params
            'batch_size': 64,
            'dropout': 0.2,
            'weight_decay': tune.loguniform(1e-5, 1e-3),
        },
        num_samples=8,  # population size
        scheduler=pbt_scheduler,
        resources_per_trial={'cpu': 2, 'gpu': 1},
    )
    
    return result


if __name__ == "__main__":
    ray.init()
    run_ray_tune()
    ray.shutdown()
```

### Optuna vs Ray Tune Comparison

| Feature | Optuna | Ray Tune |
|---------|--------|----------|
| **Best for** | Single-machine, quick experiments | Multi-GPU/multi-machine |
| **Search algorithms** | TPE (built-in) | Wraps Optuna, HyperOpt, Ax, etc. |
| **Pruning** | MedianPruner, HyperbandPruner | ASHA, PBT, HyperBand |
| **Distributed** | Limited (via storage backends) | Native (Ray cluster) |
| **Visualization** | optuna-dashboard | TensorBoard, Weights & Biases |
| **Learning curve** | Easy | Medium |
| **Use when** | <4 GPUs, rapid prototyping | 4+ GPUs, production tuning |

---

## 11.7 Neural Architecture Search (NAS)

### What It Is
NAS automates the design of neural network architectures themselves. Instead of humans designing layers and connections, an algorithm searches over possible architectures to find the best one. Think of it as **using AI to design AI**.

### Why It Matters
- Discovered architectures that outperform human-designed ones (NASNet, EfficientNet)
- Can find optimal architectures for specific hardware constraints
- Removes human bias from architecture design
- EfficientNet's base architecture (B0) was found via NAS

### Types of NAS

```
NAS Approaches:
━━━━━━━━━━━━━━

1. Reinforcement Learning NAS (original, very expensive)
   Controller (RNN) → generates architecture → trains it → reward = accuracy
   Problem: ~1000 GPU-days per search

2. Differentiable NAS (DARTS) — make architecture choice differentiable
   Treat architecture as continuous relaxation → optimize with gradient descent
   Cost: ~1-4 GPU-days

3. One-Shot NAS — train a super-network once, extract sub-networks
   Train massive network with all possible paths → evaluate sub-paths
   Cost: ~1 GPU-day

4. Hardware-Aware NAS — optimize for specific hardware (latency, memory)
   Adds latency/FLOPS as constraint to the objective
```

### Code Example — Simple DARTS-style Search

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# Available operations (search space)
OPERATIONS = {
    'conv_3x3': lambda C: nn.Sequential(
        nn.Conv2d(C, C, 3, padding=1, bias=False),
        nn.BatchNorm2d(C),
        nn.ReLU()
    ),
    'conv_5x5': lambda C: nn.Sequential(
        nn.Conv2d(C, C, 5, padding=2, bias=False),
        nn.BatchNorm2d(C),
        nn.ReLU()
    ),
    'sep_conv_3x3': lambda C: nn.Sequential(
        nn.Conv2d(C, C, 3, padding=1, groups=C, bias=False),  # depthwise
        nn.Conv2d(C, C, 1, bias=False),                        # pointwise
        nn.BatchNorm2d(C),
        nn.ReLU()
    ),
    'max_pool_3x3': lambda C: nn.MaxPool2d(3, stride=1, padding=1),
    'skip_connect': lambda C: nn.Identity(),
    'none': lambda C: Zero(C),  # no connection
}


class Zero(nn.Module):
    """Zero operation (no connection)."""
    def __init__(self, C):
        super().__init__()
    def forward(self, x):
        return torch.zeros_like(x)


class MixedOperation(nn.Module):
    """
    DARTS mixed operation: weighted sum of all candidate operations.
    Architecture weights (alphas) determine which operations are important.
    """
    def __init__(self, C):
        super().__init__()
        self.ops = nn.ModuleList([
            op_fn(C) for op_fn in OPERATIONS.values()
        ])
        # Architecture parameters (learnable!)
        self.alphas = nn.Parameter(torch.zeros(len(OPERATIONS)))
    
    def forward(self, x):
        # Softmax over architecture weights
        weights = F.softmax(self.alphas, dim=0)
        
        # Weighted sum of all operations
        return sum(w * op(x) for w, op in zip(weights, self.ops))


class SearchableCell(nn.Module):
    """A cell with mixed operations — architecture is searched."""
    def __init__(self, C, num_nodes=4):
        super().__init__()
        self.num_nodes = num_nodes
        
        # Each edge has a mixed operation
        self.edges = nn.ModuleList()
        for i in range(num_nodes):
            for j in range(i + 2):  # connect to all previous nodes + 2 inputs
                self.edges.append(MixedOperation(C))
    
    def forward(self, x):
        states = [x, x]  # two input states
        
        edge_idx = 0
        for i in range(self.num_nodes):
            # Sum of all incoming edges
            incoming = []
            for j in range(len(states)):
                incoming.append(self.edges[edge_idx](states[j]))
                edge_idx += 1
            states.append(sum(incoming))
        
        # Concatenate intermediate nodes as output
        return torch.cat(states[2:], dim=1)


class DARTSNetwork(nn.Module):
    """Simplified DARTS searchable network."""
    def __init__(self, C=16, num_classes=10, num_cells=8, num_nodes=4):
        super().__init__()
        
        self.stem = nn.Sequential(
            nn.Conv2d(3, C, 3, padding=1, bias=False),
            nn.BatchNorm2d(C)
        )
        
        self.cells = nn.ModuleList([
            SearchableCell(C, num_nodes) for _ in range(num_cells)
        ])
        
        # After concatenating num_nodes intermediate features
        self.classifier = nn.Linear(C * num_nodes, num_classes)
        self.pool = nn.AdaptiveAvgPool2d(1)
    
    def forward(self, x):
        x = self.stem(x)
        for cell in self.cells:
            x = cell(x)
            # Only take first C channels (simplified)
            x = x[:, :x.shape[1]//self.cells[0].num_nodes, :, :]
        x = self.pool(x).flatten(1)
        return self.classifier(x)
    
    def architecture_parameters(self):
        """Get all alpha parameters (architecture weights)."""
        for cell in self.cells:
            for edge in cell.edges:
                yield edge.alphas
    
    def network_parameters(self):
        """Get all non-architecture parameters (weights)."""
        arch_params = set(id(p) for p in self.architecture_parameters())
        for p in self.parameters():
            if id(p) not in arch_params:
                yield p


def train_darts(model, train_loader, val_loader, epochs=50, device='cuda'):
    """
    DARTS bilevel optimization:
    - Inner loop: optimize network weights on train data
    - Outer loop: optimize architecture params on validation data
    """
    model = model.to(device)
    
    # Two separate optimizers!
    weight_optimizer = torch.optim.SGD(
        model.network_parameters(), lr=0.025, momentum=0.9, weight_decay=3e-4
    )
    arch_optimizer = torch.optim.Adam(
        model.architecture_parameters(), lr=3e-4, weight_decay=1e-3
    )
    
    criterion = nn.CrossEntropyLoss()
    
    for epoch in range(epochs):
        model.train()
        
        for (train_batch, val_batch) in zip(train_loader, val_loader):
            train_images, train_labels = train_batch
            val_images, val_labels = val_batch
            
            train_images = train_images.to(device)
            train_labels = train_labels.to(device)
            val_images = val_images.to(device)
            val_labels = val_labels.to(device)
            
            # Step 1: Update architecture parameters on validation data
            arch_optimizer.zero_grad()
            val_loss = criterion(model(val_images), val_labels)
            val_loss.backward()
            arch_optimizer.step()
            
            # Step 2: Update network weights on training data
            weight_optimizer.zero_grad()
            train_loss = criterion(model(train_images), train_labels)
            train_loss.backward()
            weight_optimizer.step()
        
        if (epoch + 1) % 10 == 0:
            # Show which operations are being selected
            print(f"\nEpoch {epoch+1} — Selected operations:")
            op_names = list(OPERATIONS.keys())
            for i, edge in enumerate(model.cells[0].edges):
                weights = F.softmax(edge.alphas, dim=0)
                best_op = op_names[weights.argmax().item()]
                print(f"  Edge {i}: {best_op} (weight={weights.max():.3f})")
```

---

## 11.8 Learning Rate Strategies Deep Dive

### Why Learning Rate Scheduling Matters

A fixed learning rate is suboptimal:
- **Too high at start** → unstable, diverges
- **Too high at end** → can't settle into minimum
- **Too low throughout** → painfully slow, stuck in bad minima

### Common Schedulers Compared

```
Learning Rate over Training:

1. Step Decay:         2. Cosine Annealing:    3. OneCycleLR:
   LR                     LR                      LR
   ▲                      ▲                       ▲
   │────┐                 │╲                      │    ╱╲
   │    │                 │  ╲                    │   ╱  ╲
   │    └────┐            │   ╲                   │  ╱    ╲
   │         │            │    ╲                  │ ╱      ╲
   │         └───         │     ╲_╱╲_            │╱        ╲___
   └──────────── epoch    └──────────── epoch    └──────────── epoch

4. Warmup + Decay:     5. Cyclic LR:           6. ReduceLROnPlateau:
   LR                     LR                      LR
   ▲                      ▲                       ▲
   │  ╱────╲              │ ╱╲  ╱╲  ╱╲          │────────┐
   │ ╱      ╲             │╱  ╲╱  ╲╱  ╲         │        └───┐
   │╱        ╲            │               ╲      │            └──
   │          ╲___        │                      │
   └──────────── epoch    └──────────── epoch    └──────────── epoch
                                                  (drops when loss plateaus)
```

### Code Example — All Schedulers

```python
import torch
import torch.optim as optim
import matplotlib.pyplot as plt
import numpy as np

def visualize_schedulers():
    """Compare different LR schedulers side by side."""
    
    # Dummy model for optimizer
    model = torch.nn.Linear(10, 10)
    epochs = 100
    steps_per_epoch = 100
    total_steps = epochs * steps_per_epoch
    
    schedulers = {}
    
    # 1. StepLR — reduce by factor every N epochs
    opt = optim.SGD(model.parameters(), lr=0.1)
    schedulers['StepLR (step=30, γ=0.1)'] = optim.lr_scheduler.StepLR(
        opt, step_size=30, gamma=0.1
    )
    
    # 2. CosineAnnealingLR — smooth decay following cosine curve
    opt = optim.SGD(model.parameters(), lr=0.1)
    schedulers['CosineAnnealing'] = optim.lr_scheduler.CosineAnnealingLR(
        opt, T_max=epochs
    )
    
    # 3. CosineAnnealingWarmRestarts — periodic restarts
    opt = optim.SGD(model.parameters(), lr=0.1)
    schedulers['CosineWarmRestarts (T0=20)'] = optim.lr_scheduler.CosineAnnealingWarmRestarts(
        opt, T_0=20, T_mult=2
    )
    
    # 4. OneCycleLR — the super-convergence schedule
    opt = optim.SGD(model.parameters(), lr=0.1)
    schedulers['OneCycleLR'] = optim.lr_scheduler.OneCycleLR(
        opt, max_lr=0.1, total_steps=total_steps
    )
    
    # 5. ExponentialLR — exponential decay
    opt = optim.SGD(model.parameters(), lr=0.1)
    schedulers['ExponentialLR (γ=0.95)'] = optim.lr_scheduler.ExponentialLR(
        opt, gamma=0.95
    )
    
    # Track LRs
    fig, axes = plt.subplots(2, 3, figsize=(15, 8))
    axes = axes.flatten()
    
    for idx, (name, scheduler) in enumerate(schedulers.items()):
        lrs = []
        for epoch in range(epochs):
            lrs.append(scheduler.optimizer.param_groups[0]['lr'])
            if 'OneCycle' in name:
                for _ in range(steps_per_epoch):
                    scheduler.step()
            else:
                scheduler.step()
        
        axes[idx].plot(lrs)
        axes[idx].set_title(name)
        axes[idx].set_xlabel('Epoch')
        axes[idx].set_ylabel('Learning Rate')
        axes[idx].grid(True)
    
    plt.tight_layout()
    plt.savefig('lr_schedulers_comparison.png', dpi=150)
    plt.show()


# --- Production-Ready LR Schedule (Warmup + Cosine Decay) ---
class WarmupCosineScheduler:
    """
    Linear warmup followed by cosine decay.
    This is the most common schedule for training transformers.
    """
    def __init__(self, optimizer, warmup_steps, total_steps, min_lr=1e-6):
        self.optimizer = optimizer
        self.warmup_steps = warmup_steps
        self.total_steps = total_steps
        self.min_lr = min_lr
        self.base_lrs = [group['lr'] for group in optimizer.param_groups]
        self.current_step = 0
    
    def step(self):
        self.current_step += 1
        
        if self.current_step <= self.warmup_steps:
            # Linear warmup
            scale = self.current_step / self.warmup_steps
        else:
            # Cosine decay
            progress = (self.current_step - self.warmup_steps) / (
                self.total_steps - self.warmup_steps
            )
            scale = 0.5 * (1 + np.cos(np.pi * progress))
        
        for param_group, base_lr in zip(self.optimizer.param_groups, self.base_lrs):
            param_group['lr'] = max(self.min_lr, base_lr * scale)
    
    def get_lr(self):
        return [group['lr'] for group in self.optimizer.param_groups]


# Usage:
# optimizer = optim.AdamW(model.parameters(), lr=3e-4)
# scheduler = WarmupCosineScheduler(optimizer, warmup_steps=1000, total_steps=100000)
# 
# for step in range(total_steps):
#     loss = train_step()
#     optimizer.step()
#     scheduler.step()


# --- The OneCycle Policy (Leslie Smith's Super-Convergence) ---
def train_with_one_cycle(model, train_loader, epochs=30, max_lr=0.01):
    """
    OneCycleLR achieves 'super-convergence' — training faster with better results.
    
    Key insight: Start low, ramp up to high LR (exploration), 
    then decay to very low LR (fine convergence).
    """
    optimizer = optim.SGD(model.parameters(), lr=max_lr/25, momentum=0.9, weight_decay=1e-4)
    
    # OneCycleLR handles everything
    scheduler = optim.lr_scheduler.OneCycleLR(
        optimizer,
        max_lr=max_lr,
        epochs=epochs,
        steps_per_epoch=len(train_loader),
        pct_start=0.3,      # 30% of training is warmup
        anneal_strategy='cos',
        div_factor=25,       # initial_lr = max_lr / 25
        final_div_factor=1e4  # final_lr = initial_lr / 10000
    )
    
    criterion = nn.CrossEntropyLoss()
    
    for epoch in range(epochs):
        model.train()
        for images, labels in train_loader:
            optimizer.zero_grad()
            loss = criterion(model(images), labels)
            loss.backward()
            optimizer.step()
            scheduler.step()  # step every BATCH, not every epoch!
    
    return model
```

### Which Scheduler to Use?

| Scenario | Recommended Scheduler | Reason |
|----------|----------------------|--------|
| Training from scratch (CNN) | OneCycleLR | Super-convergence, fastest training |
| Fine-tuning pretrained model | CosineAnnealing with warmup | Gentle start, smooth decay |
| Training Transformers | Linear warmup + Cosine decay | Standard for all modern transformers |
| Long training (100+ epochs) | CosineAnnealingWarmRestarts | Periodic restarts escape local minima |
| Don't know what to use | ReduceLROnPlateau | Adaptive, no tuning needed |
| Quick experiments | Constant LR with Adam | Adam adapts per-parameter |

---

## 11.9 Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Tuning too many params at once | Can't find good combinations efficiently | Tune in priority order: LR → batch size → architecture → regularization |
| Not using log-uniform for LR | Linear sampling wastes budget (0.05 is as far from 0.01 as 0.09 is) | Always sample LR in log space |
| Tuning on test set | Data leakage — overfit to test set | Use train/val/test split. Only touch test set once at the very end |
| Not using early stopping during search | Wasting compute on bad trials | Use Optuna pruning or ASHA scheduler |
| Ignoring batch size effects | LR that works for BS=32 won't work for BS=256 | Scale LR linearly with batch size: `new_lr = old_lr * (new_bs / old_bs)` |
| Tuning augmentation last | Good augmentation changes optimal LR/regularization | Include augmentation in the search from the start |
| Not saving trial results | Can't analyze what worked | Always log to W&B, MLflow, or Optuna storage |
| Running too few trials | Didn't explore enough | Minimum: 20 trials for Bayesian, 50 for random |

### Pro Tips

> **Tip 1**: Use **Weights & Biases Sweeps** for visualization + Bayesian optimization in a single tool. Free for academics and small teams.

> **Tip 2**: **Freeze-then-unfreeze** during tuning. Find optimal LR for frozen backbone, then find optimal LR for unfrozen backbone separately.

> **Tip 3**: For production models, run **3 seeds** with the best hyperparameters to ensure stability. Report mean ± std.

> **Tip 4**: The **batch size → learning rate** linear scaling rule: if you double batch size, double the learning rate. But also add warmup proportional to the scaling factor.

---

## 11.10 Interview Questions

### Conceptual Questions

**Q1: Why is random search often better than grid search for hyperparameter tuning?**
> When some hyperparameters are more important than others (which is always true), grid search wastes trials testing unimportant parameter values while only testing a few values of important ones. Random search tests unique values for each parameter in every trial, giving better coverage of the important dimensions with the same budget.

**Q2: Explain the difference between Bayesian Optimization and Random Search. When would you use each?**
> Bayesian Optimization builds a surrogate model (GP/TPE) of the objective function and uses an acquisition function to intelligently pick the next point to evaluate. It's better when evaluations are expensive (each trial takes hours) and you have <100 trials budget. Random search is better for high-dimensional spaces (>20 hyperparameters), parallel evaluation without communication, and when you have large compute budget. In practice: use Bayesian (Optuna/TPE) for DL tuning.

**Q3: What is the relationship between batch size and learning rate?**
> Linear scaling rule: when you scale batch size by factor $k$, also scale learning rate by $k$. Intuition: larger batch = less noisy gradients = can take bigger steps safely. However, this breaks for very large batches (>8K). Also need warmup proportional to scaling. Large batch training often generalizes worse — implicit regularization of small-batch SGD noise helps find flatter minima.

**Q4: Explain Population-Based Training (PBT). How is it different from standard HPO?**
> Standard HPO: try different hyperparameter configs independently, pick the best final result. PBT: train a population of models in parallel, periodically have poor performers "exploit" good performers (copy their weights) and "explore" (mutate their hyperparameters). Key difference: PBT can adapt hyperparameters DURING training — e.g., start with high LR and decay it, or increase augmentation strength over time. This adapts the schedule jointly with training dynamics.

**Q5: How do you handle hyperparameter tuning when you have limited compute budget?**
> 1) Start with known good defaults from papers/blogs for your architecture. 2) Use LR finder to nail the most important parameter first. 3) Use successive halving/ASHA to kill bad trials early. 4) Search in priority order: LR, batch size, then other params. 5) Use transfer — if a similar task was tuned before, start from those values. 6) Use smaller proxy tasks (subset of data, fewer epochs) for initial screening, then verify top-3 configs on full data.

**Q6: What is the "super-convergence" phenomenon with OneCycle policy?**
> Leslie Smith discovered that using a very high max learning rate with the one-cycle schedule (warmup to high LR, then decay) can train models in ~1/5 the usual epochs while achieving better final accuracy. The high LR phase acts as regularization (noisy updates prevent sharp minima) and exploration (escapes local optima), while the annealing phase fine-tunes into a good flat minimum. It works best with SGD + momentum.

---

## 11.11 Quick Reference

| Concept | Recommendation |
|---------|---------------|
| **Best starting LR** | 3e-4 (Adam), 0.1 (SGD+momentum), 1e-4 (fine-tuning) |
| **LR sampling** | Always log-uniform: `10^uniform(-5, -1)` |
| **Batch size rule** | Start 32-64, scale LR linearly with BS |
| **Best HPO library** | Optuna (single machine), Ray Tune (distributed) |
| **Best scheduler (CNN)** | OneCycleLR |
| **Best scheduler (Transformer)** | Warmup + Cosine Decay |
| **Minimum trials** | 20 (Bayesian), 50 (Random) |
| **Pruning** | Always use (ASHA or MedianPruner) |
| **NAS** | Use DARTS for custom architectures, otherwise use EfficientNet |
| **Priority order** | LR → batch size → architecture → regularization → augmentation |

### Hyperparameter Quick-Start Cheat Sheet

```python
# Copy-paste starting configuration for most DL tasks:

config = {
    # Optimization
    'optimizer': 'AdamW',
    'lr': 3e-4,               # THE starting point
    'weight_decay': 1e-2,     # AdamW default
    'gradient_clip': 1.0,     # always clip
    
    # Training
    'batch_size': 64,
    'epochs': 100,
    'warmup_epochs': 5,       # ~5% of total
    
    # Regularization
    'dropout': 0.1,           # transformers
    'label_smoothing': 0.1,   # helps generalization
    
    # LR Schedule
    'scheduler': 'cosine',    # with warmup
    'min_lr': 1e-6,
    
    # Data
    'augmentation': 'medium', # RandAugment(N=2, M=9)
}
```

---

*Next: [Chapter 12 — Deep Learning Project Lifecycle](12-DL-Project-Lifecycle.md)*
