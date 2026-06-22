# Chapter 03: Image Segmentation

## Table of Contents
- [3.1 What is Image Segmentation?](#31-what-is-image-segmentation)
- [3.2 Types of Segmentation](#32-types-of-segmentation)
- [3.3 Semantic Segmentation — Pixel-Level Classification](#33-semantic-segmentation--pixel-level-classification)
- [3.4 U-Net — The Legendary Architecture](#34-u-net--the-legendary-architecture)
- [3.5 DeepLab Family](#35-deeplab-family)
- [3.6 Instance Segmentation — Mask R-CNN](#36-instance-segmentation--mask-r-cnn)
- [3.7 Panoptic Segmentation](#37-panoptic-segmentation)
- [3.8 Loss Functions for Segmentation](#38-loss-functions-for-segmentation)
- [3.9 Evaluation Metrics](#39-evaluation-metrics)
- [3.10 Data Preparation and Annotation](#310-data-preparation-and-annotation)
- [3.11 Modern Approaches (SAM, Segment Everything)](#311-modern-approaches-sam-segment-everything)
- [3.12 Practical End-to-End Project](#312-practical-end-to-end-project)
- [Quick Reference](#quick-reference)

---

## 3.1 What is Image Segmentation?

### What It Is
Image segmentation assigns a **label to every single pixel** in an image, not just to the whole image. If classification asks "Is there a cat?", segmentation asks "Which exact pixels ARE the cat?"

Analogy: Classification is like saying "this photo has a person." Segmentation is like carefully cutting out the person's silhouette with scissors — you know exactly which pixels belong to the person and which are background.

### Why It Matters
- **Autonomous driving:** Know EXACTLY where the road, cars, pedestrians are (pixel-perfect)
- **Medical imaging:** Precisely outline tumors, organs, lesions for measurement
- **Photo editing:** Remove backgrounds, change sky, apply effects to specific regions
- **Robotics:** Understand scene geometry for manipulation
- **Agriculture:** Measure crop area, detect diseased regions from drone images

### The Segmentation Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                        Input Image                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Classification:  "There is a cat and a dog"                     │
│                                                                   │
│  Object Detection: [cat: box(10,20,150,200)] [dog: box(200,50,350,250)] │
│                                                                   │
│  Semantic Seg:    Every pixel labeled as cat/dog/background       │
│                   (can't distinguish individual instances)         │
│                                                                   │
│  Instance Seg:    Every pixel labeled as cat_1/dog_1/dog_2/bg    │
│                   (separate masks per object)                     │
│                                                                   │
│  Panoptic Seg:   Every pixel labeled (instances + stuff)          │
│                   cat_1/dog_1/sky/road/tree                       │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.2 Types of Segmentation

### Comparison

| Type | Distinguishes Instances? | Covers All Pixels? | Example Output |
|------|-------------------------|--------------------|-|
| Semantic | ❌ All cats = "cat" | ✅ | Pixel-wise class map |
| Instance | ✅ Cat_1 ≠ Cat_2 | ❌ Only "things" | Per-object binary masks |
| Panoptic | ✅ | ✅ | Instance masks + stuff labels |

**"Things" vs "Stuff":**
- **Things:** Countable objects (person, car, dog) → Instance segmentation
- **Stuff:** Amorphous regions (sky, road, grass) → Semantic segmentation
- **Panoptic:** Both things AND stuff → Complete scene understanding

### Visual Example

```
Input Image:          Semantic:            Instance:           Panoptic:
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  🌤️ sky      │    │  SSSSSSSSSS  │    │  ..........  │    │  sky_stuff   │
│              │    │  SSSSSSSSSS  │    │  ..........  │    │  sky_stuff   │
│  🐈 🐈      │    │  CCCCCC CCCC │    │  111111 2222 │    │  cat1  cat2  │
│              │    │              │    │              │    │              │
│  🌿 grass   │    │  GGGGGGGGGG  │    │  ..........  │    │  grass_stuff │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                    S=sky, C=cat, G=grass  1=cat1, 2=cat2    Everything labeled
```

---

## 3.3 Semantic Segmentation — Pixel-Level Classification

### What It Is
Classify every pixel independently into one of $K$ classes. The output is a **dense prediction map** with the same spatial dimensions as the input.

### How It Works

**The fundamental challenge:** CNNs reduce spatial resolution through pooling/striding. But segmentation NEEDS full resolution output. How do we get it back?

**Solution approaches:**

```
Approach 1: Encoder-Decoder (U-Net style)
Input → [Downsample] → [Bottleneck] → [Upsample] → Output
             ↓                              ↑
        Skip Connections ────────────────────┘

Approach 2: Dilated/Atrous Convolutions (DeepLab style)  
Input → [Conv without reducing resolution, but large receptive field] → Output

Approach 3: Feature Pyramid (FPN style)
Input → [Multi-scale features] → [Combine all scales] → Output
```

**Mathematical formulation:**

Given input $X \in \mathbb{R}^{H \times W \times 3}$, produce output $Y \in \mathbb{R}^{H \times W \times K}$ where $K$ = number of classes.

For each pixel $(i, j)$:
$$\hat{y}_{i,j} = \text{argmax}_k \; f(X)_{i,j,k}$$

The model output before argmax is a $K$-channel "logit map" — softmax across channels gives per-pixel class probabilities.

### Fully Convolutional Networks (FCN) — The Foundation

The first successful deep segmentation approach. Key insight: replace FC layers with 1×1 convolutions → output a spatial map.

```python
import torch
import torch.nn as nn
import torchvision.models as models

class SimpleFCN(nn.Module):
    """Simplified FCN for understanding the concept."""
    def __init__(self, num_classes=21):
        super().__init__()
        # Use pretrained ResNet as encoder
        resnet = models.resnet50(weights='IMAGENET1K_V1')
        
        # Take all layers except avgpool and fc
        self.encoder = nn.Sequential(*list(resnet.children())[:-2])
        # Output: (B, 2048, H/32, W/32)
        
        # 1x1 conv to get class scores
        self.classifier = nn.Conv2d(2048, num_classes, kernel_size=1)
        
        # Upsample back to input resolution
        self.upsample = nn.Upsample(scale_factor=32, mode='bilinear', 
                                     align_corners=False)
    
    def forward(self, x):
        features = self.encoder(x)           # (B, 2048, H/32, W/32)
        scores = self.classifier(features)    # (B, K, H/32, W/32)
        output = self.upsample(scores)        # (B, K, H, W)
        return output

# Test
model = SimpleFCN(num_classes=21)
x = torch.randn(1, 3, 512, 512)
output = model(x)
print(f"Input: {x.shape} → Output: {output.shape}")
# Input: (1, 3, 512, 512) → Output: (1, 21, 512, 512)
```

> **⚠️ Problem:** Simply upsampling 32× produces very coarse boundaries. That's why U-Net and DeepLab were invented.

---

## 3.4 U-Net — The Legendary Architecture

### What It Is
U-Net is an encoder-decoder architecture with **skip connections** that pass high-resolution features from the encoder directly to the decoder. The architecture forms a "U" shape when drawn.

### Why It Matters
- **Medical imaging gold standard:** Still the go-to architecture for medical segmentation
- **Works with very little data:** Original paper used ~30 training images!
- **Elegant design:** Influenced almost every subsequent segmentation architecture
- **Versatile:** Used in diffusion models (Stable Diffusion uses a U-Net!)

### How It Works

```
Input (572×572×1)
    │
    ├─[Conv 3×3]──[Conv 3×3]─→ (568×568×64) ─────────────────────────┐ Skip
    │                                                                   │
    ├─[MaxPool 2×2]                                                    │
    │                                                                   │
    ├─[Conv 3×3]──[Conv 3×3]─→ (280×280×128) ───────────────────┐     │
    │                                                             │     │
    ├─[MaxPool 2×2]                                              │     │
    │                                                             │     │
    ├─[Conv 3×3]──[Conv 3×3]─→ (136×136×256) ─────────────┐     │     │
    │                                                       │     │     │
    ├─[MaxPool 2×2]                                        │     │     │
    │                                                       │     │     │
    ├─[Conv 3×3]──[Conv 3×3]─→ (64×64×512) ─────────┐     │     │     │
    │                                                 │     │     │     │
    ├─[MaxPool 2×2]                                  │     │     │     │
    │                                                 │     │     │     │
    └─[Conv 3×3]──[Conv 3×3]─→ (28×28×1024)         │     │     │     │
                    BOTTLENECK                        │     │     │     │
    ┌─[UpConv 2×2]─────────────────────────────────  │     │     │     │
    │                                                 │     │     │     │
    ├─[Concatenate] ←─────────────────────────────────┘     │     │     │
    ├─[Conv 3×3]──[Conv 3×3]─→ (52×52×512)                 │     │     │
    │                                                       │     │     │
    ├─[UpConv 2×2]                                         │     │     │
    ├─[Concatenate] ←──────────────────────────────────────┘     │     │
    ├─[Conv 3×3]──[Conv 3×3]─→ (100×100×256)                    │     │
    │                                                             │     │
    ├─[UpConv 2×2]                                               │     │
    ├─[Concatenate] ←────────────────────────────────────────────┘     │
    ├─[Conv 3×3]──[Conv 3×3]─→ (196×196×128)                          │
    │                                                                   │
    ├─[UpConv 2×2]                                                     │
    ├─[Concatenate] ←──────────────────────────────────────────────────┘
    ├─[Conv 3×3]──[Conv 3×3]─→ (388×388×64)
    │
    └─[Conv 1×1]─→ (388×388×num_classes)   OUTPUT
```

**Why skip connections matter:**
- Encoder captures **WHAT** (semantic meaning) but loses **WHERE** (spatial detail)
- Decoder needs to recover spatial precision for pixel-level output
- Skip connections provide the "WHERE" information directly from the encoder

### Code Examples

```python
import torch
import torch.nn as nn

class DoubleConv(nn.Module):
    """Two consecutive 3x3 convolutions with BatchNorm and ReLU."""
    def __init__(self, in_channels, out_channels):
        super().__init__()
        self.double_conv = nn.Sequential(
            nn.Conv2d(in_channels, out_channels, 3, padding=1, bias=False),
            nn.BatchNorm2d(out_channels),
            nn.ReLU(inplace=True),
            nn.Conv2d(out_channels, out_channels, 3, padding=1, bias=False),
            nn.BatchNorm2d(out_channels),
            nn.ReLU(inplace=True),
        )
    
    def forward(self, x):
        return self.double_conv(x)

class UNet(nn.Module):
    """Standard U-Net implementation."""
    def __init__(self, in_channels=3, num_classes=1, features=[64, 128, 256, 512]):
        super().__init__()
        self.downs = nn.ModuleList()   # Encoder blocks
        self.ups = nn.ModuleList()     # Decoder blocks
        self.pool = nn.MaxPool2d(2, 2)
        
        # Encoder (downsampling path)
        for feature in features:
            self.downs.append(DoubleConv(in_channels, feature))
            in_channels = feature
        
        # Bottleneck
        self.bottleneck = DoubleConv(features[-1], features[-1] * 2)
        
        # Decoder (upsampling path)
        for feature in reversed(features):
            # Transposed conv for upsampling
            self.ups.append(nn.ConvTranspose2d(feature * 2, feature, 
                                               kernel_size=2, stride=2))
            # Double conv after concatenation (feature*2 input because of skip)
            self.ups.append(DoubleConv(feature * 2, feature))
        
        # Final 1x1 conv to get class predictions
        self.final_conv = nn.Conv2d(features[0], num_classes, kernel_size=1)
    
    def forward(self, x):
        skip_connections = []
        
        # Encoder
        for down in self.downs:
            x = down(x)
            skip_connections.append(x)  # Save for skip connection
            x = self.pool(x)
        
        # Bottleneck
        x = self.bottleneck(x)
        
        # Decoder
        skip_connections = skip_connections[::-1]  # Reverse order
        for idx in range(0, len(self.ups), 2):
            x = self.ups[idx](x)      # Upsample
            skip = skip_connections[idx // 2]
            
            # Handle size mismatch (if input isn't perfectly divisible)
            if x.shape != skip.shape:
                x = nn.functional.interpolate(x, size=skip.shape[2:])
            
            x = torch.cat([skip, x], dim=1)  # Concatenate skip connection
            x = self.ups[idx + 1](x)          # Double conv
        
        return self.final_conv(x)

# === Test U-Net ===
model = UNet(in_channels=3, num_classes=1)  # Binary segmentation
x = torch.randn(1, 3, 256, 256)
output = model(x)
print(f"Input: {x.shape} → Output: {output.shape}")
# Input: (1, 3, 256, 256) → Output: (1, 1, 256, 256)

# === Parameter count ===
params = sum(p.numel() for p in model.parameters())
print(f"Parameters: {params:,}")  # ~31M for standard U-Net
```

### U-Net Variants

| Variant | Key Difference | Use Case |
|---------|---------------|----------|
| U-Net | Original, simple | Medical (small datasets) |
| U-Net++ | Nested skip connections | Better feature fusion |
| Attention U-Net | Attention gates on skips | Focus on relevant features |
| ResUNet | ResNet blocks in encoder | Deeper without degradation |
| U-Net with pretrained encoder | ImageNet encoder | General segmentation |

```python
# === Using segmentation_models_pytorch (SMP) — industry standard ===
import segmentation_models_pytorch as smp

# U-Net with pretrained ResNet-34 encoder (much better than training from scratch)
model = smp.Unet(
    encoder_name='resnet34',
    encoder_weights='imagenet',
    in_channels=3,
    classes=1,  # Binary segmentation
)

# Other architectures available:
model_unetpp = smp.UnetPlusPlus(encoder_name='efficientnet-b4', 
                                 encoder_weights='imagenet', classes=5)
model_fpn = smp.FPN(encoder_name='resnet50', encoder_weights='imagenet', classes=5)
model_deeplabv3 = smp.DeepLabV3Plus(encoder_name='resnet101', 
                                     encoder_weights='imagenet', classes=21)
```

> **Pro Tip:** In practice, always use `segmentation_models_pytorch` (SMP) library. It gives you any encoder (ResNet, EfficientNet, etc.) with any decoder (U-Net, FPN, DeepLab) with pretrained weights — in one line.

---

## 3.5 DeepLab Family

### What It Is
DeepLab uses **atrous (dilated) convolutions** to maintain spatial resolution while increasing the receptive field. Instead of downsampling aggressively, it "spreads out" the convolution kernel.

### Why It Matters
- **No information loss:** Avoids aggressive downsampling
- **Multi-scale:** ASPP module captures features at multiple scales simultaneously
- **State-of-the-art:** DeepLabV3+ was the dominant approach before transformers

### How It Works

**Atrous (Dilated) Convolution:**

Standard 3×3 convolution: receptive field = 3×3
Dilated 3×3 with rate=2: receptive field = 5×5 (same parameters!)
Dilated 3×3 with rate=4: receptive field = 9×9 (still same parameters!)

```
Standard Conv (rate=1):    Dilated Conv (rate=2):    Dilated Conv (rate=4):
┌─●─●─●─┐                 ┌─●─○─●─○─●─┐            ┌─●─○─○─○─●─○─○─○─●─┐
│ ● ● ● │                 │ ○ ○ ○ ○ ○ │            │ ○ ○ ○ ○ ○ ○ ○ ○ ○ │
│ ● ● ● │                 │ ● ○ ● ○ ● │            │ ○ ○ ○ ○ ○ ○ ○ ○ ○ │
└────────┘                 │ ○ ○ ○ ○ ○ │            │ ○ ○ ○ ○ ○ ○ ○ ○ ○ │
                           │ ● ○ ● ○ ● │            │ ● ○ ○ ○ ● ○ ○ ○ ● │
RF: 3×3                    └───────────┘            │ ... (continues)    │
9 params                    RF: 5×5                  RF: 9×9
                            9 params!                 9 params!
● = sampled position, ○ = skipped
```

**ASPP (Atrous Spatial Pyramid Pooling):**

```
                    ┌─── 1×1 Conv ──────────────┐
                    │                             │
Input Features ─────├─── 3×3 Conv, rate=6 ───────├──── Concatenate ──→ Output
                    │                             │
                    ├─── 3×3 Conv, rate=12 ──────┤
                    │                             │
                    ├─── 3×3 Conv, rate=18 ──────┤
                    │                             │
                    └─── Global Avg Pool ────────┘

Each branch captures context at a different scale!
```

**DeepLabV3+ architecture:**

```
Input → Encoder (ResNet/Xception) → ASPP → Decoder → Output
              │ (low-level features)           ↑
              └────────────────────────────────┘
                    (skip connection from early layer)
```

### Code Examples

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# === Atrous/Dilated Convolution ===
# Standard: kernel touches adjacent pixels
conv_standard = nn.Conv2d(64, 64, kernel_size=3, padding=1, dilation=1)

# Dilated: kernel skips pixels (larger receptive field, same params)
conv_dilated_6 = nn.Conv2d(64, 64, kernel_size=3, padding=6, dilation=6)
conv_dilated_12 = nn.Conv2d(64, 64, kernel_size=3, padding=12, dilation=12)

# === ASPP Module ===
class ASPP(nn.Module):
    """Atrous Spatial Pyramid Pooling."""
    def __init__(self, in_channels, out_channels=256, rates=[6, 12, 18]):
        super().__init__()
        modules = []
        
        # 1×1 convolution
        modules.append(nn.Sequential(
            nn.Conv2d(in_channels, out_channels, 1, bias=False),
            nn.BatchNorm2d(out_channels),
            nn.ReLU(inplace=True)
        ))
        
        # Atrous convolutions at different rates
        for rate in rates:
            modules.append(nn.Sequential(
                nn.Conv2d(in_channels, out_channels, 3, padding=rate, 
                         dilation=rate, bias=False),
                nn.BatchNorm2d(out_channels),
                nn.ReLU(inplace=True)
            ))
        
        # Global average pooling branch
        modules.append(nn.Sequential(
            nn.AdaptiveAvgPool2d(1),
            nn.Conv2d(in_channels, out_channels, 1, bias=False),
            nn.BatchNorm2d(out_channels),
            nn.ReLU(inplace=True)
        ))
        
        self.convs = nn.ModuleList(modules)
        
        # Projection after concatenation
        self.project = nn.Sequential(
            nn.Conv2d(out_channels * (len(rates) + 2), out_channels, 1, bias=False),
            nn.BatchNorm2d(out_channels),
            nn.ReLU(inplace=True),
            nn.Dropout(0.5)
        )
    
    def forward(self, x):
        results = []
        for conv in self.convs[:-1]:
            results.append(conv(x))
        
        # Global pooling branch — upsample back to spatial size
        global_feat = self.convs[-1](x)
        global_feat = F.interpolate(global_feat, size=x.shape[2:], mode='bilinear',
                                     align_corners=False)
        results.append(global_feat)
        
        x = torch.cat(results, dim=1)
        return self.project(x)

# === Using pretrained DeepLabV3+ ===
from torchvision.models.segmentation import deeplabv3_resnet101

model = deeplabv3_resnet101(weights='COCO_WITH_VOC_LABELS_V1')
model.eval()

# Inference
x = torch.randn(1, 3, 512, 512)
with torch.no_grad():
    output = model(x)
print(f"Output shape: {output['out'].shape}")  # (1, 21, 512, 512) — 21 VOC classes
```

---

## 3.6 Instance Segmentation — Mask R-CNN

### What It Is
Instance segmentation detects individual objects AND produces a pixel-perfect mask for each one. It's object detection + segmentation combined.

### Why It Matters
- **Counting:** Know exactly how many objects (semantic seg can't count)
- **Individual tracking:** Track specific objects across frames
- **Precise measurement:** Measure area/dimensions of individual objects
- **Robotics:** Know which pixels belong to which graspable object

### How It Works

**Mask R-CNN = Faster R-CNN + Mask Branch**

```
                                    ┌──→ Classification (what class?)
                                    │
Input → Backbone → RPN → RoI Align ├──→ Bounding Box (where?)
         (ResNet)                    │
                                    └──→ Mask Branch (which pixels?)
                                         14×14 or 28×28 binary mask per RoI
```

**Key innovation — RoI Align (vs RoI Pool):**
- RoI Pool uses integer quantization → misaligns pixels
- RoI Align uses bilinear interpolation → pixel-perfect alignment
- This small change gave significant mask quality improvement

### Code Examples

```python
import torch
import torchvision
from torchvision.models.detection import maskrcnn_resnet50_fpn_v2

# === Pretrained Mask R-CNN ===
model = maskrcnn_resnet50_fpn_v2(weights='COCO_V1')
model.eval()

# Inference
img = torch.randn(3, 480, 640)  # Single image (C, H, W), values in [0, 1]
with torch.no_grad():
    predictions = model([img])

# Output format:
pred = predictions[0]
print(f"Boxes: {pred['boxes'].shape}")   # (N, 4) — [x1, y1, x2, y2]
print(f"Labels: {pred['labels'].shape}") # (N,) — class indices
print(f"Scores: {pred['scores'].shape}") # (N,) — confidence scores
print(f"Masks: {pred['masks'].shape}")   # (N, 1, H, W) — soft masks [0, 1]

# === Filter by confidence ===
threshold = 0.7
keep = pred['scores'] > threshold
boxes = pred['boxes'][keep]
labels = pred['labels'][keep]
masks = pred['masks'][keep]  # Soft masks

# Convert soft mask to binary
binary_masks = (masks > 0.5).squeeze(1)  # (N, H, W) boolean

# === Fine-tune Mask R-CNN on custom dataset ===
from torchvision.models.detection import MaskRCNN
from torchvision.models.detection.rpn import AnchorGenerator

def get_custom_maskrcnn(num_classes):
    """Create Mask R-CNN for custom dataset."""
    # Load pretrained model
    model = maskrcnn_resnet50_fpn_v2(weights='COCO_V1')
    
    # Replace box predictor (classification head)
    in_features = model.roi_heads.box_predictor.cls_score.in_features
    model.roi_heads.box_predictor = torchvision.models.detection.faster_rcnn.FastRCNNPredictor(
        in_features, num_classes)
    
    # Replace mask predictor
    in_features_mask = model.roi_heads.mask_predictor.conv5_mask.in_channels
    hidden_layer = 256
    model.roi_heads.mask_predictor = torchvision.models.detection.mask_rcnn.MaskRCNNPredictor(
        in_features_mask, hidden_layer, num_classes)
    
    return model

# Custom dataset format for Mask R-CNN:
class CustomSegDataset(torch.utils.data.Dataset):
    def __init__(self, images, annotations):
        self.images = images
        self.annotations = annotations
    
    def __getitem__(self, idx):
        img = self.images[idx]  # (C, H, W) tensor, float [0, 1]
        
        # Target must contain:
        target = {
            'boxes': torch.tensor([[x1, y1, x2, y2], ...], dtype=torch.float32),
            'labels': torch.tensor([class_id, ...], dtype=torch.int64),
            'masks': torch.tensor(binary_masks, dtype=torch.uint8),  # (N, H, W)
            'area': torch.tensor([area1, area2, ...], dtype=torch.float32),
            'iscrowd': torch.zeros(num_objects, dtype=torch.int64),
        }
        return img, target
    
    def __len__(self):
        return len(self.images)
```

---

## 3.7 Panoptic Segmentation

### What It Is
Panoptic segmentation provides a **complete, unified scene understanding**: every pixel gets a label, countable objects (things) get instance IDs, and uncountable regions (stuff) get semantic labels.

### How It Works

```
Input Image
    │
    ├───→ Semantic Segmentation Branch → Stuff predictions (sky, road, grass)
    │
    ├───→ Instance Segmentation Branch → Thing predictions (car_1, person_2)
    │
    └───→ Panoptic Fusion Module → Combined output
          (resolves overlaps, assigns every pixel exactly one label)
```

**Panoptic Quality (PQ) metric:**

$$PQ = \underbrace{\frac{\sum_{(p,g) \in TP} IoU(p,g)}{|TP|}}_{\text{Segmentation Quality (SQ)}} \times \underbrace{\frac{|TP|}{|TP| + \frac{1}{2}|FP| + \frac{1}{2}|FN|}}_{\text{Recognition Quality (RQ)}}$$

### Code Examples

```python
# === Using Detectron2 for Panoptic Segmentation ===
# pip install detectron2

from detectron2 import model_zoo
from detectron2.engine import DefaultPredictor
from detectron2.config import get_cfg

cfg = get_cfg()
cfg.merge_from_file(model_zoo.get_config_file(
    "COCO-PanopticSegmentation/panoptic_fpn_R_101_3x.yaml"))
cfg.MODEL.WEIGHTS = model_zoo.get_checkpoint_url(
    "COCO-PanopticSegmentation/panoptic_fpn_R_101_3x.yaml")
predictor = DefaultPredictor(cfg)

# Run panoptic segmentation
# outputs["panoptic_seg"] = (panoptic_map, segments_info)
# panoptic_map: (H, W) tensor where each pixel has a unique ID
# segments_info: list of dicts with category_id, id, isthing, area
```

---

## 3.8 Loss Functions for Segmentation

### What It Is
Loss functions measure how wrong the predicted segmentation is. Choosing the right loss is CRITICAL because segmentation has unique challenges: class imbalance (background dominates), fuzzy boundaries, and tiny objects.

### Why It Matters
Using CrossEntropy alone often gives poor results because:
- 90%+ of pixels might be background → model learns to predict "background everywhere"
- Small objects contribute negligibly to the loss → model ignores them
- Boundaries are hard → standard losses don't penalize boundary errors enough

### The Loss Functions

#### Cross-Entropy Loss (Baseline)

$$\mathcal{L}_{CE} = -\frac{1}{N} \sum_{i=1}^{N} \sum_{k=1}^{K} y_{ik} \log(\hat{y}_{ik})$$

Where $N$ = number of pixels, $K$ = number of classes.

#### Dice Loss (Overlap-based)

Based on the Dice coefficient (F1 score for segmentation):

$$\mathcal{L}_{Dice} = 1 - \frac{2 \sum_i p_i g_i + \epsilon}{\sum_i p_i + \sum_i g_i + \epsilon}$$

Where $p_i$ = predicted probability, $g_i$ = ground truth, $\epsilon$ = smoothing factor.

#### Focal Loss (Handles class imbalance)

$$\mathcal{L}_{Focal} = -\alpha_t (1 - p_t)^\gamma \log(p_t)$$

Where $\gamma$ (focusing parameter) down-weights easy examples. Typically $\gamma = 2$.

#### Combined Loss (Best practice)

$$\mathcal{L} = \lambda_1 \mathcal{L}_{CE} + \lambda_2 \mathcal{L}_{Dice}$$

### Code Examples

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

# === Dice Loss ===
class DiceLoss(nn.Module):
    """Dice loss for binary or multi-class segmentation."""
    def __init__(self, smooth=1.0):
        super().__init__()
        self.smooth = smooth
    
    def forward(self, pred, target):
        """
        pred: (B, C, H, W) — logits or probabilities
        target: (B, H, W) — class indices (for multi-class)
               or (B, 1, H, W) — binary target
        """
        if pred.shape[1] == 1:  # Binary
            pred = torch.sigmoid(pred)
            pred_flat = pred.view(-1)
            target_flat = target.float().view(-1)
            intersection = (pred_flat * target_flat).sum()
            return 1 - (2. * intersection + self.smooth) / (
                pred_flat.sum() + target_flat.sum() + self.smooth)
        else:  # Multi-class
            pred = F.softmax(pred, dim=1)
            num_classes = pred.shape[1]
            target_one_hot = F.one_hot(target, num_classes).permute(0, 3, 1, 2).float()
            
            total_loss = 0
            for c in range(num_classes):
                pred_c = pred[:, c]
                target_c = target_one_hot[:, c]
                intersection = (pred_c * target_c).sum()
                total_loss += 1 - (2. * intersection + self.smooth) / (
                    pred_c.sum() + target_c.sum() + self.smooth)
            return total_loss / num_classes

# === Focal Loss ===
class FocalLoss(nn.Module):
    """Focal loss for handling class imbalance."""
    def __init__(self, alpha=0.25, gamma=2.0):
        super().__init__()
        self.alpha = alpha
        self.gamma = gamma
    
    def forward(self, pred, target):
        """
        pred: (B, C, H, W) logits
        target: (B, H, W) class indices
        """
        ce_loss = F.cross_entropy(pred, target, reduction='none')
        pt = torch.exp(-ce_loss)  # Probability of correct class
        focal_loss = self.alpha * (1 - pt) ** self.gamma * ce_loss
        return focal_loss.mean()

# === Combined Loss (RECOMMENDED) ===
class SegmentationLoss(nn.Module):
    """Combined CE + Dice loss — best practice for segmentation."""
    def __init__(self, ce_weight=1.0, dice_weight=1.0, num_classes=1):
        super().__init__()
        self.ce_weight = ce_weight
        self.dice_weight = dice_weight
        self.dice = DiceLoss()
        self.num_classes = num_classes
    
    def forward(self, pred, target):
        if self.num_classes == 1:
            ce = F.binary_cross_entropy_with_logits(pred, target.float())
        else:
            ce = F.cross_entropy(pred, target)
        dice = self.dice(pred, target)
        return self.ce_weight * ce + self.dice_weight * dice

# === Boundary Loss (for precise edges) ===
class BoundaryLoss(nn.Module):
    """Penalizes predictions far from ground truth boundaries."""
    def __init__(self):
        super().__init__()
    
    def forward(self, pred, dist_map):
        """
        pred: (B, 1, H, W) sigmoid output
        dist_map: (B, 1, H, W) distance transform of GT boundary
                  (precomputed: positive inside, negative outside)
        """
        return (pred * dist_map).mean()

# === Usage ===
criterion = SegmentationLoss(ce_weight=1.0, dice_weight=1.0, num_classes=1)
pred = model(images)        # (B, 1, H, W)
loss = criterion(pred, masks)  # masks: (B, 1, H, W)
```

### Loss Function Selection Guide

| Scenario | Recommended Loss | Why |
|----------|-----------------|-----|
| Balanced classes | CE + Dice | Standard and effective |
| Severe imbalance | Focal + Dice | Focal handles imbalance |
| Small objects critical | Dice (per-instance) | Scale-invariant |
| Precise boundaries needed | CE + Dice + Boundary | Boundary loss helps edges |
| Binary segmentation | BCE + Dice | Simple and effective |
| Multi-class | CE + Dice (per-class) | Handles each class fairly |

---

## 3.9 Evaluation Metrics

### Core Metrics

#### IoU (Intersection over Union) / Jaccard Index

The **gold standard** metric for segmentation:

$$IoU = \frac{|P \cap G|}{|P \cup G|} = \frac{TP}{TP + FP + FN}$$

```
Prediction:          Ground Truth:        Intersection:        Union:
██████░░░░           ░░████████░░         ░░████░░░░░░         ██████████░░
██████░░░░           ░░████████░░         ░░████░░░░░░         ██████████░░

IoU = Intersection / Union = (overlapping pixels) / (all marked pixels)
```

#### mIoU (Mean IoU)
Average IoU across all classes:
$$mIoU = \frac{1}{K} \sum_{k=1}^{K} IoU_k$$

#### Dice Coefficient / F1 Score
$$Dice = \frac{2|P \cap G|}{|P| + |G|} = \frac{2 \cdot TP}{2 \cdot TP + FP + FN}$$

**Relationship:** $Dice = \frac{2 \cdot IoU}{1 + IoU}$

#### Pixel Accuracy
$$PA = \frac{\text{correctly classified pixels}}{\text{total pixels}}$$

> **⚠️ Warning:** Pixel accuracy is MISLEADING for imbalanced datasets. If 95% of pixels are background, predicting "everything is background" gives 95% accuracy!

### Code Examples

```python
import numpy as np
import torch

def compute_iou(pred_mask, gt_mask, num_classes):
    """Compute per-class IoU."""
    ious = []
    for cls in range(num_classes):
        pred_cls = (pred_mask == cls)
        gt_cls = (gt_mask == cls)
        
        intersection = (pred_cls & gt_cls).sum()
        union = (pred_cls | gt_cls).sum()
        
        if union == 0:
            ious.append(float('nan'))  # Class not present
        else:
            ious.append(intersection / union)
    
    return ious

def compute_dice(pred_mask, gt_mask):
    """Compute Dice coefficient for binary masks."""
    intersection = (pred_mask & gt_mask).sum()
    return (2 * intersection) / (pred_mask.sum() + gt_mask.sum() + 1e-8)

# === Comprehensive evaluation ===
def evaluate_segmentation(model, dataloader, num_classes, device):
    """Full evaluation with all metrics."""
    model.eval()
    
    # Confusion matrix accumulator
    conf_matrix = np.zeros((num_classes, num_classes), dtype=np.int64)
    
    with torch.no_grad():
        for images, masks in dataloader:
            images = images.to(device)
            outputs = model(images)
            preds = outputs.argmax(dim=1).cpu().numpy()  # (B, H, W)
            masks = masks.numpy()
            
            for pred, gt in zip(preds, masks):
                for cls in range(num_classes):
                    pred_cls = (pred == cls)
                    gt_cls = (gt == cls)
                    conf_matrix[cls, cls] += (pred_cls & gt_cls).sum()  # TP
                    # ... fill full confusion matrix
    
    # Compute metrics from confusion matrix
    ious = []
    for i in range(num_classes):
        tp = conf_matrix[i, i]
        fp = conf_matrix[:, i].sum() - tp
        fn = conf_matrix[i, :].sum() - tp
        iou = tp / (tp + fp + fn + 1e-8)
        ious.append(iou)
    
    miou = np.nanmean(ious)
    pixel_acc = np.diag(conf_matrix).sum() / conf_matrix.sum()
    
    return {
        'mIoU': miou,
        'per_class_IoU': ious,
        'pixel_accuracy': pixel_acc,
    }
```

### Metric Comparison

| Metric | Range | Pros | Cons | Use When |
|--------|-------|------|------|----------|
| mIoU | [0, 1] | Standard, class-balanced | Sensitive to small classes | Multi-class benchmarks |
| Dice | [0, 1] | Similar to IoU, used in medical | Slightly optimistic vs IoU | Binary/medical segmentation |
| Pixel Accuracy | [0, 1] | Simple | Misleading with imbalance | Never use alone |
| Boundary F1 | [0, 1] | Focuses on edge quality | Extra computation | When boundaries matter |

---

## 3.10 Data Preparation and Annotation

### What It Is
Segmentation requires **per-pixel annotations** — far more expensive than bounding boxes. Understanding efficient annotation strategies is essential for practical projects.

### Annotation Formats

| Format | Description | Used By |
|--------|-------------|---------|
| PNG mask | Grayscale/color image, pixel value = class ID | Most frameworks |
| COCO RLE | Run-length encoding of binary masks | COCO dataset, Detectron2 |
| Polygon | List of vertex coordinates | LabelMe, CVAT |
| NIFTI | Medical imaging format (.nii.gz) | Medical segmentation |

### Code Examples

```python
import cv2
import numpy as np
import json
from pathlib import Path

# === Load segmentation mask ===
# Format 1: Class-indexed PNG (pixel value = class ID)
mask = cv2.imread('mask.png', cv2.IMREAD_GRAYSCALE)  # Values: 0, 1, 2, ...
num_classes = mask.max() + 1

# Format 2: Color-coded mask → class indices
color_mask = cv2.imread('color_mask.png')
# Define color → class mapping
color_map = {
    (0, 0, 0): 0,       # Black → background
    (255, 0, 0): 1,     # Red → class 1
    (0, 255, 0): 2,     # Green → class 2
    (0, 0, 255): 3,     # Blue → class 3
}

# Convert color mask to class indices
class_mask = np.zeros(color_mask.shape[:2], dtype=np.uint8)
for color, class_id in color_map.items():
    match = np.all(color_mask == color, axis=2)
    class_mask[match] = class_id

# === Custom Dataset for Segmentation ===
from torch.utils.data import Dataset
from torchvision import transforms

class SegmentationDataset(Dataset):
    """Custom segmentation dataset."""
    def __init__(self, image_dir, mask_dir, transform=None):
        self.image_paths = sorted(Path(image_dir).glob('*.jpg'))
        self.mask_dir = Path(mask_dir)
        self.transform = transform
    
    def __len__(self):
        return len(self.image_paths)
    
    def __getitem__(self, idx):
        # Load image
        img_path = self.image_paths[idx]
        image = cv2.imread(str(img_path))
        image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
        
        # Load corresponding mask
        mask_path = self.mask_dir / f"{img_path.stem}.png"
        mask = cv2.imread(str(mask_path), cv2.IMREAD_GRAYSCALE)
        
        # Apply transforms (Albumentations handles both image AND mask)
        if self.transform:
            transformed = self.transform(image=image, mask=mask)
            image = transformed['image']
            mask = transformed['mask']
        
        return image, mask.long()

# === Augmentation for segmentation (must transform mask too!) ===
import albumentations as A
from albumentations.pytorch import ToTensorV2

seg_transform = A.Compose([
    A.RandomResizedCrop(512, 512, scale=(0.5, 1.0)),
    A.HorizontalFlip(p=0.5),
    A.VerticalFlip(p=0.5),  # If orientation doesn't matter (e.g., satellite)
    A.RandomRotate90(p=0.5),
    A.ColorJitter(brightness=0.3, contrast=0.3, saturation=0.3, hue=0.1),
    A.GaussianBlur(blur_limit=7, p=0.3),
    A.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
    ToTensorV2(),
])
# NOTE: geometric transforms are applied to BOTH image and mask automatically!
# Color transforms are applied to image ONLY (masks are not affected)
```

> **⚠️ Critical:** When augmenting segmentation data, geometric transforms (flip, rotate, crop) must be applied identically to BOTH image and mask. Albumentations handles this automatically. If using torchvision transforms, you must apply them to both manually.

---

## 3.11 Modern Approaches (SAM, Segment Everything)

### Segment Anything Model (SAM) — Meta, 2023

**What it is:** A foundation model for segmentation that can segment ANY object in ANY image with various prompts (points, boxes, text).

**Why it's revolutionary:**
- Zero-shot: works on unseen object types without retraining
- Multiple prompt types: point, box, text, or automatic
- Trained on 11M images, 1.1B masks

```
SAM Architecture:
┌──────────────┐     ┌───────────────┐     ┌──────────────┐
│ Image Encoder│     │ Prompt Encoder│     │ Mask Decoder │
│ (ViT-H)     │  +  │ (points/boxes)│  →  │ (lightweight)│  → Masks
│ Run once     │     │ Run per prompt│     │              │
└──────────────┘     └───────────────┘     └──────────────┘
```

```python
# === Using SAM ===
from segment_anything import sam_model_registry, SamPredictor

# Load model
sam = sam_model_registry["vit_h"](checkpoint="sam_vit_h.pth")
predictor = SamPredictor(sam)

# Set image (encodes once, reuse for multiple prompts)
image = cv2.imread("photo.jpg")
image_rgb = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
predictor.set_image(image_rgb)

# Prompt with points
input_point = np.array([[500, 375]])  # (x, y) coordinates
input_label = np.array([1])  # 1 = foreground, 0 = background

masks, scores, logits = predictor.predict(
    point_coords=input_point,
    point_labels=input_label,
    multimask_output=True,  # Returns 3 masks at different granularities
)

# Prompt with bounding box
input_box = np.array([425, 600, 700, 875])  # [x1, y1, x2, y2]
masks, scores, logits = predictor.predict(
    box=input_box,
    multimask_output=False,
)

# === Automatic mask generation (segment everything) ===
from segment_anything import SamAutomaticMaskGenerator

mask_generator = SamAutomaticMaskGenerator(
    model=sam,
    points_per_side=32,        # Grid density
    pred_iou_thresh=0.88,      # Filter low-quality masks
    stability_score_thresh=0.95,
    min_mask_region_area=100,  # Remove tiny masks
)

masks = mask_generator.generate(image_rgb)
# Returns list of dicts: {'segmentation', 'area', 'bbox', 'predicted_iou', ...}
print(f"Found {len(masks)} segments")
```

### SAM 2 (2024) — Video Extension
Extends SAM to video: segment and track objects across frames with a single click.

---

## 3.12 Practical End-to-End Project

### Binary Segmentation: Road Surface Detection

```python
"""
Complete segmentation project: Binary road segmentation.
"""
import torch
import torch.nn as nn
from torch.utils.data import DataLoader
import segmentation_models_pytorch as smp
import albumentations as A
from albumentations.pytorch import ToTensorV2
import numpy as np
from pathlib import Path

# ============= Configuration =============
class Config:
    img_size = 512
    batch_size = 8
    epochs = 50
    lr = 1e-4
    encoder = 'efficientnet-b3'
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

cfg = Config()

# ============= Transforms =============
train_transform = A.Compose([
    A.RandomResizedCrop(cfg.img_size, cfg.img_size, scale=(0.5, 1.0)),
    A.HorizontalFlip(p=0.5),
    A.RandomBrightnessContrast(brightness_limit=0.3, contrast_limit=0.3, p=0.5),
    A.GaussNoise(var_limit=(10, 50), p=0.3),
    A.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
    ToTensorV2(),
])

val_transform = A.Compose([
    A.Resize(cfg.img_size, cfg.img_size),
    A.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
    ToTensorV2(),
])

# ============= Model =============
model = smp.UnetPlusPlus(
    encoder_name=cfg.encoder,
    encoder_weights='imagenet',
    in_channels=3,
    classes=1,                    # Binary: 1 output channel
    activation=None,              # Raw logits (apply sigmoid in loss/inference)
)
model = model.to(cfg.device)

# ============= Loss & Optimizer =============
class DiceBCELoss(nn.Module):
    def __init__(self):
        super().__init__()
    
    def forward(self, pred, target):
        bce = nn.functional.binary_cross_entropy_with_logits(pred, target)
        pred_sig = torch.sigmoid(pred)
        intersection = (pred_sig * target).sum()
        dice = 1 - (2 * intersection + 1) / (pred_sig.sum() + target.sum() + 1)
        return bce + dice

criterion = DiceBCELoss()
optimizer = torch.optim.AdamW(model.parameters(), lr=cfg.lr, weight_decay=0.01)
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=cfg.epochs)

# ============= Training Loop =============
best_iou = 0.0

for epoch in range(cfg.epochs):
    # Train
    model.train()
    train_loss = 0
    for images, masks in train_loader:
        images = images.to(cfg.device)
        masks = masks.float().unsqueeze(1).to(cfg.device)  # (B, 1, H, W)
        
        preds = model(images)
        loss = criterion(preds, masks)
        
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        train_loss += loss.item()
    
    scheduler.step()
    
    # Validate
    model.eval()
    val_iou = 0
    num_batches = 0
    with torch.no_grad():
        for images, masks in val_loader:
            images = images.to(cfg.device)
            masks = masks.to(cfg.device)
            
            preds = torch.sigmoid(model(images)) > 0.5  # Binary prediction
            masks_bool = masks.bool()
            
            # Compute IoU
            intersection = (preds.squeeze(1) & masks_bool).float().sum((1, 2))
            union = (preds.squeeze(1) | masks_bool).float().sum((1, 2))
            iou = (intersection / (union + 1e-8)).mean()
            val_iou += iou.item()
            num_batches += 1
    
    val_iou /= num_batches
    print(f"Epoch {epoch+1}/{cfg.epochs} | Loss: {train_loss/len(train_loader):.4f} | Val IoU: {val_iou:.4f}")
    
    if val_iou > best_iou:
        best_iou = val_iou
        torch.save(model.state_dict(), 'best_seg_model.pth')

# ============= Inference =============
def predict_mask(model, image_path, transform, device, threshold=0.5):
    """Run segmentation inference on a single image."""
    import cv2
    
    image = cv2.imread(str(image_path))
    image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
    original_size = image.shape[:2]
    
    # Preprocess
    transformed = transform(image=image)
    input_tensor = transformed['image'].unsqueeze(0).to(device)
    
    # Predict
    model.eval()
    with torch.no_grad():
        pred = torch.sigmoid(model(input_tensor))
    
    # Post-process
    mask = (pred > threshold).squeeze().cpu().numpy().astype(np.uint8)
    
    # Resize back to original
    mask = cv2.resize(mask, (original_size[1], original_size[0]), 
                      interpolation=cv2.INTER_NEAREST)  # NEAREST for masks!
    
    return mask
```

---

## Interview Questions

1. **Q: What's the difference between semantic, instance, and panoptic segmentation?**
   - A: Semantic assigns a class to every pixel but doesn't distinguish instances (all cats = "cat"). Instance detects individual objects with separate masks (cat_1, cat_2) but only for "things," not "stuff." Panoptic does both — every pixel gets a label, and individual things get unique IDs.

2. **Q: Why does U-Net use skip connections? How do they differ from ResNet's skip connections?**
   - A: U-Net skip connections concatenate encoder features with decoder features to recover spatial detail lost during downsampling. ResNet's skip connections ADD features (residual learning) within the same resolution. U-Net's are across resolution levels (encoder→decoder), ResNet's are within the same level.

3. **Q: Why is Dice loss preferred over CrossEntropy for medical segmentation?**
   - A: Medical images often have tiny target regions (e.g., tumor = 2% of image). CE loss is dominated by the background class. Dice loss directly optimizes the overlap metric and is inherently balanced — it cares about the ratio, not absolute pixel counts.

4. **Q: Explain dilated/atrous convolutions and their advantage.**
   - A: Dilated convolutions insert gaps between kernel weights, increasing receptive field without adding parameters or reducing resolution. A 3×3 kernel with dilation=2 has a 5×5 receptive field but only 9 parameters. This avoids the information loss from pooling while maintaining computational efficiency.

5. **Q: How would you handle class imbalance in segmentation?**
   - A: Multiple strategies: (1) Use Dice or Focal loss instead of/alongside CE; (2) Weight the CE loss inversely proportional to class frequency; (3) Oversample patches containing rare classes; (4) Use a combined loss (CE + Dice); (5) For extreme imbalance, use class-specific Dice and average.

6. **Q: What is IoU and why is it better than pixel accuracy?**
   - A: IoU = intersection/union of predicted and ground truth regions. If 95% of pixels are background, predicting all-background gives 95% pixel accuracy but 0% IoU for foreground classes. IoU penalizes both false positives and false negatives equally.

7. **Q: How does Mask R-CNN differ from a semantic segmentation model?**
   - A: Mask R-CNN first detects objects (regions of interest), then performs binary segmentation WITHIN each detected box. This gives instance-level masks. Semantic models produce a single dense map for the whole image. Mask R-CNN = detection + per-instance segmentation.

8. **Q: What is SAM and why is it significant?**
   - A: Segment Anything Model is a foundation model for segmentation. It can segment any object with minimal prompts (point, box) without retraining. It's significant because it's a zero-shot segmentation model — analogous to GPT for text, but for visual segmentation.

---

## Quick Reference

### Architecture Selection

| Task | Architecture | Library |
|------|-------------|---------|
| Binary segmentation | U-Net / U-Net++ | segmentation_models_pytorch |
| Multi-class semantic | DeepLabV3+ | torchvision / SMP |
| Instance segmentation | Mask R-CNN | Detectron2 / torchvision |
| Panoptic segmentation | Panoptic FPN | Detectron2 |
| Zero-shot segmentation | SAM | segment-anything |
| Medical imaging | U-Net + pretrained encoder | SMP / MONAI |

### Loss Function Cheat Sheet

| Loss | Best For | Formula Key |
|------|---------|-------------|
| BCE | Binary, balanced | $-y\log p - (1-y)\log(1-p)$ |
| CE | Multi-class, balanced | $-\sum y_k \log p_k$ |
| Dice | Imbalanced, overlap-focused | $1 - 2|P \cap G| / (|P| + |G|)$ |
| Focal | Severe imbalance | $(1-p_t)^\gamma \cdot CE$ |
| BCE + Dice | Binary (default choice) | Combine both |
| CE + Dice | Multi-class (default choice) | Combine both |

### Metric Ranges (What's "Good"?)

| Metric | Poor | Acceptable | Good | Excellent |
|--------|------|-----------|------|-----------|
| mIoU | < 0.3 | 0.3–0.5 | 0.5–0.7 | > 0.7 |
| Dice | < 0.5 | 0.5–0.7 | 0.7–0.85 | > 0.85 |
| Pixel Acc | < 0.8 | 0.8–0.9 | 0.9–0.95 | > 0.95 |

### Common Hyperparameters

| Parameter | Typical Value | Notes |
|-----------|--------------|-------|
| Input size | 256–512 | Larger = better but slower |
| Encoder | EfficientNet-B3/B4 | Good accuracy/speed |
| Batch size | 4–16 | Limited by GPU memory |
| Learning rate | 1e-4 | With pretrained encoder |
| Loss | BCE + Dice | Binary; CE + Dice for multi-class |
| Optimizer | AdamW | weight_decay=0.01 |
| Epochs | 50–100 | With early stopping |
| Threshold | 0.5 | Tune on validation set |
