# 📘 Chapter 05: EfficientDet — Google's Scalable Object Detection

> **Goal**: Understand Google's approach to object detection that achieves better accuracy with FEWER parameters and computations than YOLO or Faster R-CNN, using smart scaling and a novel feature fusion technique called BiFPN.

---

## 📑 Table of Contents

1. [What is EfficientDet & Why It Matters](#1-what-is-efficientdet--why-it-matters)
2. [The EfficientNet Backbone (Foundation)](#2-the-efficientnet-backbone-foundation)
3. [BiFPN — Bidirectional Feature Pyramid Network](#3-bifpn--bidirectional-feature-pyramid-network)
4. [Compound Scaling — The Key Innovation](#4-compound-scaling--the-key-innovation)
5. [EfficientDet Architecture (Complete)](#5-efficientdet-architecture-complete)
6. [EfficientDet Model Family (D0 to D7)](#6-efficientdet-model-family-d0-to-d7)
7. [Training EfficientDet](#7-training-efficientdet)
8. [Inference & Code Examples](#8-inference--code-examples)
9. [EfficientDet vs YOLO vs Faster R-CNN](#9-efficientdet-vs-yolo-vs-faster-r-cnn)
10. [Deployment & Optimization](#10-deployment--optimization)
11. [Real-World Use Cases](#11-real-world-use-cases)
12. [EfficientDet Limitations & Successors](#12-efficientdet-limitations--successors)

---

## 1. What is EfficientDet & Why It Matters

### 💡 The Core Idea

> "Instead of making models bigger randomly, SCALE everything (resolution, depth, width) in a PRINCIPLED way to get maximum accuracy per computation."

### 🏢 Who Uses EfficientDet

| Company | Use Case |
|---------|----------|
| **Google** | Google Lens, Google Photos |
| **Waymo** | Self-driving car perception |
| **Google Cloud** | AutoML Vision, Vertex AI |
| **Qualcomm** | On-device AI (Snapdragon) |
| **MediaTek** | Mobile AI chips |
| **Various robotics** | Efficient detection on limited hardware |

### The Problem EfficientDet Solves

```
Before EfficientDet (2019):
  Want more accuracy? → Make model BIGGER (more layers, wider)
  Want faster?        → Make model SMALLER (less layers, narrower)
  
  Problem: Random scaling is INEFFICIENT!
           You add 10× more computation but only get +2% accuracy.

EfficientDet's insight:
  Scale backbone (depth/width), neck (features), and resolution TOGETHER
  in a BALANCED way → Get maximum accuracy per FLOP!

Result: 
  EfficientDet-D0: Same accuracy as YOLOv3 with 28× fewer FLOPs!
  EfficientDet-D7: Best accuracy with fewer FLOPs than prior SOTA!
```

### Paper Info

```
Title: "EfficientDet: Scalable and Efficient Object Detection"
Authors: Mingxing Tan, Ruoming Pang, Quoc V. Le (Google Brain)
Published: CVPR 2020
Key contributions:
  1. BiFPN (Bidirectional Feature Pyramid Network)
  2. Compound scaling for object detection
  3. New SOTA: better accuracy at 4-9× fewer FLOPs
```

---

## 2. The EfficientNet Backbone (Foundation)

### 💡 You MUST Understand EfficientNet to Understand EfficientDet

> **EfficientNet** is the backbone (feature extractor) used in EfficientDet. It was the first network to systematically study HOW to scale neural networks efficiently.

### The Scaling Problem

```
Traditional scaling approaches (pick ONE):
  1. Make it DEEPER (more layers):     ResNet-18 → ResNet-152
  2. Make it WIDER (more channels):     64 filters → 256 filters
  3. Use higher RESOLUTION:             224×224 → 512×512

Problem: Each alone gives diminishing returns!

EfficientNet's solution: COMPOUND SCALING
  → Scale ALL THREE dimensions TOGETHER in a balanced ratio!
```

### Compound Scaling Formula

```
depth:      d = α^φ
width:      w = β^φ  
resolution: r = γ^φ

Where:
  φ (phi) = compound coefficient (user-defined: 0, 1, 2, ... 7)
  α, β, γ = constants found by neural architecture search
  
  For EfficientNet: α=1.2, β=1.1, γ=1.15
  Constraint: α × β² × γ² ≈ 2 (doubles FLOPs for each φ increment)

Example:
  φ=0: EfficientNet-B0 (baseline, 5.3M params)
  φ=1: EfficientNet-B1 (7.8M params, +1.2% accuracy)
  φ=2: EfficientNet-B2 (9.2M params)
  ...
  φ=7: EfficientNet-B7 (66M params, SOTA on ImageNet!)
```

### EfficientNet Family

| Model | Params | Top-1 Acc | FLOPs | Resolution |
|-------|--------|-----------|-------|-----------|
| EfficientNet-B0 | 5.3M | 77.1% | 0.39B | 224 |
| EfficientNet-B1 | 7.8M | 79.1% | 0.70B | 240 |
| EfficientNet-B2 | 9.2M | 80.1% | 1.0B | 260 |
| EfficientNet-B3 | 12M | 81.6% | 1.8B | 300 |
| EfficientNet-B4 | 19M | 82.9% | 4.2B | 380 |
| EfficientNet-B5 | 30M | 83.6% | 9.9B | 456 |
| EfficientNet-B6 | 43M | 84.0% | 19B | 528 |
| EfficientNet-B7 | 66M | 84.3% | 37B | 600 |

### MBConv Block (EfficientNet's Building Block)

```
Input
  │
  ▼
┌─────────────────────────┐
│  1×1 Conv (Expand)      │  ← Increase channels by expansion ratio (6×)
│  + BatchNorm + Swish    │
├─────────────────────────┤
│  3×3/5×5 Depthwise Conv │  ← Apply filter per channel (efficient!)
│  + BatchNorm + Swish    │
├─────────────────────────┤
│  Squeeze-and-Excitation │  ← Channel attention (which channels matter?)
│  (SE block)             │
├─────────────────────────┤
│  1×1 Conv (Project)     │  ← Reduce channels back
│  + BatchNorm            │
└─────────────────────────┘
  │
  + (Skip connection if input/output same size)
  │
  ▼
Output

Key: Depthwise Separable Conv = 8-9× fewer operations than standard conv!
```

### Why EfficientNet as Backbone?

```
Comparison at similar accuracy (~80% ImageNet Top-1):

Model           Params    FLOPs     Accuracy
ResNet-50       25M       4.1B      76.1%
ResNet-152      60M       11.5B     77.8%
EfficientNet-B1 7.8M      0.70B     79.1%  ← 8× fewer params, 6× fewer FLOPs!

EfficientNet gives MORE accuracy with MUCH less computation!
That's why it's perfect for detection (saves compute budget for neck+head).
```

---

## 3. BiFPN — Bidirectional Feature Pyramid Network

### 💡 The Second Key Innovation

> Standard FPN flows information only TOP-DOWN. BiFPN adds BOTTOM-UP connections AND learnable weights to control how features are fused.

### Evolution of Feature Fusion

```
(a) FPN (2017):              Only top-down
    P7 ← P6 ← P5 ← P4 ← P3
    
(b) PANet (2018):            Top-down + Bottom-up
    P7 ← P6 ← P5 ← P4 ← P3
    P7 → P6 → P5 → P4 → P3
    
(c) NAS-FPN (2019):          Neural Architecture Search (irregular)
    Complex cross-connections found by AutoML
    
(d) BiFPN (2020):            Simplified bidirectional + WEIGHTED fusion
    Remove nodes with single input (they contribute less)
    Add weighted fusion (learn importance of each input)
    Stack multiple BiFPN layers (repeat for better fusion)
```

### BiFPN Architecture

```
Standard FPN (Top-down only):
  P3 ────────────────── P3_out
        ↑
  P4 ───┘──────────── P4_out
        ↑
  P5 ───┘──────────── P5_out
        ↑
  P6 ───┘──────────── P6_out
        ↑
  P7 ──────────────── P7_out


BiFPN (Bidirectional + Weighted):

 INPUT          TOP-DOWN           BOTTOM-UP          OUTPUT
  P3 ─────────────────────────────── P3_td ──────── P3_out
       ╲                            ╱     ╲
  P4 ────── P4_td ──────── P4_td ──────── P4_out
       ╲    ╱                      ╱     ╲
  P5 ────── P5_td ──────── P5_td ──────── P5_out
       ╲    ╱                      ╱     ╲
  P6 ────── P6_td ──────── P6_td ──────── P6_out
       ╲    ╱                      ╱
  P7 ────── P7_td ────────────────────── P7_out

Key differences from FPN:
  1. Bidirectional (info flows BOTH ways)
  2. Weighted fusion (learned weights per connection)
  3. Stacked (repeat BiFPN layer multiple times: 3-7 times)
  4. Removed single-input nodes (efficiency)
```

### Weighted Feature Fusion

```
Standard fusion:     Output = Feature_1 + Feature_2 + Feature_3  (equal weight)

BiFPN weighted fusion (Fast Normalized):

                     w₁·Feature_1 + w₂·Feature_2 + w₃·Feature_3
  Output = ──────────────────────────────────────────────────────
                            w₁ + w₂ + w₃ + ε

Where:
  w₁, w₂, w₃ = LEARNABLE weights (initialized to 1)
  ε = 0.0001 (for numerical stability)
  
  After training, the network LEARNS which connections matter most!
  
  Example: For detecting small objects, w(high_res_feature) becomes larger.
           For large objects, w(low_res_feature) becomes larger.
```

### 📝 Code: BiFPN Layer (Simplified)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class BiFPNLayer(nn.Module):
    """Single BiFPN layer with weighted feature fusion."""
    
    def __init__(self, num_channels=64, num_levels=5):
        super().__init__()
        self.num_levels = num_levels
        
        # Learnable fusion weights (one per input at each fusion node)
        # Top-down weights
        self.w_td = nn.ParameterList([
            nn.Parameter(torch.ones(2)) for _ in range(num_levels - 1)
        ])
        # Bottom-up weights
        self.w_bu = nn.ParameterList([
            nn.Parameter(torch.ones(3)) for _ in range(num_levels - 1)
        ])
        
        # Depthwise separable convs for each level
        self.td_convs = nn.ModuleList([
            self._make_conv(num_channels) for _ in range(num_levels - 1)
        ])
        self.bu_convs = nn.ModuleList([
            self._make_conv(num_channels) for _ in range(num_levels - 1)
        ])
        
        self.epsilon = 1e-4
    
    def _make_conv(self, channels):
        return nn.Sequential(
            nn.Conv2d(channels, channels, 3, padding=1, groups=channels),  # Depthwise
            nn.Conv2d(channels, channels, 1),  # Pointwise
            nn.BatchNorm2d(channels),
            nn.SiLU(inplace=True)
        )
    
    def _weighted_fusion(self, features, weights):
        """Fast normalized weighted fusion."""
        w = F.relu(weights)  # Ensure positive
        w_sum = w.sum() + self.epsilon
        
        fused = sum(w[i] * f for i, f in enumerate(features))
        return fused / w_sum
    
    def forward(self, features):
        """
        features: list of [P3, P4, P5, P6, P7] (low to high level)
        """
        # Top-down pathway
        td_features = [features[-1]]  # Start from P7 (deepest)
        for i in range(self.num_levels - 2, -1, -1):
            # Upsample deeper feature
            upsampled = F.interpolate(td_features[0], size=features[i].shape[2:])
            # Weighted fusion
            fused = self._weighted_fusion([features[i], upsampled], self.w_td[i])
            td_features.insert(0, self.td_convs[i](fused))
        
        # Bottom-up pathway
        bu_features = [td_features[0]]  # Start from P3 (shallowest)
        for i in range(1, self.num_levels):
            # Downsample shallower feature
            downsampled = F.adaptive_avg_pool2d(bu_features[-1], td_features[i].shape[2:])
            # Weighted fusion (3 inputs: original + top-down + bottom-up)
            fused = self._weighted_fusion(
                [features[i], td_features[i], downsampled], 
                self.w_bu[i-1]
            )
            bu_features.append(self.bu_convs[i-1](fused))
        
        return bu_features


class BiFPN(nn.Module):
    """Stack multiple BiFPN layers."""
    
    def __init__(self, num_channels=64, num_layers=3, num_levels=5):
        super().__init__()
        self.layers = nn.ModuleList([
            BiFPNLayer(num_channels, num_levels) for _ in range(num_layers)
        ])
    
    def forward(self, features):
        for layer in self.layers:
            features = layer(features)
        return features
```

### BiFPN vs Other Necks

| Neck | Connections | Weighted? | Layers | AP (COCO) | Params |
|------|-------------|-----------|--------|-----------|--------|
| FPN | Top-down only | No | 1 | 37.0 | Baseline |
| PANet | Top-down + Bottom-up | No | 1 | 38.5 | +15% |
| NAS-FPN | AutoML-found | No | 7 | 39.9 | +3× |
| **BiFPN** | Bidirectional | **Yes** | 3-7 | **40.3** | +3% |

> 💡 BiFPN gets better accuracy than NAS-FPN with much fewer parameters, because weighted fusion lets the network LEARN what matters!

---

## 4. Compound Scaling — The Key Innovation

### 💡 "Scale Everything Together, Not Independently"

> Just like EfficientNet scales backbone depth/width/resolution together, EfficientDet scales the ENTIRE detector: backbone + BiFPN + head + resolution.

### What Gets Scaled

```
As you go from EfficientDet-D0 → D7:

┌────────────────────────────────────────────────────────┐
│  Component        │  Scaling                           │
├───────────────────┼────────────────────────────────────┤
│  Backbone         │  EfficientNet-B0 → B6             │
│  BiFPN channels   │  64 → 384                         │
│  BiFPN layers     │  3 → 8 (repeated stacking)        │
│  Box/Class head   │  3 → 5 conv layers                │
│  Input resolution │  512 → 1536                       │
└────────────────────────────────────────────────────────┘

Scaling is NOT random — it follows a formula tied to φ (phi)!
```

### Scaling Rules

```python
# EfficientDet compound scaling rules
def get_efficientdet_config(phi):
    """
    phi: 0-7 (compound coefficient)
    """
    configs = {
        # phi: (backbone, resolution, bifpn_channels, bifpn_layers, head_layers)
        0: ('B0', 512,  64,  3, 3),
        1: ('B1', 640,  88,  4, 3),
        2: ('B2', 768,  112, 5, 3),
        3: ('B3', 896,  160, 6, 4),
        4: ('B4', 1024, 224, 7, 4),
        5: ('B5', 1280, 288, 7, 4),
        6: ('B6', 1280, 384, 8, 5),
        7: ('B6', 1536, 384, 8, 5),  # B6 backbone reused
    }
    return configs[phi]
```

### Visual: How Compound Scaling Works

```
EfficientDet-D0 (Small, Fast):
┌─────────┐   ┌───────┐   ┌────┐
│ B0      │──▶│BiFPN  │──▶│Head│──▶ Detect
│(small)  │   │(3 lyr)│   │(3) │
└─────────┘   └───────┘   └────┘
Input: 512×512, BiFPN: 64 channels

EfficientDet-D4 (Balanced):
┌──────────────┐   ┌──────────────┐   ┌──────┐
│ B4           │──▶│ BiFPN        │──▶│ Head │──▶ Detect
│ (medium)     │   │ (7 layers!)  │   │ (4)  │
└──────────────┘   └──────────────┘   └──────┘
Input: 1024×1024, BiFPN: 224 channels

EfficientDet-D7 (Maximum Accuracy):
┌──────────────────┐   ┌───────────────────┐   ┌────────┐
│ B6              │──▶│ BiFPN             │──▶│  Head  │──▶ Detect
│ (large)         │   │ (8 layers!!!)     │   │  (5)   │
└──────────────────┘   └───────────────────┘   └────────┘
Input: 1536×1536, BiFPN: 384 channels
```

---

## 5. EfficientDet Architecture (Complete)

### Full Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                      EFFICIENTDET ARCHITECTURE                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  INPUT IMAGE (e.g., 512×512×3 for D0)                                │
│       │                                                                │
│       ▼                                                                │
│  ┌────────────────────────────────────────┐                           │
│  │          BACKBONE                       │                           │
│  │       EfficientNet-B0/B1/.../B6         │                           │
│  │                                         │                           │
│  │  Extracts multi-scale features:         │                           │
│  │    P3 (64×64), P4 (32×32), P5 (16×16) │                           │
│  │    P6 (8×8), P7 (4×4)                 │                           │
│  └───────────────┬────────────────────────┘                           │
│                  │ [P3, P4, P5, P6, P7]                               │
│                  ▼                                                      │
│  ┌────────────────────────────────────────┐                           │
│  │            BiFPN (NECK)                 │                           │
│  │                                         │                           │
│  │  • Bidirectional feature fusion         │                           │
│  │  • Weighted connections (learnable)     │                           │
│  │  • Stacked 3-8 times (depending on D#) │                           │
│  │  • Depthwise separable convolutions     │                           │
│  │                                         │                           │
│  │  [P3, P4, P5, P6, P7] → [P3', P4', P5', P6', P7']              │
│  └───────────────┬────────────────────────┘                           │
│                  │                                                      │
│          ┌───────┴───────┐                                             │
│          ▼               ▼                                             │
│  ┌──────────────┐ ┌──────────────┐                                   │
│  │  CLASS HEAD  │ │  BOX HEAD    │  (Shared across all levels!)       │
│  │              │ │              │                                     │
│  │  3-5 Conv   │ │  3-5 Conv    │  + Depthwise separable             │
│  │  layers     │ │  layers      │  + BatchNorm + Swish               │
│  │  → K×A cls  │ │  → 4×A boxes │  A = num_anchors (9)              │
│  │  (per level)│ │  (per level) │  K = num_classes                   │
│  └──────────────┘ └──────────────┘                                   │
│                                                                        │
│  POST-PROCESSING:                                                     │
│  → Confidence threshold → NMS → Final detections                      │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### Anchors in EfficientDet

```
At each feature level (P3-P7), EfficientDet uses:
  3 aspect ratios: {1:1, 1.4:1, 1:1.4} (≈ {1:2, 1:1, 2:1} equivalent)
  3 scales: {2^0, 2^(1/3), 2^(2/3)} per level
  
  Total: 3 × 3 = 9 anchors per feature map cell

Anchor sizes per level:
  P3: base_size = 32   → anchors: 32, 40, 50 (× 3 ratios)
  P4: base_size = 64   → anchors: 64, 80, 101
  P5: base_size = 128  → anchors: 128, 161, 203
  P6: base_size = 256  → anchors: 256, 322, 406
  P7: base_size = 512  → anchors: 512, 645, 812
```

### Loss Function

```
Total Loss = α × Focal Loss (classification) + Smooth L1 Loss (box regression)

Focal Loss (same as RetinaNet!):
  FL(p) = -α_t × (1-p_t)^γ × log(p_t)
  Where γ=1.5, α=0.25

Smooth L1 Loss:
  For box regression [dx, dy, dw, dh] offsets

Why Focal Loss?
  → EfficientDet is one-stage (like RetinaNet)
  → Same class imbalance problem (99.9% background)
  → Focal Loss down-weights easy negatives
```

---

## 6. EfficientDet Model Family (D0 to D7)

### Complete Model Specifications

| Model | Backbone | Input | BiFPN Ch | BiFPN Layers | Head Layers | Params | FLOPs | AP (COCO) |
|-------|----------|-------|----------|-------------|-------------|--------|-------|-----------|
| **D0** | B0 | 512 | 64 | 3 | 3 | 3.9M | 2.5B | 33.8% |
| **D1** | B1 | 640 | 88 | 4 | 3 | 6.6M | 6.1B | 39.6% |
| **D2** | B2 | 768 | 112 | 5 | 3 | 8.1M | 11B | 43.0% |
| **D3** | B3 | 896 | 160 | 6 | 4 | 12M | 25B | 45.8% |
| **D4** | B4 | 1024 | 224 | 7 | 4 | 21M | 55B | 49.4% |
| **D5** | B5 | 1280 | 288 | 7 | 4 | 34M | 135B | 50.7% |
| **D6** | B6 | 1280 | 384 | 8 | 5 | 52M | 226B | 51.7% |
| **D7** | B6 | 1536 | 384 | 8 | 5 | 52M | 325B | 52.2% |

### Efficiency Comparison (The WOW Moment)

```
At similar accuracy (~43% AP COCO):
  EfficientDet-D2:  11B FLOPs,  8.1M params
  YOLOv3-608:       141B FLOPs, 65M params    ← 13× more FLOPs!
  RetinaNet-R50:    97B FLOPs,  34M params    ← 9× more FLOPs!
  Faster R-CNN-R101: 172B FLOPs, 60M params   ← 16× more FLOPs!

EfficientDet does MORE with MUCH LESS!
```

### Visual: FLOPs vs Accuracy

```
AP (COCO)
52% │                                        ★ D7
    │                                   ★ D6
50% │                              ★ D5
    │                         ★ D4
48% │
    │
46% │                    ★ D3
    │
44% │               ★ D2
    │          ★ D1          ★ YOLOv4
42% │
    │
40% │     ★ D1
    │                        ★ RetinaNet-R101
38% │
    │
36% │ ★ D0
    │                   ★ Faster R-CNN
34% │ 
    └──────────────────────────────────────────────
    0   10B  25B  50B  100B  150B  200B  300B  FLOPs
    
    EfficientDet = Upper-left corner (best accuracy per FLOP!)
```

### Which EfficientDet to Choose?

```
Mobile/IoT (minimal compute):          → D0 (3.9M params, 33.8% AP)
Edge device (Jetson Nano):             → D1 (6.6M params, 39.6% AP)
Balanced (server, real-time needed):   → D2/D3 (8-12M params, 43-45% AP)
High accuracy (batch processing):      → D4/D5 (21-34M params, 49-50% AP)
Maximum accuracy (research/medical):   → D6/D7 (52M params, 51-52% AP)
```

---

## 7. Training EfficientDet

### 📝 Code: Training with Official TF Implementation

```python
# Google's official implementation (TensorFlow/TPU)
# Repository: google/automl/efficientdet

# Clone repo
# git clone https://github.com/google/automl
# cd automl/efficientdet

# Training command (TPU):
"""
python main.py \
    --mode=train_and_eval \
    --train_file_pattern=tfrecords/train* \
    --val_file_pattern=tfrecords/val* \
    --model_name=efficientdet-d0 \
    --model_dir=/tmp/efficientdet-d0 \
    --backbone_ckpt=efficientnet-b0 \
    --train_batch_size=64 \
    --eval_batch_size=64 \
    --num_epochs=300 \
    --hparams="num_classes=90,moving_average_decay=0.9998"
"""
```

### 📝 Code: Training with PyTorch (Yet-Another-EfficientDet)

```python
# Most popular PyTorch implementation
# pip install yet-another-efficientdet-pytorch
# OR use: https://github.com/zylo117/Yet-Another-EfficientDet-Pytorch

import torch
from torch.utils.data import DataLoader
from backbone import EfficientDetBackbone
from efficientdet.dataset import CocoDataset
from efficientdet.loss import FocalLoss

# Load model
model = EfficientDetBackbone(
    compound_coef=0,            # D0
    num_classes=len(obj_list),
    ratios=[(1.0, 1.0), (1.4, 0.7), (0.7, 1.4)],
    scales=[2**0, 2**(1/3), 2**(2/3)]
)

# Load pre-trained weights
model.load_state_dict(torch.load('efficientdet-d0.pth'))

# Training setup
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4)
scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(optimizer, patience=3)

# Training loop
for epoch in range(num_epochs):
    model.train()
    for images, annotations in train_loader:
        optimizer.zero_grad()
        
        # Forward pass
        cls_loss, reg_loss = model(images, annotations)
        loss = cls_loss + reg_loss
        
        # Backward pass
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), 0.1)
        optimizer.step()
    
    # Validate
    model.eval()
    # ... evaluate on val set
    scheduler.step(val_loss)
```

### 📝 Code: Training with Ultralytics (Easiest!)

```python
# Ultralytics also supports EfficientDet-style models via their ecosystem
# But the most direct way is using their YOLO which has similar BiFPN concepts

# For pure EfficientDet, use timm + custom training:
import timm
import torch

# Load EfficientNet backbone from timm
backbone = timm.create_model('efficientnet_b0', pretrained=True, features_only=True)

# Get feature map sizes
dummy = torch.randn(1, 3, 512, 512)
features = backbone(dummy)
for i, f in enumerate(features):
    print(f"Level {i}: {f.shape}")
# Level 0: [1, 16, 256, 256]   (stride 2)
# Level 1: [1, 24, 128, 128]   (stride 4)  → P3
# Level 2: [1, 40, 64, 64]     (stride 8)  → P4
# Level 3: [1, 112, 32, 32]    (stride 16) → P5
# Level 4: [1, 320, 16, 16]    (stride 32) → P6
```

### Training Hyperparameters (Recommended)

| Hyperparameter | D0-D2 | D3-D5 | D6-D7 |
|----------------|-------|-------|-------|
| Batch size | 128 | 64 | 32 |
| Learning rate | 0.08 | 0.04 | 0.02 |
| Optimizer | SGD + momentum 0.9 | Same | Same |
| LR Schedule | Cosine decay | Same | Same |
| Epochs | 300 | 300 | 300 |
| Weight decay | 4e-5 | 4e-5 | 4e-5 |
| Augmentation | AutoAugment | Same | Same |
| EMA | 0.9998 | 0.9998 | 0.9998 |

---

## 8. Inference & Code Examples

### 📝 Code: Inference with Pre-trained EfficientDet (TF)

```python
# Using TensorFlow Hub
import tensorflow as tf
import tensorflow_hub as hub
import numpy as np
from PIL import Image

# Load model from TF Hub
detector = hub.load(
    "https://tfhub.dev/tensorflow/efficientdet/d0/1"
)

# Prepare image
image = Image.open("test.jpg")
image_np = np.array(image)
input_tensor = tf.convert_to_tensor(image_np)
input_tensor = input_tensor[tf.newaxis, ...]  # Add batch dim

# Run detection
results = detector(input_tensor)

# Process results
boxes = results['detection_boxes'][0].numpy()      # [y1, x1, y2, x2] normalized
scores = results['detection_scores'][0].numpy()     # Confidence scores
classes = results['detection_classes'][0].numpy()    # Class IDs (1-indexed)

# Filter by confidence
threshold = 0.5
for i, score in enumerate(scores):
    if score > threshold:
        y1, x1, y2, x2 = boxes[i]
        cls = int(classes[i])
        print(f"Class {cls}: {score:.3f} at [{x1:.3f}, {y1:.3f}, {x2:.3f}, {y2:.3f}]")
```

### 📝 Code: Inference with PyTorch (zylo117)

```python
# pip install yet-another-efficientdet-pytorch

import torch
import cv2
import numpy as np
from backbone import EfficientDetBackbone
from efficientdet.utils import BBoxTransform, ClipBoxes
from utils.utils import preprocess, invert_affine, postprocess

# Parameters
compound_coef = 0  # D0
threshold = 0.5
iou_threshold = 0.5
obj_list = ['person', 'bicycle', 'car', ...]  # COCO classes

# Load model
model = EfficientDetBackbone(
    compound_coef=compound_coef,
    num_classes=len(obj_list)
)
model.load_state_dict(torch.load(f'efficientdet-d{compound_coef}.pth'))
model.eval()
model.cuda()

# Prepare image
image = cv2.imread('test.jpg')
ori_imgs, framed_imgs, framed_metas = preprocess(
    image, max_size=512  # Input size for D0
)

# Run inference
with torch.no_grad():
    x = torch.stack([torch.from_numpy(fi).cuda() for fi in framed_imgs], 0)
    x = x.permute(0, 3, 1, 2).float()  # HWC → CHW
    
    features, regression, classification, anchors = model(x)
    
    # Post-process
    out = postprocess(
        x, anchors, regression, classification,
        threshold, iou_threshold
    )

# Visualize
for i, (roi, cls_id, score) in enumerate(zip(out[0]['rois'], out[0]['class_ids'], out[0]['scores'])):
    x1, y1, x2, y2 = roi.astype(int)
    label = f"{obj_list[cls_id]}: {score:.2f}"
    cv2.rectangle(image, (x1, y1), (x2, y2), (0, 255, 0), 2)
    cv2.putText(image, label, (x1, y1-10), cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 255, 0), 2)

cv2.imshow("EfficientDet", image)
cv2.waitKey(0)
```

### 📝 Code: Batch Inference for Production

```python
import torch
import time

model = load_efficientdet('d2')
model.eval().cuda()

# Batch processing
images = load_batch("images/", batch_size=32)

start = time.time()
with torch.no_grad():
    for batch in images:
        predictions = model(batch.cuda())
elapsed = time.time() - start

total_images = len(images) * 32
print(f"Processed {total_images} images in {elapsed:.2f}s")
print(f"Speed: {total_images/elapsed:.1f} images/sec")
```

---

## 9. EfficientDet vs YOLO vs Faster R-CNN

### Head-to-Head Comparison

| Aspect | EfficientDet | YOLOv8 | Faster R-CNN |
|--------|-------------|--------|-------------|
| **Type** | One-stage, anchor-based | One-stage, anchor-free | Two-stage |
| **Backbone** | EfficientNet | CSPDarknet | ResNet/ResNeXt |
| **Neck** | BiFPN (weighted) | PANet | FPN |
| **Key Innovation** | Compound scaling | Speed + UX | RPN |
| **Best At** | Efficiency (acc/FLOP) | Speed + ease of use | Accuracy |
| **Mobile-friendly** | ✅ (D0-D2) | ✅ (nano/small) | ❌ |
| **Ease of Use** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Community** | Medium | Massive | Large |
| **Framework** | TF (official), PyTorch (3rd party) | PyTorch (Ultralytics) | PyTorch (torchvision/Detectron2) |

### Accuracy vs Speed at Similar Compute

```
At ~10 GFLOPs:
  EfficientDet-D1: 39.6% AP
  YOLOv8s:         44.9% AP     ← YOLOv8 wins (newer, more tricks)
  
At ~50 GFLOPs:
  EfficientDet-D4: 49.4% AP
  YOLOv8l:         52.9% AP     ← YOLOv8 still wins

BUT: EfficientDet was published in 2020, YOLOv8 in 2023.
     At the TIME of publication, EfficientDet was SOTA.
     The scaling PRINCIPLES (BiFPN, compound scaling) 
     influenced later designs including YOLOv8's PANet!
```

### When to Choose EfficientDet Over YOLO

```
Choose EfficientDet when:
✅ Using TensorFlow ecosystem (TF Serving, TF Lite, Coral)
✅ Need efficient scaling for custom hardware constraints
✅ Google Cloud / Vertex AI deployment
✅ Working with Google's AutoML Vision (uses EfficientDet internally)
✅ Need TFLite-optimized mobile model
✅ Academic research comparing with 2020 era detectors

Choose YOLO when:
✅ Need easiest developer experience (95% of cases)
✅ PyTorch ecosystem
✅ Real-time video processing
✅ Quick prototyping
✅ Multi-task (detect + segment + pose)
✅ Active community support
```

---

## 10. Deployment & Optimization

### TFLite Deployment (Mobile/Edge)

```python
# Convert EfficientDet to TFLite
import tensorflow as tf

# Load saved model
model = tf.saved_model.load('efficientdet-d0-saved-model/')

# Convert to TFLite
converter = tf.lite.TFLiteConverter.from_saved_model('efficientdet-d0-saved-model/')
converter.optimizations = [tf.lite.Optimize.DEFAULT]  # Quantization
converter.target_spec.supported_types = [tf.float16]  # FP16

tflite_model = converter.convert()

# Save
with open('efficientdet_d0.tflite', 'wb') as f:
    f.write(tflite_model)

print(f"Model size: {len(tflite_model) / 1024 / 1024:.1f} MB")
```

### Google Coral (Edge TPU) Deployment

```bash
# EfficientDet is OPTIMIZED for Google Coral Edge TPU!
# Convert to Edge TPU format:

# Step 1: Convert to TFLite with full integer quantization
python convert_to_tflite.py \
    --model_name=efficientdet-d0 \
    --ckpt_path=efficientdet-d0/ \
    --quantize=True

# Step 2: Compile for Edge TPU
edgetpu_compiler efficientdet_d0_quant.tflite

# Result: efficientdet_d0_quant_edgetpu.tflite
# Runs at 30+ FPS on $60 Coral USB Accelerator!
```

### 📝 Code: Running on Coral Edge TPU

```python
import tflite_runtime.interpreter as tflite
import numpy as np
import cv2

# Load Edge TPU model
interpreter = tflite.Interpreter(
    model_path='efficientdet_d0_edgetpu.tflite',
    experimental_delegates=[tflite.load_delegate('libedgetpu.so.1')]
)
interpreter.allocate_tensors()

# Get input/output details
input_details = interpreter.get_input_details()
output_details = interpreter.get_output_details()

# Prepare image
image = cv2.imread('test.jpg')
input_size = input_details[0]['shape'][1:3]  # e.g., (512, 512)
resized = cv2.resize(image, tuple(input_size))
input_data = np.expand_dims(resized, axis=0).astype(np.uint8)

# Run inference
interpreter.set_tensor(input_details[0]['index'], input_data)
interpreter.invoke()

# Get results
boxes = interpreter.get_tensor(output_details[0]['index'])[0]
classes = interpreter.get_tensor(output_details[1]['index'])[0]
scores = interpreter.get_tensor(output_details[2]['index'])[0]

# Process detections
for i in range(len(scores)):
    if scores[i] > 0.5:
        y1, x1, y2, x2 = boxes[i]
        print(f"Class {int(classes[i])}: {scores[i]:.2f}")
```

### ONNX Export & Inference

```python
# Export to ONNX
import torch

model = load_efficientdet_pytorch('d2')
model.eval()

dummy_input = torch.randn(1, 3, 768, 768)
torch.onnx.export(
    model, dummy_input, "efficientdet_d2.onnx",
    opset_version=11,
    input_names=['input'],
    output_names=['boxes', 'scores', 'classes']
)

# Run with ONNX Runtime
import onnxruntime as ort

session = ort.InferenceSession("efficientdet_d2.onnx")
results = session.run(None, {"input": input_array})
```

---

## 11. Real-World Use Cases

### 🏢 Case Study 1: Google Lens

```
App: Google Lens (available on every Android phone)
Task: Identify objects, translate text, find products
Model: EfficientDet-Lite (optimized for mobile)
Why EfficientDet: 
  - Runs on-device (no internet needed)
  - TFLite optimized  
  - Low latency on Qualcomm/MediaTek chips
  - Battery efficient
Scale: 1B+ Android devices
```

### 🏢 Case Study 2: Waymo Self-Driving

```
Task: Real-time perception for autonomous vehicles
Models: EfficientDet variants (multi-camera, multi-task)
Why: Excellent accuracy per compute budget
     Multiple cameras = need EFFICIENT models
     Safety-critical = need HIGH accuracy
     On-vehicle compute is limited
Pipeline:
  8 cameras × EfficientDet → 3D fusion → Planning → Driving
```

### 🏢 Case Study 3: Retail (Smart Shelves)

```
Task: Detect products on shelves, identify gaps
Hardware: Qualcomm Snapdragon (edge AI)
Model: EfficientDet-D0/D1 with TFLite
Why: Small enough for edge SoC, accurate enough for retail
Accuracy: 92% product detection
Speed: 25 FPS on Snapdragon 888
```

### 🏢 Case Study 4: Medical Imaging

```
Task: Detect polyps in colonoscopy video
Model: EfficientDet-D3 fine-tuned on medical data
Why: High accuracy needed (medical), reasonable speed for real-time video
Setup:
  - 5000 labeled colonoscopy frames
  - Transfer learning from COCO
  - Fine-tuned for 50 epochs
  - 30 FPS inference on GTX 1080
Result: 96% sensitivity, 2% false positive rate
```

---

## 12. EfficientDet Limitations & Successors

### ⚠️ Limitations

| Limitation | Details |
|-----------|---------|
| **Slower than YOLO** | At same accuracy, newer YOLO is faster |
| **TF-centric** | Official code is TensorFlow, PyTorch ports are 3rd-party |
| **Anchor-based** | Still uses anchors (YOLOv8+ is anchor-free, simpler) |
| **Training complexity** | Requires careful hyperparameter tuning |
| **Not unified** | Separate models for detection/segmentation (YOLO does both) |
| **Less active community** | Smaller community vs Ultralytics YOLO |
| **2020 paper** | Many newer techniques have surpassed it |

### Successors & Influenced Models

| Model | Year | Relation to EfficientDet |
|-------|------|--------------------------|
| **EfficientDetV2** | 2021 | Updated training, better accuracy |
| **EfficientDet-Lite** | 2021 | Mobile-optimized variants (Coral) |
| **YOLOX** | 2021 | Adopted decoupled head idea |
| **YOLOv8** | 2023 | PANet influenced by BiFPN concepts |
| **RT-DETR** | 2023 | Efficient scaling principles |
| **EfficientViT** | 2023 | Efficient vision transformers (successor idea) |

### EfficientDet's Legacy

```
EfficientDet may not be the #1 choice TODAY, but it:

✅ PROVED that systematic scaling beats random scaling
✅ INTRODUCED weighted feature fusion (BiFPN) — adopted everywhere
✅ SET the efficiency standard (accuracy per FLOP)
✅ ENABLED mobile detection (D0-Lite on phones/Coral)
✅ INFLUENCED every detector that came after it

Think of it as: "The model that taught the field HOW to scale efficiently"
```

---

## 📋 Chapter Summary

### What You Learned

```
✅ EfficientNet backbone: Compound scaling (depth × width × resolution)
✅ BiFPN: Bidirectional + weighted feature fusion (smarter than FPN)
✅ Compound scaling for detection: Scale everything together
✅ D0-D7 family: From 3.9M params (mobile) to 52M (SOTA)
✅ Efficiency matters: EfficientDet-D2 ≈ YOLOv3 accuracy at 13× fewer FLOPs
✅ Deployment: TFLite, Coral Edge TPU, ONNX
✅ Real-world: Google Lens, Waymo, medical imaging
✅ When to use: TF ecosystem, mobile, efficiency-critical applications
```

### 🔥 Interview Quick-Fire

| Question | Answer |
|----------|--------|
| "What is EfficientDet's key innovation?" | BiFPN (weighted bidirectional feature fusion) + compound scaling |
| "What is BiFPN?" | Bidirectional FPN with learnable fusion weights, stacked multiple times |
| "How does compound scaling work?" | Scale backbone, neck, head, and resolution together using coefficient φ |
| "EfficientDet vs YOLO?" | EfficientDet: more efficient per FLOP. YOLO: faster, easier to use, newer |
| "What is EfficientNet?" | CNN backbone that scales depth/width/resolution with formula α^φ, β^φ, γ^φ |
| "What's the MBConv block?" | Depthwise separable conv + SE attention — efficient building block |
| "Why weighted fusion?" | Different features have different importance; let network LEARN the weights |
| "Best for mobile?" | EfficientDet-D0 or EfficientDet-Lite (TFLite/Coral optimized) |

---

### ➡️ Next Chapter: [06 - TensorFlow Object Detection API](./06-TensorFlow-OD-API.md)

> Now you understand EfficientDet's efficiency principles. Next, we'll explore Google's TensorFlow Object Detection API — a unified framework that lets you train and deploy many architectures (SSD, Faster R-CNN, EfficientDet) within one pipeline.

---

*[← Back to Index](./00-INDEX.md) | [← Previous: 04 - YOLO Family](./04-YOLO-Family-Complete-Guide.md)*
