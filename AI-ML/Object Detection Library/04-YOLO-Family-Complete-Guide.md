# 📘 Chapter 04: YOLO Family — Complete Guide (v1 to v11)

> **Goal**: Master the most popular object detection library in the world. From understanding YOLO's revolutionary idea to training production models with Ultralytics YOLOv8/v11.

---

## 📑 Table of Contents

1. [Why YOLO Dominates the Industry](#1-why-yolo-dominates-the-industry)
2. [YOLOv1 — The Original Revolution (2016)](#2-yolov1--the-original-revolution-2016)
3. [YOLOv2 / YOLO9000 — Better, Faster, Stronger (2017)](#3-yolov2--yolo9000--better-faster-stronger-2017)
4. [YOLOv3 — The Practical Workhorse (2018)](#4-yolov3--the-practical-workhorse-2018)
5. [YOLOv4 — Bag of Freebies & Specials (2020)](#5-yolov4--bag-of-freebies--specials-2020)
6. [YOLOv5 — Ultralytics Era Begins (2020)](#6-yolov5--ultralytics-era-begins-2020)
7. [YOLOv6, v7 — Speed Wars (2022)](#7-yolov6-v7--speed-wars-2022)
8. [YOLOv8 — The Modern Standard (2023)](#8-yolov8--the-modern-standard-2023)
9. [YOLOv9, v10, v11 — Latest Innovations (2024)](#9-yolov9-v10-v11--latest-innovations-2024)
10. [Ultralytics Library — Complete Usage Guide](#10-ultralytics-library--complete-usage-guide)
11. [Training YOLO on Custom Dataset (Step-by-Step)](#11-training-yolo-on-custom-dataset-step-by-step)
12. [YOLO for Different Tasks (Detect, Segment, Pose, OBB)](#12-yolo-for-different-tasks-detect-segment-pose-obb)
13. [YOLO Deployment (Export & Optimize)](#13-yolo-deployment-export--optimize)
14. [YOLO Tips, Tricks & Best Practices](#14-yolo-tips-tricks--best-practices)
15. [Real-World YOLO Projects & Company Use Cases](#15-real-world-yolo-projects--company-use-cases)

---

## 1. Why YOLO Dominates the Industry

### 📊 YOLO by the Numbers

```
✅ 50,000+ GitHub stars (Ultralytics)
✅ 200M+ pip installs
✅ Used by 100,000+ companies worldwide
✅ #1 most deployed object detection model
✅ 5 lines of code to detect objects
✅ Runs on: GPU, CPU, Mobile, Edge, Browser
```

### 🏢 Companies Using YOLO in Production

| Company | Use Case | YOLO Version |
|---------|----------|-------------|
| **Tesla** | Object detection for Autopilot R&D | Custom (inspired by YOLO) |
| **Samsung** | Camera AI, Scene detection | YOLOv5/v8 (mobile) |
| **Airbus** | Satellite image analysis | YOLOv8 |
| **Walmart** | Shelf monitoring | YOLOv5/v8 |
| **John Deere** | Crop/weed detection | YOLOv8 |
| **Hyundai** | ADAS development | YOLOv7/v8 |
| **Philips** | Medical device imaging | YOLOv8 |
| **DHL** | Package sorting & damage detection | YOLOv5 |
| **Shell** | Pipeline inspection (drones) | YOLOv8 |
| **Nike** | Visual search, product detection | YOLOv8 |

### Why Everyone Chooses YOLO

| Reason | Explanation |
|--------|------------|
| **Easiest to use** | 5 lines to train, 3 lines to predict |
| **Fastest** | 200+ FPS on GPU, 30+ FPS on CPU |
| **Best docs** | Ultralytics docs are industry-leading |
| **Multi-task** | Detection + Segmentation + Pose + Classification + OBB |
| **Export anywhere** | ONNX, TensorRT, CoreML, TFLite, OpenVINO, NCNN |
| **Active community** | Daily updates, Discord, GitHub support |
| **Production-proven** | Running in millions of production systems |
| **Free** | AGPL-3.0 (or Enterprise license for commercial) |

---

## 2. YOLOv1 — The Original Revolution (2016)

### 💡 The Big Idea: "You Only Look Once"

> Instead of scanning the image thousands of times (sliding window, R-CNN), look at the ENTIRE image ONCE and predict everything in one shot!

### 🔥 The Revolutionary Paper

```
Title: "You Only Look Once: Unified, Real-Time Object Detection"
Authors: Joseph Redmon, Santosh Divvala, Ross Girshick, Ali Farhadi
Key Claim: "We frame detection as a REGRESSION problem" 
           (not classification of proposals!)
```

### How YOLOv1 Works

```
CORE IDEA: Divide image into a GRID, each cell predicts boxes + classes

Step 1: Divide image into S×S grid (S=7)
┌───┬───┬───┬───┬───┬───┬───┐
│   │   │   │   │   │   │   │
├───┼───┼───┼───┼───┼───┼───┤
│   │   │   │   │   │   │   │
├───┼───┼───┼───┼───┼───┼───┤
│   │   │ ● │   │   │   │   │  ← Center of object falls in this cell
├───┼───┼───┼───┼───┼───┼───┤     → This cell is RESPONSIBLE for detecting it
│   │   │   │   │   │   │   │
├───┼───┼───┼───┼───┼───┼───┤
│   │   │   │   │   │   │   │
├───┼───┼───┼───┼───┼───┼───┤
│   │   │   │   │   │   │   │
├───┼───┼───┼───┼───┼───┼───┤
│   │   │   │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┘

Step 2: Each cell predicts B bounding boxes (B=2) + C class probabilities

Each box prediction = [x, y, w, h, confidence]
  x, y = center of box RELATIVE to grid cell (0-1)
  w, h = width, height RELATIVE to image (0-1)
  confidence = P(Object) × IoU(pred, truth)

Step 3: Output tensor
  → S × S × (B×5 + C) = 7 × 7 × (2×5 + 20) = 7 × 7 × 30

  For PASCAL VOC (20 classes):
  Each cell predicts: 2 boxes (5 values each) + 20 class probabilities = 30 values
```

### YOLOv1 Network Architecture

```
Input: 448 × 448 × 3

┌─────────────────────────────────────────────────────────┐
│  24 Convolutional Layers          2 Fully Connected     │
│                                                         │
│  448×448 → 7×7×1024        →  4096  →  7×7×30          │
│                                                         │
│  (Inspired by GoogLeNet)      (Flatten)  (Reshape)     │
└─────────────────────────────────────────────────────────┘

Output: 7 × 7 × 30 tensor
  = 7×7 grid cells
  × (2 boxes × [x, y, w, h, conf] + 20 class probs)
```

### YOLOv1 Loss Function

```
Loss = λ_coord × [Box center loss + Box size loss]
     + [Confidence loss for cells WITH objects]
     + λ_noobj × [Confidence loss for cells WITHOUT objects]
     + [Class probability loss]

Where:
  λ_coord = 5 (emphasize box accuracy)
  λ_noobj = 0.5 (de-emphasize empty cells)
  
  Box size uses √(w) and √(h) instead of w, h
  → Small errors in small boxes matter MORE than in big boxes
```

### YOLOv1 Performance

| Metric | Value |
|--------|-------|
| mAP (VOC 2007) | 63.4% |
| Speed | **45 FPS** (real-time!) |
| Fast YOLO | 155 FPS (smaller network) |

### ⚠️ YOLOv1 Limitations

```
❌ Only 2 boxes per cell → struggles with groups (birds flock)
❌ 7×7 grid = coarse → poor on small objects
❌ Localization errors (bounding boxes not precise)
❌ No multi-scale detection
❌ mAP lower than Faster R-CNN (63% vs 73%)
```

---

## 3. YOLOv2 / YOLO9000 — Better, Faster, Stronger (2017)

### 💡 Key Improvements

| Trick | What It Does | mAP Gain |
|-------|-------------|----------|
| **Batch Normalization** | After every conv layer | +2% |
| **High Resolution** | Train classifier at 448×448 | +4% |
| **Anchor Boxes** | Use predefined boxes (like Faster R-CNN) | +recall |
| **Dimension Clusters** | K-means on dataset to find best anchors | +5% |
| **Direct Location** | Predict relative to grid cell (sigmoid) | +stability |
| **Multi-Scale Training** | Random input size (320-608) every 10 batches | +robustness |
| **Darknet-19** | New lighter backbone (19 conv layers) | +speed |
| **Passthrough Layer** | Bring fine-grained features to detection | +1% small objects |

### YOLOv2 Architecture: Darknet-19

```
Input: 416 × 416 × 3

Conv layers (19 conv + 5 maxpool):
416×416 → 208×208 → 104×104 → 52×52 → 26×26 → 13×13

Output: 13 × 13 × (5 × (5 + 20)) = 13 × 13 × 125

  13×13 grid (finer than 7×7!)
  5 anchor boxes per cell
  Each anchor: [tx, ty, tw, th, objectness] + 20 classes
```

### YOLO9000: Detect 9000 Object Types!

```
Key Idea: Jointly train on ImageNet (classification, 9000 classes) 
          + COCO (detection, 80 classes)

WordTree: Hierarchical class structure
  - "dog" is a child of "animal"
  - "golden retriever" is a child of "dog"
  - Predict at multiple hierarchy levels
  
Result: Can detect 9000+ categories!
(But accuracy on rare classes is low)
```

### YOLOv2 Performance

| Model | Input | mAP (VOC) | FPS |
|-------|-------|-----------|-----|
| YOLOv2 | 416×416 | 76.8% | 67 |
| YOLOv2 | 544×544 | 78.6% | 40 |
| Faster R-CNN | ~1000×600 | 73.2% | 7 |
| SSD512 | 512×512 | 76.8% | 19 |

> YOLOv2: Higher accuracy than Faster R-CNN at 10× the speed!

---

## 4. YOLOv3 — The Practical Workhorse (2018)

### 💡 "The One That Made YOLO Production-Ready"

> YOLOv3 introduced multi-scale detection (detecting at 3 different scales), making it actually good at small objects. This is where YOLO became PRACTICAL for real work.

### Key Innovations

| Feature | Description |
|---------|------------|
| **Darknet-53** | Deeper backbone (53 layers, residual connections) |
| **Multi-scale detection** | Predict at 3 scales (13×13, 26×26, 52×52) |
| **Better anchors** | 9 anchors (3 per scale), from k-means clustering |
| **Independent logistic classifiers** | Multi-label (object can be "person" AND "athlete") |
| **No softmax** | Binary cross-entropy per class (multi-label) |

### YOLOv3 Architecture (Important!)

```
Input: 416 × 416 × 3

BACKBONE: Darknet-53
  ├── 53 convolutional layers
  ├── Residual connections (like ResNet)
  └── No fully-connected layers

NECK: Feature Pyramid (FPN-like)
  ├── Upsample deep features
  └── Concatenate with earlier layers

DETECTION AT 3 SCALES:
  ├── Scale 1: 13×13  → Detect LARGE objects  (3 anchors)
  ├── Scale 2: 26×26  → Detect MEDIUM objects (3 anchors)
  └── Scale 3: 52×52  → Detect SMALL objects  (3 anchors)

Total predictions: (13² + 26² + 52²) × 3 = 10,647 boxes!

Each prediction: [tx, ty, tw, th, objectness, class1, class2, ..., class80]
  = 4 + 1 + 80 = 85 values per box (for COCO)
```

### Visual: Multi-Scale Detection

```
Image with objects of different sizes:
┌─────────────────────────────────┐
│                                 │
│    🚗 (large)                   │
│                    🐕 (medium)  │
│  ✈️ (tiny in sky)              │
│                                 │
└─────────────────────────────────┘

Detection scales:
13×13 grid → catches 🚗 (large objects, stride 32)
26×26 grid → catches 🐕 (medium objects, stride 16)  
52×52 grid → catches ✈️ (small objects, stride 8)
```

### YOLOv3 Anchor Boxes (COCO)

```
Scale 1 (13×13, large objects):   (116,90), (156,198), (373,326)
Scale 2 (26×26, medium objects):  (30,61),  (62,45),   (59,119)
Scale 3 (52×52, small objects):   (10,13),  (16,30),   (33,23)

These are found by running k-means clustering on COCO bounding boxes!
```

### YOLOv3 Performance

| Model | mAP-50 (COCO) | mAP (COCO) | FPS (Titan X) |
|-------|---------------|------------|---------------|
| YOLOv3-416 | 55.3% | 31.0% | 35 |
| YOLOv3-608 | 57.9% | 33.0% | 20 |
| RetinaNet-101 | 57.5% | 34.4% | 5 |
| Faster R-CNN+FPN | 59.1% | 36.2% | 6 |

### 💡 Joseph Redmon's Farewell

```
After YOLOv3, Joseph Redmon (YOLO creator) quit computer vision research,
citing ethical concerns about military use and privacy implications.

His final paper ended with:
"I stopped doing CV research because I saw the impact my work was having... 
 I have fun with the work but the military applications worry me."

This led to YOLO development being continued by OTHERS:
- YOLOv4: Alexey Bochkovskiy (Darknet maintainer)
- YOLOv5-v11: Ultralytics (Glenn Jocher)
```

---

## 5. YOLOv4 — Bag of Freebies & Specials (2020)

### 💡 "A Systematic Study of Every Trick That Works"

> YOLOv4 by Alexey Bochkovskiy wasn't just a new model — it was a COMPREHENSIVE study of which tricks improve detection. It categorized them into "freebies" (no cost) and "specials" (slight cost).

### Bag of Freebies (Improve accuracy, NO speed cost)

| Trick | Category | Effect |
|-------|----------|--------|
| CutMix augmentation | Data augmentation | Better generalization |
| Mosaic augmentation | Data augmentation | See 4 images at once! |
| DropBlock | Regularization | Prevents overfitting |
| Label Smoothing | Training | More robust predictions |
| CIoU Loss | Loss function | Better box regression |
| Self-Adversarial Training | Training | Harder training examples |

### Bag of Specials (Improve accuracy, SMALL speed cost)

| Trick | Category | Effect |
|-------|----------|--------|
| CSPNet backbone | Architecture | Better gradient flow |
| Mish activation | Activation | Smoother gradients |
| SPP (Spatial Pyramid Pooling) | Neck | Multi-scale context |
| PANet | Neck | Better feature aggregation |
| DIoU-NMS | Post-processing | Better duplicate removal |

### 🔥 Mosaic Augmentation (Game-Changer!)

```
Instead of training on 1 image, combine 4 images into ONE:

┌──────────┬──────────┐
│  Image1  │  Image2  │
│  (cats)  │ (street) │
├──────────┼──────────┤
│  Image3  │  Image4  │
│  (park)  │ (office) │
└──────────┴──────────┘

Benefits:
✅ Sees 4× more context per batch
✅ Reduces need for large batch sizes
✅ Better for small object detection
✅ Free augmentation (no inference cost)
```

### YOLOv4 Architecture

```
BACKBONE: CSPDarknet53 (Cross-Stage Partial connections)
NECK:     SPP + PANet (Path Aggregation Network)
HEAD:     YOLOv3-style (3 scales)

CSPDarknet53 = Darknet53 + Cross-Stage Partial connections
  → Reduces computation while maintaining accuracy
  → Better gradient flow

SPP Block:
  Input → MaxPool(5×5) ──┐
  Input → MaxPool(9×9) ──┼── Concatenate → Output
  Input → MaxPool(13×13)─┘
  Input ─────────────────┘
  
  → Captures multi-scale local context without resizing
```

### YOLOv4 Performance

| Model | AP (COCO) | FPS (V100) |
|-------|-----------|------------|
| YOLOv3 | 33.0% | 35 |
| EfficientDet-D2 | 43.0% | 26 |
| **YOLOv4** | **43.5%** | **62** |
| YOLOv4-CSP | 47.5% | 73 |

> YOLOv4 matched EfficientDet accuracy at 2.5× the speed!

---

## 6. YOLOv5 — Ultralytics Era Begins (2020)

### 💡 "Making YOLO Accessible to Everyone"

> YOLOv5 by Glenn Jocher (Ultralytics) wasn't a research paper — it was a SOFTWARE PRODUCT. Written in PyTorch (not Darknet), with incredible UX, documentation, and ecosystem.

### ⚠️ The Controversy

```
YOLOv5 was released DAYS after YOLOv4, causing controversy:
- No paper published
- Named "v5" without community consensus
- Some argued it should be called "YOLOv4-PyTorch"

BUT: It won by ADOPTION. The developer experience was SO good 
     that everyone just started using it.

Today: Ultralytics is the de-facto YOLO standard.
```

### Why YOLOv5 Won (Developer Experience)

```python
# YOLOv4 (Darknet): Compile C code, edit .cfg files, complex commands
./darknet detector train data/obj.data cfg/yolov4.cfg yolov4.conv.137

# YOLOv5 (Ultralytics): pip install + 3 lines of Python
pip install ultralytics
from ultralytics import YOLO
model = YOLO('yolov5s.pt')
results = model('image.jpg')
```

### YOLOv5 Model Sizes

```
YOLOv5 comes in 5 sizes (like T-shirt sizing):

┌──────────────────────────────────────────────────────┐
│  Model    │ Params │  mAP   │   FPS    │  Use Case  │
├───────────┼────────┼────────┼──────────┼────────────┤
│  YOLOv5n  │  1.9M  │ 28.0%  │  Ultra   │  Mobile    │
│  YOLOv5s  │  7.2M  │ 37.4%  │  Fast    │  Edge      │
│  YOLOv5m  │ 21.2M  │ 45.4%  │  Medium  │  Balanced  │
│  YOLOv5l  │ 46.5M  │ 49.0%  │  Slower  │  Accuracy  │
│  YOLOv5x  │ 86.7M  │ 50.7%  │  Slowest │  Research  │
└──────────────────────────────────────────────────────┘

Rule: n < s < m < l < x (size/accuracy increasing, speed decreasing)
```

### YOLOv5 Key Features

```
✅ PyTorch native (easy to modify, debug)
✅ Auto-anchor (automatically finds best anchors for your dataset)
✅ Auto-augmentation (Mosaic, MixUp, HSV, flip, scale)
✅ Multi-GPU training (DDP)
✅ Weights & Biases integration
✅ TensorBoard logging
✅ Export to 12+ formats
✅ Hyperparameter evolution (auto-tune)
✅ Comet, ClearML integration
✅ REST API for serving
```

---

## 7. YOLOv6, v7 — Speed Wars (2022)

### YOLOv6 (by Meituan, June 2022)

```
Focus: INDUSTRIAL deployment speed (especially for Meituan's delivery robots)

Key innovations:
- EfficientRep backbone (hardware-aware design)
- Rep-PAN neck (re-parameterizable)
- Task Alignment Learning (TAL) for label assignment
- Self-distillation training strategy

Performance (COCO):
  YOLOv6-N: 37.5% AP, 1187 FPS (T4 TensorRT)
  YOLOv6-S: 45.0% AP, 484 FPS
  YOLOv6-M: 50.0% AP, 226 FPS
  YOLOv6-L: 52.8% AP, 116 FPS
```

### YOLOv7 (by WongKinYiu, July 2022)

```
Focus: SOTA accuracy + speed without increasing cost

Key innovations:
- E-ELAN: Extended efficient layer aggregation
- Compound model scaling (scale width/depth/resolution together)
- Re-parameterized convolutions
- Coarse-to-fine auxiliary head training
- Planned re-parameterized model

Performance (COCO):
  YOLOv7: 51.4% AP at 161 FPS (V100)
  YOLOv7-X: 53.1% AP at 114 FPS
  YOLOv7-E6: 55.9% AP at 56 FPS

Paper: "YOLOv7: Trainable bag-of-freebies sets new state-of-the-art 
        for real-time object detectors"
```

### The "Version War" Problem

```
2022 was chaotic for YOLO:
- YOLOv6 (Meituan) - June 2022
- YOLOv7 (WongKinYiu) - July 2022  
- YOLO-NAS (Deci.ai) - 2023
- Gold-YOLO - 2023

Each claimed "state-of-the-art" but used different benchmarks/settings!

Resolution: Ultralytics unified everything with YOLOv8 (Jan 2023)
            → Single ecosystem, consistent benchmarking
```

---

## 8. YOLOv8 — The Modern Standard (2023)

### 💡 "The Most Important YOLO for Practitioners"

> **YOLOv8** is the version you should learn and use TODAY. It's anchor-free, unified (detect/segment/pose/classify), has the best developer experience, and is the current industry standard.

### 🔥 What Makes YOLOv8 Special

| Feature | YOLOv5 (old) | YOLOv8 (new) |
|---------|-------------|-------------|
| Anchors | Anchor-based | **Anchor-free** |
| Head | Coupled head | **Decoupled head** |
| Label assignment | IoU-based | **Task-Aligned Assignment** |
| Loss | CIoU + BCE | **DFL + CIoU + BCE** |
| Tasks | Detection only | **Detect + Segment + Pose + Classify + OBB** |
| API | `model.train()` | Same + CLI + Python + REST |

### YOLOv8 Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        YOLOv8 ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  INPUT (640×640×3)                                               │
│       │                                                           │
│       ▼                                                           │
│  ┌──────────────┐                                                │
│  │  BACKBONE    │  CSPDarknet (C2f blocks instead of C3)         │
│  │  (Feature    │  • Faster than YOLOv5's C3 blocks              │
│  │   Extractor) │  • Better gradient flow                        │
│  └──────┬───────┘                                                │
│         │ Multi-scale features (P3, P4, P5)                      │
│         ▼                                                         │
│  ┌──────────────┐                                                │
│  │    NECK      │  PANet (Path Aggregation Network)              │
│  │  (Feature    │  • Top-down + Bottom-up paths                  │
│  │   Fusion)    │  • C2f blocks for feature fusion               │
│  └──────┬───────┘                                                │
│         │                                                         │
│         ▼                                                         │
│  ┌──────────────────────────────────────────┐                    │
│  │         DECOUPLED HEAD (NEW!)             │                    │
│  │                                           │                    │
│  │  ┌──────────────┐   ┌──────────────┐    │                    │
│  │  │Classification│   │  Regression  │    │ ← SEPARATE heads!  │
│  │  │    Branch    │   │    Branch    │    │   (not shared)      │
│  │  │              │   │              │    │                      │
│  │  │  Conv→Conv   │   │  Conv→Conv   │    │                    │
│  │  │  →Classes    │   │  →BBox+DFL   │    │                    │
│  │  └──────────────┘   └──────────────┘    │                    │
│  └──────────────────────────────────────────┘                    │
│                                                                   │
│  OUTPUT: Boxes (cx, cy, w, h) + Class probabilities              │
│          → No objectness score! (anchor-free)                    │
│          → Distribution Focal Loss for box regression            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Anchor-Free Detection (YOLOv8's Approach)

```
OLD (Anchor-based, YOLOv5):
  Predefined anchors → Predict offsets from anchors → Box
  Problems: Anchor design is dataset-specific, complex

NEW (Anchor-free, YOLOv8):
  Each grid point → DIRECTLY predict center (cx, cy) + size (w, h) → Box
  Benefits: Simpler, no anchor tuning, better generalization

How it works:
  For each point on the feature map:
  1. Predict: "How far is the object center from this point?" (regression)
  2. Predict: "How wide/tall is the object?" (regression)
  3. Predict: "What class is it?" (classification)
  
  No predefined box shapes needed!
```

### YOLOv8 Model Variants

| Model | Params | mAP^val 50-95 | Speed (A100 TensorRT) | Use Case |
|-------|--------|---------------|----------------------|----------|
| YOLOv8n | 3.2M | 37.3% | 0.99ms | Mobile, Edge |
| YOLOv8s | 11.2M | 44.9% | 1.20ms | Edge, Jetson |
| YOLOv8m | 25.9M | 50.2% | 1.83ms | Balanced |
| YOLOv8l | 43.7M | 52.9% | 2.39ms | High accuracy |
| YOLOv8x | 68.2M | 53.9% | 3.53ms | Maximum accuracy |

### 📝 Code: YOLOv8 in 5 Lines

```python
from ultralytics import YOLO

# Load model (auto-downloads weights)
model = YOLO("yolov8n.pt")  # nano model

# Predict on an image
results = model("street.jpg")

# Show results
results[0].show()  # Display with boxes drawn
```

### 📝 Code: YOLOv8 Complete Inference

```python
from ultralytics import YOLO
import cv2

# Load model
model = YOLO("yolov8m.pt")  # medium model for balance

# Run inference
results = model("street.jpg", conf=0.5, iou=0.45)

# Access results
for result in results:
    boxes = result.boxes  # Boxes object
    
    # Get box coordinates
    print(f"Boxes (xyxy): {boxes.xyxy}")      # [x1, y1, x2, y2]
    print(f"Boxes (xywh): {boxes.xywh}")      # [cx, cy, w, h]
    print(f"Confidence:   {boxes.conf}")       # confidence scores
    print(f"Class IDs:    {boxes.cls}")        # class indices
    
    # Get class names
    for box in boxes:
        cls_id = int(box.cls[0])
        conf = float(box.conf[0])
        coords = box.xyxy[0].tolist()
        name = model.names[cls_id]
        print(f"  {name}: {conf:.2f} at {coords}")

# Save annotated image
results[0].save("output.jpg")

# Or get the plotted image as numpy array
annotated = results[0].plot()
cv2.imshow("YOLOv8", annotated)
cv2.waitKey(0)
```

### 📝 Code: YOLOv8 on Video (Real-Time)

```python
from ultralytics import YOLO
import cv2

model = YOLO("yolov8n.pt")  # Use nano for real-time

# Open video or webcam
cap = cv2.VideoCapture(0)  # 0 for webcam, or "video.mp4"

while cap.isOpened():
    ret, frame = cap.read()
    if not ret:
        break
    
    # Run detection
    results = model(frame, stream=True, conf=0.5)
    
    for result in results:
        annotated = result.plot()
        cv2.imshow("YOLOv8 Real-Time", annotated)
    
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
```

---

## 9. YOLOv9, v10, v11 — Latest Innovations (2024)

### YOLOv9 (Feb 2024) — Programmable Gradient Information

```
Key Innovation: PGI (Programmable Gradient Information)
  → Solves the "information bottleneck" problem in deep networks
  → Information gets LOST as it passes through many layers
  → PGI adds auxiliary reversible branches to preserve information

Architecture: GELAN (Generalized Efficient Layer Aggregation Network)
  → More flexible than CSP blocks
  → Can use various computation blocks

Performance:
  YOLOv9-C: 53.0% AP, 7.6M params
  YOLOv9-E: 55.6% AP, 57.3M params
  
  Better accuracy than YOLOv8 with FEWER parameters!
```

### YOLOv10 (May 2024) — NMS-Free YOLO

```
Key Innovation: Eliminated NMS entirely!
  → Consistent Dual Assignments during training
  → One-to-many assignment for learning + One-to-one for inference
  → No NMS post-processing needed = lower latency!

Other innovations:
  → Spatial-channel decoupled downsampling
  → Rank-guided block design
  → Large-kernel convolutions

Performance:
  YOLOv10-N: 38.5% AP, 2.3M params, 1.84ms latency
  YOLOv10-S: 46.3% AP, 7.2M params, 2.49ms
  YOLOv10-M: 51.1% AP, 15.4M params, 4.74ms
  YOLOv10-L: 53.2% AP, 24.4M params, 7.28ms
  YOLOv10-X: 54.4% AP, 29.5M params, 10.70ms
```

### YOLOv11 (Oct 2024) — Ultralytics Latest

```
Key Innovations:
  → C3k2 blocks (improved C2f with kernel-size 2 efficiency)
  → SPPF → Enhanced spatial pyramid pooling
  → C2PSA (Cross-Stage Partial with Spatial Attention)
  → Improved attention mechanisms
  → Better small object detection

Performance:
  YOLO11n: 39.5% AP, 2.6M params, 1.5ms
  YOLO11s: 47.0% AP, 9.4M params, 2.5ms
  YOLO11m: 51.5% AP, 20.1M params, 4.7ms
  YOLO11l: 53.4% AP, 25.3M params, 6.2ms
  YOLO11x: 54.7% AP, 56.9M params, 11.3ms

Supports: Detection, Segmentation, Pose, OBB, Classification
```

### 📝 Code: Using Latest YOLO Models

```python
from ultralytics import YOLO

# YOLOv8
model_v8 = YOLO("yolov8n.pt")

# YOLOv9
model_v9 = YOLO("yolov9c.pt")

# YOLOv10 (NMS-free!)
model_v10 = YOLO("yolov10n.pt")

# YOLOv11 (latest)
model_v11 = YOLO("yolo11n.pt")

# They ALL use the same API!
results = model_v11("image.jpg")
results[0].show()
```

### Version Comparison (Which to Use?)

| Version | Best For | Key Advantage |
|---------|----------|---------------|
| YOLOv5 | Legacy projects, stability | Most battle-tested |
| YOLOv8 | **Most projects (recommended)** | Best balance, great docs |
| YOLOv9 | Maximum accuracy per param | PGI, fewer params for same AP |
| YOLOv10 | Low-latency deployment | No NMS overhead |
| YOLOv11 | Latest & greatest | Best accuracy, attention mechanisms |

### 💡 Recommendation

```
🏆 For most users:           Use YOLOv8 (most stable, best ecosystem)
🏆 For cutting-edge:         Use YOLO11 (latest improvements)  
🏆 For minimal latency:      Use YOLOv10 (NMS-free)
🏆 For accuracy research:    Use YOLOv9 (PGI innovation)
```

---

## 10. Ultralytics Library — Complete Usage Guide

### Installation

```bash
# Install ultralytics (includes YOLOv5, v8, v9, v10, v11)
pip install ultralytics

# Verify installation
yolo check

# Check version
python -c "import ultralytics; print(ultralytics.__version__)"
```

### Three Ways to Use YOLO

#### Method 1: CLI (Command Line Interface)

```bash
# Predict
yolo predict model=yolov8n.pt source="image.jpg" conf=0.5

# Train
yolo train model=yolov8n.pt data=coco128.yaml epochs=100 imgsz=640

# Validate
yolo val model=yolov8n.pt data=coco.yaml

# Export
yolo export model=yolov8n.pt format=onnx

# Track (video object tracking)
yolo track model=yolov8n.pt source="video.mp4" tracker=bytetrack.yaml
```

#### Method 2: Python API (Most Common)

```python
from ultralytics import YOLO

# ─── PREDICT ───
model = YOLO("yolov8n.pt")
results = model("image.jpg")
results = model(["img1.jpg", "img2.jpg"])         # Batch
results = model("video.mp4", stream=True)          # Video
results = model(0)                                  # Webcam
results = model("screen")                           # Screenshot
results = model("https://example.com/image.jpg")   # URL

# ─── TRAIN ───
model = YOLO("yolov8n.pt")
results = model.train(data="dataset.yaml", epochs=100, imgsz=640, batch=16)

# ─── VALIDATE ───
results = model.val(data="dataset.yaml")

# ─── EXPORT ───
model.export(format="onnx")       # ONNX
model.export(format="torchscript")
model.export(format="engine")     # TensorRT
model.export(format="coreml")     # Apple CoreML
model.export(format="tflite")     # TensorFlow Lite

# ─── TRACK ───
results = model.track("video.mp4", persist=True, tracker="bytetrack.yaml")
```

#### Method 3: REST API

```python
# Start server
# yolo serve model=yolov8n.pt

# Client code
import requests

url = "http://localhost:5000/predict"
files = {"image": open("test.jpg", "rb")}
response = requests.post(url, files=files)
predictions = response.json()
```

### Detailed Results Object

```python
from ultralytics import YOLO

model = YOLO("yolov8m.pt")
results = model("street.jpg")

# Access everything from results
result = results[0]

# ─── Bounding Boxes ───
print(result.boxes.xyxy)    # [x1, y1, x2, y2] tensor
print(result.boxes.xywh)    # [cx, cy, w, h] tensor
print(result.boxes.xywhn)   # normalized [cx, cy, w, h]
print(result.boxes.xyxyn)   # normalized [x1, y1, x2, y2]
print(result.boxes.conf)    # confidence scores
print(result.boxes.cls)     # class indices
print(result.boxes.data)    # all box data combined

# ─── Image Info ───
print(result.orig_img.shape)  # Original image (numpy)
print(result.orig_shape)      # (height, width)
print(result.path)            # Image path

# ─── Class Names ───
print(result.names)           # {0: 'person', 1: 'bicycle', ...}

# ─── Visualization ───
annotated = result.plot()     # Get annotated image as numpy array
result.show()                 # Display in window
result.save("output.jpg")     # Save to file

# ─── Convert to different formats ───
boxes_list = result.boxes.xyxy.tolist()  # Python list
boxes_numpy = result.boxes.xyxy.numpy()  # NumPy array
boxes_pandas = result.pandas()            # Pandas DataFrame

# ─── Speed info ───
print(result.speed)  # {'preprocess': 1.5, 'inference': 3.2, 'postprocess': 1.1}
```

---

## 11. Training YOLO on Custom Dataset (Step-by-Step)

### 🔥 This is what 90% of YOLO users actually need to do!

### Step 1: Prepare Your Dataset

```
Dataset structure (YOLO format):
my_dataset/
├── images/
│   ├── train/
│   │   ├── img001.jpg
│   │   ├── img002.jpg
│   │   └── ...
│   └── val/
│       ├── img100.jpg
│       └── ...
├── labels/
│   ├── train/
│   │   ├── img001.txt    ← One .txt per image
│   │   ├── img002.txt
│   │   └── ...
│   └── val/
│       ├── img100.txt
│       └── ...
└── dataset.yaml
```

### Step 2: Label Format (YOLO Format)

```
Each .txt file contains ONE line per object:
<class_id> <center_x> <center_y> <width> <height>

All values are NORMALIZED (0.0 to 1.0 relative to image size)

Example (img001.txt):
0 0.5 0.4 0.3 0.6    ← class 0, center at (50%, 40%), size (30%, 60%)
1 0.2 0.7 0.15 0.2   ← class 1, center at (20%, 70%), size (15%, 20%)
2 0.8 0.3 0.1 0.1    ← class 2, center at (80%, 30%), size (10%, 10%)

If image has NO objects → empty .txt file (or no file)
```

### Step 3: Create dataset.yaml

```yaml
# dataset.yaml
path: /path/to/my_dataset    # Root directory
train: images/train           # Relative to path
val: images/val               # Relative to path
test: images/test             # Optional

# Classes
names:
  0: helmet
  1: no_helmet
  2: person

# OR simply:
# names: ['helmet', 'no_helmet', 'person']

# Number of classes (auto-detected from names, but good to specify)
nc: 3
```

### Step 4: Labeling Tools

| Tool | Type | Best For | Link |
|------|------|----------|------|
| **Roboflow** | Cloud | Easiest, auto-export YOLO format | roboflow.com |
| **CVAT** | Cloud/Self-host | Teams, video annotation | cvat.ai |
| **Label Studio** | Self-host | ML-assisted labeling | labelstud.io |
| **LabelImg** | Desktop | Simple, lightweight | GitHub |
| **V7 (Darwin)** | Cloud | Auto-labeling with AI | v7labs.com |

### Step 5: Train!

```python
from ultralytics import YOLO

# Start from pre-trained COCO model (transfer learning!)
model = YOLO("yolov8m.pt")  # medium model

# Train on your custom dataset
results = model.train(
    data="dataset.yaml",
    epochs=100,
    imgsz=640,
    batch=16,
    name="my_experiment",
    
    # ─── Key Hyperparameters ───
    lr0=0.01,              # Initial learning rate
    lrf=0.01,              # Final learning rate (lr0 * lrf)
    momentum=0.937,        # SGD momentum
    weight_decay=0.0005,   # L2 regularization
    warmup_epochs=3.0,     # Warmup epochs
    
    # ─── Augmentation ───
    hsv_h=0.015,           # HSV-Hue augmentation
    hsv_s=0.7,             # HSV-Saturation augmentation
    hsv_v=0.4,             # HSV-Value augmentation
    degrees=0.0,           # Rotation (+/- deg)
    translate=0.1,         # Translation (+/- fraction)
    scale=0.5,             # Scale (+/- gain)
    flipud=0.0,            # Flip up-down probability
    fliplr=0.5,            # Flip left-right probability
    mosaic=1.0,            # Mosaic augmentation probability
    mixup=0.0,             # MixUp augmentation probability
    
    # ─── Device ───
    device=0,              # GPU 0 (or "cpu", or "0,1" for multi-GPU)
    
    # ─── Other ───
    patience=50,           # Early stopping patience
    save=True,             # Save checkpoints
    plots=True,            # Save training plots
)
```

### Step 6: Evaluate & Use

```python
# Validate on test set
metrics = model.val(data="dataset.yaml")
print(f"mAP50: {metrics.box.map50:.3f}")
print(f"mAP50-95: {metrics.box.map:.3f}")

# Predict on new images
model = YOLO("runs/detect/my_experiment/weights/best.pt")
results = model("new_image.jpg", conf=0.5)
results[0].show()
```

### Step 7: Training Outputs

```
After training, you'll find:
runs/detect/my_experiment/
├── weights/
│   ├── best.pt          ← Best model (highest mAP)
│   └── last.pt          ← Last epoch model
├── results.csv          ← Metrics per epoch
├── results.png          ← Training curves plot
├── confusion_matrix.png ← Confusion matrix
├── F1_curve.png         ← F1 vs confidence
├── P_curve.png          ← Precision vs confidence
├── R_curve.png          ← Recall vs confidence
├── PR_curve.png         ← Precision-Recall curve
├── val_batch0_pred.jpg  ← Sample predictions
└── args.yaml            ← Training config
```

### 📝 Code: Convert Other Formats to YOLO

```python
# Convert COCO JSON to YOLO format
import json
import os

def coco_to_yolo(coco_json_path, output_dir, image_width, image_height):
    """Convert COCO format annotations to YOLO format."""
    with open(coco_json_path) as f:
        coco = json.load(f)
    
    os.makedirs(output_dir, exist_ok=True)
    
    # Create image_id → annotations mapping
    img_anns = {}
    for ann in coco['annotations']:
        img_id = ann['image_id']
        if img_id not in img_anns:
            img_anns[img_id] = []
        img_anns[img_id].append(ann)
    
    # Convert each image's annotations
    for img in coco['images']:
        img_id = img['id']
        img_w = img['width']
        img_h = img['height']
        
        # Create label file
        label_file = os.path.join(output_dir, 
                                   img['file_name'].replace('.jpg', '.txt'))
        
        lines = []
        if img_id in img_anns:
            for ann in img_anns[img_id]:
                # COCO bbox: [x, y, width, height] (absolute, top-left)
                x, y, w, h = ann['bbox']
                
                # Convert to YOLO: [cx, cy, w, h] (normalized, center)
                cx = (x + w/2) / img_w
                cy = (y + h/2) / img_h
                nw = w / img_w
                nh = h / img_h
                
                class_id = ann['category_id'] - 1  # COCO is 1-indexed
                lines.append(f"{class_id} {cx:.6f} {cy:.6f} {nw:.6f} {nh:.6f}")
        
        with open(label_file, 'w') as f:
            f.write('\n'.join(lines))

# Usage
coco_to_yolo("annotations.json", "labels/train/", 640, 480)
```

---

## 12. YOLO for Different Tasks (Detect, Segment, Pose, OBB)

### YOLOv8/v11 Supports 5 Tasks!

```
┌─────────────────────────────────────────────────────────────┐
│                     YOLO MULTI-TASK                          │
├──────────────┬──────────────┬──────────────┬───────────────┤
│  DETECTION   │ SEGMENTATION │    POSE      │     OBB       │
│              │              │              │               │
│  ┌──────┐   │  ┌──────┐   │   ○          │  ╱──────╲     │
│  │ BOX  │   │  │██████│   │  /|\         │ ╱  BOX   ╲    │
│  │      │   │  │██████│   │  / \         │ ╲ ROTATED╱    │
│  └──────┘   │  └──────┘   │  Keypoints   │  ╲──────╱     │
│              │  (pixel mask)│              │  (angled box) │
├──────────────┼──────────────┼──────────────┼───────────────┤
│ yolov8n.pt   │yolov8n-seg.pt│yolov8n-pose.pt│yolov8n-obb.pt│
└──────────────┴──────────────┴──────────────┴───────────────┘
```

### Task 1: Object Detection (Default)

```python
from ultralytics import YOLO

model = YOLO("yolov8n.pt")
results = model("image.jpg")

# Access boxes
for box in results[0].boxes:
    print(f"{model.names[int(box.cls)]}: {float(box.conf):.2f}")
```

### Task 2: Instance Segmentation

```python
from ultralytics import YOLO

# Load segmentation model
model = YOLO("yolov8n-seg.pt")
results = model("image.jpg")

# Access masks
for result in results:
    if result.masks is not None:
        masks = result.masks.data        # (N, H, W) binary masks
        boxes = result.boxes.xyxy        # Corresponding boxes
        classes = result.boxes.cls       # Class IDs
        
        # Get mask for first detection
        mask_0 = masks[0].cpu().numpy()  # Binary mask (H×W)
        print(f"Mask shape: {mask_0.shape}")

# Train segmentation on custom data
model = YOLO("yolov8n-seg.pt")
model.train(data="seg_dataset.yaml", epochs=100, task="segment")
```

### Task 3: Pose Estimation (Keypoints)

```python
from ultralytics import YOLO

# Load pose model (COCO: 17 keypoints per person)
model = YOLO("yolov8n-pose.pt")
results = model("person.jpg")

# Access keypoints
for result in results:
    if result.keypoints is not None:
        keypoints = result.keypoints.data  # (N, 17, 3) → [x, y, confidence]
        
        # 17 COCO keypoints:
        # 0: nose, 1: left_eye, 2: right_eye, 3: left_ear, 4: right_ear
        # 5: left_shoulder, 6: right_shoulder, 7: left_elbow, 8: right_elbow
        # 9: left_wrist, 10: right_wrist, 11: left_hip, 12: right_hip
        # 13: left_knee, 14: right_knee, 15: left_ankle, 16: right_ankle
        
        for person_kpts in keypoints:
            nose = person_kpts[0]  # [x, y, conf]
            print(f"Nose at: ({nose[0]:.0f}, {nose[1]:.0f}), conf: {nose[2]:.2f}")
```

### Task 4: Oriented Bounding Boxes (OBB)

```python
from ultralytics import YOLO

# OBB for rotated objects (aerial/satellite imagery, text detection)
model = YOLO("yolov8n-obb.pt")
results = model("satellite.jpg")

# Access oriented boxes
for result in results:
    if result.obb is not None:
        # OBB format: [cx, cy, w, h, angle]
        obb_data = result.obb.data
        print(f"Oriented boxes: {obb_data.shape}")

# Great for:
# - Aerial/satellite imagery (buildings at angles)
# - Document text detection (rotated text)
# - Industrial measurement (angled parts)
```

### Task 5: Image Classification

```python
from ultralytics import YOLO

# Classification (NOT detection — whole image classification)
model = YOLO("yolov8n-cls.pt")
results = model("cat.jpg")

# Access predictions
for result in results:
    probs = result.probs  # Probabilities for each class
    top5 = probs.top5     # Top 5 class indices
    top5_conf = probs.top5conf  # Top 5 confidences
    
    print(f"Top prediction: {model.names[probs.top1]} ({probs.top1conf:.2f})")
```

---

## 13. YOLO Deployment (Export & Optimize)

### Export Formats

```python
from ultralytics import YOLO

model = YOLO("yolov8n.pt")

# Export to various formats
model.export(format="onnx")          # ONNX (universal)
model.export(format="torchscript")   # PyTorch mobile
model.export(format="engine")        # TensorRT (NVIDIA GPU)
model.export(format="coreml")        # Apple iOS/macOS
model.export(format="tflite")        # Android/Edge
model.export(format="openvino")      # Intel hardware
model.export(format="ncnn")          # Mobile (C++)
model.export(format="paddle")        # PaddlePaddle
```

### Supported Export Formats (Complete)

| Format | CLI Flag | File | Platform | Speed Boost |
|--------|----------|------|----------|-------------|
| PyTorch | - | .pt | GPU/CPU | Baseline |
| ONNX | onnx | .onnx | Cross-platform | 1.5-2× |
| TorchScript | torchscript | .torchscript | Mobile | 1.2× |
| **TensorRT** | engine | .engine | **NVIDIA GPU** | **3-5×** |
| **CoreML** | coreml | .mlpackage | **Apple** | 2-3× |
| **TFLite** | tflite | .tflite | **Android/Edge** | 2-4× |
| OpenVINO | openvino | _openvino/ | Intel CPU/GPU | 2-3× |
| NCNN | ncnn | .ncnn | Mobile C++ | 2× |
| Paddle | paddle | .pdmodel | Baidu ecosystem | - |

### 📝 Code: Deploy with TensorRT (Fastest on NVIDIA)

```python
from ultralytics import YOLO

# Export to TensorRT
model = YOLO("yolov8n.pt")
model.export(format="engine", half=True, imgsz=640)  # FP16 for max speed

# Use TensorRT model (same API!)
trt_model = YOLO("yolov8n.engine")
results = trt_model("image.jpg")  # 3-5× faster than PyTorch!
```

### 📝 Code: Deploy on Edge (Jetson Nano/Xavier)

```python
# On Jetson device:
from ultralytics import YOLO

# Export with TensorRT (optimized for Jetson)
model = YOLO("yolov8n.pt")
model.export(format="engine", half=True, imgsz=320)  # Smaller input for edge

# Run inference
model = YOLO("yolov8n.engine")
results = model(0)  # Webcam on Jetson
```

### 📝 Code: Deploy as REST API (Production)

```python
# server.py
from ultralytics import YOLO
from flask import Flask, request, jsonify
import cv2
import numpy as np

app = Flask(__name__)
model = YOLO("yolov8n.pt")

@app.route('/detect', methods=['POST'])
def detect():
    # Read image from request
    file = request.files['image']
    img_bytes = file.read()
    nparr = np.frombuffer(img_bytes, np.uint8)
    image = cv2.imdecode(nparr, cv2.IMREAD_COLOR)
    
    # Run detection
    results = model(image, conf=0.5)
    
    # Format response
    detections = []
    for box in results[0].boxes:
        detections.append({
            'class': model.names[int(box.cls)],
            'confidence': float(box.conf),
            'bbox': box.xyxy[0].tolist()
        })
    
    return jsonify({'detections': detections})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### Quantization (Make Models Smaller & Faster)

```python
# INT8 Quantization (4× smaller, 2-4× faster)
model = YOLO("yolov8n.pt")
model.export(
    format="engine",
    int8=True,              # INT8 quantization
    data="coco128.yaml",    # Calibration data needed for INT8
    imgsz=640
)

# The exported model is ~4× smaller with minimal accuracy loss!
# FP32: 6.2MB → INT8: 1.6MB
# Speed: ~2-4× faster on supported hardware
```

---

## 14. YOLO Tips, Tricks & Best Practices

### ✅ Training Best Practices

| Tip | Why |
|-----|-----|
| **Always start from pre-trained** | Transfer learning > training from scratch (3-5× faster, better accuracy) |
| **Use medium model (m) first** | Then scale up/down based on results |
| **640 input size** | Standard COCO size, good balance |
| **Batch size as large as GPU allows** | Better gradients, faster convergence |
| **Train for 100+ epochs** | YOLO needs time to converge (check for overfitting) |
| **Use patience=50** | Early stopping prevents wasting time |
| **Validate on holdout set** | NEVER train on validation data! |
| **At least 100 images/class** | More = better (1000+ is ideal) |
| **Include background images** | 10% of dataset should have NO objects (reduces FP) |
| **Diverse backgrounds** | Don't just photograph objects on white background |

### ⚠️ Common Mistakes

| Mistake | Fix |
|---------|-----|
| Too few images | Get at least 100-1000 per class |
| Images too similar | Add variety (lighting, angle, background) |
| Wrong label format | YOLO uses normalized center format, NOT corner! |
| Overfitting | Add augmentation, reduce model size, more data |
| Classes imbalanced | Oversample rare classes or use class weights |
| Wrong input size | Match training and inference `imgsz` |
| Forgot to freeze backbone | For small datasets, freeze backbone first |
| Confidence too low | Start with conf=0.5, adjust based on use case |

### 📝 Hyperparameter Tuning Cheat Sheet

```python
# For SMALL datasets (< 500 images):
model.train(
    data="data.yaml",
    epochs=300,          # More epochs needed
    imgsz=640,
    batch=8,             # Small batch for small dataset
    lr0=0.001,           # Lower learning rate
    freeze=10,           # Freeze first 10 backbone layers!
    augment=True,        # Heavy augmentation
    mosaic=1.0,
    mixup=0.1,
    degrees=10,
    scale=0.9,
    translate=0.2,
)

# For LARGE datasets (> 10,000 images):
model.train(
    data="data.yaml",
    epochs=100,
    imgsz=640,
    batch=32,            # Larger batch
    lr0=0.01,            # Standard learning rate
    freeze=0,            # Don't freeze (enough data)
    mosaic=1.0,
    close_mosaic=10,     # Disable mosaic last 10 epochs
)

# For REAL-TIME deployment:
model = YOLO("yolov8n.pt")  # Use nano!
model.train(
    data="data.yaml",
    epochs=200,
    imgsz=320,           # Smaller input = faster
    batch=64,
)
```

### 📝 Data Augmentation Visualization

```python
from ultralytics import YOLO
import cv2

# See what augmentation does to your images
model = YOLO("yolov8n.pt")
model.train(
    data="data.yaml",
    epochs=1,
    plots=True,  # Saves augmented batch visualizations!
)
# Check: runs/detect/train/train_batch0.jpg → shows mosaic + augmentations
```

### Improving mAP Checklist

```
□ More diverse training data (different angles, lighting, sizes)
□ Better labels (tight boxes, correct classes)
□ Include hard negatives (similar objects that are NOT the target)
□ Increase input size (640→1280) if GPU allows
□ Use larger model (n→s→m→l→x)
□ Train longer (more epochs, check if still improving)
□ Add background images (0-10% of dataset)
□ Adjust anchors (or use anchor-free YOLOv8+)
□ Use test-time augmentation (TTA): model.predict(augment=True)
□ Ensemble multiple models
□ Use multi-scale training
□ Fine-tune confidence/NMS thresholds for your use case
```

---

## 15. Real-World YOLO Projects & Company Use Cases

### 🏢 Case Study 1: Airbus — Satellite Object Detection

```
Challenge: Detect aircraft, ships, and vehicles in satellite imagery
Solution: YOLOv8 with OBB (oriented bounding boxes)
Why YOLO: Fast processing of thousands of satellite images daily
Setup:
  - Custom dataset: 50,000+ labeled satellite images
  - YOLOv8l-obb for maximum accuracy
  - Deployed on cloud GPU cluster
  - Processes new satellite passes in minutes
Result: 94% detection rate, 10× faster than manual analysis
```

### 🏢 Case Study 2: Walmart — Shelf Monitoring

```
Challenge: Detect out-of-stock products, wrong placements
Solution: YOLOv5/v8 on edge devices (cameras in aisles)
Pipeline:
  Camera → YOLOv8n (edge) → Detect products → Compare with planogram
  → Alert staff if: empty shelf, wrong product, price tag missing
  
Scale: 4,700+ stores, millions of detections/day
```

### 🏢 Case Study 3: Construction Safety

```
Challenge: Detect workers without helmets/vests in real-time
Solution: YOLOv8 for PPE (Personal Protective Equipment) detection
Classes: helmet, no_helmet, vest, no_vest, person
Hardware: NVIDIA Jetson on construction cameras
Speed: 30 FPS real-time alerting
Action: Alarm sounds if worker detected without PPE
```

### 🏢 Case Study 4: Agriculture — Crop Disease Detection

```
Challenge: Detect plant diseases early (before visible to human eye)
Solution: YOLOv8 on drone imagery
Pipeline:
  Drone (DJI) → Capture images → YOLOv8 → Detect diseased areas
  → Generate field map → Targeted spraying (saves 60% chemicals)
  
Model: YOLOv8m fine-tuned on 20,000 crop images
Accuracy: 91% disease detection rate
```

### 🏢 Case Study 5: Traffic Management

```
Challenge: Count vehicles, detect violations, manage congestion
Solution: YOLOv8 + ByteTrack (tracking)
Classes: car, truck, bus, motorcycle, bicycle, pedestrian

Pipeline:
  Traffic Camera → YOLOv8s → Detect vehicles → ByteTrack (track IDs)
  → Count per lane → Analyze speed → Detect red-light violations
  → Adjust traffic signal timing dynamically

Deployed: 2000+ intersections
Speed: YOLOv8n at 60 FPS per camera
```

### 📝 Project Template: Build Your Own YOLO Project

```python
"""
Template for any YOLO project.
Replace dataset and classes for your use case.
"""
from ultralytics import YOLO
import cv2

# ═══════════════════════════════════════════
# STEP 1: TRAIN
# ═══════════════════════════════════════════
model = YOLO("yolov8m.pt")  # Start from pre-trained

model.train(
    data="my_dataset.yaml",   # Your dataset config
    epochs=100,
    imgsz=640,
    batch=16,
    name="my_project_v1",
    patience=30,
)

# ═══════════════════════════════════════════
# STEP 2: EVALUATE
# ═══════════════════════════════════════════
best_model = YOLO("runs/detect/my_project_v1/weights/best.pt")
metrics = best_model.val()
print(f"mAP50: {metrics.box.map50:.3f}")
print(f"mAP50-95: {metrics.box.map:.3f}")

# ═══════════════════════════════════════════
# STEP 3: PREDICT
# ═══════════════════════════════════════════
results = best_model("test_image.jpg", conf=0.5)
results[0].show()

# ═══════════════════════════════════════════
# STEP 4: EXPORT & DEPLOY
# ═══════════════════════════════════════════
best_model.export(format="onnx")  # For production

# ═══════════════════════════════════════════
# STEP 5: REAL-TIME INFERENCE
# ═══════════════════════════════════════════
deployed_model = YOLO("runs/detect/my_project_v1/weights/best.onnx")

cap = cv2.VideoCapture(0)
while True:
    ret, frame = cap.read()
    results = deployed_model(frame, conf=0.5)
    cv2.imshow("Detection", results[0].plot())
    if cv2.waitKey(1) == ord('q'):
        break
cap.release()
```

---

## 📋 Chapter Summary

### YOLO Version Decision Guide

```
┌─────────────────────────────────────────────────────────┐
│            WHICH YOLO VERSION SHOULD I USE?              │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  "I'm learning YOLO for the first time"                │
│  → YOLOv8 (best docs, best ecosystem)                  │
│                                                         │
│  "I need maximum production stability"                  │
│  → YOLOv8 (most battle-tested of modern versions)      │
│                                                         │
│  "I need absolute latest accuracy"                      │
│  → YOLO11 (newest improvements)                        │
│                                                         │
│  "I need minimum latency (no NMS overhead)"            │
│  → YOLOv10 (NMS-free design)                           │
│                                                         │
│  "I have a legacy project on YOLOv5"                   │
│  → Stay on v5 OR migrate to v8 (same API)              │
│                                                         │
│  "I need segmentation too"                              │
│  → YOLOv8-seg or YOLO11-seg                            │
│                                                         │
│  "I need pose estimation"                               │
│  → YOLOv8-pose or YOLO11-pose                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### YOLO Model Size Selection

```
Mobile/Edge (Raspberry Pi, Jetson Nano):  → YOLOv8n (nano)
Edge GPU (Jetson Xavier/Orin):            → YOLOv8s (small)
Server (balanced):                         → YOLOv8m (medium)
Server (accuracy-critical):               → YOLOv8l/x (large/xlarge)
```

### 🔥 Interview Quick-Fire

| Question | Answer |
|----------|--------|
| "How does YOLO differ from R-CNN?" | YOLO: one-stage, single pass, grid-based. R-CNN: two-stage, proposal+classify |
| "What does YOLO stand for?" | You Only Look Once — processes image in single forward pass |
| "How does YOLOv1 work?" | Divide image into S×S grid, each cell predicts B boxes + C classes |
| "What's anchor-free detection?" | YOLOv8+ directly predicts center + size, no predefined box templates |
| "How to handle small objects in YOLO?" | Use larger input size (1280), add P2 layer, more small-object training data |
| "YOLOv8 vs YOLOv5?" | v8: anchor-free, decoupled head, better accuracy. v5: more stable, legacy |
| "How to deploy YOLO?" | Export to ONNX/TensorRT/CoreML → run on target hardware |
| "What is Mosaic augmentation?" | Combines 4 training images into 1, increases diversity without extra data |
| "YOLO limitations?" | Struggles with very small/dense objects, fixed input size, real-time needs GPU |

---

### ➡️ Next Chapter: [05 - EfficientDet](./05-EfficientDet.md)

> Now you've mastered YOLO! Next, we'll look at Google's EfficientDet — which takes a different approach to the speed-accuracy tradeoff using compound scaling and BiFPN.

---

*[← Back to Index](./00-INDEX.md) | [← Previous: 03 - DL Evolution](./03-DL-Evolution-RCNN-Family.md)*
