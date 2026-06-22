# Chapter 05: CNN with PyTorch

## Table of Contents
- [1. Convolutional Neural Networks — The Vision Engine](#1-convolutional-neural-networks)
- [2. Core Building Blocks](#2-core-building-blocks)
- [3. Building CNNs in PyTorch](#3-building-cnns-in-pytorch)
- [4. Classic CNN Architectures](#4-classic-cnn-architectures)
- [5. Image Classification Pipeline](#5-image-classification-pipeline)
- [6. Using Pretrained Models (torchvision)](#6-using-pretrained-models)
- [7. Visualizing CNNs — What Does the Network See?](#7-visualizing-cnns)
- [8. Advanced CNN Techniques](#8-advanced-cnn-techniques)
- [9. Common Mistakes](#9-common-mistakes)
- [10. Interview Questions](#10-interview-questions)
- [11. Quick Reference](#11-quick-reference)

---

## 1. Convolutional Neural Networks — The Vision Engine {#1-convolutional-neural-networks}

### What It Is
A CNN is a neural network designed specifically for processing grid-like data (images, audio spectrograms). Instead of looking at every pixel independently, it uses **small sliding filters (kernels)** that scan across the image to detect patterns — edges, textures, shapes, and eventually entire objects.

Think of it like reading with a magnifying glass: you slide it across the page, focusing on small regions at a time, and your brain combines all those local observations into understanding the full page.

### Why It Matters
- CNNs are the **backbone of computer vision**: image classification, object detection, segmentation, face recognition
- They are **translation invariant** — a cat in the top-left looks the same as a cat in the bottom-right
- They use **far fewer parameters** than fully connected networks (weight sharing via kernels)
- Every major vision system (self-driving cars, medical imaging, surveillance) uses CNNs

### How It Works — Intuition

```
Input Image (3×224×224)
    │
    ▼
┌───────────────────────────┐
│  CONV + ReLU              │  Detect edges, colors, textures
│  (many small 3×3 filters) │
├───────────────────────────┤
│  MaxPool (2×2)            │  Shrink spatial dimensions (downsample)
├───────────────────────────┤
│  CONV + ReLU              │  Detect patterns (corners, circles)
├───────────────────────────┤
│  MaxPool (2×2)            │  Shrink again
├───────────────────────────┤
│  CONV + ReLU              │  Detect parts (eyes, wheels, wings)
├───────────────────────────┤
│  MaxPool / AdaptivePool   │  Shrink to fixed size
├───────────────────────────┤
│  Flatten                  │  Convert 2D feature map to 1D vector
├───────────────────────────┤
│  FC (Linear) + ReLU       │  Combine features
├───────────────────────────┤
│  FC (Linear)              │  Output class scores (logits)
└───────────────────────────┘
    │
    ▼
  [cat: 0.9, dog: 0.05, bird: 0.05]
```

**Key insight**: As you go deeper, the network sees **larger regions** of the image (receptive field grows) and detects **more abstract concepts** (edges → textures → parts → objects).

---

## 2. Core Building Blocks {#2-core-building-blocks}

### Conv2d — The Convolution Layer

```
Convolution Operation (simplified):

Input (1 channel):           Kernel (3×3):        Output:
┌───┬───┬───┬───┬───┐       ┌───┬───┬───┐
│ 1 │ 2 │ 3 │ 0 │ 1 │       │ 1 │ 0 │-1 │       ┌───┬───┬───┐
├───┼───┼───┼───┼───┤       ├───┼───┼───┤       │-1 │-3 │ 2 │
│ 0 │ 1 │ 2 │ 3 │ 1 │   *   │ 1 │ 0 │-1 │   =   ├───┼───┼───┤
├───┼───┼───┼───┼───┤       ├───┼───┼───┤       │ 1 │-1 │ 1 │
│ 1 │ 0 │ 1 │ 2 │ 0 │       │ 1 │ 0 │-1 │       └───┴───┴───┘
├───┼───┼───┼───┼───┤       └───┴───┴───┘
│ 2 │ 1 │ 0 │ 1 │ 1 │        (edge detector)
├───┼───┼───┼───┼───┤
│ 1 │ 1 │ 2 │ 0 │ 2 │
└───┴───┴───┴───┴───┘

Kernel slides across input, computing dot product at each position.
```

```python
import torch
import torch.nn as nn

# ═══════════════════════════════════════════
# Conv2d — 2D Convolution
# ═══════════════════════════════════════════
conv = nn.Conv2d(
    in_channels=3,      # Input channels (3 for RGB)
    out_channels=64,     # Number of filters (output channels)
    kernel_size=3,       # 3×3 kernel (most common)
    stride=1,            # Slide 1 pixel at a time
    padding=1,           # Add 1 pixel border (preserves spatial size)
    bias=True            # Add bias term (default True)
)

# Input shape: (batch, channels, height, width) = (N, C, H, W)
x = torch.randn(1, 3, 224, 224)  # 1 image, 3 channels, 224×224
out = conv(x)
print(out.shape)  # torch.Size([1, 64, 224, 224])

# With kernel_size=3, stride=1, padding=1: output size = input size ✓
# This is called "same" padding
```

### Output Size Formula

$$H_{out} = \left\lfloor\frac{H_{in} + 2 \times \text{padding} - \text{kernel\_size}}{\text{stride}}\right\rfloor + 1$$

```python
# Examples:
# Input: 224×224, kernel=3, stride=1, padding=1 → (224+2-3)/1+1 = 224  ← same
# Input: 224×224, kernel=3, stride=2, padding=1 → (224+2-3)/2+1 = 112  ← halved
# Input: 224×224, kernel=7, stride=2, padding=3 → (224+6-7)/2+1 = 112  ← halved
# Input: 224×224, kernel=3, stride=1, padding=0 → (224+0-3)/1+1 = 222  ← shrinks
```

### How Many Parameters Does Conv2d Have?

$$\text{params} = \text{out\_channels} \times (\text{in\_channels} \times k_h \times k_w + 1)$$

```python
conv = nn.Conv2d(3, 64, kernel_size=3, padding=1)
params = sum(p.numel() for p in conv.parameters())
print(f"Parameters: {params}")  # 64 × (3 × 3 × 3 + 1) = 64 × 28 = 1,792

# Compare with a fully connected layer for the same input:
fc = nn.Linear(3 * 224 * 224, 64 * 224 * 224)
# That would be 3×224×224 × 64×224×224 ≈ 461 BILLION parameters!
# Conv2d: 1,792 params. THIS is why CNNs work for images.
```

### MaxPool2d — Downsampling

```
MaxPool2d(2, 2):

Input (4×4):              Output (2×2):
┌───┬───┬───┬───┐        ┌───┬───┐
│ 1 │ 3 │ 2 │ 4 │        │ 3 │ 4 │  ← max of each 2×2 region
├───┼───┼───┼───┤   →    ├───┼───┤
│ 5 │ 2 │ 1 │ 0 │        │ 5 │ 3 │
├───┼───┼───┼───┤        └───┴───┘
│ 0 │ 1 │ 3 │ 2 │
├───┼───┼───┼───┤        Spatial size: 4×4 → 2×2 (halved!)
│ 1 │ 0 │ 2 │ 1 │        No learnable parameters
└───┴───┴───┴───┘
```

```python
# MaxPool2d — takes the maximum value in each pooling window
pool = nn.MaxPool2d(
    kernel_size=2,   # 2×2 pooling window
    stride=2         # Move 2 pixels (non-overlapping)
)

x = torch.randn(1, 64, 224, 224)
out = pool(x)
print(out.shape)  # torch.Size([1, 64, 112, 112])  ← spatial dims halved

# AvgPool2d — takes the average instead
avg_pool = nn.AvgPool2d(kernel_size=2, stride=2)

# AdaptiveAvgPool2d — output ANY fixed size (most flexible)
adaptive_pool = nn.AdaptiveAvgPool2d((1, 1))  # Output: (N, C, 1, 1)
out = adaptive_pool(torch.randn(1, 512, 7, 7))
print(out.shape)  # torch.Size([1, 512, 1, 1])  ← Global Average Pooling!
```

### BatchNorm2d — Stabilize Training

```python
# BatchNorm normalizes activations across the batch dimension
# Dramatically helps CNN training: faster convergence, higher accuracy
bn = nn.BatchNorm2d(
    num_features=64  # Number of channels (must match conv output channels)
)

# Standard pattern: Conv → BatchNorm → ReLU
layer = nn.Sequential(
    nn.Conv2d(3, 64, 3, padding=1),
    nn.BatchNorm2d(64),
    nn.ReLU(inplace=True)  # inplace=True saves memory (modifies tensor in-place)
)
```

> **Pro Tip**: The order `Conv → BN → ReLU` is the standard convention. Some researchers argue for `Conv → ReLU → BN`, but the difference is usually negligible. Be consistent within your model.

### Dropout2d — Regularization for CNNs

```python
# Dropout2d drops entire CHANNELS (not individual pixels)
# This is more effective for CNNs than regular Dropout
dropout = nn.Dropout2d(p=0.2)  # Drop 20% of channels

# Regular Dropout drops individual elements
# Use Dropout for fully connected layers, Dropout2d for conv layers
```

---

## 3. Building CNNs in PyTorch {#3-building-cnns-in-pytorch}

### Your First CNN

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SimpleCNN(nn.Module):
    """Simple CNN for CIFAR-10 (32×32 RGB images, 10 classes)."""
    
    def __init__(self, num_classes=10):
        super().__init__()
        
        # Feature extractor (convolutional layers)
        self.features = nn.Sequential(
            # Block 1: 3 → 32 channels, 32×32 → 16×16
            nn.Conv2d(3, 32, kernel_size=3, padding=1),   # (N, 32, 32, 32)
            nn.BatchNorm2d(32),
            nn.ReLU(inplace=True),
            nn.Conv2d(32, 32, kernel_size=3, padding=1),  # (N, 32, 32, 32)
            nn.BatchNorm2d(32),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2, 2),                            # (N, 32, 16, 16)
            nn.Dropout2d(0.25),
            
            # Block 2: 32 → 64 channels, 16×16 → 8×8
            nn.Conv2d(32, 64, kernel_size=3, padding=1),   # (N, 64, 16, 16)
            nn.BatchNorm2d(64),
            nn.ReLU(inplace=True),
            nn.Conv2d(64, 64, kernel_size=3, padding=1),   # (N, 64, 16, 16)
            nn.BatchNorm2d(64),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2, 2),                            # (N, 64, 8, 8)
            nn.Dropout2d(0.25),
            
            # Block 3: 64 → 128 channels, 8×8 → 4×4
            nn.Conv2d(64, 128, kernel_size=3, padding=1),  # (N, 128, 8, 8)
            nn.BatchNorm2d(128),
            nn.ReLU(inplace=True),
            nn.Conv2d(128, 128, kernel_size=3, padding=1), # (N, 128, 8, 8)
            nn.BatchNorm2d(128),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2, 2),                            # (N, 128, 4, 4)
            nn.Dropout2d(0.25),
        )
        
        # Classifier (fully connected layers)
        self.classifier = nn.Sequential(
            nn.Flatten(),                         # (N, 128*4*4) = (N, 2048)
            nn.Linear(128 * 4 * 4, 512),
            nn.ReLU(inplace=True),
            nn.Dropout(0.5),
            nn.Linear(512, num_classes),           # (N, 10) — raw logits
        )
    
    def forward(self, x):
        x = self.features(x)
        x = self.classifier(x)
        return x

# Create model and test
model = SimpleCNN(num_classes=10)
x = torch.randn(4, 3, 32, 32)  # Batch of 4 CIFAR images
out = model(x)
print(f"Output shape: {out.shape}")  # torch.Size([4, 10])

# Count parameters
total_params = sum(p.numel() for p in model.parameters())
trainable_params = sum(p.numel() for p in model.parameters() if p.requires_grad)
print(f"Total parameters: {total_params:,}")       # ~600K
print(f"Trainable parameters: {trainable_params:,}")
```

### CNN with Functional API (More Flexible)

```python
class FlexibleCNN(nn.Module):
    """CNN using functional API for more control in forward()."""
    
    def __init__(self, num_classes=10):
        super().__init__()
        # Only define layers with learnable parameters
        self.conv1 = nn.Conv2d(3, 32, 3, padding=1)
        self.bn1 = nn.BatchNorm2d(32)
        self.conv2 = nn.Conv2d(32, 64, 3, padding=1)
        self.bn2 = nn.BatchNorm2d(64)
        self.conv3 = nn.Conv2d(64, 128, 3, padding=1)
        self.bn3 = nn.BatchNorm2d(128)
        
        self.pool = nn.AdaptiveAvgPool2d((1, 1))  # Global Average Pooling
        self.fc = nn.Linear(128, num_classes)
        self.dropout = nn.Dropout(0.5)
    
    def forward(self, x):
        # Block 1
        x = F.relu(self.bn1(self.conv1(x)))
        x = F.max_pool2d(x, 2)
        
        # Block 2
        x = F.relu(self.bn2(self.conv2(x)))
        x = F.max_pool2d(x, 2)
        
        # Block 3
        x = F.relu(self.bn3(self.conv3(x)))
        
        # Global Average Pooling (replaces flatten + large FC layer)
        x = self.pool(x)          # (N, 128, 1, 1)
        x = x.view(x.size(0), -1) # (N, 128)
        
        x = self.dropout(x)
        x = self.fc(x)            # (N, num_classes)
        return x
```

> **Pro Tip**: Global Average Pooling (`AdaptiveAvgPool2d((1,1))`) is preferred over flattening large feature maps. It drastically reduces parameters and makes the model accept any input size.

### Building Blocks as Reusable Modules

```python
class ConvBlock(nn.Module):
    """Reusable Conv → BN → ReLU block."""
    
    def __init__(self, in_ch, out_ch, kernel_size=3, stride=1, padding=1):
        super().__init__()
        self.block = nn.Sequential(
            nn.Conv2d(in_ch, out_ch, kernel_size, stride, padding, bias=False),
            nn.BatchNorm2d(out_ch),
            nn.ReLU(inplace=True)
        )
    
    def forward(self, x):
        return self.block(x)

# Note: bias=False when using BatchNorm (BN has its own bias term)

class ResidualBlock(nn.Module):
    """Basic residual block (skip connection)."""
    
    def __init__(self, channels):
        super().__init__()
        self.block = nn.Sequential(
            nn.Conv2d(channels, channels, 3, padding=1, bias=False),
            nn.BatchNorm2d(channels),
            nn.ReLU(inplace=True),
            nn.Conv2d(channels, channels, 3, padding=1, bias=False),
            nn.BatchNorm2d(channels),
        )
    
    def forward(self, x):
        residual = x
        out = self.block(x)
        out += residual  # Skip connection!
        out = F.relu(out)
        return out
```

---

## 4. Classic CNN Architectures {#4-classic-cnn-architectures}

### Architecture Evolution

```
Year   Model          Key Innovation                    Top-5 Error
1998   LeNet-5        First practical CNN               N/A
2012   AlexNet        Deep CNN + ReLU + Dropout         16.4%
2014   VGG            Small 3×3 kernels, very deep      7.3%
2014   GoogLeNet      Inception modules (multi-scale)   6.7%
2015   ResNet         Skip connections (152 layers!)     3.6%
2017   DenseNet       Dense connections                  ~3.5%
2019   EfficientNet   Compound scaling                   ~2.9%
2020   ViT            Vision Transformer (no convs!)     ~2.5%
```

### LeNet-5 (The Pioneer)

```python
class LeNet5(nn.Module):
    """LeNet-5 (1998) — first successful CNN for digit recognition."""
    
    def __init__(self):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(1, 6, 5),        # (1, 28, 28) → (6, 24, 24)
            nn.Tanh(),
            nn.AvgPool2d(2, 2),        # (6, 24, 24) → (6, 12, 12)
            nn.Conv2d(6, 16, 5),       # (6, 12, 12) → (16, 8, 8)
            nn.Tanh(),
            nn.AvgPool2d(2, 2),        # (16, 8, 8) → (16, 4, 4)
        )
        self.classifier = nn.Sequential(
            nn.Flatten(),
            nn.Linear(16 * 4 * 4, 120),
            nn.Tanh(),
            nn.Linear(120, 84),
            nn.Tanh(),
            nn.Linear(84, 10),
        )
    
    def forward(self, x):
        return self.classifier(self.features(x))
```

### VGG-Style (Small Kernels, Deep Networks)

```python
class VGGBlock(nn.Module):
    """VGG building block: multiple 3×3 convs followed by max pool."""
    
    def __init__(self, in_ch, out_ch, num_convs):
        super().__init__()
        layers = []
        for i in range(num_convs):
            layers.append(nn.Conv2d(in_ch if i == 0 else out_ch, out_ch, 3, padding=1))
            layers.append(nn.BatchNorm2d(out_ch))  # Added (not in original VGG)
            layers.append(nn.ReLU(inplace=True))
        layers.append(nn.MaxPool2d(2, 2))
        self.block = nn.Sequential(*layers)
    
    def forward(self, x):
        return self.block(x)

class VGG11(nn.Module):
    """VGG-11 style network."""
    
    def __init__(self, num_classes=10):
        super().__init__()
        self.features = nn.Sequential(
            VGGBlock(3, 64, 1),    # 224→112
            VGGBlock(64, 128, 1),  # 112→56
            VGGBlock(128, 256, 2), # 56→28
            VGGBlock(256, 512, 2), # 28→14
            VGGBlock(512, 512, 2), # 14→7
        )
        self.classifier = nn.Sequential(
            nn.AdaptiveAvgPool2d((1, 1)),
            nn.Flatten(),
            nn.Linear(512, num_classes),
        )
    
    def forward(self, x):
        return self.classifier(self.features(x))
```

### ResNet (Skip Connections — The Game Changer)

```
Why Skip Connections?

Without skip connection:          With skip connection (ResNet):
                                  
Input ──▶ Conv ──▶ Conv ──▶ Out   Input ──▶ Conv ──▶ Conv ──┐
                                    │                        │
                                    └────────── (+) ─────────┘
                                              identity        
                                              shortcut        

Problem without: Deeper networks get WORSE (degradation problem)
Solution: Let the network learn F(x) = H(x) - x (residual)
           Output = F(x) + x
If the layer is useless, F(x) ≈ 0, so output ≈ x (identity)
This makes it easy to train very deep networks (100+ layers)
```

```python
class BasicBlock(nn.Module):
    """ResNet Basic Block (used in ResNet-18, ResNet-34)."""
    expansion = 1
    
    def __init__(self, in_channels, out_channels, stride=1, downsample=None):
        super().__init__()
        self.conv1 = nn.Conv2d(in_channels, out_channels, 3, stride, 1, bias=False)
        self.bn1 = nn.BatchNorm2d(out_channels)
        self.conv2 = nn.Conv2d(out_channels, out_channels, 3, 1, 1, bias=False)
        self.bn2 = nn.BatchNorm2d(out_channels)
        self.downsample = downsample  # Adjusts identity path if dimensions change
    
    def forward(self, x):
        identity = x
        
        out = F.relu(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        
        if self.downsample is not None:
            identity = self.downsample(x)  # Match dimensions for addition
        
        out += identity  # ← THE SKIP CONNECTION
        out = F.relu(out)
        return out

class Bottleneck(nn.Module):
    """ResNet Bottleneck Block (used in ResNet-50, 101, 152).
    
    1×1 conv (reduce channels) → 3×3 conv → 1×1 conv (expand channels)
    More efficient: fewer parameters than two 3×3 convs.
    """
    expansion = 4
    
    def __init__(self, in_channels, mid_channels, stride=1, downsample=None):
        super().__init__()
        out_channels = mid_channels * self.expansion
        
        self.conv1 = nn.Conv2d(in_channels, mid_channels, 1, bias=False)  # 1×1
        self.bn1 = nn.BatchNorm2d(mid_channels)
        self.conv2 = nn.Conv2d(mid_channels, mid_channels, 3, stride, 1, bias=False)  # 3×3
        self.bn2 = nn.BatchNorm2d(mid_channels)
        self.conv3 = nn.Conv2d(mid_channels, out_channels, 1, bias=False)  # 1×1
        self.bn3 = nn.BatchNorm2d(out_channels)
        self.downsample = downsample
    
    def forward(self, x):
        identity = x
        
        out = F.relu(self.bn1(self.conv1(x)))
        out = F.relu(self.bn2(self.conv2(out)))
        out = self.bn3(self.conv3(out))
        
        if self.downsample is not None:
            identity = self.downsample(x)
        
        out += identity
        out = F.relu(out)
        return out
```

### Architecture Comparison

| Architecture | Layers | Params | Key Feature | Best For |
|-------------|--------|--------|-------------|----------|
| LeNet-5 | 5 | 60K | Pioneer CNN | MNIST/small images |
| AlexNet | 8 | 61M | ReLU, Dropout, GPU training | Historical importance |
| VGG-16 | 16 | 138M | All 3×3 convs, very uniform | Feature extraction |
| ResNet-50 | 50 | 25.6M | Skip connections | General classification |
| ResNet-152 | 152 | 60.2M | Very deep residuals | High accuracy tasks |
| EfficientNet-B0 | ~18 | 5.3M | Compound scaling | Mobile/efficient |
| EfficientNet-B7 | ~66 | 66M | Scaled up | Best accuracy/compute |

---

## 5. Image Classification Pipeline {#5-image-classification-pipeline}

### Complete CIFAR-10 Training Example

```python
import torch
import torch.nn as nn
import torch.optim as optim
import torchvision
import torchvision.transforms as transforms
from torch.utils.data import DataLoader
import time

# ══════════════════════════════════════════════════════════════
# CONFIGURATION
# ══════════════════════════════════════════════════════════════
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
print(f"Using device: {device}")

BATCH_SIZE = 128
NUM_EPOCHS = 50
LEARNING_RATE = 0.1
WEIGHT_DECAY = 5e-4

# ══════════════════════════════════════════════════════════════
# DATA
# ══════════════════════════════════════════════════════════════
train_transform = transforms.Compose([
    transforms.RandomCrop(32, padding=4),
    transforms.RandomHorizontalFlip(),
    transforms.ToTensor(),
    transforms.Normalize((0.4914, 0.4822, 0.4465), (0.2470, 0.2435, 0.2616)),
])

test_transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((0.4914, 0.4822, 0.4465), (0.2470, 0.2435, 0.2616)),
])

train_dataset = torchvision.datasets.CIFAR10(
    root='./data', train=True, download=True, transform=train_transform
)
test_dataset = torchvision.datasets.CIFAR10(
    root='./data', train=False, download=True, transform=test_transform
)

train_loader = DataLoader(train_dataset, batch_size=BATCH_SIZE, shuffle=True,
                          num_workers=4, pin_memory=True)
test_loader = DataLoader(test_dataset, batch_size=BATCH_SIZE, shuffle=False,
                         num_workers=4, pin_memory=True)

# ══════════════════════════════════════════════════════════════
# MODEL — ResNet-style for CIFAR
# ══════════════════════════════════════════════════════════════
class CIFARResNet(nn.Module):
    def __init__(self, num_classes=10):
        super().__init__()
        
        # Initial conv (no pooling — 32×32 is already small)
        self.conv1 = nn.Conv2d(3, 64, 3, 1, 1, bias=False)
        self.bn1 = nn.BatchNorm2d(64)
        
        # Residual blocks
        self.layer1 = self._make_layer(64, 64, num_blocks=2, stride=1)   # 32×32
        self.layer2 = self._make_layer(64, 128, num_blocks=2, stride=2)  # 16×16
        self.layer3 = self._make_layer(128, 256, num_blocks=2, stride=2) # 8×8
        self.layer4 = self._make_layer(256, 512, num_blocks=2, stride=2) # 4×4
        
        self.avgpool = nn.AdaptiveAvgPool2d((1, 1))
        self.fc = nn.Linear(512, num_classes)
    
    def _make_layer(self, in_ch, out_ch, num_blocks, stride):
        downsample = None
        if stride != 1 or in_ch != out_ch:
            downsample = nn.Sequential(
                nn.Conv2d(in_ch, out_ch, 1, stride, bias=False),
                nn.BatchNorm2d(out_ch),
            )
        
        layers = [BasicBlock(in_ch, out_ch, stride, downsample)]
        for _ in range(1, num_blocks):
            layers.append(BasicBlock(out_ch, out_ch))
        return nn.Sequential(*layers)
    
    def forward(self, x):
        x = F.relu(self.bn1(self.conv1(x)))
        x = self.layer1(x)
        x = self.layer2(x)
        x = self.layer3(x)
        x = self.layer4(x)
        x = self.avgpool(x)
        x = x.view(x.size(0), -1)
        x = self.fc(x)
        return x

model = CIFARResNet().to(device)

# ══════════════════════════════════════════════════════════════
# TRAINING SETUP
# ══════════════════════════════════════════════════════════════
criterion = nn.CrossEntropyLoss()

# SGD with momentum and weight decay — standard for CNNs
optimizer = optim.SGD(model.parameters(), lr=LEARNING_RATE,
                      momentum=0.9, weight_decay=WEIGHT_DECAY)

# Cosine annealing schedule — smooth decay
scheduler = optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=NUM_EPOCHS)

# ══════════════════════════════════════════════════════════════
# TRAINING LOOP
# ══════════════════════════════════════════════════════════════
best_acc = 0.0

for epoch in range(NUM_EPOCHS):
    # ── Train ──
    model.train()
    train_loss, correct, total = 0, 0, 0
    
    for inputs, targets in train_loader:
        inputs, targets = inputs.to(device), targets.to(device)
        
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        train_loss += loss.item() * inputs.size(0)
        _, predicted = outputs.max(1)
        total += targets.size(0)
        correct += predicted.eq(targets).sum().item()
    
    train_acc = 100. * correct / total
    
    # ── Test ──
    model.eval()
    test_loss, correct, total = 0, 0, 0
    
    with torch.no_grad():
        for inputs, targets in test_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            loss = criterion(outputs, targets)
            
            test_loss += loss.item() * inputs.size(0)
            _, predicted = outputs.max(1)
            total += targets.size(0)
            correct += predicted.eq(targets).sum().item()
    
    test_acc = 100. * correct / total
    scheduler.step()
    
    if test_acc > best_acc:
        best_acc = test_acc
        torch.save(model.state_dict(), 'best_cifar_model.pth')
    
    print(f"Epoch [{epoch+1}/{NUM_EPOCHS}] "
          f"Train Acc: {train_acc:.1f}% | Test Acc: {test_acc:.1f}% | "
          f"Best: {best_acc:.1f}%")

# Expected: ~93% test accuracy with this setup
```

---

## 6. Using Pretrained Models (torchvision) {#6-using-pretrained-models}

### What It Is
Pretrained models are CNNs already trained on ImageNet (1.2M images, 1000 classes). You can use them directly for inference or fine-tune them for your own task. This is called **transfer learning** and is the standard approach in practice.

### Loading Pretrained Models

```python
import torchvision.models as models

# ═══════════════════════════════════════════
# Modern API (PyTorch ≥ 1.13) — use weights enum
# ═══════════════════════════════════════════
from torchvision.models import (
    resnet18, ResNet18_Weights,
    resnet50, ResNet50_Weights,
    efficientnet_b0, EfficientNet_B0_Weights,
    vit_b_16, ViT_B_16_Weights,
)

# ResNet-18 with ImageNet weights
model = resnet18(weights=ResNet18_Weights.IMAGENET1K_V1)

# ResNet-50 (best available weights)
model = resnet50(weights=ResNet50_Weights.DEFAULT)

# EfficientNet-B0
model = efficientnet_b0(weights=EfficientNet_B0_Weights.DEFAULT)

# Without pretrained weights (random initialization)
model = resnet18(weights=None)

# ═══════════════════════════════════════════
# Get transforms used during pretraining
# ═══════════════════════════════════════════
weights = ResNet50_Weights.DEFAULT
preprocess = weights.transforms()
# This automatically gives you the correct resize, crop, normalize!

# ═══════════════════════════════════════════
# Legacy API (still works)
# ═══════════════════════════════════════════
model = models.resnet18(pretrained=True)  # Deprecated but works
```

### Modifying for Custom Number of Classes

```python
import torch.nn as nn
import torchvision.models as models
from torchvision.models import ResNet50_Weights

# Load pretrained ResNet-50
model = models.resnet50(weights=ResNet50_Weights.DEFAULT)

# Check the original classifier
print(model.fc)  # Linear(in_features=2048, out_features=1000)

# Replace the last layer for YOUR number of classes
num_classes = 5  # e.g., 5 types of flowers
model.fc = nn.Linear(model.fc.in_features, num_classes)

# Now the model outputs 5 logits instead of 1000

# ═══════════════════════════════════════════
# For EfficientNet (different attribute name)
# ═══════════════════════════════════════════
model = models.efficientnet_b0(weights=EfficientNet_B0_Weights.DEFAULT)
print(model.classifier)  # Sequential(Dropout, Linear(1280, 1000))
model.classifier[1] = nn.Linear(1280, num_classes)

# ═══════════════════════════════════════════
# For VGG (another attribute name)
# ═══════════════════════════════════════════
model = models.vgg16(weights='DEFAULT')
model.classifier[6] = nn.Linear(4096, num_classes)
```

### Feature Extraction vs Fine-Tuning

```python
# ═══════════════════════════════════════════
# FEATURE EXTRACTION: Freeze all layers, train only the new classifier
# Use when: small dataset, similar to ImageNet
# ═══════════════════════════════════════════
model = models.resnet50(weights=ResNet50_Weights.DEFAULT)

# Freeze ALL parameters
for param in model.parameters():
    param.requires_grad = False

# Replace and unfreeze the classifier
model.fc = nn.Linear(2048, num_classes)
# model.fc.parameters() have requires_grad=True by default (newly created)

# Only optimize the classifier parameters
optimizer = optim.Adam(model.fc.parameters(), lr=1e-3)

# ═══════════════════════════════════════════
# FINE-TUNING: Train all layers with small learning rate
# Use when: larger dataset, somewhat different from ImageNet
# ═══════════════════════════════════════════
model = models.resnet50(weights=ResNet50_Weights.DEFAULT)
model.fc = nn.Linear(2048, num_classes)

# Different learning rates for pretrained vs new layers
optimizer = optim.AdamW([
    {'params': model.conv1.parameters(), 'lr': 1e-5},
    {'params': model.layer1.parameters(), 'lr': 1e-5},
    {'params': model.layer2.parameters(), 'lr': 1e-5},
    {'params': model.layer3.parameters(), 'lr': 1e-4},
    {'params': model.layer4.parameters(), 'lr': 1e-4},
    {'params': model.fc.parameters(), 'lr': 1e-3},
], weight_decay=0.01)

# ═══════════════════════════════════════════
# GRADUAL UNFREEZING: Start frozen, unfreeze progressively
# Best of both worlds
# ═══════════════════════════════════════════
model = models.resnet50(weights=ResNet50_Weights.DEFAULT)
model.fc = nn.Linear(2048, num_classes)

# Phase 1: Train only classifier (5 epochs)
for param in model.parameters():
    param.requires_grad = False
model.fc.weight.requires_grad = True
model.fc.bias.requires_grad = True

# Phase 2: Unfreeze layer4 (5 more epochs)
for param in model.layer4.parameters():
    param.requires_grad = True

# Phase 3: Unfreeze everything (remaining epochs)
for param in model.parameters():
    param.requires_grad = True
```

### Available Pretrained Models

| Model | Params | Top-1 Acc | Speed | Use Case |
|-------|--------|-----------|-------|----------|
| resnet18 | 11.7M | 69.8% | Very Fast | Quick experiments |
| resnet50 | 25.6M | 80.9% | Fast | Standard baseline |
| efficientnet_b0 | 5.3M | 77.7% | Fast | Mobile/edge deployment |
| efficientnet_b4 | 19.3M | 83.4% | Medium | Good accuracy/speed |
| convnext_base | 88.6M | 84.1% | Medium | Modern CNN |
| vit_b_16 | 86.6M | 81.1% | Medium | Transformer-based |
| swin_t | 28.3M | 81.5% | Medium | Hierarchical transformer |

---

## 7. Visualizing CNNs — What Does the Network See? {#7-visualizing-cnns}

### Visualizing Learned Filters

```python
import matplotlib.pyplot as plt

def visualize_filters(model, layer_name='conv1'):
    """Visualize the first convolutional layer's filters."""
    # Get the first conv layer weights
    if hasattr(model, layer_name):
        weights = getattr(model, layer_name).weight.data.cpu()
    else:
        # For Sequential models
        weights = model.features[0].weight.data.cpu()
    
    # Normalize for display
    weights = (weights - weights.min()) / (weights.max() - weights.min())
    
    n_filters = min(weights.shape[0], 64)
    fig, axes = plt.subplots(8, 8, figsize=(10, 10))
    
    for i, ax in enumerate(axes.flat):
        if i < n_filters:
            if weights.shape[1] == 3:  # RGB filters
                ax.imshow(weights[i].permute(1, 2, 0))
            else:  # Grayscale
                ax.imshow(weights[i, 0], cmap='gray')
        ax.axis('off')
    
    plt.suptitle(f'Learned Filters: {layer_name}')
    plt.tight_layout()
    plt.show()
```

### Feature Map Visualization

```python
class FeatureMapExtractor:
    """Extract intermediate feature maps using hooks."""
    
    def __init__(self, model, target_layers):
        self.features = {}
        self.hooks = []
        
        for name, module in model.named_modules():
            if name in target_layers:
                hook = module.register_forward_hook(
                    self._make_hook(name)
                )
                self.hooks.append(hook)
    
    def _make_hook(self, name):
        def hook_fn(module, input, output):
            self.features[name] = output.detach().cpu()
        return hook_fn
    
    def remove_hooks(self):
        for hook in self.hooks:
            hook.remove()

# Usage
model = models.resnet18(weights=ResNet18_Weights.DEFAULT)
model.eval()

extractor = FeatureMapExtractor(model, ['layer1', 'layer2', 'layer3', 'layer4'])

# Forward pass
x = torch.randn(1, 3, 224, 224)
_ = model(x)

# Visualize feature maps
for name, feat in extractor.features.items():
    print(f"{name}: shape = {feat.shape}")
    # layer1: shape = torch.Size([1, 64, 56, 56])
    # layer2: shape = torch.Size([1, 128, 28, 28])
    # layer3: shape = torch.Size([1, 256, 14, 14])
    # layer4: shape = torch.Size([1, 512, 7, 7])

extractor.remove_hooks()
```

---

## 8. Advanced CNN Techniques {#8-advanced-cnn-techniques}

### Depthwise Separable Convolution (MobileNet)

```python
class DepthwiseSeparableConv(nn.Module):
    """
    Split standard convolution into:
    1. Depthwise: one filter per input channel (spatial filtering)
    2. Pointwise: 1×1 conv to mix channels
    
    Reduces computation by ~8-9x for 3×3 kernels!
    Standard 3×3 conv: C_in × C_out × 3 × 3 multiplications
    Depthwise separable: C_in × 3 × 3 + C_in × C_out multiplications
    """
    
    def __init__(self, in_ch, out_ch, stride=1):
        super().__init__()
        self.depthwise = nn.Conv2d(in_ch, in_ch, 3, stride, 1,
                                    groups=in_ch, bias=False)  # groups=in_ch!
        self.pointwise = nn.Conv2d(in_ch, out_ch, 1, bias=False)  # 1×1 conv
        self.bn1 = nn.BatchNorm2d(in_ch)
        self.bn2 = nn.BatchNorm2d(out_ch)
    
    def forward(self, x):
        x = F.relu(self.bn1(self.depthwise(x)))
        x = F.relu(self.bn2(self.pointwise(x)))
        return x
```

### Squeeze-and-Excitation Block (Channel Attention)

```python
class SEBlock(nn.Module):
    """
    Squeeze-and-Excitation: learn to weight channels by importance.
    
    1. Squeeze: Global average pool → one number per channel
    2. Excitation: FC → ReLU → FC → Sigmoid → channel weights
    3. Scale: Multiply each channel by its learned weight
    """
    
    def __init__(self, channels, reduction=16):
        super().__init__()
        self.squeeze = nn.AdaptiveAvgPool2d(1)
        self.excitation = nn.Sequential(
            nn.Linear(channels, channels // reduction, bias=False),
            nn.ReLU(inplace=True),
            nn.Linear(channels // reduction, channels, bias=False),
            nn.Sigmoid()
        )
    
    def forward(self, x):
        b, c, _, _ = x.size()
        # Squeeze
        y = self.squeeze(x).view(b, c)
        # Excitation
        y = self.excitation(y).view(b, c, 1, 1)
        # Scale
        return x * y.expand_as(x)
```

### 1×1 Convolutions — Channel Mixing

```python
# 1×1 convolution: doesn't look at spatial neighbors
# Purpose: change number of channels (dimensionality reduction)
# Used extensively in Inception, ResNet bottleneck, modern architectures

# Reduce channels: 256 → 64 (cheaper than 3×3 conv on 256 channels)
reduce = nn.Conv2d(256, 64, kernel_size=1)  # Only 256×64 = 16,384 params

# Increase channels
expand = nn.Conv2d(64, 256, kernel_size=1)
```

---

## 9. Common Mistakes {#9-common-mistakes}

| Mistake | Problem | Fix |
|---------|---------|-----|
| Wrong input shape order | PyTorch uses (N, C, H, W), not (N, H, W, C) | Transpose with `.permute(0, 3, 1, 2)` |
| Not using `bias=False` with BatchNorm | Redundant parameters (BN has its own bias) | Set `bias=False` in Conv2d when followed by BN |
| Using huge FC after flatten | Millions of unnecessary parameters | Use AdaptiveAvgPool2d((1,1)) before FC |
| Not using pretrained models | Training from scratch wastes time | Always start with pretrained weights for real projects |
| Same augmentation for train and val | Noisy evaluation metrics | Separate transform pipelines |
| Forgetting `model.eval()` for inference | Dropout + BN behave wrong | Always call `model.eval()` |
| Using large kernels (5×5, 7×7) everywhere | Too many parameters, slow | Use 3×3 kernels (stack two 3×3 ≈ one 5×5 receptive field) |
| Not using skip connections for deep CNNs | Gradient vanishing, degradation | Use ResNet-style skip connections for >20 layers |
| Ignoring input size through the network | Shape mismatch at flatten/FC | Track output size manually or print shapes |

---

## 10. Interview Questions {#10-interview-questions}

**Q1: Why are 3×3 kernels preferred over larger kernels?**
> Two stacked 3×3 convolutions have the same receptive field as one 5×5 but with fewer parameters (2×9=18 vs 25) and more non-linearity (two ReLUs). Three 3×3s match a 7×7 (27 vs 49 params). This was the key insight from VGGNet.

**Q2: What is the receptive field and why does it matter?**
> The receptive field is the region of the input image that influences a particular output neuron. Deeper layers have larger receptive fields. It matters because the network needs a large enough receptive field to "see" the entire object it's classifying. Pooling, strided convolutions, and dilated convolutions all increase the receptive field.

**Q3: Explain the difference between stride and pooling for downsampling.**
> Both reduce spatial dimensions. Strided convolution learns the downsampling (learnable parameters), while max pooling has no learnable parameters (takes the max). Modern architectures often prefer strided convolutions (more flexible) or a mix. Max pooling provides translation invariance but discards spatial information.

**Q4: What problem do skip connections (residual connections) solve?**
> The degradation problem: very deep networks perform worse than shallower ones (not overfitting, but optimization difficulty). Skip connections allow gradients to flow directly through the identity path, making it easier to train very deep networks. The network learns residuals F(x) = H(x) - x, which is easier to optimize than H(x) directly.

**Q5: How does Global Average Pooling (GAP) compare to fully connected layers?**
> GAP converts each feature map to a single number (its average), creating a vector of length = number of channels. Benefits: (1) Massive parameter reduction, (2) No overfitting from large FC layers, (3) Input size agnostic, (4) Acts as structural regularizer. Used in most modern architectures.

**Q6: When would you use depthwise separable convolutions?**
> When you need a lightweight model for mobile or edge deployment. They reduce computation by ~8-9x compared to standard convolutions with minimal accuracy loss. Used in MobileNet, EfficientNet, and Xception. Trade-off: slightly lower representational capacity.

**Q7: What is batch normalization and why is it critical for CNNs?**
> BN normalizes activations within each mini-batch to have zero mean and unit variance, then applies learned scale and shift. Benefits: (1) Allows higher learning rates, (2) Reduces sensitivity to initialization, (3) Acts as regularizer, (4) Smoother loss landscape. Applied after convolution, before activation in the standard pattern.

---

## 11. Quick Reference {#11-quick-reference}

### Layer Output Size Calculator

```python
def conv_output_size(h_in, kernel, stride=1, padding=0, dilation=1):
    """Calculate output height/width after convolution."""
    return (h_in + 2*padding - dilation*(kernel-1) - 1) // stride + 1

# Examples
print(conv_output_size(224, 3, 1, 1))  # 224 (same padding)
print(conv_output_size(224, 7, 2, 3))  # 112 (ResNet first conv)
print(conv_output_size(32, 3, 1, 1))   # 32  (CIFAR same padding)
```

### CNN Architecture Template

```python
class MyCNN(nn.Module):
    def __init__(self, num_classes):
        super().__init__()
        self.features = nn.Sequential(
            # Block 1: input → 64 channels
            nn.Conv2d(3, 64, 3, padding=1, bias=False),
            nn.BatchNorm2d(64), nn.ReLU(inplace=True),
            nn.MaxPool2d(2),
            # Block 2: 64 → 128 channels
            nn.Conv2d(64, 128, 3, padding=1, bias=False),
            nn.BatchNorm2d(128), nn.ReLU(inplace=True),
            nn.MaxPool2d(2),
            # Block 3: 128 → 256 channels
            nn.Conv2d(128, 256, 3, padding=1, bias=False),
            nn.BatchNorm2d(256), nn.ReLU(inplace=True),
            nn.AdaptiveAvgPool2d((1, 1)),  # Global Average Pooling
        )
        self.classifier = nn.Sequential(
            nn.Flatten(),
            nn.Dropout(0.5),
            nn.Linear(256, num_classes),
        )
    
    def forward(self, x):
        return self.classifier(self.features(x))
```

### Pretrained Model Quick Start

```python
import torchvision.models as models

# Load pretrained + modify classifier
model = models.resnet50(weights='DEFAULT')
model.fc = nn.Linear(2048, YOUR_NUM_CLASSES)

# Get correct preprocessing
weights = models.ResNet50_Weights.DEFAULT
preprocess = weights.transforms()

# Inference
model.eval()
with torch.no_grad():
    output = model(preprocess(image).unsqueeze(0))
    predicted_class = output.argmax(1).item()
```

### Key Imports for CNN Work

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import torch.optim as optim
import torchvision
import torchvision.transforms as transforms
import torchvision.models as models
from torch.utils.data import DataLoader
```

---

*Previous: [04-Data-Loading-and-Transforms](04-Data-Loading-and-Transforms.md) | Next: [06-RNN-and-NLP-with-PyTorch](06-RNN-and-NLP-with-PyTorch.md)*
