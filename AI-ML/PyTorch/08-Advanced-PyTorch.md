# Chapter 08: Advanced PyTorch

## Table of Contents
- [Custom Layers and Modules](#custom-layers-and-modules)
- [Hooks — Peeking Inside Networks](#hooks--peeking-inside-networks)
- [Mixed Precision Training](#mixed-precision-training)
- [Gradient Accumulation](#gradient-accumulation)
- [Gradient Clipping](#gradient-clipping)
- [Custom Loss Functions](#custom-loss-functions)
- [Custom Learning Rate Schedulers](#custom-learning-rate-schedulers)
- [Dynamic Computation Graphs](#dynamic-computation-graphs)
- [Memory Optimization](#memory-optimization)
- [Debugging PyTorch Models](#debugging-pytorch-models)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Custom Layers and Modules

### What It Is

Custom layers let you create your own building blocks that plug into PyTorch's `nn.Module` system — just like LEGO bricks you design yourself. Whenever PyTorch's built-in layers (Linear, Conv2d, etc.) can't do what you need, you build your own.

### Why It Matters

- Research papers constantly propose new layer types (Squeeze-and-Excitation, Spatial Attention, etc.)
- Production models often need domain-specific operations
- Understanding custom modules is essential for reading and implementing papers

### Building a Custom Layer

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# ============================================================
# BASIC CUSTOM LAYER: Learnable Scaling
# ============================================================

class LearnableScale(nn.Module):
    """
    A simple layer that learns a per-channel scaling factor.
    Like BatchNorm's gamma, but without normalization.
    """
    def __init__(self, num_features):
        super().__init__()
        # nn.Parameter tells PyTorch: "this tensor has learnable weights"
        # It automatically registers for gradient computation and optimizer updates
        self.scale = nn.Parameter(torch.ones(num_features))  # Initialize to 1
        self.bias = nn.Parameter(torch.zeros(num_features))  # Initialize to 0
    
    def forward(self, x):
        # x shape: (batch, channels, height, width) for images
        # Reshape scale to broadcast: (1, C, 1, 1)
        return x * self.scale.view(1, -1, 1, 1) + self.bias.view(1, -1, 1, 1)

# Usage
scale_layer = LearnableScale(64)
x = torch.randn(8, 64, 32, 32)  # batch=8, channels=64, 32x32
out = scale_layer(x)
print(out.shape)  # torch.Size([8, 64, 32, 32])

# Verify parameters are registered
for name, param in scale_layer.named_parameters():
    print(f"{name}: shape={param.shape}, requires_grad={param.requires_grad}")
# scale: shape=torch.Size([64]), requires_grad=True
# bias: shape=torch.Size([64]), requires_grad=True
```

### Squeeze-and-Excitation Block (Real Research Example)

```python
class SEBlock(nn.Module):
    """
    Squeeze-and-Excitation Block (Hu et al., 2018).
    
    Learns to re-weight channels based on global information.
    
    Think of it as: "Which channels are important for THIS input?"
    
    ┌──────────────────────────────────────────────────────┐
    │  Input: (B, C, H, W)                                 │
    │     │                                                 │
    │     ▼                                                 │
    │  [Global Avg Pool] → (B, C, 1, 1)    ← SQUEEZE       │
    │     │                                                 │
    │     ▼                                                 │
    │  [FC → ReLU → FC → Sigmoid] → (B, C, 1, 1) ← EXCITE │
    │     │                                                 │
    │     ▼                                                 │
    │  [Multiply with Input] → (B, C, H, W) ← SCALE       │
    └──────────────────────────────────────────────────────┘
    """
    def __init__(self, channels, reduction=16):
        super().__init__()
        self.squeeze = nn.AdaptiveAvgPool2d(1)  # Global average pooling
        self.excitation = nn.Sequential(
            nn.Linear(channels, channels // reduction, bias=False),
            nn.ReLU(inplace=True),
            nn.Linear(channels // reduction, channels, bias=False),
            nn.Sigmoid()
        )
    
    def forward(self, x):
        b, c, _, _ = x.size()
        
        # Squeeze: (B, C, H, W) → (B, C, 1, 1) → (B, C)
        y = self.squeeze(x).view(b, c)
        
        # Excitation: (B, C) → (B, C//r) → (B, C) with sigmoid
        y = self.excitation(y).view(b, c, 1, 1)
        
        # Scale: multiply each channel by its importance weight
        return x * y  # Broadcasting: (B,C,H,W) * (B,C,1,1)

# Usage
se_block = SEBlock(channels=256, reduction=16)
x = torch.randn(4, 256, 14, 14)
out = se_block(x)
print(out.shape)  # torch.Size([4, 256, 14, 14])
```

### Custom Module with Sub-Modules

```python
class ResidualBlock(nn.Module):
    """
    Residual block with optional squeeze-and-excitation.
    
    ┌─────────┐
    │  Input   │──────────────────────┐
    └────┬─────┘                      │ (skip connection)
         │                            │
    ┌────▼─────┐                      │
    │  Conv3x3 │                      │
    │  BN+ReLU │                      │
    └────┬─────┘                      │
         │                            │
    ┌────▼─────┐                      │
    │  Conv3x3 │                      │
    │  BN      │                      │
    └────┬─────┘                      │
         │                            │
    ┌────▼─────┐                      │
    │ SE Block │ (optional)           │
    └────┬─────┘                      │
         │         ┌──────────────────┘
    ┌────▼─────────▼────┐
    │       ADD          │
    │      + ReLU        │
    └────────┬───────────┘
             │
        ┌────▼─────┐
        │  Output   │
        └──────────┘
    """
    def __init__(self, in_channels, out_channels, stride=1, use_se=True):
        super().__init__()
        
        self.conv1 = nn.Conv2d(in_channels, out_channels, 3, stride, 1, bias=False)
        self.bn1 = nn.BatchNorm2d(out_channels)
        self.conv2 = nn.Conv2d(out_channels, out_channels, 3, 1, 1, bias=False)
        self.bn2 = nn.BatchNorm2d(out_channels)
        self.relu = nn.ReLU(inplace=True)
        
        # Optional SE block
        self.se = SEBlock(out_channels) if use_se else nn.Identity()
        
        # Skip connection: adjust dimensions if needed
        self.skip = nn.Identity()
        if stride != 1 or in_channels != out_channels:
            self.skip = nn.Sequential(
                nn.Conv2d(in_channels, out_channels, 1, stride, bias=False),
                nn.BatchNorm2d(out_channels)
            )
    
    def forward(self, x):
        identity = self.skip(x)
        
        out = self.relu(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        out = self.se(out)  # Channel attention
        
        out += identity  # Residual connection
        out = self.relu(out)
        return out

# Build a small network from custom blocks
class CustomNet(nn.Module):
    def __init__(self, num_classes=10):
        super().__init__()
        self.stem = nn.Sequential(
            nn.Conv2d(3, 64, 7, 2, 3, bias=False),
            nn.BatchNorm2d(64),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(3, 2, 1)
        )
        self.layer1 = ResidualBlock(64, 128, stride=2, use_se=True)
        self.layer2 = ResidualBlock(128, 256, stride=2, use_se=True)
        self.layer3 = ResidualBlock(256, 512, stride=2, use_se=True)
        self.head = nn.Sequential(
            nn.AdaptiveAvgPool2d(1),
            nn.Flatten(),
            nn.Linear(512, num_classes)
        )
    
    def forward(self, x):
        x = self.stem(x)
        x = self.layer1(x)
        x = self.layer2(x)
        x = self.layer3(x)
        x = self.head(x)
        return x

model = CustomNet(num_classes=10)
x = torch.randn(2, 3, 224, 224)
out = model(x)
print(out.shape)  # torch.Size([2, 10])
print(f"Parameters: {sum(p.numel() for p in model.parameters()):,}")
```

### Registering Buffers (Non-Learnable State)

```python
class LayerNormCustom(nn.Module):
    """
    Use register_buffer for tensors that:
    - Should be saved/loaded with the model
    - Should move to GPU with model.to(device)
    - But should NOT be updated by the optimizer
    """
    def __init__(self, num_features, eps=1e-5):
        super().__init__()
        self.eps = eps
        # Parameters: learnable
        self.gamma = nn.Parameter(torch.ones(num_features))
        self.beta = nn.Parameter(torch.zeros(num_features))
        
        # Buffer: not learnable, but part of model state
        # Example: a running count of forward passes
        self.register_buffer('num_batches_tracked', torch.tensor(0, dtype=torch.long))
    
    def forward(self, x):
        self.num_batches_tracked += 1
        
        mean = x.mean(dim=-1, keepdim=True)
        var = x.var(dim=-1, keepdim=True, unbiased=False)
        x_norm = (x - mean) / torch.sqrt(var + self.eps)
        return self.gamma * x_norm + self.beta

# Buffers move with the model
layer = LayerNormCustom(256)
layer = layer.to('cuda' if torch.cuda.is_available() else 'cpu')
# num_batches_tracked is also on the same device now

# Buffers are saved in state_dict
print(layer.state_dict().keys())
# odict_keys(['gamma', 'beta', 'num_batches_tracked'])
```

---

## Hooks — Peeking Inside Networks

### What Hooks Are

**Simple Explanation:** Hooks are like wiretaps for neural networks. They let you intercept data flowing through any layer — see the inputs, outputs, and gradients — without modifying the model's code.

### Why Hooks Matter

1. **Debugging**: See what each layer receives and produces
2. **Feature visualization**: Extract intermediate features (e.g., for Grad-CAM)
3. **Gradient analysis**: Monitor gradient flow to detect vanishing/exploding gradients
4. **Pruning**: Identify unimportant neurons based on activation patterns
5. **Model interpretability**: Understand what the model "sees"

### Types of Hooks

```
┌──────────────────────────────────────────────────────────────┐
│                       HOOK TYPES                              │
├───────────────────┬──────────────────────────────────────────┤
│  Forward Hook     │  Called AFTER forward pass of a module    │
│                   │  Access: module, input, output            │
│                   │  Use: Extract features, log activations   │
├───────────────────┼──────────────────────────────────────────┤
│  Forward Pre-Hook │  Called BEFORE forward pass               │
│                   │  Access: module, input                    │
│                   │  Use: Modify input, log shapes            │
├───────────────────┼──────────────────────────────────────────┤
│  Backward Hook    │  Called AFTER backward pass               │
│                   │  Access: module, grad_input, grad_output  │
│                   │  Use: Gradient analysis, gradient clipping│
└───────────────────┴──────────────────────────────────────────┘
```

### Forward Hooks — Extracting Intermediate Features

```python
import torch
import torch.nn as nn
from torchvision import models
from torchvision.models import ResNet50_Weights

model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)
model.eval()

# ============================================================
# METHOD 1: Simple forward hook to capture activations
# ============================================================

# Dictionary to store intermediate outputs
activations = {}

def get_activation(name):
    """Factory function that returns a hook function."""
    def hook(module, input, output):
        # Store the output of this layer
        activations[name] = output.detach()
    return hook

# Register hooks on specific layers
hook1 = model.layer1.register_forward_hook(get_activation('layer1'))
hook2 = model.layer3.register_forward_hook(get_activation('layer3'))
hook3 = model.layer4.register_forward_hook(get_activation('layer4'))

# Run forward pass
x = torch.randn(1, 3, 224, 224)
with torch.no_grad():
    output = model(x)

# Access intermediate features
for name, feat in activations.items():
    print(f"{name}: {feat.shape}")
# layer1: torch.Size([1, 256, 56, 56])
# layer3: torch.Size([1, 1024, 14, 14])
# layer4: torch.Size([1, 2048, 7, 7])

# IMPORTANT: Always remove hooks when done to prevent memory leaks!
hook1.remove()
hook2.remove()
hook3.remove()
```

### Feature Extraction with Hooks (Clean Pattern)

```python
class FeatureExtractor(nn.Module):
    """
    Clean wrapper to extract features from any layer of any model.
    Used for tasks like object detection (FPN) or style transfer.
    """
    def __init__(self, model, layer_names):
        super().__init__()
        self.model = model
        self.layer_names = layer_names
        self.features = {}
        self._hooks = []
        
        # Register hooks on named layers
        for name, module in self.model.named_modules():
            if name in layer_names:
                hook = module.register_forward_hook(self._make_hook(name))
                self._hooks.append(hook)
    
    def _make_hook(self, name):
        def hook(module, input, output):
            self.features[name] = output
        return hook
    
    def forward(self, x):
        self.features = {}  # Clear previous features
        _ = self.model(x)   # Run forward pass (hooks populate self.features)
        return {name: self.features[name] for name in self.layer_names}
    
    def remove_hooks(self):
        """Clean up hooks."""
        for hook in self._hooks:
            hook.remove()
        self._hooks.clear()

# Usage
model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)
model.eval()

extractor = FeatureExtractor(model, ['layer1', 'layer2', 'layer3', 'layer4'])
x = torch.randn(1, 3, 224, 224)

with torch.no_grad():
    features = extractor(x)

for name, feat in features.items():
    print(f"{name}: {feat.shape}")

# Don't forget to clean up
extractor.remove_hooks()
```

### Backward Hooks — Gradient Analysis

```python
import torch
import torch.nn as nn

model = nn.Sequential(
    nn.Linear(784, 256),
    nn.ReLU(),
    nn.Linear(256, 128),
    nn.ReLU(),
    nn.Linear(128, 10)
)

# Monitor gradient magnitudes to detect vanishing/exploding gradients
gradient_stats = {}

def grad_hook(name):
    def hook(module, grad_input, grad_output):
        # grad_output: gradient flowing INTO this layer (from loss)
        # grad_input: gradient flowing OUT of this layer (to earlier layers)
        if grad_output[0] is not None:
            gradient_stats[name] = {
                'mean': grad_output[0].abs().mean().item(),
                'max': grad_output[0].abs().max().item(),
                'std': grad_output[0].std().item(),
            }
    return hook

# Register backward hooks
for name, module in model.named_modules():
    if isinstance(module, nn.Linear):
        module.register_full_backward_hook(grad_hook(name))

# Forward + backward
x = torch.randn(32, 784)
y = torch.randint(0, 10, (32,))
output = model(x)
loss = nn.CrossEntropyLoss()(output, y)
loss.backward()

# Check gradient health
print("\nGradient Statistics:")
for name, stats in gradient_stats.items():
    print(f"  {name}: mean={stats['mean']:.6f}, max={stats['max']:.6f}, std={stats['std']:.6f}")

# Healthy gradients: mean ~0.01-1.0
# Vanishing: mean < 1e-6
# Exploding: mean > 100
```

### Grad-CAM Implementation (Model Interpretability)

```python
import torch
import torch.nn.functional as F
import numpy as np

class GradCAM:
    """
    Gradient-weighted Class Activation Mapping.
    Shows which parts of the image the model focuses on for a specific class.
    
    How it works:
    1. Extract feature maps from the last conv layer
    2. Get gradients of target class w.r.t. those feature maps
    3. Weight each feature map by its average gradient (importance)
    4. Sum weighted maps and apply ReLU
    """
    def __init__(self, model, target_layer):
        self.model = model
        self.model.eval()
        self.gradients = None
        self.activations = None
        
        # Register hooks on the target layer
        target_layer.register_forward_hook(self._forward_hook)
        target_layer.register_full_backward_hook(self._backward_hook)
    
    def _forward_hook(self, module, input, output):
        self.activations = output.detach()
    
    def _backward_hook(self, module, grad_input, grad_output):
        self.gradients = grad_output[0].detach()
    
    def generate(self, input_image, target_class=None):
        """Generate Grad-CAM heatmap."""
        # Forward pass
        output = self.model(input_image)
        
        if target_class is None:
            target_class = output.argmax(dim=1).item()
        
        # Backward pass for target class
        self.model.zero_grad()
        output[0, target_class].backward()
        
        # Global average pooling of gradients → importance weights
        # Shape: (1, C, H, W) → (C,)
        weights = self.gradients.mean(dim=[2, 3]).squeeze()
        
        # Weighted combination of activations
        # (C,) × (C, H, W) → (H, W)
        cam = torch.zeros(self.activations.shape[2:], dtype=torch.float32)
        for i, w in enumerate(weights):
            cam += w * self.activations[0, i]
        
        # ReLU: only keep positive contributions
        cam = F.relu(cam)
        
        # Normalize to [0, 1]
        cam = cam - cam.min()
        if cam.max() > 0:
            cam = cam / cam.max()
        
        # Resize to input image size
        cam = F.interpolate(
            cam.unsqueeze(0).unsqueeze(0),
            size=input_image.shape[2:],
            mode='bilinear',
            align_corners=False
        ).squeeze()
        
        return cam.numpy()

# Usage
from torchvision import models
from torchvision.models import ResNet50_Weights

model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)
grad_cam = GradCAM(model, model.layer4[-1])

# Generate heatmap
img = torch.randn(1, 3, 224, 224)  # Replace with real image
heatmap = grad_cam.generate(img, target_class=243)  # 243 = "bull mastiff"
print(f"Heatmap shape: {heatmap.shape}")  # (224, 224)
```

---

## Mixed Precision Training

### What It Is

**Simple Explanation:** Instead of using 32-bit numbers for everything (like using a ruler marked in millimeters), mixed precision uses 16-bit numbers where possible (ruler marked in centimeters) and 32-bit only where needed. This makes training **2x faster** while using **half the memory**.

### Why It Matters

- **2x faster training** on modern GPUs (Tensor Cores)
- **50% less GPU memory** → larger batch sizes or bigger models
- **Almost no accuracy loss** when done correctly
- Essential for training large models (GPT, ViT, etc.)

### How It Works

```
┌──────────────────────────────────────────────────────────────┐
│             MIXED PRECISION TRAINING FLOW                     │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Master Weights (FP32)                                       │
│       │                                                      │
│       ▼  Copy to FP16                                        │
│  FP16 Weights ──► Forward Pass (FP16) ──► FP16 Loss         │
│                                                │              │
│                                    Loss Scaling (×65536)      │
│                                                │              │
│                                                ▼              │
│                          Backward Pass (FP16 gradients)       │
│                                                │              │
│                                    Unscale gradients (÷65536) │
│                                                │              │
│                                                ▼              │
│  Master Weights (FP32) ◄── Update with FP32 gradients       │
│                                                               │
│  Why loss scaling?                                           │
│  FP16 can't represent very small gradients (< 6e-8).         │
│  Scaling up the loss prevents gradients from becoming zero.   │
└──────────────────────────────────────────────────────────────┘
```

### Data Types Comparison

| Type | Bits | Range | Precision | Use Case |
|------|------|-------|-----------|----------|
| FP32 | 32 | ±3.4e38 | ~7 decimal digits | Master weights, loss, small gradients |
| FP16 | 16 | ±65504 | ~3.5 decimal digits | Forward/backward pass |
| BF16 | 16 | ±3.4e38 | ~3 decimal digits | Same range as FP32, less precision |
| TF32 | 19 | ±3.4e38 | ~3.5 decimal digits | NVIDIA Ampere+ internal format |

> **Pro Tip:** BFloat16 (BF16) is preferred over FP16 on Ampere+ GPUs (A100, RTX 3090+) because it has the same range as FP32 and rarely needs loss scaling.

### PyTorch AMP (Automatic Mixed Precision)

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.amp import autocast, GradScaler

# ============================================================
# STANDARD MIXED PRECISION TRAINING
# ============================================================

model = nn.Sequential(
    nn.Linear(784, 512),
    nn.ReLU(),
    nn.Linear(512, 256),
    nn.ReLU(),
    nn.Linear(256, 10)
).cuda()

optimizer = optim.Adam(model.parameters(), lr=1e-3)
criterion = nn.CrossEntropyLoss()

# GradScaler handles loss scaling automatically
scaler = GradScaler('cuda')

for epoch in range(10):
    for images, labels in train_loader:
        images, labels = images.cuda(), labels.cuda()
        
        optimizer.zero_grad()
        
        # Step 1: Forward pass in mixed precision
        with autocast('cuda'):  # Automatically chooses FP16/FP32 per operation
            outputs = model(images)
            loss = criterion(outputs, labels)
        # NOTE: loss is still in FP32 here
        
        # Step 2: Backward pass with scaled loss
        scaler.scale(loss).backward()
        # Gradients are now scaled (in FP16)
        
        # Step 3: Unscale gradients and step optimizer
        scaler.step(optimizer)
        # scaler.step does two things:
        # 1. Unscale gradients (÷ scale factor)
        # 2. Call optimizer.step() IF no inf/nan gradients
        
        # Step 4: Update the scale factor
        scaler.update()
        # If inf/nan detected: increase scale less aggressively
        # If no issues: increase scale for more precision
```

### AMP with Gradient Clipping

```python
import torch
from torch.amp import autocast, GradScaler

scaler = GradScaler('cuda')

for images, labels in train_loader:
    images, labels = images.cuda(), labels.cuda()
    
    optimizer.zero_grad()
    
    with autocast('cuda'):
        outputs = model(images)
        loss = criterion(outputs, labels)
    
    scaler.scale(loss).backward()
    
    # IMPORTANT: Unscale BEFORE clipping!
    # Gradients must be in their true scale for clip to work correctly
    scaler.unscale_(optimizer)
    
    # Now clip in FP32
    torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
    
    scaler.step(optimizer)
    scaler.update()
```

### BFloat16 (Simpler, No Scaler Needed)

```python
import torch
from torch.amp import autocast

# BFloat16 on Ampere+ GPUs — no GradScaler needed!
# BF16 has the same range as FP32, so underflow is rarely an issue

for images, labels in train_loader:
    images, labels = images.cuda(), labels.cuda()
    
    optimizer.zero_grad()
    
    with autocast('cuda', dtype=torch.bfloat16):
        outputs = model(images)
        loss = criterion(outputs, labels)
    
    loss.backward()       # No scaler needed!
    optimizer.step()

# Check if BF16 is supported
print(torch.cuda.is_bf16_supported())  # True on Ampere+
```

### What autocast Handles Automatically

```python
# autocast promotes/demotes operations intelligently:

# Operations that run in FP16 (safe, benefit from speed):
#   - Linear layers (matmul)
#   - Convolutions
#   - BMM (batch matrix multiply)

# Operations that stay in FP32 (need precision):
#   - Loss functions
#   - Softmax
#   - Layer normalization
#   - Log, exp, pow
#   - Reductions (sum, mean)

# You can check what autocast does:
with autocast('cuda'):
    # This matmul runs in FP16
    a = torch.randn(100, 100, device='cuda')
    b = torch.randn(100, 100, device='cuda')
    c = a @ b
    print(c.dtype)  # torch.float16
    
    # This softmax stays in FP32
    d = torch.softmax(c, dim=-1)
    print(d.dtype)  # torch.float32
```

---

## Gradient Accumulation

### What It Is

**Simple Explanation:** Imagine you can only carry 2 grocery bags at a time, but you need to bring 8 bags inside. You make 4 trips, accumulating all bags before you're done. Gradient accumulation works the same way — when your GPU can't fit a large batch, you process smaller mini-batches and accumulate their gradients before updating weights.

### Why It Matters

$$\text{Effective Batch Size} = \text{Mini-Batch Size} \times \text{Accumulation Steps}$$

- Train with batch size 256 even if your GPU only fits batch size 32
- Larger effective batch sizes often improve training stability
- Essential for training large models (LLMs, ViT) on limited hardware

### How It Works

```
┌──────────────────────────────────────────────────────────────────┐
│              GRADIENT ACCUMULATION (accum_steps = 4)              │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Mini-batch 1: forward → backward → gradients += ∇L₁            │
│  Mini-batch 2: forward → backward → gradients += ∇L₂            │
│  Mini-batch 3: forward → backward → gradients += ∇L₃            │
│  Mini-batch 4: forward → backward → gradients += ∇L₄            │
│                                                                   │
│  NOW: optimizer.step()  →  weights -= lr * (∇L₁+∇L₂+∇L₃+∇L₄)/4 │
│       optimizer.zero_grad()                                       │
│                                                                   │
│  Equivalent to batch_size = 4 × mini_batch_size                  │
└──────────────────────────────────────────────────────────────────┘
```

### Implementation

```python
import torch
import torch.nn as nn
import torch.optim as optim

model = nn.Linear(784, 10).cuda()
optimizer = optim.Adam(model.parameters(), lr=1e-3)
criterion = nn.CrossEntropyLoss()

# ============================================================
# GRADIENT ACCUMULATION
# ============================================================

accumulation_steps = 4  # Effective batch = 32 × 4 = 128
mini_batch_size = 32

optimizer.zero_grad()  # Zero once at the start

for step, (images, labels) in enumerate(train_loader):
    images, labels = images.cuda(), labels.cuda()
    
    # Forward pass
    outputs = model(images)
    loss = criterion(outputs, labels)
    
    # IMPORTANT: Normalize loss by accumulation steps
    # This ensures the total gradient magnitude matches
    # what you'd get from a single large batch
    loss = loss / accumulation_steps
    
    # Backward pass — gradients ACCUMULATE (no zero_grad yet!)
    loss.backward()
    
    # Update weights every `accumulation_steps` mini-batches
    if (step + 1) % accumulation_steps == 0:
        optimizer.step()
        optimizer.zero_grad()

# Handle remaining batches at the end of epoch
if (step + 1) % accumulation_steps != 0:
    optimizer.step()
    optimizer.zero_grad()
```

### Gradient Accumulation + Mixed Precision

```python
import torch
from torch.amp import autocast, GradScaler

scaler = GradScaler('cuda')
accumulation_steps = 4

optimizer.zero_grad()

for step, (images, labels) in enumerate(train_loader):
    images, labels = images.cuda(), labels.cuda()
    
    # Forward in mixed precision
    with autocast('cuda'):
        outputs = model(images)
        loss = criterion(outputs, labels) / accumulation_steps
    
    # Backward with scaled loss
    scaler.scale(loss).backward()
    
    if (step + 1) % accumulation_steps == 0:
        # Unscale, clip, step, update — all at once
        scaler.unscale_(optimizer)
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        scaler.step(optimizer)
        scaler.update()
        optimizer.zero_grad()
```

---

## Gradient Clipping

### What It Is

**Simple Explanation:** Gradient clipping is like a speed limiter on a car. When gradients (the "speed" of weight updates) get too large, you cap them to prevent the model from making wild, destructive updates. This is especially critical for RNNs and Transformers.

### Two Types of Gradient Clipping

```python
import torch
import torch.nn as nn

model = nn.Linear(100, 10)
optimizer = torch.optim.SGD(model.parameters(), lr=0.01)

# Dummy forward/backward
x = torch.randn(8, 100)
loss = model(x).sum()
loss.backward()

# ============================================================
# TYPE 1: Clip by NORM (recommended)
# ============================================================
# Scales ALL gradients proportionally if total norm exceeds max_norm
# Preserves gradient direction, only reduces magnitude

total_norm = torch.nn.utils.clip_grad_norm_(
    model.parameters(),
    max_norm=1.0,       # Maximum allowed gradient norm
    norm_type=2         # L2 norm (default)
)
print(f"Gradient norm before clipping: {total_norm:.4f}")

# How it works mathematically:
# if ||g|| > max_norm:
#     g = g × (max_norm / ||g||)

# ============================================================
# TYPE 2: Clip by VALUE (less common)
# ============================================================
# Clips each gradient element independently
# Can change gradient direction!

torch.nn.utils.clip_grad_value_(
    model.parameters(),
    clip_value=0.5  # Each gradient element is clamped to [-0.5, 0.5]
)
```

### When to Use Which

| Method | Use When | Pros | Cons |
|--------|----------|------|------|
| Clip by norm | RNNs, Transformers, most cases | Preserves direction | Doesn't cap individual elements |
| Clip by value | Specific debugging scenarios | Simple, per-element control | Can distort gradient direction |

### Monitoring Gradient Norms

```python
def get_gradient_norm(model):
    """Calculate the total gradient norm across all parameters."""
    total_norm = 0.0
    for p in model.parameters():
        if p.grad is not None:
            total_norm += p.grad.data.norm(2).item() ** 2
    return total_norm ** 0.5

# During training, log gradient norms
for epoch in range(num_epochs):
    for batch in train_loader:
        # ... forward, loss, backward ...
        
        grad_norm = get_gradient_norm(model)
        
        # Clip
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        
        optimizer.step()
        optimizer.zero_grad()
        
        # Log for monitoring
        # writer.add_scalar('grad_norm', grad_norm, global_step)
        
        if grad_norm > 10.0:
            print(f"WARNING: Large gradient norm: {grad_norm:.2f}")
```

---

## Custom Loss Functions

### What It Is

Sometimes the built-in loss functions (CrossEntropy, MSE, etc.) don't capture what you actually want to optimize. Custom losses let you design exactly what "wrong" means for your specific problem.

### Focal Loss (For Class Imbalance)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class FocalLoss(nn.Module):
    """
    Focal Loss (Lin et al., 2017) — designed for severe class imbalance.
    
    Standard CE: L = -log(p_t)
    Focal Loss:  L = -α_t × (1 - p_t)^γ × log(p_t)
    
    The (1 - p_t)^γ term DOWN-WEIGHTS easy examples (high p_t)
    and focuses learning on HARD examples (low p_t).
    
    When γ = 0, this reduces to standard cross-entropy.
    When γ = 2 (recommended), easy examples with p_t > 0.5 
    are down-weighted by 4× or more.
    """
    def __init__(self, alpha=None, gamma=2.0, reduction='mean'):
        super().__init__()
        self.alpha = alpha  # Per-class weights (tensor of shape [C])
        self.gamma = gamma  # Focusing parameter
        self.reduction = reduction
    
    def forward(self, inputs, targets):
        # inputs: (N, C) raw logits
        # targets: (N,) class indices
        
        ce_loss = F.cross_entropy(inputs, targets, reduction='none')
        p_t = torch.exp(-ce_loss)  # Probability of true class
        
        focal_weight = (1 - p_t) ** self.gamma
        loss = focal_weight * ce_loss
        
        if self.alpha is not None:
            alpha_t = self.alpha[targets]
            loss = alpha_t * loss
        
        if self.reduction == 'mean':
            return loss.mean()
        elif self.reduction == 'sum':
            return loss.sum()
        return loss

# Usage
num_classes = 5
# Class weights: higher for rare classes
alpha = torch.tensor([0.1, 0.1, 0.3, 0.7, 1.0]).cuda()
criterion = FocalLoss(alpha=alpha, gamma=2.0)

logits = torch.randn(32, num_classes).cuda()
targets = torch.randint(0, num_classes, (32,)).cuda()
loss = criterion(logits, targets)
print(f"Focal loss: {loss.item():.4f}")
```

### Label Smoothing Loss

```python
class LabelSmoothingLoss(nn.Module):
    """
    Label Smoothing: Instead of hard targets [0, 0, 1, 0],
    use soft targets [0.033, 0.033, 0.9, 0.033].
    
    Prevents overconfidence and improves generalization.
    
    Formula: y_smooth = (1 - ε) × y_hard + ε / K
    where ε is the smoothing factor and K is the number of classes.
    """
    def __init__(self, num_classes, smoothing=0.1):
        super().__init__()
        self.num_classes = num_classes
        self.smoothing = smoothing
        self.confidence = 1.0 - smoothing
    
    def forward(self, logits, targets):
        log_probs = F.log_softmax(logits, dim=-1)
        
        # Create smooth targets
        with torch.no_grad():
            smooth_targets = torch.full_like(log_probs, self.smoothing / self.num_classes)
            smooth_targets.scatter_(1, targets.unsqueeze(1), self.confidence)
        
        # KL divergence
        loss = (-smooth_targets * log_probs).sum(dim=-1)
        return loss.mean()

# Usage
criterion = LabelSmoothingLoss(num_classes=10, smoothing=0.1)

# Pro tip: PyTorch 1.10+ has built-in label smoothing:
criterion_builtin = nn.CrossEntropyLoss(label_smoothing=0.1)
```

### Contrastive Loss (For Representation Learning)

```python
class NTXentLoss(nn.Module):
    """
    Normalized Temperature-scaled Cross-Entropy Loss (SimCLR).
    
    Used in self-supervised learning to pull positive pairs together
    and push negative pairs apart in embedding space.
    """
    def __init__(self, temperature=0.5):
        super().__init__()
        self.temperature = temperature
        self.criterion = nn.CrossEntropyLoss()
    
    def forward(self, z_i, z_j):
        """
        z_i, z_j: embeddings of two augmented views of the same batch
        Shape: (batch_size, embedding_dim)
        """
        batch_size = z_i.size(0)
        
        # Normalize embeddings
        z_i = F.normalize(z_i, dim=1)
        z_j = F.normalize(z_j, dim=1)
        
        # Concatenate: [z_i; z_j] → (2B, D)
        z = torch.cat([z_i, z_j], dim=0)
        
        # Compute similarity matrix: (2B, 2B)
        sim = torch.mm(z, z.t()) / self.temperature
        
        # Mask out self-similarities (diagonal)
        mask = torch.eye(2 * batch_size, device=z.device).bool()
        sim.masked_fill_(mask, -1e9)
        
        # Positive pairs: (i, i+B) and (i+B, i)
        labels = torch.cat([
            torch.arange(batch_size, 2 * batch_size),
            torch.arange(batch_size)
        ]).to(z.device)
        
        return self.criterion(sim, labels)

# Usage
z_i = torch.randn(64, 128)  # Embeddings from augmentation 1
z_j = torch.randn(64, 128)  # Embeddings from augmentation 2
loss = NTXentLoss(temperature=0.5)(z_i, z_j)
```

---

## Custom Learning Rate Schedulers

### What It Is

Learning rate schedulers adjust the learning rate during training. While PyTorch has built-in schedulers, you sometimes need custom schedules for specific training recipes.

### Warmup + Cosine Decay (The Modern Standard)

```python
import math
import torch.optim as optim

class WarmupCosineScheduler:
    """
    Warmup: linearly increase LR from 0 to max_lr
    Cosine decay: smoothly decrease LR to min_lr
    
    This is THE standard schedule for training Transformers.
    
    LR
    ^
    |    /‾‾‾‾\
    |   /       \
    |  /         \
    | /           \_______
    |/                    
    └────────────────────► epoch
     warmup    cosine decay
    """
    def __init__(self, optimizer, warmup_steps, total_steps, min_lr=1e-7):
        self.optimizer = optimizer
        self.warmup_steps = warmup_steps
        self.total_steps = total_steps
        self.min_lr = min_lr
        self.base_lrs = [pg['lr'] for pg in optimizer.param_groups]
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
            scale = 0.5 * (1 + math.cos(math.pi * progress))
        
        for pg, base_lr in zip(self.optimizer.param_groups, self.base_lrs):
            pg['lr'] = max(self.min_lr, base_lr * scale)
    
    def get_lr(self):
        return [pg['lr'] for pg in self.optimizer.param_groups]

# Usage
model_params = [torch.nn.Parameter(torch.randn(10))]
optimizer = optim.Adam(model_params, lr=1e-3)

total_steps = len(train_loader) * num_epochs
warmup_steps = int(0.1 * total_steps)  # 10% warmup

scheduler = WarmupCosineScheduler(optimizer, warmup_steps, total_steps)

for epoch in range(num_epochs):
    for batch in train_loader:
        # ... train ...
        scheduler.step()  # Call per step, not per epoch
```

### Using PyTorch's Built-In LambdaLR

```python
import torch.optim as optim
import math

optimizer = optim.Adam(model.parameters(), lr=1e-3)

# Same warmup + cosine as above, but using LambdaLR
warmup_steps = 500
total_steps = 5000

def lr_lambda(step):
    if step < warmup_steps:
        return step / warmup_steps
    progress = (step - warmup_steps) / (total_steps - warmup_steps)
    return max(0.0, 0.5 * (1 + math.cos(math.pi * progress)))

scheduler = optim.lr_scheduler.LambdaLR(optimizer, lr_lambda)
```

---

## Dynamic Computation Graphs

### What It Is

**Simple Explanation:** PyTorch builds the computation graph on-the-fly as your code runs (like drawing a map while you walk), rather than defining it upfront (like following a pre-drawn map). This means you can use Python `if/else`, `for` loops, and any Python logic in your model.

### Why It Matters

- **Debug easily**: Use `print()`, `pdb`, breakpoints — it's just Python
- **Variable-length inputs**: Naturally handle sequences of different lengths
- **Conditional computation**: Different inputs can take different paths
- **Research flexibility**: Quickly experiment with novel architectures

### Dynamic Branching

```python
import torch
import torch.nn as nn

class DynamicNet(nn.Module):
    """
    A network that uses different computation paths
    based on the input — impossible with static graphs!
    """
    def __init__(self):
        super().__init__()
        self.shared = nn.Linear(784, 256)
        self.branch_easy = nn.Linear(256, 10)
        self.branch_hard = nn.Sequential(
            nn.Linear(256, 128),
            nn.ReLU(),
            nn.Linear(128, 64),
            nn.ReLU(),
            nn.Linear(64, 10)
        )
        self.confidence = nn.Linear(256, 1)
    
    def forward(self, x):
        features = torch.relu(self.shared(x))
        
        # Dynamic decision: use easy or hard branch based on input
        conf = torch.sigmoid(self.confidence(features))
        
        if conf.mean() > 0.5:
            # Easy path — fewer computations
            return self.branch_easy(features)
        else:
            # Hard path — more computations for difficult inputs
            return self.branch_hard(features)

# This works because PyTorch builds the graph dynamically!
model = DynamicNet()
x = torch.randn(8, 784)
output = model(x)  # Graph depends on the actual input values
```

### Variable-Length Sequences with Loops

```python
class RecurrentWithEarlyStop(nn.Module):
    """
    Process a sequence and stop early if a condition is met.
    This is natural in PyTorch but impossible in static graph frameworks.
    """
    def __init__(self, input_size, hidden_size):
        super().__init__()
        self.rnn_cell = nn.GRUCell(input_size, hidden_size)
        self.classifier = nn.Linear(hidden_size, 1)
        self.threshold = 0.95
    
    def forward(self, x):
        # x: (batch, seq_len, features)
        batch_size, seq_len, _ = x.shape
        h = torch.zeros(batch_size, self.rnn_cell.hidden_size, device=x.device)
        
        for t in range(seq_len):
            h = self.rnn_cell(x[:, t, :], h)
            
            # Check confidence — stop early if very confident
            confidence = torch.sigmoid(self.classifier(h))
            if confidence.mean() > self.threshold:
                print(f"Early stop at step {t}/{seq_len}")
                break
        
        return h
```

---

## Memory Optimization

### What It Is

GPU memory is precious and limited. Memory optimization techniques let you train larger models or use larger batch sizes without running out of VRAM.

### Technique 1: Gradient Checkpointing

```python
import torch
import torch.nn as nn
from torch.utils.checkpoint import checkpoint

class MemoryEfficientModel(nn.Module):
    """
    Gradient Checkpointing: Trade compute for memory.
    
    Normal:       Save all intermediate activations → lots of memory
    Checkpointed: Save only some activations, recompute others during backward
    
    Memory savings: ~60-70% reduction
    Speed cost:    ~20-30% slower (due to recomputation)
    """
    def __init__(self):
        super().__init__()
        self.block1 = nn.Sequential(
            nn.Conv2d(3, 64, 3, padding=1), nn.BatchNorm2d(64), nn.ReLU()
        )
        self.block2 = nn.Sequential(
            nn.Conv2d(64, 128, 3, padding=1), nn.BatchNorm2d(128), nn.ReLU()
        )
        self.block3 = nn.Sequential(
            nn.Conv2d(128, 256, 3, padding=1), nn.BatchNorm2d(256), nn.ReLU()
        )
        self.head = nn.Sequential(
            nn.AdaptiveAvgPool2d(1), nn.Flatten(), nn.Linear(256, 10)
        )
    
    def forward(self, x):
        # Checkpoint each block: activations are NOT stored
        # They will be recomputed during backward pass
        x = checkpoint(self.block1, x, use_reentrant=False)
        x = checkpoint(self.block2, x, use_reentrant=False)
        x = checkpoint(self.block3, x, use_reentrant=False)
        x = self.head(x)
        return x

# For pretrained models, checkpoint specific layers:
from torchvision import models
from torchvision.models import ResNet50_Weights

model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)

# Wrap each layer with checkpointing
original_forward = model.layer3.forward
model.layer3.forward = lambda x: checkpoint(original_forward, x, use_reentrant=False)
```

### Technique 2: In-Place Operations

```python
# In-place operations save memory by modifying tensors directly
# instead of creating new ones

# ✅ In-place (saves memory)
x = torch.relu_(x)      # Note the underscore!
x.add_(1.0)
x.mul_(0.5)
nn.ReLU(inplace=True)   # inplace parameter

# ❌ Out-of-place (creates new tensor)
x = torch.relu(x)       # Old x still in memory until GC
x = x + 1.0
x = x * 0.5

# WARNING: In-place ops can break autograd if the tensor is needed for backward
# PyTorch will raise an error if this happens. Safe to use on activations
# that aren't used again.
```

### Technique 3: Emptying Cache and Monitoring

```python
import torch

# Check GPU memory usage
def print_gpu_memory():
    if torch.cuda.is_available():
        allocated = torch.cuda.memory_allocated() / 1024**3
        reserved = torch.cuda.memory_reserved() / 1024**3
        max_allocated = torch.cuda.max_memory_allocated() / 1024**3
        print(f"Allocated: {allocated:.2f} GB")
        print(f"Reserved:  {reserved:.2f} GB")
        print(f"Max Allocated: {max_allocated:.2f} GB")

print_gpu_memory()

# Free unused cached memory
torch.cuda.empty_cache()

# Reset peak memory stats
torch.cuda.reset_peak_memory_stats()

# Context manager for memory tracking
# Useful for profiling which operations use the most memory
```

### Technique 4: Efficient Data Types

```python
# Use the smallest data type that works

# For model weights during inference:
model.half()  # Convert to FP16 — 50% memory savings

# For integer labels:
labels = torch.tensor([1, 2, 3], dtype=torch.long)    # 8 bytes each
labels = torch.tensor([1, 2, 3], dtype=torch.int32)    # 4 bytes each (if < 2B classes)

# For boolean masks:
mask = torch.tensor([True, False, True])  # 1 byte each vs 4 for float
```

### Technique 5: Delete Tensors Early

```python
# Explicitly delete large tensors when done
def train_step(model, images, labels):
    outputs = model(images)
    loss = criterion(outputs, labels)
    loss.backward()
    
    # Delete intermediates that are no longer needed
    del outputs  # Free the output tensor
    # torch.cuda.empty_cache()  # Only if needed; has overhead
    
    return loss.item()  # .item() returns a Python float, not a tensor
```

---

## Debugging PyTorch Models

### Detecting NaN/Inf

```python
import torch

# ============================================================
# METHOD 1: Anomaly detection mode
# ============================================================
torch.autograd.set_detect_anomaly(True)

# This will raise an error with a traceback pointing to the
# exact operation that produced NaN/Inf
# WARNING: Significant performance overhead. Use only for debugging!

model = torch.nn.Linear(10, 5)
x = torch.randn(3, 10)

with torch.autograd.detect_anomaly():
    output = model(x)
    output = output / 0  # Will raise with detailed traceback!
    loss = output.sum()
    loss.backward()

# ============================================================
# METHOD 2: Manual NaN checking
# ============================================================
def check_nan(tensor, name=""):
    """Check if a tensor contains NaN or Inf values."""
    if torch.isnan(tensor).any():
        print(f"WARNING: NaN detected in {name}!")
        return True
    if torch.isinf(tensor).any():
        print(f"WARNING: Inf detected in {name}!")
        return True
    return False

# Use during training
for name, param in model.named_parameters():
    if param.grad is not None:
        check_nan(param.grad, f"gradient of {name}")
    check_nan(param.data, f"weight of {name}")
```

### Shape Debugging

```python
class DebugModel(nn.Module):
    """Model with shape assertions for debugging."""
    
    def __init__(self):
        super().__init__()
        self.conv1 = nn.Conv2d(3, 64, 3, padding=1)
        self.pool = nn.MaxPool2d(2, 2)
        self.fc = nn.Linear(64 * 16 * 16, 10)
    
    def forward(self, x):
        # Print shapes at each step
        print(f"Input: {x.shape}")          # (B, 3, 32, 32)
        
        x = self.pool(torch.relu(self.conv1(x)))
        print(f"After conv1+pool: {x.shape}")  # (B, 64, 16, 16)
        
        x = x.view(x.size(0), -1)
        print(f"After flatten: {x.shape}")     # (B, 64*16*16)
        
        x = self.fc(x)
        print(f"After fc: {x.shape}")          # (B, 10)
        
        return x

# Better approach: use hooks for shape debugging without modifying code
def shape_hook(name):
    def hook(module, input, output):
        in_shape = input[0].shape if isinstance(input, tuple) else input.shape
        out_shape = output.shape if isinstance(output, torch.Tensor) else "N/A"
        print(f"{name}: {in_shape} → {out_shape}")
    return hook

# Register on all layers
for name, module in model.named_modules():
    if len(list(module.children())) == 0:  # Only leaf modules
        module.register_forward_hook(shape_hook(name))
```

### Profiling Performance

```python
import torch
from torch.profiler import profile, record_function, ProfilerActivity

# Profile CPU and GPU operations
with profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    record_shapes=True,
    with_stack=True
) as prof:
    for step, (images, labels) in enumerate(train_loader):
        if step >= 5:
            break
        images, labels = images.cuda(), labels.cuda()
        
        with record_function("forward"):
            outputs = model(images)
        
        with record_function("loss"):
            loss = criterion(outputs, labels)
        
        with record_function("backward"):
            loss.backward()
        
        optimizer.step()
        optimizer.zero_grad()

# Print summary
print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=10))

# Export for Chrome trace viewer
# prof.export_chrome_trace("trace.json")
```

---

## Common Mistakes

### 1. Not Using `use_reentrant=False` in Checkpointing

```python
# ❌ WRONG (deprecated behavior)
x = checkpoint(block, x)  # Default use_reentrant=True has subtle bugs

# ✅ CORRECT
x = checkpoint(block, x, use_reentrant=False)
```

### 2. Forgetting to Scale Loss in Gradient Accumulation

```python
# ❌ WRONG: Gradients are 4x too large
loss = criterion(outputs, labels)
loss.backward()

# ✅ CORRECT: Normalize by accumulation steps
loss = criterion(outputs, labels) / accumulation_steps
loss.backward()
```

### 3. Calling `scaler.step()` Without `scaler.unscale_()` When Clipping

```python
# ❌ WRONG: Clipping scaled gradients (thresholds are wrong)
scaler.scale(loss).backward()
torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)  # Clips scaled gradients!
scaler.step(optimizer)

# ✅ CORRECT: Unscale first
scaler.scale(loss).backward()
scaler.unscale_(optimizer)  # Unscale gradients to their true values
torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
scaler.step(optimizer)
```

### 4. Memory Leaks from Storing Tensors with Gradients

```python
# ❌ WRONG: Storing tensors keeps the entire computation graph alive
all_losses = []
for batch in train_loader:
    loss = criterion(model(batch), labels)
    all_losses.append(loss)  # Entire graph is kept in memory!

# ✅ CORRECT: Detach or use .item()
all_losses = []
for batch in train_loader:
    loss = criterion(model(batch), labels)
    all_losses.append(loss.item())  # Only stores the float value
```

### 5. Custom Layer Not Registering Parameters

```python
# ❌ WRONG: Using raw tensors — optimizer won't see them!
class BadLayer(nn.Module):
    def __init__(self):
        super().__init__()
        self.weight = torch.randn(10, 10)  # NOT a Parameter!
    
    def forward(self, x):
        return x @ self.weight

# ✅ CORRECT: Use nn.Parameter
class GoodLayer(nn.Module):
    def __init__(self):
        super().__init__()
        self.weight = nn.Parameter(torch.randn(10, 10))
    
    def forward(self, x):
        return x @ self.weight
```

### 6. Hooks Not Removed — Causing Memory Leaks

```python
# ❌ WRONG: Hook lives forever, accumulating activations
model.layer1.register_forward_hook(my_hook)
# Training for 1000 epochs... hooks keep running!

# ✅ CORRECT: Store handle and remove when done
handle = model.layer1.register_forward_hook(my_hook)
# ... use the hook ...
handle.remove()  # Clean up!
```

---

## Interview Questions

### Q1: What is mixed precision training and why does it work?

**Answer:** Mixed precision training uses FP16 for most computations (forward/backward pass) and FP32 for critical operations (loss computation, weight updates). It works because:
1. Modern GPUs have Tensor Cores optimized for FP16 → 2x speedup
2. Most weights and activations don't need 32-bit precision
3. Loss scaling prevents gradient underflow in FP16
4. Master weights in FP32 ensure accurate weight updates

Key components: `autocast` (automatic dtype selection), `GradScaler` (loss scaling).

### Q2: Explain gradient accumulation. When and why would you use it?

**Answer:** Gradient accumulation simulates large batch sizes by accumulating gradients over multiple mini-batches before calling `optimizer.step()`. 

Use when:
- GPU memory can't fit the desired batch size
- Larger batch sizes improve training stability (common in NLP)
- Training large models (LLMs, ViT)

Critical detail: Must divide loss by accumulation steps to keep gradient magnitudes correct.

### Q3: What is gradient checkpointing? What's the tradeoff?

**Answer:** Gradient checkpointing saves memory by not storing intermediate activations during the forward pass. Instead, it recomputes them during the backward pass.

- **Memory savings**: ~60-70% reduction
- **Speed cost**: ~20-30% slower due to recomputation
- **When to use**: Training very large models that don't fit in GPU memory

### Q4: What is the difference between `nn.Parameter` and `register_buffer`?

**Answer:**

| Feature | `nn.Parameter` | `register_buffer` |
|---------|----------------|-------------------|
| `requires_grad` | `True` (default) | `False` |
| Updated by optimizer | Yes | No |
| Saved in `state_dict` | Yes | Yes |
| Moves with `.to(device)` | Yes | Yes |
| Example use | Learnable weights | Running statistics (BN mean/var), positional encodings |

### Q5: How do forward hooks differ from backward hooks?

**Answer:**
- **Forward hooks** are called after a module's `forward()` method. They receive `(module, input, output)`. Used for: extracting features, logging activations, modifying outputs.
- **Backward hooks** are called after gradients are computed. They receive `(module, grad_input, grad_output)`. Used for: gradient analysis, gradient modification, detecting vanishing gradients.

Both must be explicitly removed to prevent memory leaks.

### Q6: What is `torch.autograd.detect_anomaly()` and when should you use it?

**Answer:** It's a debugging tool that tracks the operations that produce NaN/Inf values and provides a traceback to the exact line of code. Use it when training produces NaN losses. Don't use in production — it has significant performance overhead.

### Q7: How does Focal Loss address class imbalance?

**Answer:** Focal Loss adds a modulating factor $(1 - p_t)^\gamma$ to cross-entropy loss. When the model is confident about an easy example ($p_t$ is high), the factor becomes small, down-weighting that example. When the model struggles with a hard example ($p_t$ is low), the factor stays near 1. This focuses training on hard, misclassified examples rather than easy, well-classified ones. With $\gamma = 2$, an easy example with $p_t = 0.9$ is weighted 100× less than a hard example with $p_t = 0.1$.

---

## Quick Reference

### Memory Optimization Techniques

| Technique | Memory Savings | Speed Impact | Complexity |
|-----------|---------------|--------------|------------|
| Mixed Precision (FP16) | ~50% | 2x faster | Low |
| Gradient Accumulation | Enables large batches | Neutral | Low |
| Gradient Checkpointing | ~60-70% | 20-30% slower | Low |
| In-place Operations | ~5-10% | Neutral | Low |
| Delete intermediates | Variable | Neutral | Low |
| Model parallelism | Unlimited | Slower (communication) | High |

### Hook Types Cheat Sheet

| Hook | Registration | Callback Args | Use Case |
|------|-------------|---------------|----------|
| Forward | `module.register_forward_hook(fn)` | `(module, input, output)` | Feature extraction |
| Forward Pre | `module.register_forward_pre_hook(fn)` | `(module, input)` | Input modification |
| Backward | `module.register_full_backward_hook(fn)` | `(module, grad_in, grad_out)` | Gradient analysis |

### Custom Module Checklist

```
✅ Inherit from nn.Module
✅ Call super().__init__() in __init__
✅ Use nn.Parameter for learnable weights
✅ Use register_buffer for non-learnable state
✅ Define forward() method
✅ Use nn.ModuleList/nn.ModuleDict for dynamic sub-modules
✅ All sub-modules as attributes (not in plain Python lists)
```

### Mixed Precision Template

```python
from torch.amp import autocast, GradScaler
scaler = GradScaler('cuda')

# Training step:
with autocast('cuda'):
    output = model(input)
    loss = criterion(output, target)
scaler.scale(loss).backward()
scaler.unscale_(optimizer)                                    # Before clipping
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm)  # Optional
scaler.step(optimizer)
scaler.update()
optimizer.zero_grad()
```

### Gradient Accumulation Template

```python
accumulation_steps = N
optimizer.zero_grad()
for i, (x, y) in enumerate(loader):
    loss = criterion(model(x), y) / accumulation_steps
    loss.backward()
    if (i + 1) % accumulation_steps == 0:
        optimizer.step()
        optimizer.zero_grad()
```
