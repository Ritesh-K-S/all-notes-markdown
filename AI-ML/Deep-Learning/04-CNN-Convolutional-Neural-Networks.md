# Chapter 04: Convolutional Neural Networks (CNNs)

## Table of Contents
- [What is a CNN?](#what-is-a-cnn)
- [Why CNNs Matter](#why-cnns-matter)
- [Core Building Blocks](#core-building-blocks)
- [Convolution Operation](#convolution-operation)
- [Pooling Layers](#pooling-layers)
- [CNN Architecture Design](#cnn-architecture-design)
- [Landmark Architectures](#landmark-architectures)
- [Code Examples](#code-examples)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is a CNN?

A Convolutional Neural Network is a type of neural network designed specifically to process **grid-like data** — images, audio spectrograms, or even time-series data arranged in a grid.

### Simple Analogy (For a 15-year-old)

Imagine you're looking for your friend in a crowd photo. You don't scan every pixel — you look for **patterns**: hair color, face shape, clothing. A CNN does exactly this:
1. First, it detects **edges** (outlines)
2. Then, it combines edges into **textures** (hair, skin)
3. Then textures into **parts** (eyes, nose)
4. Finally, parts into **objects** (face → your friend)

Each layer looks for increasingly complex patterns, just like your brain!

---

## Why CNNs Matter

### Real-World Applications
| Domain | Application | Example |
|--------|------------|---------|
| Healthcare | Medical Imaging | Detecting tumors in X-rays |
| Autonomous Vehicles | Object Detection | Recognizing pedestrians, traffic signs |
| Agriculture | Crop Monitoring | Identifying diseased plants from drone images |
| Security | Face Recognition | Unlocking your phone |
| Manufacturing | Quality Control | Detecting defective products on assembly lines |
| Social Media | Content Moderation | Filtering inappropriate images |

### Why Not Just Use Fully Connected Networks?

A 224×224 RGB image has **150,528** input values. A fully connected layer with 1000 neurons would need **150 million parameters** — just for the first layer! This is:
- Computationally expensive
- Prone to overfitting
- Ignores spatial structure

CNNs solve all three problems through **parameter sharing** and **local connectivity**.

---

## Core Building Blocks

### The Three Key Ideas

```
┌─────────────────────────────────────────────────────────┐
│                CNN Design Principles                      │
├─────────────────────────────────────────────────────────┤
│ 1. LOCAL CONNECTIVITY    → Each neuron sees only a       │
│                            small region (receptive field)│
│                                                         │
│ 2. PARAMETER SHARING     → Same filter scans the entire │
│                            image (weight sharing)        │
│                                                         │
│ 3. TRANSLATION INVARIANCE→ Detect pattern regardless    │
│                            of position in image          │
└─────────────────────────────────────────────────────────┘
```

---

## Convolution Operation

### What is Convolution?

Convolution is a mathematical operation where a small matrix (called a **kernel** or **filter**) slides over the input and computes element-wise multiplication + sum at each position.

### Visual Explanation

```
Input Image (5×5)          Kernel (3×3)         Output (3×3)
┌───┬───┬───┬───┬───┐    ┌───┬───┬───┐       ┌───┬───┬───┐
│ 1 │ 0 │ 1 │ 0 │ 1 │    │ 1 │ 0 │ 1 │       │ 4 │ 3 │ 4 │
├───┼───┼───┼───┼───┤    ├───┼───┼───┤       ├───┼───┼───┤
│ 0 │ 1 │ 0 │ 1 │ 0 │    │ 0 │ 1 │ 0 │       │ 2 │ 4 │ 3 │
├───┼───┼───┼───┼───┤    ├───┼───┼───┤       ├───┼───┼───┤
│ 1 │ 0 │ 1 │ 0 │ 1 │    │ 1 │ 0 │ 1 │       │ 4 │ 3 │ 4 │
├───┼───┼───┼───┼───┤    └───┴───┴───┘       └───┴───┴───┘
│ 0 │ 1 │ 0 │ 1 │ 0 │
├───┼───┼───┼───┼───┤     Computation at (0,0):
│ 1 │ 0 │ 1 │ 0 │ 1 │     (1×1)+(0×0)+(1×1)+
└───┴───┴───┴───┴───┘     (0×0)+(1×1)+(0×0)+
                           (1×1)+(0×0)+(1×1) = 4
```

### Mathematical Formula

For a 2D input $I$ and kernel $K$, the convolution at position $(i, j)$ is:

$$S(i, j) = (I * K)(i, j) = \sum_m \sum_n I(i+m, j+n) \cdot K(m, n)$$

For a multi-channel input (e.g., RGB with $C$ channels):

$$S(i, j) = \sum_{c=1}^{C} \sum_m \sum_n I(c, i+m, j+n) \cdot K(c, m, n) + b$$

Where $b$ is the bias term.

### Key Parameters

| Parameter | What It Controls | Effect of Increasing |
|-----------|-----------------|---------------------|
| **Kernel Size** | Size of the filter (e.g., 3×3, 5×5) | Larger receptive field, more params |
| **Stride** | How many pixels the filter moves | Smaller output, downsampling |
| **Padding** | Zero-padding around input | Preserves spatial dimensions |
| **Dilation** | Spacing between kernel elements | Larger receptive field without more params |

### Output Size Formula

$$O = \left\lfloor \frac{W - K + 2P}{S} \right\rfloor + 1$$

Where:
- $W$ = Input size (width or height)
- $K$ = Kernel size
- $P$ = Padding
- $S$ = Stride

> **Critical Formula** — You WILL be asked this in interviews. Memorize it!

### Padding Types

```
"valid" (no padding):     "same" (zero padding):
Input: 5×5               Input: 5×5 → Pad to 7×7
Kernel: 3×3              Kernel: 3×3
Output: 3×3              Output: 5×5 (same as input!)

┌─────────┐              ┌─0─0─0─0─0─0─0─┐
│ x x x x x│              │0│x x x x x│0│
│ x x x x x│              │0│x x x x x│0│
│ x x x x x│              │0│x x x x x│0│
│ x x x x x│              │0│x x x x x│0│
│ x x x x x│              │0│x x x x x│0│
└─────────┘              └─0─0─0─0─0─0─0─┘
```

### Stride Visualization

```
Stride = 1:                 Stride = 2:
Filter moves 1 pixel       Filter moves 2 pixels
at a time → fine detail     at a time → downsamples

[■ ■ ■] → [■ ■ ■]         [■ ■ ■] ─ ─ → [■ ■ ■]
 . . . . .   . . . . .      . . . . .       . . . . .
```

### Number of Parameters in a Conv Layer

$$\text{Params} = (K_h \times K_w \times C_{in} + 1) \times C_{out}$$

Where:
- $K_h, K_w$ = kernel height and width
- $C_{in}$ = input channels
- $C_{out}$ = output channels (number of filters)
- $+1$ = bias per filter

**Example**: Conv layer with 64 filters of size 3×3 on RGB input:
$(3 \times 3 \times 3 + 1) \times 64 = 1,792$ parameters

### What Do Filters Learn?

```
Layer 1 (Low-level):    Layer 3 (Mid-level):    Layer 5+ (High-level):
┌───┐  ┌───┐           ┌─────────┐             ┌───────────┐
│///│  │───│           │  eyes   │             │   face    │
│///│  │───│           │  ears   │             │   car     │
│///│  │───│           │  wheels │             │   dog     │
└───┘  └───┘           └─────────┘             └───────────┘
Edges, Gradients        Textures, Parts         Objects, Scenes
```

### 1×1 Convolutions (Network-in-Network)

A 1×1 convolution doesn't look at spatial patterns — it operates across **channels**:

```
Input: H×W×256 channels
1×1 Conv with 64 filters
Output: H×W×64 channels

Purpose: Channel-wise dimensionality reduction
         (Like PCA across channels)
```

Used extensively in Inception and ResNet architectures.

---

## Pooling Layers

### What is Pooling?

Pooling **downsamples** feature maps, reducing spatial dimensions while retaining important information.

### Types of Pooling

```
Input (4×4):                Max Pooling (2×2, stride 2):
┌───┬───┬───┬───┐         ┌───┬───┐
│ 1 │ 3 │ 2 │ 1 │         │ 4 │ 6 │   ← Takes maximum
├───┼───┼───┼───┤         ├───┼───┤     from each 2×2 region
│ 4 │ 2 │ 6 │ 1 │         │ 8 │ 4 │
├───┼───┼───┼───┤         └───┴───┘
│ 1 │ 8 │ 3 │ 0 │
├───┼───┼───┼───┤         Average Pooling (2×2, stride 2):
│ 2 │ 1 │ 4 │ 2 │         ┌─────┬─────┐
└───┴───┴───┴───┘         │ 2.5 │ 2.5 │  ← Takes average
                           ├─────┼─────┤    from each 2×2 region
                           │ 3.0 │ 2.25│
                           └─────┴─────┘
```

### Global Average Pooling (GAP)

Replaces fully connected layers at the end of CNNs:

```
Input: 7×7×512 → GAP → 1×1×512 (one value per channel)

Advantages:
- No learnable parameters (reduces overfitting)
- Enforces correspondence between feature maps and categories
- More robust to spatial translations
```

### Max Pooling vs Average Pooling

| Aspect | Max Pooling | Average Pooling |
|--------|------------|-----------------|
| What it captures | Strongest activation (presence of feature) | Overall activation (how much of feature) |
| Use case | Feature detection | Preserving background info |
| Gradient flow | Only through max element | Through all elements |
| Modern preference | ✅ Most common | Used in final layers (GAP) |

---

## CNN Architecture Design

### Typical CNN Structure

```
Input → [CONV → ReLU → CONV → ReLU → POOL] × N → FLATTEN → FC → FC → Softmax
         └────────── Feature Extraction ──────────┘  └── Classification ──┘
```

### Design Rules of Thumb

1. **Start with small kernels** (3×3) — they're more efficient than large ones
2. **Double channels when halving spatial dims**: 64→128→256→512
3. **Use BatchNorm after every Conv layer** (before or after activation)
4. **Prefer stride-2 conv over pooling** in modern architectures
5. **Use Global Average Pooling** instead of flattening + FC

### Receptive Field

The **receptive field** is the region of the input that influences a particular neuron's output.

```
Layer 1 (3×3 conv): Each output pixel sees 3×3 of input
Layer 2 (3×3 conv): Each output pixel sees 5×5 of input
Layer 3 (3×3 conv): Each output pixel sees 7×7 of input

Two 3×3 convs = one 5×5 conv (same receptive field)
Three 3×3 convs = one 7×7 conv (same receptive field)

But 3×3 × 2 has FEWER parameters:
  2 × (3×3×C×C) = 18C² vs 1 × (5×5×C×C) = 25C²
```

> **Key Insight**: Stacking small filters is better than using one large filter — same receptive field, fewer parameters, more non-linearity.

---

## Landmark Architectures

### Architecture Evolution Timeline

```
1998    2012      2014       2014      2015      2016      2017
 │       │         │          │         │         │         │
LeNet  AlexNet  VGGNet    GoogLeNet  ResNet   DenseNet  SENet
 │       │         │          │         │         │         │
 ▼       ▼         ▼          ▼         ▼         ▼         ▼
Basic  GPU+ReLU  Deeper    Inception  Skip      Dense    Channel
CNN    Dropout   3×3 only  Module     Connects  Connects  Attention
```

### LeNet-5 (1998) — Yann LeCun

The grandfather of all CNNs. Designed for handwritten digit recognition.

```
Architecture:
Input(32×32×1) → Conv(5×5,6) → Pool → Conv(5×5,16) → Pool → FC(120) → FC(84) → Output(10)

Key Innovations:
- First successful CNN application
- Used tanh activation (ReLU wasn't popular yet)
- Proved convolutions work for pattern recognition
```

### AlexNet (2012) — Krizhevsky et al.

Won ImageNet 2012 by a massive margin. Started the deep learning revolution.

```
Architecture:
Input(227×227×3) → Conv(11×11,96,s=4) → MaxPool → Conv(5×5,256) → MaxPool
→ Conv(3×3,384) → Conv(3×3,384) → Conv(3×3,256) → MaxPool → FC(4096) → FC(4096) → FC(1000)

Key Innovations:
- ReLU activation (faster training than tanh/sigmoid)
- Dropout (0.5) for regularization
- Data augmentation (flips, crops, color jittering)
- GPU training (split across 2 GPUs)
- Local Response Normalization (replaced by BatchNorm later)
```

### VGGNet (2014) — Simonyan & Zisserman

Philosophy: **Go deeper with only 3×3 convolutions**.

```
VGG-16 Architecture:
Input(224×224×3)
├─ Block 1: 2× Conv(3×3, 64)  → MaxPool → 112×112×64
├─ Block 2: 2× Conv(3×3, 128) → MaxPool → 56×56×128
├─ Block 3: 3× Conv(3×3, 256) → MaxPool → 28×28×256
├─ Block 4: 3× Conv(3×3, 512) → MaxPool → 14×14×512
├─ Block 5: 3× Conv(3×3, 512) → MaxPool → 7×7×512
├─ FC(4096) → FC(4096) → FC(1000)
└─ Total Parameters: ~138 Million

Key Insight:
- 3×3 is the smallest filter that captures left/right, up/down, center
- Stack of 3×3 = large receptive field + fewer params + more non-linearity
```

### GoogLeNet / Inception (2014) — Szegedy et al.

Philosophy: **Why choose one filter size when you can use them all?**

```
Inception Module:
                    ┌─────────────────┐
                    │   Previous Layer  │
                    └─────────┬─────────┘
           ┌──────────┬───────┼───────┬──────────┐
           │          │       │       │          │
        ┌──▼──┐   ┌──▼──┐ ┌──▼──┐ ┌──▼──┐      │
        │1×1  │   │1×1  │ │1×1  │ │3×3  │      │
        │Conv │   │Conv │ │Conv │ │Pool │      │
        └──┬──┘   └──┬──┘ └──┬──┘ └──┬──┘      │
           │      ┌──▼──┐ ┌──▼──┐    │         │
           │      │3×3  │ │5×5  │ ┌──▼──┐      │
           │      │Conv │ │Conv │ │1×1  │      │
           │      └──┬──┘ └──┬──┘ │Conv │      │
           │          │       │    └──┬──┘      │
           └──────────┴───────┴───────┴──────────┘
                         Concatenate
                    (along channel dim)

Key Innovations:
- Multiple filter sizes in parallel (multi-scale features)
- 1×1 convolutions for dimensionality reduction (bottleneck)
- Global Average Pooling (no FC layers → fewer params)
- Only 5M parameters (vs VGG's 138M!)
- Auxiliary classifiers for training deeper networks
```

### ResNet (2015) — He et al.

Philosophy: **If deeper is better, but training degrades — let the network learn RESIDUALS.**

```
The Degradation Problem:
- Adding more layers should at least not HURT (identity mapping)
- But plain deep networks perform WORSE than shallow ones
- Not overfitting — training error is higher too!

The Solution — Skip (Residual) Connections:

Plain Block:              Residual Block:
┌────────┐               ┌────────┐
│ Input x │               │ Input x │──────────────┐
└────┬───┘               └────┬───┘              │
     │                        │                   │
┌────▼───┐               ┌────▼───┐              │
│ Conv    │               │ Conv    │              │ (identity
├────────┤               ├────────┤              │  shortcut)
│ BN+ReLU│               │ BN+ReLU│              │
├────────┤               ├────────┤              │
│ Conv    │               │ Conv    │              │
├────────┤               ├────────┤              │
│ BN      │               │ BN      │              │
└────┬───┘               └────┬───┘              │
     │                        │                   │
┌────▼───┐               ┌────▼───────────────▼──┐
│ ReLU   │               │    F(x) + x    (ADD)  │
└────────┘               ├────────────────────────┤
                         │       ReLU              │
Output: H(x) = F(x)     └────────────────────────┘
                         Output: H(x) = F(x) + x
```

**Why it works mathematically:**

Instead of learning $H(x)$ directly, the network learns:
$$F(x) = H(x) - x$$

So the output is $H(x) = F(x) + x$

If the optimal mapping is close to identity, it's easier to learn $F(x) \approx 0$ than $H(x) \approx x$.

**Gradient flow:**
$$\frac{\partial \mathcal{L}}{\partial x} = \frac{\partial \mathcal{L}}{\partial H} \cdot \frac{\partial H}{\partial x} = \frac{\partial \mathcal{L}}{\partial H} \cdot \left(1 + \frac{\partial F}{\partial x}\right)$$

The "1" ensures gradients always flow — **no vanishing gradients!**

```
ResNet Variants:
┌──────────┬────────┬─────────────┬──────────────┐
│ Model    │ Layers │ Params      │ Top-5 Error  │
├──────────┼────────┼─────────────┼──────────────┤
│ ResNet-18│ 18     │ 11.7M       │ 10.92%       │
│ ResNet-34│ 34     │ 21.8M       │ 8.58%        │
│ ResNet-50│ 50     │ 25.6M       │ 7.13%        │
│ ResNet-101│101    │ 44.5M       │ 6.44%        │
│ ResNet-152│152    │ 60.2M       │ 6.16%        │
└──────────┴────────┴─────────────┴──────────────┘
```

**Bottleneck Block (ResNet-50+):**
```
Input (256 channels)
    │
┌───▼────┐
│ 1×1, 64 │  ← Reduce dimensions
├────────┤
│ 3×3, 64 │  ← Process at lower dims
├────────┤
│ 1×1,256│  ← Restore dimensions
└───┬────┘
    + ← Add skip connection
    │
Output (256 channels)
```

### Architecture Comparison

| Architecture | Year | Params | Top-5 Error | Key Innovation |
|-------------|------|--------|-------------|----------------|
| LeNet-5 | 1998 | 60K | — | First CNN |
| AlexNet | 2012 | 60M | 16.4% | ReLU + GPU + Dropout |
| VGG-16 | 2014 | 138M | 7.3% | Deeper with 3×3 only |
| GoogLeNet | 2014 | 5M | 6.7% | Inception module |
| ResNet-152 | 2015 | 60M | 3.6% | Skip connections |

---

## Code Examples

### Basic CNN from Scratch (PyTorch)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SimpleCNN(nn.Module):
    """A simple CNN for CIFAR-10 classification (10 classes, 32x32 RGB images)"""
    
    def __init__(self, num_classes=10):
        super(SimpleCNN, self).__init__()
        
        # Feature extraction layers
        # Conv2d(in_channels, out_channels, kernel_size, padding)
        self.conv1 = nn.Conv2d(3, 32, kernel_size=3, padding=1)   # 32x32x3 → 32x32x32
        self.bn1 = nn.BatchNorm2d(32)                              # Normalize activations
        
        self.conv2 = nn.Conv2d(32, 64, kernel_size=3, padding=1)  # 16x16x32 → 16x16x64
        self.bn2 = nn.BatchNorm2d(64)
        
        self.conv3 = nn.Conv2d(64, 128, kernel_size=3, padding=1) # 8x8x64 → 8x8x128
        self.bn3 = nn.BatchNorm2d(128)
        
        self.pool = nn.MaxPool2d(2, 2)  # Halves spatial dimensions
        
        # Classification layers
        self.fc1 = nn.Linear(128 * 4 * 4, 256)  # After 3 pools: 32→16→8→4
        self.fc2 = nn.Linear(256, num_classes)
        self.dropout = nn.Dropout(0.5)
    
    def forward(self, x):
        # Block 1: Conv → BN → ReLU → Pool
        x = self.pool(F.relu(self.bn1(self.conv1(x))))  # 32x32 → 16x16
        
        # Block 2: Conv → BN → ReLU → Pool
        x = self.pool(F.relu(self.bn2(self.conv2(x))))  # 16x16 → 8x8
        
        # Block 3: Conv → BN → ReLU → Pool
        x = self.pool(F.relu(self.bn3(self.conv3(x))))  # 8x8 → 4x4
        
        # Flatten and classify
        x = x.view(x.size(0), -1)      # Flatten: (batch, 128*4*4)
        x = self.dropout(F.relu(self.fc1(x)))
        x = self.fc2(x)                 # Raw logits (use CrossEntropyLoss)
        return x

# Instantiate and inspect
model = SimpleCNN()
print(f"Total parameters: {sum(p.numel() for p in model.parameters()):,}")
# Output: Total parameters: 295,818

# Test with random input
sample = torch.randn(1, 3, 32, 32)  # (batch=1, channels=3, H=32, W=32)
output = model(sample)
print(f"Output shape: {output.shape}")  # torch.Size([1, 10])
```

### Training Loop for CNN

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader
import torchvision
import torchvision.transforms as transforms

# ─── Data Preparation ─────────────────────────────────────
transform_train = transforms.Compose([
    transforms.RandomHorizontalFlip(),          # Data augmentation
    transforms.RandomCrop(32, padding=4),       # Random crop with padding
    transforms.ToTensor(),                      # Convert to tensor [0, 1]
    transforms.Normalize(                       # Normalize per channel
        mean=[0.4914, 0.4822, 0.4465],
        std=[0.2470, 0.2435, 0.2616]
    )
])

transform_test = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.4914, 0.4822, 0.4465],
                         std=[0.2470, 0.2435, 0.2616])
])

# Download CIFAR-10
trainset = torchvision.datasets.CIFAR10(root='./data', train=True,
                                         download=True, transform=transform_train)
testset = torchvision.datasets.CIFAR10(root='./data', train=False,
                                        download=True, transform=transform_test)

trainloader = DataLoader(trainset, batch_size=128, shuffle=True, num_workers=2)
testloader = DataLoader(testset, batch_size=128, shuffle=False, num_workers=2)

# ─── Training Setup ───────────────────────────────────────
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = SimpleCNN().to(device)
criterion = nn.CrossEntropyLoss()              # Includes softmax internally
optimizer = optim.Adam(model.parameters(), lr=0.001, weight_decay=1e-4)
scheduler = optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=50)

# ─── Training Loop ────────────────────────────────────────
num_epochs = 50

for epoch in range(num_epochs):
    model.train()
    running_loss = 0.0
    correct = 0
    total = 0
    
    for images, labels in trainloader:
        images, labels = images.to(device), labels.to(device)
        
        optimizer.zero_grad()           # Clear previous gradients
        outputs = model(images)         # Forward pass
        loss = criterion(outputs, labels)  # Compute loss
        loss.backward()                 # Backward pass (compute gradients)
        optimizer.step()                # Update weights
        
        # Track metrics
        running_loss += loss.item()
        _, predicted = outputs.max(1)
        total += labels.size(0)
        correct += predicted.eq(labels).sum().item()
    
    scheduler.step()  # Update learning rate
    
    train_acc = 100. * correct / total
    
    # ─── Validation ────────────────────────────────────
    model.eval()
    test_correct = 0
    test_total = 0
    
    with torch.no_grad():  # No gradients needed for evaluation
        for images, labels in testloader:
            images, labels = images.to(device), labels.to(device)
            outputs = model(images)
            _, predicted = outputs.max(1)
            test_total += labels.size(0)
            test_correct += predicted.eq(labels).sum().item()
    
    test_acc = 100. * test_correct / test_total
    
    if (epoch + 1) % 10 == 0:
        print(f"Epoch [{epoch+1}/{num_epochs}] "
              f"Loss: {running_loss/len(trainloader):.4f} "
              f"Train Acc: {train_acc:.2f}% "
              f"Test Acc: {test_acc:.2f}%")
```

### ResNet Implementation from Scratch

```python
import torch
import torch.nn as nn

class BasicBlock(nn.Module):
    """Basic residual block for ResNet-18/34"""
    expansion = 1  # Output channels = planes * expansion
    
    def __init__(self, in_channels, out_channels, stride=1, downsample=None):
        super(BasicBlock, self).__init__()
        
        # First conv: may downsample spatially (stride > 1)
        self.conv1 = nn.Conv2d(in_channels, out_channels, kernel_size=3,
                               stride=stride, padding=1, bias=False)
        self.bn1 = nn.BatchNorm2d(out_channels)
        
        # Second conv: always stride=1
        self.conv2 = nn.Conv2d(out_channels, out_channels, kernel_size=3,
                               stride=1, padding=1, bias=False)
        self.bn2 = nn.BatchNorm2d(out_channels)
        
        # Skip connection: if dimensions change, we need a projection
        self.downsample = downsample  # 1x1 conv to match dimensions
        self.relu = nn.ReLU(inplace=True)
    
    def forward(self, x):
        identity = x  # Save for skip connection
        
        out = self.relu(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        
        # If input/output dims don't match, project the identity
        if self.downsample is not None:
            identity = self.downsample(x)
        
        out += identity  # THE skip connection (element-wise addition)
        out = self.relu(out)
        return out


class BottleneckBlock(nn.Module):
    """Bottleneck block for ResNet-50/101/152"""
    expansion = 4  # Output channels = planes * 4
    
    def __init__(self, in_channels, planes, stride=1, downsample=None):
        super(BottleneckBlock, self).__init__()
        
        # 1×1 conv: reduce channels
        self.conv1 = nn.Conv2d(in_channels, planes, kernel_size=1, bias=False)
        self.bn1 = nn.BatchNorm2d(planes)
        
        # 3×3 conv: spatial processing at reduced dimensions
        self.conv2 = nn.Conv2d(planes, planes, kernel_size=3,
                               stride=stride, padding=1, bias=False)
        self.bn2 = nn.BatchNorm2d(planes)
        
        # 1×1 conv: restore channels (expand by 4x)
        self.conv3 = nn.Conv2d(planes, planes * self.expansion,
                               kernel_size=1, bias=False)
        self.bn3 = nn.BatchNorm2d(planes * self.expansion)
        
        self.downsample = downsample
        self.relu = nn.ReLU(inplace=True)
    
    def forward(self, x):
        identity = x
        
        out = self.relu(self.bn1(self.conv1(x)))   # 256 → 64
        out = self.relu(self.bn2(self.conv2(out)))  # 64 → 64 (spatial)
        out = self.bn3(self.conv3(out))             # 64 → 256
        
        if self.downsample is not None:
            identity = self.downsample(x)
        
        out += identity
        out = self.relu(out)
        return out


class ResNet(nn.Module):
    def __init__(self, block, layers, num_classes=1000):
        """
        Args:
            block: BasicBlock or BottleneckBlock
            layers: list of 4 ints (blocks per stage), e.g., [3,4,6,3] for ResNet-50
            num_classes: output classes
        """
        super(ResNet, self).__init__()
        self.in_channels = 64
        
        # Initial conv (stem)
        self.conv1 = nn.Conv2d(3, 64, kernel_size=7, stride=2, padding=3, bias=False)
        self.bn1 = nn.BatchNorm2d(64)
        self.relu = nn.ReLU(inplace=True)
        self.maxpool = nn.MaxPool2d(kernel_size=3, stride=2, padding=1)
        
        # Residual stages
        self.layer1 = self._make_layer(block, 64, layers[0], stride=1)
        self.layer2 = self._make_layer(block, 128, layers[1], stride=2)
        self.layer3 = self._make_layer(block, 256, layers[2], stride=2)
        self.layer4 = self._make_layer(block, 512, layers[3], stride=2)
        
        # Classification head
        self.avgpool = nn.AdaptiveAvgPool2d((1, 1))  # Global Average Pooling
        self.fc = nn.Linear(512 * block.expansion, num_classes)
        
        # Weight initialization (Kaiming/He)
        for m in self.modules():
            if isinstance(m, nn.Conv2d):
                nn.init.kaiming_normal_(m.weight, mode='fan_out', nonlinearity='relu')
            elif isinstance(m, nn.BatchNorm2d):
                nn.init.constant_(m.weight, 1)
                nn.init.constant_(m.bias, 0)
    
    def _make_layer(self, block, planes, num_blocks, stride):
        downsample = None
        
        # Need projection if stride changes or channels don't match
        if stride != 1 or self.in_channels != planes * block.expansion:
            downsample = nn.Sequential(
                nn.Conv2d(self.in_channels, planes * block.expansion,
                          kernel_size=1, stride=stride, bias=False),
                nn.BatchNorm2d(planes * block.expansion)
            )
        
        layers = []
        layers.append(block(self.in_channels, planes, stride, downsample))
        self.in_channels = planes * block.expansion
        
        for _ in range(1, num_blocks):
            layers.append(block(self.in_channels, planes))
        
        return nn.Sequential(*layers)
    
    def forward(self, x):
        # Stem
        x = self.maxpool(self.relu(self.bn1(self.conv1(x))))  # 224→56
        
        # Residual stages
        x = self.layer1(x)  # 56×56
        x = self.layer2(x)  # 28×28
        x = self.layer3(x)  # 14×14
        x = self.layer4(x)  # 7×7
        
        # Classification
        x = self.avgpool(x)        # 7×7 → 1×1
        x = torch.flatten(x, 1)   # (batch, 512*expansion)
        x = self.fc(x)            # (batch, num_classes)
        return x


# Create standard ResNet variants
def resnet18():  return ResNet(BasicBlock, [2, 2, 2, 2])
def resnet34():  return ResNet(BasicBlock, [3, 4, 6, 3])
def resnet50():  return ResNet(BottleneckBlock, [3, 4, 6, 3])
def resnet101(): return ResNet(BottleneckBlock, [3, 4, 23, 3])
def resnet152(): return ResNet(BottleneckBlock, [3, 8, 36, 3])

# Test
model = resnet50(num_classes=10)
x = torch.randn(2, 3, 224, 224)
print(f"Output: {model(x).shape}")  # torch.Size([2, 10])
print(f"Parameters: {sum(p.numel() for p in model.parameters()):,}")
# Output: Parameters: 23,528,522
```

### Using Pretrained Models (Transfer Learning)

```python
import torch
import torchvision.models as models
import torch.nn as nn

# ─── Load Pretrained ResNet-50 ─────────────────────────────
model = models.resnet50(weights='IMAGENET1K_V2')  # Pretrained on ImageNet

# Freeze all layers (feature extractor)
for param in model.parameters():
    param.requires_grad = False

# Replace the final FC layer for your task (e.g., 5 classes)
num_features = model.fc.in_features  # 2048
model.fc = nn.Sequential(
    nn.Dropout(0.3),
    nn.Linear(num_features, 256),
    nn.ReLU(),
    nn.Dropout(0.2),
    nn.Linear(256, 5)  # 5 output classes
)

# Only train the new FC layers
optimizer = torch.optim.Adam(model.fc.parameters(), lr=0.001)

# Pro Tip: After a few epochs, unfreeze last few layers for fine-tuning
# for param in model.layer4.parameters():
#     param.requires_grad = True
# optimizer = torch.optim.Adam(model.parameters(), lr=1e-5)  # Lower LR!
```

### Visualizing CNN Filters and Feature Maps

```python
import torch
import torchvision.models as models
import matplotlib.pyplot as plt
import numpy as np

# Load pretrained model
model = models.resnet50(weights='IMAGENET1K_V2')

# ─── Visualize First Layer Filters ────────────────────────
filters = model.conv1.weight.data.cpu()  # Shape: (64, 3, 7, 7)
print(f"First layer filters shape: {filters.shape}")

fig, axes = plt.subplots(8, 8, figsize=(12, 12))
for i, ax in enumerate(axes.flat):
    # Normalize filter for visualization
    f = filters[i]
    f = (f - f.min()) / (f.max() - f.min())
    ax.imshow(f.permute(1, 2, 0))  # (C, H, W) → (H, W, C)
    ax.axis('off')
plt.suptitle('First Layer Conv Filters (7×7)')
plt.tight_layout()
plt.savefig('conv1_filters.png')

# ─── Visualize Feature Maps (Activations) ─────────────────
from torchvision import transforms
from PIL import Image

# Hook to capture intermediate activations
activations = {}
def hook_fn(name):
    def hook(module, input, output):
        activations[name] = output.detach()
    return hook

# Register hooks on specific layers
model.layer1.register_forward_hook(hook_fn('layer1'))
model.layer3.register_forward_hook(hook_fn('layer3'))

# Process an image
img = Image.open('sample.jpg').resize((224, 224))
transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],
                         std=[0.229, 0.224, 0.225])
])
x = transform(img).unsqueeze(0)

model.eval()
with torch.no_grad():
    _ = model(x)

# Plot feature maps
feat = activations['layer1'][0]  # (256, 56, 56)
fig, axes = plt.subplots(4, 4, figsize=(10, 10))
for i, ax in enumerate(axes.flat):
    ax.imshow(feat[i].cpu(), cmap='viridis')
    ax.axis('off')
plt.suptitle('Layer 1 Feature Maps')
plt.savefig('feature_maps.png')
```

---

## Common Mistakes

### 1. Not Normalizing Input Data
```python
# ❌ WRONG: Raw pixel values [0, 255]
transform = transforms.ToTensor()  # Only converts to [0, 1]

# ✅ CORRECT: Normalize to match pretrained model's training distribution
transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],  # ImageNet stats
                         std=[0.229, 0.224, 0.225])
])
```

### 2. Wrong Input Dimensions
```python
# ❌ WRONG: Image is (H, W, C) — NumPy/PIL format
img = np.random.rand(224, 224, 3)
model(torch.tensor(img))  # Error!

# ✅ CORRECT: PyTorch expects (B, C, H, W)
img = torch.randn(1, 3, 224, 224)  # Batch, Channels, Height, Width
model(img)
```

### 3. Forgetting to Set model.eval() During Inference
```python
# ❌ WRONG: BatchNorm and Dropout behave differently in training mode
predictions = model(test_images)

# ✅ CORRECT: Switch to eval mode
model.eval()
with torch.no_grad():  # Also disable gradient computation
    predictions = model(test_images)
```

### 4. Using Too Large a Fully Connected Layer
```python
# ❌ WRONG: Flatten a large feature map → massive FC layer
# If feature map is 32×32×512, FC would have 32*32*512 = 524,288 inputs!

# ✅ CORRECT: Use Global Average Pooling
self.gap = nn.AdaptiveAvgPool2d((1, 1))  # Any size → 1×1
# Now FC only has 512 inputs regardless of input image size
```

### 5. Not Using Data Augmentation
```python
# ❌ WRONG: Only basic transform for training
transform = transforms.Compose([transforms.ToTensor()])

# ✅ CORRECT: Augment training data (NOT test data!)
transform_train = transforms.Compose([
    transforms.RandomResizedCrop(224),
    transforms.RandomHorizontalFlip(p=0.5),
    transforms.ColorJitter(brightness=0.2, contrast=0.2),
    transforms.RandomRotation(15),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],
                         std=[0.229, 0.224, 0.225])
])
```

### 6. Calculating Output Size Wrong
```python
# Before writing FC layer, calculate or use adaptive pooling
# For input 224×224 with standard ResNet stem:
# 224 → conv7x7/s2 → 112 → maxpool/s2 → 56
# 56 → layer2/s2 → 28 → layer3/s2 → 14 → layer4/s2 → 7
# After GAP: 7×7 → 1×1

# Pro Tip: Use a forward pass to check dimensions
def check_dimensions(model, input_size=(1, 3, 224, 224)):
    x = torch.randn(*input_size)
    for name, layer in model.named_children():
        x = layer(x)
        print(f"{name}: {x.shape}")
```

---

## Interview Questions

### Conceptual Questions

**Q1: Why do we use 3×3 kernels instead of larger ones like 7×7?**

Two stacked 3×3 convolutions have the same receptive field as one 5×5 convolution, but:
- Fewer parameters: $2 \times (3^2 C^2) = 18C^2$ vs $5^2 C^2 = 25C^2$
- More non-linearity (two ReLUs instead of one)
- Deeper representation with less computation

**Q2: Explain the vanishing gradient problem in deep CNNs and how ResNet solves it.**

In very deep networks, gradients get multiplied by weight matrices repeatedly during backpropagation. If weights < 1, gradients shrink exponentially → early layers don't learn. ResNet adds skip connections: $H(x) = F(x) + x$. The gradient becomes $\frac{\partial L}{\partial x} = \frac{\partial L}{\partial H}(1 + \frac{\partial F}{\partial x})$. The "1" ensures gradient always flows.

**Q3: What's the difference between valid, same, and full padding?**
- **Valid**: No padding, output shrinks. $O = W - K + 1$
- **Same**: Pad so output = input size. $P = (K-1)/2$
- **Full**: Pad so every input pixel is visited by every kernel position. $P = K-1$

**Q4: How many parameters does a Conv layer have?**

$(K_h \times K_w \times C_{in} + 1) \times C_{out}$. The +1 is the bias. Note: parameter count is **independent of input spatial size** — this is why CNNs scale to large images!

**Q5: What is the receptive field? How do you increase it?**

The receptive field is the area of the original input that a neuron "sees." Increase via:
- Stacking more layers
- Using larger kernels
- Using pooling/strided convolutions
- Using dilated convolutions

**Q6: Why does ResNet-50 use bottleneck blocks instead of basic blocks?**

Bottleneck (1×1 → 3×3 → 1×1) reduces computation:
- Basic block on 256 channels: $2 \times 3^2 \times 256^2 = 1,179,648$ params
- Bottleneck (256→64→64→256): $256 \times 64 + 9 \times 64^2 + 64 \times 256 = 69,632$ params
- **17× fewer parameters** for the same receptive field!

**Q7: Explain 1×1 convolution. Why is it useful?**

A 1×1 conv is a channel-wise fully connected layer applied at every spatial position:
- **Dimensionality reduction** (256 channels → 64 channels)
- **Adding non-linearity** without changing spatial dims
- **Cross-channel interaction** (mixes channel information)
- Used in Inception (bottleneck), ResNet (projection), and many modern architectures

**Q8: How does Global Average Pooling (GAP) replace FC layers?**

GAP averages each feature map into a single value. For $C$ feature maps → $C$-dimensional vector. Benefits:
- No learnable parameters (less overfitting)
- Acts as structural regularizer
- Makes model input-size agnostic

### Coding Questions

**Q9: Calculate the output size for these layers:**
```
Input: 64×64×3
Conv(3×3, 32 filters, stride=1, padding=1) → ?
MaxPool(2×2, stride=2) → ?
Conv(3×3, 64 filters, stride=2, padding=0) → ?
```

Answer:
- After Conv1: $\lfloor(64 - 3 + 2)/1\rfloor + 1 = 64$. Shape: 64×64×32
- After Pool: 64/2 = 32. Shape: 32×32×32  
- After Conv2: $\lfloor(32 - 3 + 0)/2\rfloor + 1 = 15$. Shape: 15×15×64

**Q10: What's wrong with this code?**
```python
model.train()  # Training mode
for images, labels in test_loader:
    outputs = model(images)
    loss = criterion(outputs, labels)
    loss.backward()  # Computing gradients on test set!
```

Answer: Should use `model.eval()` and `torch.no_grad()` for evaluation. Computing gradients on test data wastes memory and `model.train()` means BatchNorm uses batch stats instead of running stats.

---

## Quick Reference

### CNN Cheat Sheet

| Concept | Formula/Rule | Notes |
|---------|-------------|-------|
| Output size | $\lfloor\frac{W-K+2P}{S}\rfloor + 1$ | W=input, K=kernel, P=pad, S=stride |
| Same padding | $P = (K-1)/2$ | Only for odd kernel sizes |
| Params per conv | $(K^2 \times C_{in} + 1) \times C_{out}$ | Independent of input size! |
| Receptive field | $r_l = r_{l-1} + (k_l - 1) \times \prod_{i=1}^{l-1} s_i$ | Grows with depth |
| FLOPs per conv | $K^2 \times C_{in} \times C_{out} \times H_{out} \times W_{out}$ | For computational budget |

### When to Use What

| Situation | Architecture | Why |
|-----------|-------------|-----|
| Limited data (<1K images) | Pretrained ResNet + fine-tune | Transfer learning prevents overfitting |
| Real-time inference | MobileNet / EfficientNet-B0 | Depthwise separable convs = fast |
| Maximum accuracy | EfficientNet-B7 / ConvNeXt | State-of-art but heavy |
| Small input (28×28) | Simple 3-layer CNN | Don't need deep models for MNIST |
| Very deep network needed | ResNet / DenseNet | Skip connections prevent degradation |
| Multi-scale features | Inception / FPN | Parallel branches at different scales |

### Tensor Shape Cheatsheet (PyTorch)

```
Input Image:     (Batch, Channels, Height, Width)  → (B, C, H, W)
Conv2d output:   (B, out_channels, H', W')
After Flatten:   (B, out_channels * H' * W')
FC output:       (B, num_classes)
```

### Key Hyperparameters

| Hyperparameter | Typical Values | Notes |
|---------------|---------------|-------|
| Learning rate | 1e-3 (from scratch), 1e-5 (fine-tune) | Most important! |
| Batch size | 32, 64, 128, 256 | Larger = faster but needs more memory |
| Kernel size | 3×3 (most common), 1×1, 5×5 | 3×3 is standard |
| Weight decay | 1e-4 to 1e-5 | L2 regularization |
| Dropout | 0.2–0.5 | After FC layers, not conv layers |
| Epochs | 50–200 | Use early stopping |

---

> **Pro Tips for Production:**
> 1. Always use **mixed precision training** (fp16) — 2× speedup, half memory
> 2. Use **gradient accumulation** if batch size is limited by GPU memory
> 3. **Learning rate warmup** (linear or cosine) prevents early divergence
> 4. **Test Time Augmentation (TTA)** — average predictions over augmented versions
> 5. **Label smoothing** (0.1) improves generalization
> 6. For deployment, use **torch.jit.trace** or export to **ONNX**
