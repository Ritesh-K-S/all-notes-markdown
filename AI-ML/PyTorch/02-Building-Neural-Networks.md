# Chapter 02: Building Neural Networks with nn.Module

## Table of Contents
- [1. nn.Module — The Foundation of Everything](#1-nnmodule--the-foundation-of-everything)
- [2. Layers — The Building Blocks](#2-layers--the-building-blocks)
- [3. The Forward Method](#3-the-forward-method)
- [4. Parameters and Buffers](#4-parameters-and-buffers)
- [5. Building Your First Neural Network](#5-building-your-first-neural-network)
- [6. nn.Sequential and Model Composition](#6-nnsequential-and-model-composition)
- [7. Custom Layers](#7-custom-layers)
- [8. Model Inspection and Summary](#8-model-inspection-and-summary)
- [9. Saving and Loading Models](#9-saving-and-loading-models)
- [10. Common Mistakes](#10-common-mistakes)
- [11. Interview Questions](#11-interview-questions)
- [12. Quick Reference](#12-quick-reference)

---

## 1. nn.Module — The Foundation of Everything

### What It Is
`nn.Module` is the base class for **every** neural network component in PyTorch — from a single linear layer to a 175-billion-parameter GPT model. If you're building anything in PyTorch, you're subclassing `nn.Module`.

**Analogy:** Think of `nn.Module` as a LEGO baseplate. Every LEGO structure (neural network) starts with one. Individual bricks (layers) snap onto it, and you can nest baseplates inside each other to build complex structures.

### Why It Matters
- **Organizes parameters** — automatically tracks all learnable weights
- **Enables `.to(device)`** — moves entire model to GPU in one call
- **Supports `.train()` / `.eval()`** — toggles behavior for dropout, batchnorm, etc.
- **Enables serialization** — save/load models with `torch.save()` / `torch.load()`
- **Recursion** — modules can contain other modules, forming a tree

### How It Works

```
nn.Module Tree Structure:
┌─────────────────────────────────────────┐
│  MyModel (nn.Module)                     │
│  ├── self.layer1 (nn.Linear)             │
│  │   ├── weight: Parameter [128×784]     │
│  │   └── bias: Parameter [128]           │
│  ├── self.relu (nn.ReLU)                 │
│  ├── self.layer2 (nn.Linear)             │
│  │   ├── weight: Parameter [10×128]      │
│  │   └── bias: Parameter [10]            │
│  └── self.dropout (nn.Dropout)           │
└─────────────────────────────────────────┘

When you call model.parameters(), it walks this
entire tree and yields every Parameter it finds.
```

### The Two Rules of nn.Module

```python
import torch
import torch.nn as nn

class MyModel(nn.Module):
    
    # RULE 1: Call super().__init__() in __init__
    def __init__(self):
        super().__init__()  # MUST call this — registers parameters
        self.layer1 = nn.Linear(784, 128)
        self.layer2 = nn.Linear(128, 10)
    
    # RULE 2: Define forward() — describes how data flows through the model
    def forward(self, x):
        x = self.layer1(x)
        x = torch.relu(x)
        x = self.layer2(x)
        return x

# Usage:
model = MyModel()
x = torch.randn(32, 784)    # Batch of 32, each with 784 features
output = model(x)            # Calls model.forward(x) — NEVER call .forward() directly!
print(output.shape)          # torch.Size([32, 10])
```

> **⚠️ Important:** Never call `model.forward(x)` directly. Always use `model(x)`. The `__call__` method does extra work — it runs hooks, handles training/eval mode, and then calls `forward()`.

---

## 2. Layers — The Building Blocks

### What It Is
Layers are pre-built `nn.Module` subclasses that perform specific transformations on data. PyTorch provides dozens of them. Each layer has learnable **parameters** (weights and biases) that get updated during training.

### Why It Matters
You rarely build operations from raw tensors. Layers encapsulate:
- The math (matrix multiplication, convolution, etc.)
- The parameters (automatically tracked)
- The initialization (sensible defaults)
- Forward/backward computation

### Core Layers Reference

#### Linear (Fully Connected / Dense)

The most fundamental layer. Performs: $y = xW^T + b$

```python
import torch
import torch.nn as nn

# nn.Linear(in_features, out_features, bias=True)
linear = nn.Linear(in_features=784, out_features=256)

# What's inside:
print(f"Weight shape: {linear.weight.shape}")  # torch.Size([256, 784])
print(f"Bias shape: {linear.bias.shape}")      # torch.Size([256])

# Forward pass:
x = torch.randn(32, 784)    # batch_size=32, features=784
output = linear(x)           # matrix multiply + bias
print(f"Output shape: {output.shape}")  # torch.Size([32, 256])

# Without bias:
linear_no_bias = nn.Linear(784, 256, bias=False)
print(f"Parameters: {sum(p.numel() for p in linear_no_bias.parameters())}")
# 200,704 (vs 200,960 with bias)
```

```
What nn.Linear does:

Input x          Weight W^T          Bias b          Output y
[32 × 784]   @   [784 × 256]    +   [256]    =    [32 × 256]
  (batch)         (transposed)      (broadcast)      (batch)
```

#### Activation Functions

Activations introduce **non-linearity** — without them, stacking linear layers is equivalent to a single linear layer (a linear combination of linear functions is still linear).

```python
import torch
import torch.nn as nn

x = torch.tensor([-2.0, -1.0, 0.0, 1.0, 2.0])

# === ReLU (Rectified Linear Unit) — Most Popular ===
# f(x) = max(0, x)
relu = nn.ReLU()
print(f"ReLU: {relu(x)}")  # tensor([0., 0., 0., 1., 2.])
# Pro: Fast, works well in practice, sparse activations
# Con: "Dead neurons" — if input always < 0, neuron never activates

# === LeakyReLU — Fixes dead neuron problem ===
# f(x) = x if x > 0, else 0.01x
leaky = nn.LeakyReLU(negative_slope=0.01)
print(f"LeakyReLU: {leaky(x)}")  # tensor([-0.0200, -0.0100, 0.0000, 1.0000, 2.0000])

# === Sigmoid — Squashes to (0, 1) ===
# f(x) = 1 / (1 + e^(-x))
sigmoid = nn.Sigmoid()
print(f"Sigmoid: {sigmoid(x)}")
# Use for: Binary classification output, gates in LSTM/GRU
# Problem: Vanishing gradients when x is very large or very small

# === Tanh — Squashes to (-1, 1) ===
# f(x) = (e^x - e^(-x)) / (e^x + e^(-x))
tanh = nn.Tanh()
print(f"Tanh: {tanh(x)}")
# Use for: Hidden layers in RNNs, output when range is [-1, 1]

# === GELU (Gaussian Error Linear Unit) — Used in Transformers ===
# f(x) = x * Φ(x)  where Φ is the CDF of standard normal
gelu = nn.GELU()
print(f"GELU: {gelu(x)}")
# Use for: Transformer models (BERT, GPT, ViT)

# === SiLU / Swish — Used in modern architectures ===
# f(x) = x * sigmoid(x)
silu = nn.SiLU()
print(f"SiLU: {silu(x)}")
# Use for: EfficientNet, modern CNNs

# === Softmax — Converts logits to probabilities ===
# f(x_i) = e^(x_i) / Σ e^(x_j)
logits = torch.tensor([2.0, 1.0, 0.1])
softmax = nn.Softmax(dim=0)
probs = softmax(logits)
print(f"Softmax: {probs}")       # tensor([0.6590, 0.2424, 0.0986])
print(f"Sum: {probs.sum()}")     # tensor(1.) — always sums to 1!
```

```
Activation Functions Visual:

ReLU:           Sigmoid:         Tanh:            GELU:
    |  /            ──────1         ──────1          ~ /
    | /          /                /                ~ /
────+────     ──+──────      ────+────         ──~+~~~~
    |        /      0        -1  |                /
    |                            ──────-1       ~ 
```

> **When to use which activation:**
> | Activation | Use When |
> |-----------|----------|
> | ReLU | Default for hidden layers |
> | LeakyReLU | If you see dead neurons with ReLU |
> | GELU | Transformer architectures |
> | Sigmoid | Binary classification output layer |
> | Softmax | Multi-class classification output layer |
> | Tanh | Output in [-1, 1] range, RNN hidden states |

#### Dropout — Regularization Layer

```python
import torch
import torch.nn as nn

# Randomly zeroes elements with probability p during TRAINING
# Scales remaining elements by 1/(1-p) to maintain expected value
dropout = nn.Dropout(p=0.5)  # 50% dropout rate

x = torch.ones(1, 10)

# Training mode (default)
print(f"Train: {dropout(x)}")
# Example output: tensor([[0., 2., 0., 2., 2., 0., 0., 2., 2., 0.]])
# Notice: non-zero values are scaled by 1/(1-0.5) = 2

# Eval mode — dropout is DISABLED
dropout.eval()
print(f"Eval: {dropout(x)}")
# Output: tensor([[1., 1., 1., 1., 1., 1., 1., 1., 1., 1.]])

# Common dropout rates:
# 0.1-0.3: Light regularization (default for transformers)
# 0.5: Standard (original paper)
# 0.7-0.8: Heavy regularization (very large models, small data)
```

> **⚠️ Critical:** Always call `model.eval()` before inference and `model.train()` before training! Dropout and BatchNorm behave differently in each mode.

#### BatchNorm — Normalization Layer

```python
import torch
import torch.nn as nn

# Normalizes activations across the BATCH dimension
# Makes training faster and more stable
# Formula: y = (x - mean) / sqrt(var + eps) * gamma + beta

# BatchNorm1d: for (batch, features)
bn1d = nn.BatchNorm1d(num_features=256)

# BatchNorm2d: for (batch, channels, height, width)
bn2d = nn.BatchNorm2d(num_features=64)  # 64 channels

x = torch.randn(32, 256)      # batch of 32
output = bn1d(x)
print(f"BN1d output shape: {output.shape}")  # torch.Size([32, 256])

# BatchNorm has 4 internal tensors:
print(f"Gamma (weight): {bn1d.weight.shape}")        # [256] — learnable scale
print(f"Beta (bias): {bn1d.bias.shape}")              # [256] — learnable shift
print(f"Running mean: {bn1d.running_mean.shape}")     # [256] — tracked for eval
print(f"Running var: {bn1d.running_var.shape}")        # [256] — tracked for eval

# Train mode: uses batch statistics
# Eval mode: uses running statistics (accumulated during training)
```

#### Embedding — For Discrete Data

```python
import torch
import torch.nn as nn

# Maps integer indices to dense vectors
# Used for: words, categories, user IDs, item IDs

# Embedding(num_embeddings, embedding_dim)
vocab_size = 10000     # Number of unique tokens
embed_dim = 256        # Size of each embedding vector

embedding = nn.Embedding(num_embeddings=vocab_size, embedding_dim=embed_dim)

# Input: integer indices (NOT one-hot!)
token_ids = torch.tensor([42, 1337, 7, 999])  # 4 tokens
vectors = embedding(token_ids)
print(f"Output shape: {vectors.shape}")  # torch.Size([4, 256])

# With padding index (embedding for padding token is always zero)
embedding_padded = nn.Embedding(vocab_size, embed_dim, padding_idx=0)
```

---

## 3. The Forward Method

### What It Is
The `forward()` method defines the **computation graph** — how data flows from input to output through your model. It's where you connect layers, apply activations, and implement any custom logic.

### Why It Matters
- It IS your model architecture — the forward pass defines what your network computes
- PyTorch's autograd records everything in `forward()` to compute gradients in the backward pass
- You can use any Python control flow (if/else, loops, recursion) — this is PyTorch's key advantage

### How It Works

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class FlexibleNet(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim, use_dropout=True):
        super().__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, hidden_dim)
        self.fc3 = nn.Linear(hidden_dim, output_dim)
        self.bn = nn.BatchNorm1d(hidden_dim)
        self.dropout = nn.Dropout(0.3)
        self.use_dropout = use_dropout
    
    def forward(self, x):
        # Layer 1: Linear → BatchNorm → ReLU → Dropout
        x = self.fc1(x)
        x = self.bn(x)
        x = F.relu(x)               # Functional API (no state, no __init__ needed)
        
        if self.use_dropout:         # Python conditionals — works in PyTorch!
            x = self.dropout(x)
        
        # Layer 2: Linear → ReLU
        x = F.relu(self.fc2(x))
        
        # Output layer (no activation — raw logits)
        x = self.fc3(x)
        return x

model = FlexibleNet(input_dim=784, hidden_dim=256, output_dim=10)
x = torch.randn(64, 784)
output = model(x)
print(f"Output: {output.shape}")  # torch.Size([64, 10])
```

### nn.Module Layers vs torch.nn.functional (F)

```python
import torch.nn as nn
import torch.nn.functional as F

# Two ways to apply operations:

# 1. nn.Module layer (has state — define in __init__)
class ModelV1(nn.Module):
    def __init__(self):
        super().__init__()
        self.relu = nn.ReLU()        # Stateless but defined as module
        self.dropout = nn.Dropout(0.5)  # Has state (p, training mode)
    
    def forward(self, x):
        x = self.relu(x)
        x = self.dropout(x)
        return x

# 2. Functional API (no state — call directly in forward)
class ModelV2(nn.Module):
    def __init__(self):
        super().__init__()
    
    def forward(self, x):
        x = F.relu(x)                           # No state needed
        x = F.dropout(x, p=0.5, training=self.training)  # Must pass training flag!
        return x
```

| Use nn.Module Layer | Use F.functional |
|---------------------|-----------------|
| Has learnable parameters (Linear, Conv2d) | Stateless operations (relu, sigmoid) |
| Has internal state (BatchNorm, Dropout) | When you want a cleaner `__init__` |
| When using hooks | Simple activations |

> **Pro Tip:** For Dropout and BatchNorm, prefer the `nn.Module` versions — they automatically handle `train()`/`eval()` mode switching. The functional versions require you to manually pass `training=self.training`.

### Skip Connections (Residual Connections)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class ResidualBlock(nn.Module):
    """The building block of ResNet — the idea that won ImageNet 2015."""
    def __init__(self, channels):
        super().__init__()
        self.fc1 = nn.Linear(channels, channels)
        self.fc2 = nn.Linear(channels, channels)
        self.bn1 = nn.BatchNorm1d(channels)
        self.bn2 = nn.BatchNorm1d(channels)
    
    def forward(self, x):
        # Save input for skip connection
        identity = x
        
        # Main path
        out = self.fc1(x)
        out = self.bn1(out)
        out = F.relu(out)
        out = self.fc2(out)
        out = self.bn2(out)
        
        # Skip connection: add input directly to output
        out = out + identity   # ← This is the magic!
        out = F.relu(out)
        return out

# Why it works: gradients can flow directly through the skip connection,
# solving the vanishing gradient problem in very deep networks.
```

```
Skip Connection (Residual):

Input x ──┬──→ [Linear] → [BN] → [ReLU] → [Linear] → [BN] → (+) → [ReLU] → Output
           │                                                    ↑
           └────────────────────────────────────────────────────┘
                              Identity (skip)
```

---

## 4. Parameters and Buffers

### What It Is
- **Parameters** (`nn.Parameter`): Tensors that ARE updated by the optimizer (weights, biases)
- **Buffers** (`register_buffer`): Tensors that are NOT updated by the optimizer but ARE part of the model state (running mean in BatchNorm, positional encodings)

### Why It Matters
Understanding the difference ensures:
- The optimizer updates the right things
- `model.to(device)` moves everything needed
- `model.state_dict()` saves everything needed

### Parameters

```python
import torch
import torch.nn as nn

class MyLayer(nn.Module):
    def __init__(self, in_features, out_features):
        super().__init__()
        # nn.Parameter: tells PyTorch "this is learnable — include in .parameters()"
        self.weight = nn.Parameter(torch.randn(out_features, in_features))
        self.bias = nn.Parameter(torch.zeros(out_features))
    
    def forward(self, x):
        return x @ self.weight.T + self.bias

layer = MyLayer(10, 5)

# Iterate over parameters
for name, param in layer.named_parameters():
    print(f"{name}: shape={param.shape}, requires_grad={param.requires_grad}")
# weight: shape=torch.Size([5, 10]), requires_grad=True
# bias: shape=torch.Size([5]), requires_grad=True

# Count total parameters
total_params = sum(p.numel() for p in layer.parameters())
trainable_params = sum(p.numel() for p in layer.parameters() if p.requires_grad)
print(f"Total: {total_params}, Trainable: {trainable_params}")
# Total: 55, Trainable: 55
```

### Buffers

```python
import torch
import torch.nn as nn

class LayerWithBuffer(nn.Module):
    def __init__(self):
        super().__init__()
        self.linear = nn.Linear(10, 5)
        
        # Buffer: moves with .to(device), saved in state_dict, but NOT trained
        self.register_buffer('running_count', torch.zeros(1))
        
        # Persistent=False: won't be in state_dict (temporary tracking)
        self.register_buffer('step', torch.zeros(1), persistent=False)
    
    def forward(self, x):
        self.running_count += 1     # Track how many times forward was called
        self.step += 1
        return self.linear(x)

model = LayerWithBuffer()

# Buffers are NOT in .parameters() but ARE in .state_dict()
print(f"Parameters: {list(model.named_parameters())}")
print(f"Buffers: {list(model.named_buffers())}")
print(f"State dict keys: {model.state_dict().keys()}")
# dict_keys(['linear.weight', 'linear.bias', 'running_count'])
# Note: 'step' is not there (persistent=False)
```

### Freezing Parameters (Transfer Learning)

```python
import torch
import torch.nn as nn

class PretrainedModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.backbone = nn.Sequential(
            nn.Linear(784, 256),
            nn.ReLU(),
            nn.Linear(256, 128),
            nn.ReLU()
        )
        self.classifier = nn.Linear(128, 10)
    
    def forward(self, x):
        features = self.backbone(x)
        return self.classifier(features)

model = PretrainedModel()

# Freeze backbone (don't train it)
for param in model.backbone.parameters():
    param.requires_grad = False

# Only classifier parameters will be updated
trainable = [p for p in model.parameters() if p.requires_grad]
print(f"Trainable params: {sum(p.numel() for p in trainable)}")
# Only classifier: 128*10 + 10 = 1290

# Pass only trainable params to optimizer
optimizer = torch.optim.Adam(
    filter(lambda p: p.requires_grad, model.parameters()),
    lr=0.001
)

# Alternative: pass specific parameter groups with different learning rates
optimizer = torch.optim.Adam([
    {'params': model.backbone.parameters(), 'lr': 1e-5},   # Fine-tune slowly
    {'params': model.classifier.parameters(), 'lr': 1e-3}  # Train faster
])
```

---

## 5. Building Your First Neural Network

### A Complete Classification Network

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MNISTClassifier(nn.Module):
    """
    Classifies 28×28 grayscale digits (0-9).
    Architecture: 784 → 512 → 256 → 128 → 10
    """
    def __init__(self):
        super().__init__()
        
        # Define layers
        self.fc1 = nn.Linear(28 * 28, 512)   # Flatten image → 512
        self.fc2 = nn.Linear(512, 256)
        self.fc3 = nn.Linear(256, 128)
        self.fc4 = nn.Linear(128, 10)         # 10 digit classes
        
        # Regularization
        self.dropout = nn.Dropout(0.2)
        self.bn1 = nn.BatchNorm1d(512)
        self.bn2 = nn.BatchNorm1d(256)
        self.bn3 = nn.BatchNorm1d(128)
    
    def forward(self, x):
        # Flatten: (batch, 1, 28, 28) → (batch, 784)
        x = x.view(x.size(0), -1)
        
        # Hidden layers with BN + ReLU + Dropout
        x = self.dropout(F.relu(self.bn1(self.fc1(x))))
        x = self.dropout(F.relu(self.bn2(self.fc2(x))))
        x = self.dropout(F.relu(self.bn3(self.fc3(x))))
        
        # Output layer — raw logits (no softmax!)
        # CrossEntropyLoss applies softmax internally
        x = self.fc4(x)
        return x

# Instantiate
model = MNISTClassifier()

# Test with random input
batch = torch.randn(64, 1, 28, 28)  # 64 images
logits = model(batch)
print(f"Output shape: {logits.shape}")  # torch.Size([64, 10])

# Get predictions
probs = F.softmax(logits, dim=1)       # Convert to probabilities
preds = logits.argmax(dim=1)            # Get predicted class
print(f"Predictions: {preds[:10]}")     # First 10 predicted digits
```

```
Architecture Diagram:

Input (28×28)          Hidden Layers                    Output
┌──────────┐    ┌────────────────────┐              ┌─────────┐
│  Flatten  │    │ Linear(784→512)    │              │ Linear  │
│ 784 units │──→ │ BatchNorm → ReLU   │──→ ... ──→  │ (128→10)│──→ Logits
│           │    │ Dropout(0.2)       │              │         │    [10]
└──────────┘    └────────────────────┘              └─────────┘
```

### Multi-Output Network (Regression + Classification)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MultiTaskModel(nn.Module):
    """Predicts both a continuous value and a category from shared features."""
    def __init__(self, input_dim):
        super().__init__()
        # Shared backbone
        self.shared = nn.Sequential(
            nn.Linear(input_dim, 256),
            nn.ReLU(),
            nn.Linear(256, 128),
            nn.ReLU()
        )
        # Task-specific heads
        self.regression_head = nn.Linear(128, 1)     # Continuous output
        self.classification_head = nn.Linear(128, 5) # 5 classes
    
    def forward(self, x):
        features = self.shared(x)
        reg_output = self.regression_head(features)
        cls_output = self.classification_head(features)
        return reg_output, cls_output  # Return both!

model = MultiTaskModel(input_dim=50)
x = torch.randn(32, 50)
reg_out, cls_out = model(x)
print(f"Regression: {reg_out.shape}")      # torch.Size([32, 1])
print(f"Classification: {cls_out.shape}")  # torch.Size([32, 5])
```

---

## 6. nn.Sequential and Model Composition

### What It Is
`nn.Sequential` is a container that chains modules in order — each module's output becomes the next module's input. Great for simple, linear architectures.

### nn.Sequential

```python
import torch.nn as nn

# === Basic Sequential ===
model = nn.Sequential(
    nn.Linear(784, 256),
    nn.ReLU(),
    nn.Dropout(0.3),
    nn.Linear(256, 128),
    nn.ReLU(),
    nn.Linear(128, 10)
)

# Forward pass: data flows through each layer in order
output = model(torch.randn(32, 784))
print(output.shape)  # torch.Size([32, 10])

# === Named Sequential (better for debugging) ===
from collections import OrderedDict

model = nn.Sequential(OrderedDict([
    ('flatten', nn.Flatten()),
    ('fc1', nn.Linear(784, 256)),
    ('relu1', nn.ReLU()),
    ('fc2', nn.Linear(256, 10)),
]))

# Access layers by name
print(model.fc1)  # Linear(in_features=784, out_features=256, bias=True)
```

### nn.ModuleList and nn.ModuleDict

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# === ModuleList — dynamic number of layers ===
class DynamicNet(nn.Module):
    def __init__(self, input_dim, hidden_dim, n_layers):
        super().__init__()
        # ⚠️ MUST use nn.ModuleList, NOT a regular Python list!
        # Regular list won't register parameters!
        self.layers = nn.ModuleList([
            nn.Linear(input_dim if i == 0 else hidden_dim, hidden_dim)
            for i in range(n_layers)
        ])
        self.output = nn.Linear(hidden_dim, 1)
    
    def forward(self, x):
        for layer in self.layers:
            x = F.relu(layer(x))
        return self.output(x)

model = DynamicNet(input_dim=50, hidden_dim=128, n_layers=5)
print(f"Number of layer groups: {len(model.layers)}")  # 5

# === ModuleDict — named module selection ===
class ConditionalNet(nn.Module):
    def __init__(self):
        super().__init__()
        self.heads = nn.ModuleDict({
            'regression': nn.Linear(128, 1),
            'classification': nn.Linear(128, 10),
            'embedding': nn.Linear(128, 64)
        })
        self.backbone = nn.Linear(784, 128)
    
    def forward(self, x, task='classification'):
        features = F.relu(self.backbone(x))
        return self.heads[task](features)  # Select head dynamically!

model = ConditionalNet()
x = torch.randn(32, 784)
cls_out = model(x, task='classification')  # Shape: [32, 10]
reg_out = model(x, task='regression')      # Shape: [32, 1]
```

> **⚠️ Critical Mistake:** Using a plain Python `list` instead of `nn.ModuleList`:
> ```python
> # WRONG — parameters NOT registered!
> self.layers = [nn.Linear(10, 10) for _ in range(5)]
> 
> # CORRECT — parameters registered properly
> self.layers = nn.ModuleList([nn.Linear(10, 10) for _ in range(5)])
> ```

---

## 7. Custom Layers

### What It Is
Sometimes built-in layers aren't enough. You can create custom layers by subclassing `nn.Module` — adding any computation, any parameter, any logic.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# === Custom attention layer ===
class SimpleSelfAttention(nn.Module):
    """Scaled dot-product self-attention (single head)."""
    def __init__(self, embed_dim):
        super().__init__()
        self.query = nn.Linear(embed_dim, embed_dim)
        self.key = nn.Linear(embed_dim, embed_dim)
        self.value = nn.Linear(embed_dim, embed_dim)
        self.scale = embed_dim ** 0.5
    
    def forward(self, x):
        # x shape: (batch, seq_len, embed_dim)
        Q = self.query(x)
        K = self.key(x)
        V = self.value(x)
        
        # Attention scores
        scores = torch.bmm(Q, K.transpose(1, 2)) / self.scale
        weights = F.softmax(scores, dim=-1)
        
        # Weighted sum of values
        output = torch.bmm(weights, V)
        return output

attn = SimpleSelfAttention(embed_dim=64)
x = torch.randn(8, 20, 64)      # batch=8, seq=20, dim=64
out = attn(x)
print(f"Attention output: {out.shape}")  # torch.Size([8, 20, 64])

# === Custom layer with learnable scaling ===
class ScaleLayer(nn.Module):
    """Learnable per-feature scaling (useful in normalizations)."""
    def __init__(self, num_features):
        super().__init__()
        self.scale = nn.Parameter(torch.ones(num_features))
    
    def forward(self, x):
        return x * self.scale

# === Custom activation function ===
class Swish(nn.Module):
    """Swish: x * sigmoid(beta * x) with learnable beta."""
    def __init__(self):
        super().__init__()
        self.beta = nn.Parameter(torch.ones(1))
    
    def forward(self, x):
        return x * torch.sigmoid(self.beta * x)
```

---

## 8. Model Inspection and Summary

### Inspecting Your Model

```python
import torch
import torch.nn as nn

class SampleModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.features = nn.Sequential(
            nn.Linear(784, 256),
            nn.ReLU(),
            nn.Linear(256, 128),
            nn.ReLU()
        )
        self.classifier = nn.Linear(128, 10)
    
    def forward(self, x):
        return self.classifier(self.features(x))

model = SampleModel()

# === Print model structure ===
print(model)
# SampleModel(
#   (features): Sequential(
#     (0): Linear(in_features=784, out_features=256, bias=True)
#     (1): ReLU()
#     (2): Linear(in_features=256, out_features=128, bias=True)
#     (3): ReLU()
#   )
#   (classifier): Linear(in_features=128, out_features=10, bias=True)
# )

# === Count parameters ===
total = sum(p.numel() for p in model.parameters())
trainable = sum(p.numel() for p in model.parameters() if p.requires_grad)
print(f"Total params: {total:,}")       # 235,146
print(f"Trainable params: {trainable:,}")

# === List all parameters with shapes ===
for name, param in model.named_parameters():
    print(f"{name:30s} {str(param.shape):20s} {param.numel():>8,}")
# features.0.weight              torch.Size([256, 784])    200,704
# features.0.bias                torch.Size([256])              256
# features.2.weight              torch.Size([128, 256])     32,768
# features.2.bias                torch.Size([128])              128
# classifier.weight              torch.Size([10, 128])       1,280
# classifier.bias                torch.Size([10])                10

# === List all modules ===
for name, module in model.named_modules():
    print(f"{name}: {module.__class__.__name__}")

# === Children (direct submodules only) ===
for name, child in model.named_children():
    print(f"{name}: {child.__class__.__name__}")
```

### Using torchinfo for Model Summary

```python
# pip install torchinfo
from torchinfo import summary

model = SampleModel()
summary(model, input_size=(32, 784), 
        col_names=["input_size", "output_size", "num_params", "trainable"])

# ==========================================================================================
# Layer (type:depth-idx)         Input Shape    Output Shape   Param #    Trainable
# ==========================================================================================
# SampleModel                    [32, 784]      [32, 10]       --         True
# ├─Sequential: 1-1              [32, 784]      [32, 128]      --         True
# │  ├─Linear: 2-1               [32, 784]      [32, 256]      200,960    True
# │  ├─ReLU: 2-2                 [32, 256]      [32, 256]      --         --
# │  ├─Linear: 2-3               [32, 256]      [32, 128]      32,896     True
# │  └─ReLU: 2-4                 [32, 128]      [32, 128]      --         --
# ├─Linear: 1-2                  [32, 128]      [32, 10]       1,290      True
# ==========================================================================================
# Total params: 235,146
# Trainable params: 235,146
```

---

## 9. Saving and Loading Models

### What It Is
Saving captures a model's learned weights (and optionally optimizer state) to disk. Loading restores them — for resuming training, deployment, or sharing.

### Two Approaches

```python
import torch
import torch.nn as nn

model = SampleModel()

# ═══════════════════════════════════════════════════════
# Method 1: Save state_dict (RECOMMENDED)
# ═══════════════════════════════════════════════════════
# Saves only the learned parameters, not the model code
torch.save(model.state_dict(), 'model_weights.pth')

# Loading — must recreate the model architecture first
loaded_model = SampleModel()                              # Create same architecture
loaded_model.load_state_dict(torch.load('model_weights.pth', weights_only=True))
loaded_model.eval()                                       # Set to eval mode!

# ═══════════════════════════════════════════════════════
# Method 2: Save entire model (NOT recommended)
# ═══════════════════════════════════════════════════════
# Uses pickle — fragile, breaks if you move/rename files
torch.save(model, 'entire_model.pth')
loaded_model = torch.load('entire_model.pth', weights_only=False)

# ═══════════════════════════════════════════════════════
# Best Practice: Save a checkpoint (for resuming training)
# ═══════════════════════════════════════════════════════
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)

checkpoint = {
    'epoch': 10,
    'model_state_dict': model.state_dict(),
    'optimizer_state_dict': optimizer.state_dict(),
    'loss': 0.25,
    'best_accuracy': 0.95
}
torch.save(checkpoint, 'checkpoint.pth')

# Resume training from checkpoint
checkpoint = torch.load('checkpoint.pth', weights_only=False)
model.load_state_dict(checkpoint['model_state_dict'])
optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
start_epoch = checkpoint['epoch']
print(f"Resuming from epoch {start_epoch}, loss={checkpoint['loss']:.4f}")
```

> **Why state_dict is better:**
> - Portable: works even if model code is reorganized
> - Smaller files: only weights, not code
> - Flexible: load weights into a slightly different architecture (with `strict=False`)
> - Secure: `weights_only=True` prevents arbitrary code execution from pickle

---

## 10. Common Mistakes

| # | Mistake | Problem | Fix |
|---|---------|---------|-----|
| 1 | Forgetting `super().__init__()` | Parameters not registered, `.to(device)` fails | Always call `super().__init__()` first |
| 2 | Using Python `list` instead of `nn.ModuleList` | Parameters invisible to optimizer | Use `nn.ModuleList` / `nn.ModuleDict` |
| 3 | Calling `model.forward(x)` | Skips hooks and mode checks | Use `model(x)` |
| 4 | Not switching `train()`/`eval()` | Dropout/BatchNorm wrong during inference | `model.eval()` before testing, `model.train()` before training |
| 5 | Applying softmax before `CrossEntropyLoss` | Double softmax — terrible gradients | `CrossEntropyLoss` already includes log-softmax |
| 6 | Wrong `view`/`reshape` dimensions | Silent data corruption (data reshuffled) | Print shapes at each step; verify with known inputs |
| 7 | Not using `weights_only=True` in `torch.load` | Security vulnerability (pickle exploits) | Always use `weights_only=True` for state dicts |

---

## 11. Interview Questions

### Conceptual

**Q1: What is `nn.Module` and why do we subclass it?**
> `nn.Module` is the base class for all neural network components. We subclass it because it: (1) automatically tracks parameters via `__init__` attribute assignment, (2) provides `.to(device)`, `.train()`, `.eval()`, `.parameters()`, `.state_dict()`, (3) integrates with autograd, (4) supports hooks for inspection, (5) enables recursive module composition.

**Q2: What's the difference between `nn.Module` layer and `F.functional` call?**
> `nn.Module` layers are classes with state (parameters, buffers, training mode). `F.functional` calls are stateless functions. Use modules for layers with parameters or state (Linear, BatchNorm, Dropout). Use functional for pure computations (relu, sigmoid). Dropout is tricky — the functional version requires you to pass `training=self.training` manually.

**Q3: How would you freeze part of a model for transfer learning?**
> Set `requires_grad = False` on the parameters you want to freeze. Pass only the unfrozen parameters to the optimizer: `optimizer = Adam(filter(lambda p: p.requires_grad, model.parameters()))`. For differential learning rates, use parameter groups.

**Q4: Why should we save `state_dict()` rather than the entire model?**
> Saving the entire model uses Python's pickle, which serializes the code structure — it breaks when files are moved, renamed, or refactored. `state_dict()` saves only the parameter values (a dict of tensor names → tensors), making it portable, smaller, and more secure (no arbitrary code execution risk).

**Q5: Explain the difference between `nn.ModuleList` and a Python list of modules.**
> `nn.ModuleList` registers its contents as submodules — their parameters appear in `model.parameters()`, they move with `model.to(device)`, and they're saved in `state_dict()`. A plain Python list is invisible to PyTorch — parameters won't be trained, moved, or saved.

### Coding

**Q6: Build a model that takes variable-length input and handles it in `forward()`.**
```python
class VariableLengthNet(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim, max_layers=5):
        super().__init__()
        self.layers = nn.ModuleList([
            nn.Linear(input_dim if i == 0 else hidden_dim, hidden_dim)
            for i in range(max_layers)
        ])
        self.output = nn.Linear(hidden_dim, output_dim)
    
    def forward(self, x, n_layers=None):
        n = n_layers or len(self.layers)
        for layer in self.layers[:n]:
            x = torch.relu(layer(x))
        return self.output(x)
```

---

## 12. Quick Reference

### Model Building Cheat Sheet

| Component | Usage | Example |
|-----------|-------|---------|
| `nn.Linear` | Fully connected layer | `nn.Linear(in, out)` |
| `nn.Conv2d` | 2D convolution | `nn.Conv2d(in_ch, out_ch, kernel)` |
| `nn.ReLU` | Activation | `nn.ReLU()` |
| `nn.Dropout` | Regularization | `nn.Dropout(p=0.5)` |
| `nn.BatchNorm1d` | Normalization | `nn.BatchNorm1d(features)` |
| `nn.Embedding` | Token → vector | `nn.Embedding(vocab, dim)` |
| `nn.Sequential` | Chain layers | `nn.Sequential(l1, l2, l3)` |
| `nn.ModuleList` | Dynamic layers | `nn.ModuleList([...])` |
| `nn.ModuleDict` | Named modules | `nn.ModuleDict({...})` |

### Essential Model Methods

| Method | Purpose |
|--------|---------|
| `model(x)` | Forward pass (calls `__call__` → `forward`) |
| `model.parameters()` | Iterator over all learnable parameters |
| `model.named_parameters()` | Iterator with names |
| `model.to(device)` | Move model to CPU/GPU |
| `model.train()` | Set training mode |
| `model.eval()` | Set evaluation mode |
| `model.state_dict()` | Get parameter dictionary |
| `model.load_state_dict(d)` | Load parameters |
| `model.zero_grad()` | Zero all parameter gradients |
| `model.children()` | Direct submodules |
| `model.modules()` | All modules recursively |

### Pattern: Standard Model Template

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MyModel(nn.Module):
    def __init__(self, config):
        super().__init__()
        # Define layers here
    
    def forward(self, x):
        # Define computation here
        return x

# Instantiate → Device → Train/Eval → Save
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = MyModel(config).to(device)
model.train()    # Before training
model.eval()     # Before inference
torch.save(model.state_dict(), 'model.pth')
```

---

*Previous: [Chapter 01 — PyTorch Fundamentals](01-PyTorch-Fundamentals.md) | Next: [Chapter 03 — Training Loop and Optimization](03-Training-Loop-and-Optimization.md)*
