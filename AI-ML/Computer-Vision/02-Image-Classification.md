# Chapter 02: Image Classification

## Table of Contents
- [2.1 What is Image Classification?](#21-what-is-image-classification)
- [2.2 From Pixels to Predictions — The Pipeline](#22-from-pixels-to-predictions--the-pipeline)
- [2.3 Convolutional Neural Networks (CNNs) — Core Architecture](#23-convolutional-neural-networks-cnns--core-architecture)
- [2.4 Landmark CNN Architectures](#24-landmark-cnn-architectures)
- [2.5 Modern Architectures (EfficientNet, ConvNeXt)](#25-modern-architectures-efficientnet-convnext)
- [2.6 Transfer Learning — The Practical Superpower](#26-transfer-learning--the-practical-superpower)
- [2.7 Data Augmentation](#27-data-augmentation)
- [2.8 Training Strategies and Tricks](#28-training-strategies-and-tricks)
- [2.9 Evaluation Metrics](#29-evaluation-metrics)
- [2.10 Practical End-to-End Project](#210-practical-end-to-end-project)
- [2.11 Common Failure Modes and Debugging](#211-common-failure-modes-and-debugging)
- [Quick Reference](#quick-reference)

---

## 2.1 What is Image Classification?

### What It Is
Image classification assigns a **single label** (or multiple labels) to an entire image. Given a photo, the model answers: "What is this?" — Is it a cat? A dog? A tumor? A defective product?

Think of it like sorting mail: you look at each letter and put it in the right mailbox based on the address.

### Why It Matters
Image classification is the **foundational task** of computer vision:
- **Medical:** Classify X-rays as normal/abnormal, skin lesions as benign/malignant
- **Manufacturing:** Detect defective products on assembly lines
- **Agriculture:** Identify crop diseases from leaf photos
- **Content moderation:** Flag inappropriate images at scale
- **Autonomous driving:** Recognize traffic signs

### Types of Classification

| Type | Output | Example |
|------|--------|---------|
| Binary | 1 class (yes/no) | Cat vs Dog, Defect vs OK |
| Multi-class | 1 of N classes | Classify into 1000 ImageNet categories |
| Multi-label | Multiple of N classes | Image has both "beach" AND "sunset" tags |

### How It Works (High Level)

```
                    ┌─────────────┐
                    │             │
 Image (224×224×3) → Feature      → Flatten → FC Layers → Softmax → [cat: 0.92]
                    │ Extraction  │                                   [dog: 0.05]
                    │ (CNN)       │                                   [bird: 0.03]
                    └─────────────┘
```

**Mathematical formulation:**

Given input image $x \in \mathbb{R}^{H \times W \times C}$ and classes $\{1, 2, ..., K\}$:

$$f(x) = \text{argmax}_{k} \; P(y=k | x; \theta)$$

Where $\theta$ are the learned parameters and the probability is computed via softmax:

$$P(y=k|x) = \frac{e^{z_k}}{\sum_{j=1}^{K} e^{z_j}}$$

---

## 2.2 From Pixels to Predictions — The Pipeline

### The Complete Training Pipeline

```
┌──────────┐    ┌─────────────┐    ┌───────────┐    ┌──────────┐    ┌──────────┐
│  Dataset │ →  │ Preprocessing│ →  │   Model   │ →  │   Loss   │ →  │ Optimizer│
│  Loading │    │ + Augment   │    │  Forward  │    │ Compute  │    │ Backward │
└──────────┘    └─────────────┘    └───────────┘    └──────────┘    └──────────┘
                                                                          │
                                                                          ↓
                                                                    Update Weights
                                                                    (Repeat Epochs)
```

### Key Components

| Component | Purpose | Common Choices |
|-----------|---------|----------------|
| Data Loader | Batch images efficiently | PyTorch DataLoader, tf.data |
| Preprocessing | Normalize, resize | ImageNet norm, resize to 224 |
| Backbone | Extract features | ResNet, EfficientNet, ViT |
| Head | Map features → classes | Linear layer(s) |
| Loss Function | Measure prediction error | CrossEntropy, BCE |
| Optimizer | Update weights | AdamW, SGD+momentum |
| Scheduler | Adjust learning rate | CosineAnnealing, OneCycleLR |

---

## 2.3 Convolutional Neural Networks (CNNs) — Core Architecture

### What It Is
A CNN is a neural network that uses **convolution operations** to automatically learn spatial features from images. Instead of manually designing edge detectors or texture filters, the CNN learns the optimal filters from data.

### Why It Matters
Before CNNs, computer vision relied on hand-crafted features (SIFT, HOG). CNNs made feature engineering automatic, leading to superhuman performance on many tasks.

### How It Works

**The Building Blocks:**

```
Input Image (224×224×3)
         │
         ↓
┌─────────────────────┐
│  Convolutional Layer │  → Learn spatial patterns (edges → textures → objects)
│  + ReLU Activation  │
├─────────────────────┤
│  Pooling Layer      │  → Reduce spatial size, add invariance
├─────────────────────┤
│  (Repeat N times)   │  → Each level learns more complex features
├─────────────────────┤
│  Global Avg Pool    │  → Collapse spatial dims to vector
├─────────────────────┤
│  Fully Connected    │  → Map features to class scores
├─────────────────────┤
│  Softmax            │  → Convert scores to probabilities
└─────────────────────┘
```

**Feature hierarchy:**

```
Layer 1: Edges, colors          (3×3 → 64 filters)
Layer 2: Textures, corners      (3×3 → 128 filters)  
Layer 3: Parts (eyes, wheels)   (3×3 → 256 filters)
Layer 4: Objects (faces, cars)  (3×3 → 512 filters)
```

**Convolution math in CNNs:**

For input feature map $X$ with $C_{in}$ channels, one convolutional filter produces:

$$Y(i,j) = \sum_{c=1}^{C_{in}} \sum_{m=0}^{k-1} \sum_{n=0}^{k-1} W_c(m,n) \cdot X_c(i+m, j+n) + b$$

Output spatial size:
$$O = \frac{I - K + 2P}{S} + 1$$

Where: $I$ = input size, $K$ = kernel size, $P$ = padding, $S$ = stride

**Parameter count for one conv layer:**
$$\text{Params} = (K \times K \times C_{in} + 1) \times C_{out}$$

### Code Examples

```python
import torch
import torch.nn as nn

# === Building a CNN from scratch ===
class SimpleCNN(nn.Module):
    """A basic CNN for image classification."""
    def __init__(self, num_classes=10):
        super().__init__()
        
        # Feature extractor
        self.features = nn.Sequential(
            # Block 1: 3 → 32 channels, spatial: 224 → 112
            nn.Conv2d(3, 32, kernel_size=3, padding=1),  # (B, 32, 224, 224)
            nn.BatchNorm2d(32),                           # Normalize activations
            nn.ReLU(inplace=True),                        # Non-linearity
            nn.MaxPool2d(2, 2),                           # (B, 32, 112, 112)
            
            # Block 2: 32 → 64 channels, spatial: 112 → 56
            nn.Conv2d(32, 64, kernel_size=3, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2, 2),                           # (B, 64, 56, 56)
            
            # Block 3: 64 → 128 channels, spatial: 56 → 28
            nn.Conv2d(64, 128, kernel_size=3, padding=1),
            nn.BatchNorm2d(128),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2, 2),                           # (B, 128, 28, 28)
            
            # Block 4: 128 → 256 channels, spatial: 28 → 14
            nn.Conv2d(128, 256, kernel_size=3, padding=1),
            nn.BatchNorm2d(256),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2, 2),                           # (B, 256, 14, 14)
        )
        
        # Classifier head
        self.classifier = nn.Sequential(
            nn.AdaptiveAvgPool2d((1, 1)),  # (B, 256, 1, 1) — works for any input size!
            nn.Flatten(),                   # (B, 256)
            nn.Dropout(0.5),               # Regularization
            nn.Linear(256, num_classes),    # (B, num_classes)
        )
    
    def forward(self, x):
        x = self.features(x)
        x = self.classifier(x)
        return x

# === Instantiate and check ===
model = SimpleCNN(num_classes=10)
dummy_input = torch.randn(1, 3, 224, 224)  # Batch of 1, RGB, 224x224
output = model(dummy_input)
print(f"Output shape: {output.shape}")  # torch.Size([1, 10])

# === Count parameters ===
total_params = sum(p.numel() for p in model.parameters())
trainable_params = sum(p.numel() for p in model.parameters() if p.requires_grad)
print(f"Total parameters: {total_params:,}")
print(f"Trainable parameters: {trainable_params:,}")
```

### Key CNN Components Explained

#### Batch Normalization
Normalizes activations within each mini-batch → faster training, allows higher learning rates.

$$\hat{x} = \frac{x - \mu_B}{\sqrt{\sigma_B^2 + \epsilon}} \quad \rightarrow \quad y = \gamma \hat{x} + \beta$$

#### Pooling
Reduces spatial dimensions and adds translation invariance.

```python
# Max pooling: takes maximum in each window (preserves strongest activation)
nn.MaxPool2d(kernel_size=2, stride=2)  # Halves spatial dims

# Average pooling: takes mean (smoother but loses sharp features)
nn.AvgPool2d(kernel_size=2, stride=2)

# Global Average Pooling: collapses entire spatial map to 1 value per channel
nn.AdaptiveAvgPool2d((1, 1))  # (B, C, H, W) → (B, C, 1, 1)
```

#### Activation Functions

| Function | Formula | Pros | Cons |
|----------|---------|------|------|
| ReLU | $\max(0, x)$ | Fast, no vanishing gradient | Dead neurons (output always 0) |
| LeakyReLU | $\max(0.01x, x)$ | No dead neurons | Slightly slower |
| GELU | $x \cdot \Phi(x)$ | Smooth, used in transformers | More compute |
| SiLU/Swish | $x \cdot \sigma(x)$ | Smooth, state-of-art | More compute |

---

## 2.4 Landmark CNN Architectures

### Why Study These?
Each architecture introduced ideas still used today. Understanding their evolution helps you choose and design models.

### Architecture Timeline

```
1998    2012      2014      2015       2015      2016       2017        2019
LeNet → AlexNet → VGG → GoogLeNet → ResNet → DenseNet → MobileNet → EfficientNet
                                        ↑
                            Most important breakthrough:
                            Skip connections (residual learning)
```

### AlexNet (2012) — The Revolution

Started the deep learning revolution. Won ImageNet 2012 by a huge margin.

**Key innovations:** ReLU, dropout, GPU training, data augmentation

```python
# AlexNet architecture (simplified)
# 5 conv layers + 3 FC layers, ~60M parameters
# Input: 227×227×3

import torchvision.models as models
alexnet = models.alexnet(pretrained=True)
```

### VGG (2014) — Simplicity and Depth

**Key insight:** Use many small (3×3) filters instead of few large filters.

Two 3×3 convs have same receptive field as one 5×5, but fewer parameters:
- One 5×5: $5 \times 5 \times C^2 = 25C^2$ parameters
- Two 3×3: $2 \times (3 \times 3 \times C^2) = 18C^2$ parameters (28% fewer!)

```python
# VGG-16: 16 layers, ~138M parameters
# Architecture: Conv blocks with increasing channels (64→128→256→512→512)
vgg16 = models.vgg16(weights='IMAGENET1K_V1')
```

### ResNet (2015) — The Most Important Architecture

**The Problem:** Very deep networks (>20 layers) actually performed WORSE than shallow ones (degradation problem — not overfitting!).

**The Solution:** Skip connections (residual learning)

$$y = F(x) + x$$

Instead of learning the full mapping $H(x)$, learn the **residual** $F(x) = H(x) - x$.

```
Standard block:                 Residual block:
x → [Conv] → [Conv] → y       x → [Conv] → [Conv] → (+) → y
                                |                       ↑
                                └───────────────────────┘
                                    (skip connection)

Why it works:
- If the optimal mapping is identity, F(x)=0 is easy to learn
- Gradients flow directly through skip connections (no vanishing)
- Enables training of 100+ layer networks
```

```python
import torch
import torch.nn as nn
import torchvision.models as models

# === ResNet variants ===
resnet18 = models.resnet18(weights='IMAGENET1K_V1')    # 11.7M params
resnet50 = models.resnet50(weights='IMAGENET1K_V1')    # 25.6M params
resnet101 = models.resnet101(weights='IMAGENET1K_V1')  # 44.5M params

# === Residual Block implementation ===
class ResidualBlock(nn.Module):
    """Basic residual block (used in ResNet-18/34)."""
    def __init__(self, in_channels, out_channels, stride=1):
        super().__init__()
        self.conv1 = nn.Conv2d(in_channels, out_channels, 3, stride=stride, padding=1, bias=False)
        self.bn1 = nn.BatchNorm2d(out_channels)
        self.conv2 = nn.Conv2d(out_channels, out_channels, 3, stride=1, padding=1, bias=False)
        self.bn2 = nn.BatchNorm2d(out_channels)
        self.relu = nn.ReLU(inplace=True)
        
        # Skip connection: if dimensions change, use 1x1 conv to match
        self.shortcut = nn.Sequential()
        if stride != 1 or in_channels != out_channels:
            self.shortcut = nn.Sequential(
                nn.Conv2d(in_channels, out_channels, 1, stride=stride, bias=False),
                nn.BatchNorm2d(out_channels)
            )
    
    def forward(self, x):
        identity = self.shortcut(x)      # Skip connection
        out = self.relu(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        out += identity                   # Add skip connection
        out = self.relu(out)
        return out

# === Bottleneck Block (used in ResNet-50/101/152) ===
class BottleneckBlock(nn.Module):
    """1x1 → 3x3 → 1x1 reduces computation."""
    expansion = 4
    
    def __init__(self, in_channels, mid_channels, stride=1):
        super().__init__()
        out_channels = mid_channels * self.expansion
        self.conv1 = nn.Conv2d(in_channels, mid_channels, 1, bias=False)
        self.bn1 = nn.BatchNorm2d(mid_channels)
        self.conv2 = nn.Conv2d(mid_channels, mid_channels, 3, stride=stride, padding=1, bias=False)
        self.bn2 = nn.BatchNorm2d(mid_channels)
        self.conv3 = nn.Conv2d(mid_channels, out_channels, 1, bias=False)
        self.bn3 = nn.BatchNorm2d(out_channels)
        self.relu = nn.ReLU(inplace=True)
        
        self.shortcut = nn.Sequential()
        if stride != 1 or in_channels != out_channels:
            self.shortcut = nn.Sequential(
                nn.Conv2d(in_channels, out_channels, 1, stride=stride, bias=False),
                nn.BatchNorm2d(out_channels)
            )
    
    def forward(self, x):
        identity = self.shortcut(x)
        out = self.relu(self.bn1(self.conv1(x)))
        out = self.relu(self.bn2(self.conv2(out)))
        out = self.bn3(self.conv3(out))
        out += identity
        out = self.relu(out)
        return out
```

### Architecture Comparison Table

| Model | Year | Params | Top-1 Acc | Key Innovation |
|-------|------|--------|-----------|----------------|
| AlexNet | 2012 | 60M | 63.3% | ReLU, dropout, GPU |
| VGG-16 | 2014 | 138M | 74.4% | Small filters, depth |
| GoogLeNet | 2014 | 6.8M | 74.8% | Inception modules |
| ResNet-50 | 2015 | 25.6M | 76.1% | Skip connections |
| ResNet-152 | 2015 | 60.2M | 78.3% | Very deep residuals |
| DenseNet-121 | 2017 | 8M | 74.4% | Dense connections |
| MobileNetV2 | 2018 | 3.4M | 72.0% | Inverted residuals, mobile |
| EfficientNet-B0 | 2019 | 5.3M | 77.1% | Compound scaling |
| EfficientNet-B7 | 2019 | 66M | 84.3% | Largest EfficientNet |
| ConvNeXt-T | 2022 | 28.6M | 82.1% | CNN matching ViT |

---

## 2.5 Modern Architectures (EfficientNet, ConvNeXt)

### EfficientNet — Smart Scaling

**The Problem:** How do you scale a network? More layers? More channels? Higher resolution? All three?

**The Solution:** Compound scaling — scale all three dimensions together using a fixed ratio.

$$\text{depth}: d = \alpha^\phi, \quad \text{width}: w = \beta^\phi, \quad \text{resolution}: r = \gamma^\phi$$
$$\text{subject to:} \quad \alpha \cdot \beta^2 \cdot \gamma^2 \approx 2$$

Where $\phi$ is a user-specified coefficient that controls total compute.

```python
import torchvision.models as models

# EfficientNet family (B0 → B7, progressively larger)
efficientnet_b0 = models.efficientnet_b0(weights='IMAGENET1K_V1')  # 5.3M params
efficientnet_b4 = models.efficientnet_b4(weights='IMAGENET1K_V1')  # 19M params

# EfficientNetV2 (faster training, progressive resizing)
efficientnet_v2_s = models.efficientnet_v2_s(weights='IMAGENET1K_V1')
```

### ConvNeXt — CNN Fights Back (2022)

ResNets "modernized" with Vision Transformer tricks. Proves CNNs can match ViTs when given the same training recipe.

**Key changes from ResNet:**
- Patchify stem (4×4 conv, stride 4) instead of 7×7 conv + maxpool
- Fewer, wider layers (like Transformers)
- LayerNorm instead of BatchNorm
- GELU instead of ReLU
- Inverted bottleneck (more channels in the middle)

```python
convnext_tiny = models.convnext_tiny(weights='IMAGENET1K_V1')   # 28.6M
convnext_base = models.convnext_base(weights='IMAGENET1K_V1')   # 88.6M
```

### When to Use What

| Scenario | Recommended Model | Why |
|----------|------------------|-----|
| Mobile/Edge deployment | MobileNetV3, EfficientNet-B0 | Small, fast |
| General classification | ResNet-50, EfficientNet-B4 | Good accuracy/speed balance |
| Maximum accuracy | EfficientNet-B7, ConvNeXt-L | Best performance |
| Limited training data | ResNet-18/34 with pretrained | Less overfitting |
| Research baseline | ResNet-50 | Universal benchmark |

---

## 2.6 Transfer Learning — The Practical Superpower

### What It Is
Instead of training from scratch on YOUR data, start with a model already trained on millions of images (ImageNet). Then **fine-tune** it for your specific task. 

Analogy: Instead of learning a new language from zero, you leverage your existing language knowledge — grammar concepts transfer, you just learn new vocabulary.

### Why It Matters
- **Less data needed:** Works with just hundreds of images (vs millions from scratch)
- **Faster training:** Converges in minutes/hours instead of days
- **Better accuracy:** Pretrained features generalize well
- **Industry standard:** 95%+ of real-world projects use transfer learning

### How It Works

**Three transfer learning strategies:**

```
Strategy 1: Feature Extraction (freeze everything, train new head)
┌─────────────────────────────────────────────────┐
│  Pretrained Backbone (FROZEN)  │  New Head (TRAIN) │
│  ResNet features (don't change) │  Linear → Classes  │
└─────────────────────────────────────────────────┘
Best when: Very little data (<1000 images), similar domain

Strategy 2: Fine-tuning (unfreeze some layers)
┌─────────────────────────────────────────────────┐
│ Early Layers │ Later Layers │  New Head  │
│  (FROZEN)    │ (TRAIN, low lr) │  (TRAIN, high lr) │
└─────────────────────────────────────────────────┘
Best when: Moderate data (1000-10000 images)

Strategy 3: Full fine-tuning (unfreeze everything, low lr)
┌─────────────────────────────────────────────────┐
│      All Layers (TRAIN with low learning rate)    │
└─────────────────────────────────────────────────┘
Best when: Large dataset, different domain (e.g., medical)
```

### Code Examples

```python
import torch
import torch.nn as nn
import torchvision.models as models
from torchvision import transforms, datasets
from torch.utils.data import DataLoader

# === Strategy 1: Feature Extraction ===
def create_feature_extractor(num_classes, freeze_backbone=True):
    """Use pretrained ResNet50 as fixed feature extractor."""
    model = models.resnet50(weights='IMAGENET1K_V1')
    
    # Freeze all backbone layers
    if freeze_backbone:
        for param in model.parameters():
            param.requires_grad = False
    
    # Replace the classification head
    # ResNet50's fc layer: Linear(2048, 1000) → Linear(2048, num_classes)
    num_features = model.fc.in_features  # 2048
    model.fc = nn.Sequential(
        nn.Dropout(0.5),
        nn.Linear(num_features, num_classes)
    )
    
    return model

# === Strategy 2: Gradual Unfreezing ===
def create_finetuned_model(num_classes):
    """Fine-tune later layers, freeze early layers."""
    model = models.resnet50(weights='IMAGENET1K_V1')
    
    # Freeze early layers (layer1, layer2)
    for name, param in model.named_parameters():
        if 'layer3' not in name and 'layer4' not in name and 'fc' not in name:
            param.requires_grad = False
    
    # Replace head
    model.fc = nn.Linear(model.fc.in_features, num_classes)
    
    # Use different learning rates for different parts
    # (shown in training section below)
    return model

# === Strategy 3: Discriminative Learning Rates ===
def get_param_groups(model, lr_backbone=1e-5, lr_head=1e-3):
    """Different learning rates for backbone vs head."""
    backbone_params = []
    head_params = []
    
    for name, param in model.named_parameters():
        if param.requires_grad:
            if 'fc' in name:
                head_params.append(param)
            else:
                backbone_params.append(param)
    
    return [
        {'params': backbone_params, 'lr': lr_backbone},
        {'params': head_params, 'lr': lr_head},
    ]

# === Complete transfer learning example ===
num_classes = 5  # e.g., 5 types of flowers

# Create model
model = create_feature_extractor(num_classes, freeze_backbone=True)
model = model.cuda()

# Define transforms (MUST match pretrained model's training transforms)
train_transforms = transforms.Compose([
    transforms.RandomResizedCrop(224),
    transforms.RandomHorizontalFlip(),
    transforms.ColorJitter(brightness=0.2, contrast=0.2),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],
                         std=[0.229, 0.224, 0.225]),
])

val_transforms = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],
                         std=[0.229, 0.224, 0.225]),
])

# Load dataset (assumes folder structure: train/class_name/images)
train_dataset = datasets.ImageFolder('data/train', transform=train_transforms)
val_dataset = datasets.ImageFolder('data/val', transform=val_transforms)

train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True, num_workers=4)
val_loader = DataLoader(val_dataset, batch_size=32, shuffle=False, num_workers=4)

# Only optimize unfrozen parameters
optimizer = torch.optim.Adam(filter(lambda p: p.requires_grad, model.parameters()),
                             lr=1e-3)
criterion = nn.CrossEntropyLoss()
```

> **Pro Tip:** Always start with frozen backbone (Strategy 1). If accuracy plateaus, gradually unfreeze later layers with a much smaller learning rate (10-100× smaller than the head).

---

## 2.7 Data Augmentation

### What It Is
Data augmentation creates **artificial training variations** by applying random transformations to images. One image becomes many different training examples. It's like teaching a child to recognize dogs by showing the same dog from different angles, in different lighting, partially obscured.

### Why It Matters
- **Prevents overfitting:** More diversity → model can't memorize
- **Simulates real-world:** Training images rarely match deployment conditions
- **Free data:** Effectively multiplies dataset size without collecting more images
- **Required for small datasets:** Without augmentation, small datasets lead to overfitting

### How It Works

**Categories of augmentation:**

| Category | Transforms | Effect |
|----------|-----------|--------|
| Geometric | Flip, rotate, scale, crop | Position/viewpoint invariance |
| Color | Brightness, contrast, saturation, hue | Lighting invariance |
| Noise/Blur | Gaussian noise, blur, JPEG artifacts | Robustness to quality |
| Occlusion | Cutout, Random Erasing, CutMix | Partial visibility robustness |
| Mix | MixUp, CutMix, Mosaic | Regularization, smoother boundaries |

### Code Examples

```python
import torchvision.transforms as T
from torchvision.transforms import v2  # New transforms API (PyTorch 2.0+)

# === Standard augmentation pipeline (classification) ===
train_transform = T.Compose([
    T.RandomResizedCrop(224, scale=(0.08, 1.0)),  # Random crop + resize
    T.RandomHorizontalFlip(p=0.5),                 # 50% horizontal flip
    T.ColorJitter(
        brightness=0.4,  # ±40% brightness
        contrast=0.4,    # ±40% contrast
        saturation=0.4,  # ±40% saturation
        hue=0.1          # ±10% hue shift
    ),
    T.RandomRotation(degrees=15),                  # ±15° rotation
    T.ToTensor(),
    T.Normalize(mean=[0.485, 0.456, 0.406],
                std=[0.229, 0.224, 0.225]),
    T.RandomErasing(p=0.25, scale=(0.02, 0.33)),  # Random rectangular mask
])

# === Advanced augmentation with Albumentations (industry standard) ===
import albumentations as A
from albumentations.pytorch import ToTensorV2

train_transform_album = A.Compose([
    A.RandomResizedCrop(224, 224, scale=(0.08, 1.0)),
    A.HorizontalFlip(p=0.5),
    A.OneOf([
        A.GaussianBlur(blur_limit=7),
        A.MotionBlur(blur_limit=7),
        A.MedianBlur(blur_limit=7),
    ], p=0.3),
    A.OneOf([
        A.OpticalDistortion(distort_limit=0.5),
        A.GridDistortion(num_steps=5),
        A.ElasticTransform(alpha=1),
    ], p=0.3),
    A.ColorJitter(brightness=0.4, contrast=0.4, saturation=0.4, hue=0.1, p=0.5),
    A.CoarseDropout(max_holes=8, max_height=32, max_width=32, p=0.3),  # Cutout
    A.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
    ToTensorV2(),
])

# Usage with Albumentations:
# transformed = train_transform_album(image=numpy_image)
# img_tensor = transformed['image']

# === MixUp implementation ===
def mixup_data(x, y, alpha=0.2):
    """MixUp: blend two images and their labels."""
    lam = np.random.beta(alpha, alpha)  # Mixing coefficient
    batch_size = x.size(0)
    index = torch.randperm(batch_size)  # Random shuffle
    
    mixed_x = lam * x + (1 - lam) * x[index]  # Blend images
    y_a, y_b = y, y[index]                      # Both labels
    return mixed_x, y_a, y_b, lam

# MixUp loss:
# loss = lam * criterion(output, y_a) + (1 - lam) * criterion(output, y_b)

# === CutMix implementation ===
def cutmix_data(x, y, alpha=1.0):
    """CutMix: paste a patch from one image onto another."""
    lam = np.random.beta(alpha, alpha)
    batch_size = x.size(0)
    index = torch.randperm(batch_size)
    
    # Random bounding box
    W, H = x.size(3), x.size(2)
    cut_ratio = np.sqrt(1 - lam)
    cut_w = int(W * cut_ratio)
    cut_h = int(H * cut_ratio)
    cx = np.random.randint(W)
    cy = np.random.randint(H)
    
    x1 = np.clip(cx - cut_w // 2, 0, W)
    x2 = np.clip(cx + cut_w // 2, 0, W)
    y1 = np.clip(cy - cut_h // 2, 0, H)
    y2 = np.clip(cy + cut_h // 2, 0, H)
    
    x[:, :, y1:y2, x1:x2] = x[index, :, y1:y2, x1:x2]
    
    # Adjust lambda based on actual cut area
    lam = 1 - (x2 - x1) * (y2 - y1) / (W * H)
    return x, y, y[index], lam
```

### Augmentation Strategy Guide

| Dataset Size | Recommended Augmentation | Intensity |
|-------------|-------------------------|-----------|
| < 1,000 | Heavy (all types + MixUp/CutMix) | Aggressive |
| 1,000 - 10,000 | Standard (geometric + color) | Moderate |
| 10,000 - 100,000 | Light (flip + crop + color jitter) | Light |
| > 100,000 | Minimal (basic flip + crop) | Very light |

> **⚠️ Warning:** Don't augment validation/test data! Only apply deterministic transforms (resize, center crop, normalize) to validation sets.

---

## 2.8 Training Strategies and Tricks

### What It Is
Training strategies are the collection of techniques that determine how efficiently and effectively a model learns. The architecture matters, but HOW you train often matters more.

### Why It Matters
The same ResNet-50 can achieve 76% or 80%+ accuracy depending on training recipe. These "tricks" are what separate competitive models from mediocre ones.

### Key Training Techniques

#### Learning Rate Scheduling

The learning rate (LR) is the **most important hyperparameter**. Too high → diverge. Too low → slow/stuck.

```python
import torch.optim as optim
from torch.optim.lr_scheduler import (
    CosineAnnealingLR, OneCycleLR, StepLR, ReduceLROnPlateau
)

optimizer = optim.AdamW(model.parameters(), lr=1e-3, weight_decay=0.01)

# === Cosine Annealing (smooth decay, very popular) ===
scheduler = CosineAnnealingLR(optimizer, T_max=100, eta_min=1e-6)

# === OneCycleLR (best for quick convergence) ===
scheduler = OneCycleLR(
    optimizer,
    max_lr=1e-3,           # Peak learning rate
    epochs=30,
    steps_per_epoch=len(train_loader),
    pct_start=0.3,         # 30% warmup, 70% decay
    anneal_strategy='cos'
)

# === ReduceLROnPlateau (adaptive, reduce when stuck) ===
scheduler = ReduceLROnPlateau(optimizer, mode='min', factor=0.1, patience=5)
# Call: scheduler.step(val_loss)

# === Warmup + Cosine (common in modern training) ===
def get_lr(epoch, warmup_epochs=5, total_epochs=100, base_lr=1e-3):
    if epoch < warmup_epochs:
        return base_lr * (epoch + 1) / warmup_epochs  # Linear warmup
    else:
        # Cosine decay
        progress = (epoch - warmup_epochs) / (total_epochs - warmup_epochs)
        return base_lr * 0.5 * (1 + np.cos(np.pi * progress))
```

```
Learning Rate Schedules:

OneCycleLR:          CosineAnnealing:      StepLR:
   /\                ──╲                   ───┐
  /  \                  ╲                     │
 /    ╲                  ╲╲                   └───┐
/      ╲──                ╲╲╲──                    └───
```

#### Complete Training Loop

```python
import torch
import torch.nn as nn
from torch.cuda.amp import autocast, GradScaler  # Mixed precision
from tqdm import tqdm

def train_one_epoch(model, loader, criterion, optimizer, scheduler, scaler, device):
    model.train()
    running_loss = 0.0
    correct = 0
    total = 0
    
    for batch_idx, (images, labels) in enumerate(tqdm(loader)):
        images, labels = images.to(device), labels.to(device)
        
        # Mixed precision forward pass (2x speedup on modern GPUs)
        with autocast():
            outputs = model(images)
            loss = criterion(outputs, labels)
        
        # Backward pass with gradient scaling
        optimizer.zero_grad()
        scaler.scale(loss).backward()
        
        # Gradient clipping (prevents exploding gradients)
        scaler.unscale_(optimizer)
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        
        scaler.step(optimizer)
        scaler.update()
        
        # Update LR (for OneCycleLR, step every batch)
        scheduler.step()
        
        # Track metrics
        running_loss += loss.item()
        _, predicted = outputs.max(1)
        total += labels.size(0)
        correct += predicted.eq(labels).sum().item()
    
    return running_loss / len(loader), 100.0 * correct / total

@torch.no_grad()
def validate(model, loader, criterion, device):
    model.eval()
    running_loss = 0.0
    correct = 0
    total = 0
    
    for images, labels in loader:
        images, labels = images.to(device), labels.to(device)
        outputs = model(images)
        loss = criterion(outputs, labels)
        
        running_loss += loss.item()
        _, predicted = outputs.max(1)
        total += labels.size(0)
        correct += predicted.eq(labels).sum().item()
    
    return running_loss / len(loader), 100.0 * correct / total

# === Full training script ===
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = model.to(device)
criterion = nn.CrossEntropyLoss(label_smoothing=0.1)  # Smoothing helps!
optimizer = optim.AdamW(model.parameters(), lr=1e-3, weight_decay=0.05)
scheduler = OneCycleLR(optimizer, max_lr=1e-3, epochs=30, 
                       steps_per_epoch=len(train_loader))
scaler = GradScaler()  # For mixed precision

best_acc = 0.0
for epoch in range(30):
    train_loss, train_acc = train_one_epoch(
        model, train_loader, criterion, optimizer, scheduler, scaler, device)
    val_loss, val_acc = validate(model, val_loader, criterion, device)
    
    print(f"Epoch {epoch+1}: Train Loss={train_loss:.4f}, Acc={train_acc:.1f}% | "
          f"Val Loss={val_loss:.4f}, Acc={val_acc:.1f}%")
    
    # Save best model
    if val_acc > best_acc:
        best_acc = val_acc
        torch.save(model.state_dict(), 'best_model.pth')
        print(f"  → Saved best model (acc: {best_acc:.1f}%)")
```

#### Essential Training Tricks

| Trick | Effect | Implementation |
|-------|--------|----------------|
| Label Smoothing | Prevents overconfidence | `CrossEntropyLoss(label_smoothing=0.1)` |
| Mixed Precision (AMP) | 2× speed, less memory | `autocast()` + `GradScaler` |
| Gradient Clipping | Prevents exploding gradients | `clip_grad_norm_(params, 1.0)` |
| Weight Decay | L2 regularization | `AdamW(weight_decay=0.01)` |
| EMA (Exponential Moving Average) | Smoother model, better generalization | Update shadow weights each step |
| Test-Time Augmentation (TTA) | +1-2% accuracy at inference | Average predictions over augmented copies |
| Stochastic Depth | Regularization for deep nets | Randomly drop entire residual blocks |

#### Early Stopping

```python
class EarlyStopping:
    """Stop training when validation metric stops improving."""
    def __init__(self, patience=7, min_delta=0.001):
        self.patience = patience
        self.min_delta = min_delta
        self.counter = 0
        self.best_score = None
        self.early_stop = False
    
    def __call__(self, val_metric):
        if self.best_score is None:
            self.best_score = val_metric
        elif val_metric < self.best_score + self.min_delta:
            self.counter += 1
            if self.counter >= self.patience:
                self.early_stop = True
        else:
            self.best_score = val_metric
            self.counter = 0
```

---

## 2.9 Evaluation Metrics

### What It Is
Metrics tell you HOW WELL your model is performing. Accuracy alone is misleading — you need the right metric for your problem.

### Core Metrics

**Accuracy:**
$$\text{Accuracy} = \frac{\text{Correct Predictions}}{\text{Total Predictions}}$$

**Precision (per class k):**
$$\text{Precision}_k = \frac{TP_k}{TP_k + FP_k}$$
"Of everything predicted as class k, how many actually were?"

**Recall (per class k):**
$$\text{Recall}_k = \frac{TP_k}{TP_k + FN_k}$$
"Of all actual class k samples, how many did we find?"

**F1 Score:**
$$F1_k = 2 \cdot \frac{\text{Precision}_k \cdot \text{Recall}_k}{\text{Precision}_k + \text{Recall}_k}$$

### When to Use Which Metric

| Scenario | Best Metric | Why |
|----------|-------------|-----|
| Balanced classes | Accuracy | All classes equally important |
| Imbalanced classes | F1-score (macro/weighted) | Accounts for class imbalance |
| Medical (miss = deadly) | Recall | Can't miss positive cases |
| Spam filter | Precision | Don't want to block real emails |
| Multi-class overall | Confusion matrix + per-class F1 | Full picture |

### Code Examples

```python
import torch
import numpy as np
from sklearn.metrics import (
    classification_report, confusion_matrix, 
    accuracy_score, f1_score, roc_auc_score
)
import seaborn as sns
import matplotlib.pyplot as plt

# === Collect predictions ===
all_preds = []
all_labels = []
all_probs = []

model.eval()
with torch.no_grad():
    for images, labels in val_loader:
        images = images.to(device)
        outputs = model(images)
        probs = torch.softmax(outputs, dim=1)
        _, preds = outputs.max(1)
        
        all_preds.extend(preds.cpu().numpy())
        all_labels.extend(labels.numpy())
        all_probs.extend(probs.cpu().numpy())

all_preds = np.array(all_preds)
all_labels = np.array(all_labels)
all_probs = np.array(all_probs)

# === Classification Report ===
class_names = ['cat', 'dog', 'bird', 'fish', 'horse']
print(classification_report(all_labels, all_preds, target_names=class_names))

# === Confusion Matrix ===
cm = confusion_matrix(all_labels, all_preds)
plt.figure(figsize=(8, 6))
sns.heatmap(cm, annot=True, fmt='d', cmap='Blues',
            xticklabels=class_names, yticklabels=class_names)
plt.xlabel('Predicted')
plt.ylabel('Actual')
plt.title('Confusion Matrix')
plt.show()

# === Top-K accuracy (used for ImageNet) ===
def topk_accuracy(output_probs, targets, k=5):
    """Percentage of samples where true label is in top-k predictions."""
    topk_preds = np.argsort(output_probs, axis=1)[:, -k:]  # Top-k indices
    correct = np.any(topk_preds == targets.reshape(-1, 1), axis=1)
    return correct.mean() * 100

top1 = topk_accuracy(all_probs, all_labels, k=1)
top5 = topk_accuracy(all_probs, all_labels, k=5)
print(f"Top-1 Accuracy: {top1:.2f}% | Top-5 Accuracy: {top5:.2f}%")
```

---

## 2.10 Practical End-to-End Project

### Complete Image Classification Pipeline

```python
"""
Complete image classification project: Classifying 5 types of flowers.
Demonstrates the full workflow from data loading to evaluation.
"""

import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, WeightedRandomSampler
from torchvision import transforms, datasets, models
from torch.cuda.amp import autocast, GradScaler
import numpy as np
from pathlib import Path

# ============= Configuration =============
class Config:
    data_dir = Path('data/flowers')
    num_classes = 5
    batch_size = 32
    num_workers = 4
    epochs = 30
    lr = 1e-3
    weight_decay = 0.01
    img_size = 224
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

cfg = Config()

# ============= Data Loading =============
train_transform = transforms.Compose([
    transforms.RandomResizedCrop(cfg.img_size, scale=(0.08, 1.0)),
    transforms.RandomHorizontalFlip(),
    transforms.RandAugment(num_ops=2, magnitude=9),  # AutoAugment-style
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
    transforms.RandomErasing(p=0.25),
])

val_transform = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(cfg.img_size),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
])

train_dataset = datasets.ImageFolder(cfg.data_dir / 'train', train_transform)
val_dataset = datasets.ImageFolder(cfg.data_dir / 'val', val_transform)

# Handle class imbalance with weighted sampling
class_counts = np.bincount([label for _, label in train_dataset.samples])
class_weights = 1.0 / class_counts
sample_weights = [class_weights[label] for _, label in train_dataset.samples]
sampler = WeightedRandomSampler(sample_weights, len(sample_weights))

train_loader = DataLoader(train_dataset, batch_size=cfg.batch_size, 
                          sampler=sampler, num_workers=cfg.num_workers, pin_memory=True)
val_loader = DataLoader(val_dataset, batch_size=cfg.batch_size,
                        shuffle=False, num_workers=cfg.num_workers, pin_memory=True)

# ============= Model =============
model = models.efficientnet_b0(weights='IMAGENET1K_V1')
model.classifier[1] = nn.Linear(model.classifier[1].in_features, cfg.num_classes)
model = model.to(cfg.device)

# ============= Training Setup =============
criterion = nn.CrossEntropyLoss(label_smoothing=0.1)
optimizer = optim.AdamW(model.parameters(), lr=cfg.lr, weight_decay=cfg.weight_decay)
scheduler = optim.lr_scheduler.OneCycleLR(
    optimizer, max_lr=cfg.lr, epochs=cfg.epochs, steps_per_epoch=len(train_loader))
scaler = GradScaler()

# ============= Training Loop =============
best_acc = 0.0
for epoch in range(cfg.epochs):
    # Train
    model.train()
    train_loss, correct, total = 0, 0, 0
    for images, labels in train_loader:
        images, labels = images.to(cfg.device), labels.to(cfg.device)
        
        with autocast():
            outputs = model(images)
            loss = criterion(outputs, labels)
        
        optimizer.zero_grad()
        scaler.scale(loss).backward()
        scaler.unscale_(optimizer)
        nn.utils.clip_grad_norm_(model.parameters(), 1.0)
        scaler.step(optimizer)
        scaler.update()
        scheduler.step()
        
        train_loss += loss.item()
        correct += (outputs.argmax(1) == labels).sum().item()
        total += labels.size(0)
    
    # Validate
    model.eval()
    val_loss, val_correct, val_total = 0, 0, 0
    with torch.no_grad():
        for images, labels in val_loader:
            images, labels = images.to(cfg.device), labels.to(cfg.device)
            outputs = model(images)
            val_loss += criterion(outputs, labels).item()
            val_correct += (outputs.argmax(1) == labels).sum().item()
            val_total += labels.size(0)
    
    train_acc = 100 * correct / total
    val_acc = 100 * val_correct / val_total
    print(f"Epoch {epoch+1}/{cfg.epochs} | "
          f"Train: {train_loss/len(train_loader):.4f}, {train_acc:.1f}% | "
          f"Val: {val_loss/len(val_loader):.4f}, {val_acc:.1f}%")
    
    if val_acc > best_acc:
        best_acc = val_acc
        torch.save(model.state_dict(), 'best_model.pth')

print(f"\nBest validation accuracy: {best_acc:.1f}%")
```

---

## 2.11 Common Failure Modes and Debugging

### Symptoms and Solutions

| Symptom | Likely Cause | Solution |
|---------|-------------|----------|
| Loss not decreasing | LR too high/low, bug in data pipeline | Reduce LR, verify data visually |
| Val acc stuck, train acc rising | Overfitting | More augmentation, dropout, less capacity |
| Both train/val stuck | Underfitting | Larger model, more epochs, higher LR |
| NaN loss | LR too high, data issue | Lower LR, check for corrupt images |
| Random accuracy (e.g., 20% for 5 classes) | Labels shuffled, wrong normalization | Verify data-label alignment |
| Works on val, fails in production | Distribution shift | Collect more representative data |

### Debugging Checklist

```python
# === 1. Verify data pipeline ===
# Always visualize a batch before training!
import matplotlib.pyplot as plt

def show_batch(loader, classes, n=8):
    images, labels = next(iter(loader))
    # Denormalize
    mean = torch.tensor([0.485, 0.456, 0.406]).view(3, 1, 1)
    std = torch.tensor([0.229, 0.224, 0.225]).view(3, 1, 1)
    images = images * std + mean
    
    fig, axes = plt.subplots(1, n, figsize=(20, 4))
    for i in range(n):
        axes[i].imshow(images[i].permute(1, 2, 0).clamp(0, 1))
        axes[i].set_title(classes[labels[i]])
        axes[i].axis('off')
    plt.show()

show_batch(train_loader, train_dataset.classes)

# === 2. Overfit on one batch (sanity check) ===
# If model can't memorize 1 batch → architecture/code bug
images, labels = next(iter(train_loader))
images, labels = images.to(device), labels.to(device)
model.train()
for i in range(100):
    outputs = model(images)
    loss = criterion(outputs, labels)
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    if i % 10 == 0:
        acc = (outputs.argmax(1) == labels).float().mean()
        print(f"Step {i}: Loss={loss.item():.4f}, Acc={acc.item():.2f}")
# Should reach 100% accuracy quickly

# === 3. Check for data leakage ===
# Ensure no overlap between train and val sets
train_files = set(Path(s[0]).name for s in train_dataset.samples)
val_files = set(Path(s[0]).name for s in val_dataset.samples)
overlap = train_files & val_files
assert len(overlap) == 0, f"Data leakage! {len(overlap)} files in both sets"
```

---

## Interview Questions

1. **Q: Why use transfer learning instead of training from scratch?**
   - A: Pretrained models have learned universal visual features (edges, textures, shapes) from millions of images. Transfer learning leverages these, requiring less data, less compute, and often achieving better accuracy. Only train from scratch when your domain is radically different (e.g., medical imaging with no natural image similarity) AND you have millions of labeled images.

2. **Q: Explain the vanishing gradient problem and how ResNet solves it.**
   - A: In deep networks, gradients shrink exponentially as they backpropagate through many layers (each multiplication by values < 1). ResNet's skip connections provide a "gradient highway" — gradients flow directly through the addition operation, bypassing potentially problematic layers.

3. **Q: What's the difference between MixUp and CutMix?**
   - A: MixUp blends entire images linearly (creates ghost-like overlays). CutMix cuts a rectangular patch from one image and pastes it on another. CutMix preserves local structure better, leading to more informative training signals and typically better performance.

4. **Q: When would you use AdamW over SGD?**
   - A: AdamW for: faster convergence, transfer learning, smaller datasets, less hyperparameter tuning needed. SGD+momentum for: larger datasets, when you can afford extensive LR tuning, often achieves slightly higher final accuracy on ImageNet-scale training.

5. **Q: Your model gets 95% accuracy on validation but fails in production. Why?**
   - A: Domain shift — validation data doesn't represent production conditions. Causes: different lighting, camera angles, image quality, class distribution, new categories. Solution: collect production-representative test set, monitor predictions in production, continuous data collection.

6. **Q: Explain label smoothing and why it helps.**
   - A: Instead of hard labels [0, 0, 1, 0], use soft labels [0.033, 0.033, 0.9, 0.033]. This prevents the model from being overconfident, improves calibration, and acts as regularization. Formula: $y_{smooth} = (1-\epsilon) \cdot y_{hot} + \epsilon / K$ where $\epsilon$ is typically 0.1.

7. **Q: How does batch size affect training?**
   - A: Larger batches → more stable gradients, can use higher LR, better GPU utilization, but may converge to sharper minima (worse generalization). Smaller batches → more noise (regularizing effect), but slower wall-clock time. Linear scaling rule: if you double batch size, double LR.

8. **Q: What is test-time augmentation (TTA)?**
   - A: At inference, create multiple augmented versions of the input (flips, crops), run each through the model, average the predictions. Typically gives +1-2% accuracy improvement at the cost of N× inference time.

---

## Quick Reference

### Model Selection Cheat Sheet

| Your Situation | Model Choice | Reason |
|---------------|-------------|--------|
| Quick prototype | ResNet-18 + pretrained | Fast, reliable baseline |
| Best accuracy (no constraint) | EfficientNet-B4/B5 or ConvNeXt | Top performance |
| Mobile deployment | MobileNetV3-Small | Designed for edge |
| < 500 training images | ResNet-18, freeze backbone | Minimal overfitting |
| 5K-50K training images | ResNet-50/EfficientNet-B0, fine-tune | Good balance |
| > 100K training images | Train from scratch possible | Enough data |

### Training Hyperparameters (Starting Points)

| Parameter | Value | Notes |
|-----------|-------|-------|
| Optimizer | AdamW | Default choice |
| Learning Rate | 1e-3 (head), 1e-5 (backbone) | Discriminative LRs |
| Weight Decay | 0.01–0.05 | Higher for larger models |
| Batch Size | 32–64 | Adjust to GPU memory |
| Epochs | 30–100 | Use early stopping |
| Label Smoothing | 0.1 | Almost always helps |
| Dropout | 0.2–0.5 | In classifier head |
| Scheduler | OneCycleLR or CosineAnnealing | Both work well |

### Common Transforms

| Phase | Transforms |
|-------|-----------|
| Train | RandomResizedCrop → HFlip → ColorJitter → Normalize → RandomErasing |
| Val/Test | Resize(256) → CenterCrop(224) → Normalize |
| TTA | Multiple crops + flips → Average predictions |
