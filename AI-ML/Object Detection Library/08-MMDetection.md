# 📘 Chapter 08: MMDetection — The Largest Model Zoo in Object Detection

> **Goal**: Master OpenMMLab's MMDetection — the framework with 300+ pre-trained models covering virtually every object detection architecture ever published. If Detectron2 is the research standard, MMDetection is the research ENCYCLOPEDIA.

---

## 📑 Table of Contents

1. [What is MMDetection & The OpenMMLab Ecosystem](#1-what-is-mmdetection--the-openmml-ecosystem)
2. [Installation & Setup](#2-installation--setup)
3. [Architecture & Design Philosophy](#3-architecture--design-philosophy)
4. [Model Zoo — 300+ Pre-trained Models](#4-model-zoo--300-pre-trained-models)
5. [Quick Start: Detection in Minutes](#5-quick-start-detection-in-minutes)
6. [The Config System (MMDetection's Superpower)](#6-the-config-system-mmdetections-superpower)
7. [Training on Custom Dataset (Step-by-Step)](#7-training-on-custom-dataset-step-by-step)
8. [Advanced Features & Techniques](#8-advanced-features--techniques)
9. [MMDetection 3.x vs 2.x](#9-mmdetection-3x-vs-2x)
10. [Evaluation, Visualization & Analysis](#10-evaluation-visualization--analysis)
11. [Deployment with MMDeploy](#11-deployment-with-mmdeploy)
12. [MMDetection vs Detectron2 vs YOLO](#12-mmdetection-vs-detectron2-vs-yolo)
13. [Real-World Use Cases & Companies](#13-real-world-use-cases--companies)
14. [Tips, Tricks & Best Practices](#14-tips-tricks--best-practices)

---

## 1. What is MMDetection & The OpenMMLab Ecosystem

### 💡 The Core Idea

> **MMDetection** is the most comprehensive open-source object detection toolbox. It implements virtually EVERY published detection method in a unified, modular framework. Think of it as the "Wikipedia of object detection architectures."

### The OpenMMLab Ecosystem

```
OpenMMLab = A FAMILY of computer vision toolboxes (all modular, all interoperable)

┌─────────────────────────────────────────────────────────┐
│                   OPENMM LAB ECOSYSTEM                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ MMDetection  │  │ MMSegmentation│  │ MMPose       │ │
│  │ (Detection)  │  │ (Semantic Seg)│  │ (Pose Est.)  │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ MMTracking   │  │ MMAction2    │  │ MMOCR        │ │
│  │ (Tracking)   │  │ (Action Rec.)│  │ (OCR/Text)   │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ MMClassify   │  │ MM3D         │  │ MMDeploy     │ │
│  │ (Classific.) │  │ (3D Detection)│  │ (Deployment) │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│                                                         │
│  ALL built on MMCV (core library) and MMEngine (runner) │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 🏢 Who Uses MMDetection

| Company/Org | Use Case |
|-------------|----------|
| **Alibaba** | Product search, visual AI |
| **SenseTime** | Smart city, autonomous driving |
| **Huawei** | HiSilicon AI chips, mobile AI |
| **Megvii (Face++)** | Face recognition, detection |
| **ByteDance/TikTok** | Content understanding, effects |
| **Tencent** | Gaming, content moderation |
| **DJI** | Drone vision, obstacle avoidance |
| **Baidu** | Apollo self-driving |
| **Academic labs worldwide** | Research papers, benchmarking |
| **NIO** | Autonomous vehicle perception |

### Key Facts

| Fact | Detail |
|------|--------|
| **Creator** | OpenMMLab (Chinese University of HK + SenseTime) |
| **Framework** | PyTorch |
| **License** | Apache 2.0 |
| **GitHub Stars** | 29K+ |
| **Models** | 300+ pre-trained configurations |
| **Architectures** | 80+ unique methods |
| **Papers Implemented** | 100+ published papers |
| **Current Version** | MMDetection 3.x (based on MMEngine) |

### Why MMDetection?

```
✅ LARGEST model zoo (300+ configs, 80+ methods)
✅ Every major architecture implemented
✅ Modular: mix and match ANY backbone + neck + head
✅ Config-driven: change entire architecture via config
✅ Reproducible: matches paper-reported numbers
✅ Active: new papers implemented within weeks of publication
✅ Well-documented with tutorials
✅ Strong in Asia (most Chinese CV companies use it)
```

---

## 2. Installation & Setup

### Installation

```bash
# Step 1: Install PyTorch (match your CUDA)
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu118

# Step 2: Install MMEngine and MMCV
pip install -U openmim
mim install mmengine
mim install "mmcv>=2.0.0"

# Step 3: Install MMDetection
mim install mmdet

# OR from source (for development):
git clone https://github.com/open-mmlab/mmdetection.git
cd mmdetection
pip install -v -e .
```

### 📝 Verification

```python
import mmdet
from mmdet.utils import register_all_modules
print(f"MMDetection version: {mmdet.__version__}")

from mmdet.apis import init_detector, inference_detector
print("✅ MMDetection installed successfully!")
```

### Project Structure

```
mmdetection/
├── configs/                    # 300+ config files organized by method
│   ├── _base_/                # Base configs (shared)
│   │   ├── datasets/          # Dataset configs
│   │   ├── models/            # Model architecture configs
│   │   ├── schedules/         # Training schedule configs
│   │   └── default_runtime.py # Runtime defaults
│   ├── faster_rcnn/           # Faster R-CNN variants
│   ├── mask_rcnn/             # Mask R-CNN variants
│   ├── retinanet/             # RetinaNet variants
│   ├── yolo/                  # YOLO in MMDet
│   ├── detr/                  # DETR variants
│   ├── dino/                  # DINO variants
│   ├── cascade_rcnn/          # Cascade R-CNN
│   ├── ssd/                   # SSD variants
│   ├── centernet/             # CenterNet
│   ├── fcos/                  # FCOS (anchor-free)
│   ├── atss/                  # ATSS
│   ├── sparse_rcnn/           # Sparse R-CNN
│   ├── deformable_detr/       # Deformable DETR
│   └── ... (80+ method folders!)
├── mmdet/
│   ├── models/                # All model implementations
│   │   ├── backbones/         # ResNet, Swin, etc.
│   │   ├── necks/             # FPN, PANet, BiFPN, etc.
│   │   ├── dense_heads/       # RetinaNet head, FCOS head, etc.
│   │   ├── roi_heads/         # R-CNN heads
│   │   ├── detectors/         # Full detector classes
│   │   └── losses/            # All loss functions
│   ├── datasets/              # Dataset implementations
│   ├── evaluation/            # Metrics
│   └── engine/                # Training hooks, runners
├── tools/
│   ├── train.py               # Training entry point
│   ├── test.py                # Testing entry point
│   └── analysis_tools/        # Visualization, analysis
└── demo/
    └── inference_demo.py
```

---

## 3. Architecture & Design Philosophy

### 💡 Everything is a Config, Everything is Modular

```
MMDetection's philosophy:
  1. Every component is independently configurable
  2. Any backbone can pair with any neck can pair with any head
  3. Switch from Faster R-CNN to DETR by changing ONE config file
  4. Reproduce ANY paper's results with their config
```

### Detector = Backbone + Neck + Head

```
┌──────────────────────────────────────────────────────────────┐
│                    MMDETECTION MODEL                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐│
│  │   BACKBONE    │──▶│    NECK      │──▶│      HEAD        ││
│  │              │   │              │   │                  ││
│  │ 20+ options: │   │ 10+ options: │   │ 30+ options:     ││
│  │ • ResNet     │   │ • FPN        │   │ • AnchorHead     ││
│  │ • ResNeXt    │   │ • PANet      │   │ • RPNHead        ││
│  │ • Swin       │   │ • BiFPN      │   │ • RetinaHead     ││
│  │ • PVT        │   │ • NASFPN     │   │ • FCOSHead       ││
│  │ • ConvNeXt   │   │ • PAFPN      │   │ • ATSSHead       ││
│  │ • EfficientNet│  │ • ChannelMapper│  │ • CascadeROI    ││
│  │ • CSPDarknet │   │ • DyHead     │   │ • MaskHead       ││
│  │ • HRNet     │   │ • YOLOF      │   │ • DETRHead       ││
│  │ • MobileNet  │   │              │   │ • DINOHead       ││
│  │ • RegNet     │   │              │   │ • SparseRCNN     ││
│  └──────────────┘   └──────────────┘   └──────────────────┘│
│                                                              │
│  Mix and match ANY combination!                              │
│  Want ResNet + BiFPN + CascadeROI? Just change the config!  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Registry System

```python
from mmdet.registry import MODELS

# Every component is registered:
# MODELS.register_module() → backbones, necks, heads, detectors, losses
# DATASETS.register_module() → dataset classes
# TRANSFORMS.register_module() → augmentation transforms
# METRICS.register_module() → evaluation metrics

# Register a custom component
@MODELS.register_module()
class MyCustomBackbone(nn.Module):
    def __init__(self, depth=50):
        super().__init__()
        # ...
    
    def forward(self, x):
        # ...
        return features

# Now usable in config:
# backbone=dict(type='MyCustomBackbone', depth=50)
```

---

## 4. Model Zoo — 300+ Pre-trained Models

### 💡 The Most Comprehensive Model Zoo in Existence

### Two-Stage Detectors

| Method | Backbone Options | COCO AP | Key Feature |
|--------|-----------------|---------|-------------|
| **Faster R-CNN** | R-50/101, X-101, Swin | 37.4-48.3 | Foundational two-stage |
| **Cascade R-CNN** | R-50/101, X-101, Swin | 40.3-52.1 | Progressive IoU refinement |
| **Mask R-CNN** | R-50/101, X-101, Swin | 38.2-49.5 | Instance segmentation |
| **Cascade Mask R-CNN** | R-50/101, Swin | 41.2-52.3 | Best two-stage + masks |
| **HTC (Hybrid Task Cascade)** | R-50/101, X-101 | 42.3-47.1 | Multi-task cascade |
| **Sparse R-CNN** | R-50/101 | 44.5-46.9 | Learnable proposals |
| **Dynamic R-CNN** | R-50 | 38.9 | Dynamic IoU thresholds |

### One-Stage Detectors

| Method | Backbone Options | COCO AP | Key Feature |
|--------|-----------------|---------|-------------|
| **RetinaNet** | R-50/101, X-101 | 36.5-41.0 | Focal Loss pioneer |
| **SSD** | VGG-16, MobileNet | 25.1-29.3 | Classic one-stage |
| **FCOS** | R-50/101, X-101 | 38.7-44.1 | Anchor-free, per-pixel |
| **ATSS** | R-50/101 | 39.4-43.6 | Adaptive training sample selection |
| **GFL** | R-50/101, X-101 | 40.2-46.0 | Generalized focal loss |
| **PAA** | R-50/101 | 40.4-43.3 | Probabilistic anchor assignment |
| **TOOD** | R-50/101 | 42.5-46.2 | Task-aligned one-stage |
| **RTMDet** | CSPNeXt-s/m/l/x | 44.5-52.8 | Real-time, SOTA one-stage |
| **YOLOX** | CSPDarknet | 40.5-51.2 | YOLO in MMDet ecosystem |
| **YOLOv5/v8** | CSPDarknet | Various | Community implementations |

### Transformer-Based Detectors

| Method | Backbone | COCO AP | Key Feature |
|--------|----------|---------|-------------|
| **DETR** | R-50/101 | 42.0-43.3 | First transformer detector |
| **Deformable DETR** | R-50 | 46.2 | Deformable attention |
| **Conditional DETR** | R-50/101 | 40.9-43.8 | Conditional cross-attention |
| **DAB-DETR** | R-50 | 42.4 | Dynamic anchor boxes |
| **DN-DETR** | R-50 | 43.4 | Denoising training |
| **DINO** | R-50, Swin-L | 49.0-63.3 | DETR with improved denoising |
| **Co-DETR** | Swin-L | 65.6 | Collaborative DETR |
| **RT-DETR** | R-50, HGNetV2 | 53.1-54.3 | Real-time DETR |
| **Grounding DINO** | Swin-T/B | 52.5-56.7 | Open-set detection |

### Anchor-Free Detectors

| Method | Backbone | COCO AP | Key Feature |
|--------|----------|---------|-------------|
| **CenterNet** | R-50/101, Hourglass | 37.4-45.1 | Center-based detection |
| **CornerNet** | Hourglass | 40.5 | Corner keypoint pairs |
| **RepPoints** | R-50/101 | 38.6-42.0 | Representative point learning |
| **FoveaBox** | R-50/101 | 36.2-40.6 | Fovea-inspired detection |

### Special & Advanced

| Method | COCO AP | Key Feature |
|--------|---------|-------------|
| **DetectoRS** | 49.6 | Recursive feature pyramid |
| **CBNet** | 53.3 | Composite backbone |
| **Swin Transformer + Cascade** | 58.7 | Vision transformer backbone |
| **DINO + Swin-L (SOTA)** | 63.3 | Current state-of-the-art |
| **Co-DETR + Swin-L** | 65.6 | Latest SOTA |

---

## 5. Quick Start: Detection in Minutes

### 📝 Code: Inference with Pre-trained Model

```python
from mmdet.apis import init_detector, inference_detector
from mmdet.utils import register_all_modules
import mmcv

# Register all modules
register_all_modules()

# Step 1: Choose config + checkpoint
config_file = 'configs/faster_rcnn/faster-rcnn_r50_fpn_1x_coco.py'
checkpoint_file = 'checkpoints/faster_rcnn_r50_fpn_1x_coco_20200130-047c8118.pth'

# Step 2: Build model
model = init_detector(config_file, checkpoint_file, device='cuda:0')

# Step 3: Run inference
result = inference_detector(model, 'test.jpg')

# Step 4: Visualize
model.show_result('test.jpg', result, out_file='output.jpg', score_thr=0.5)
```

### 📝 Code: Using MMDetection's Inference API (v3.x)

```python
from mmdet.apis import DetInferencer

# One-line setup! (auto-downloads config + weights)
inferencer = DetInferencer(model='rtmdet-l', device='cuda:0')

# Run detection
result = inferencer('test.jpg', show=True, out_dir='outputs/')

# Batch inference
results = inferencer(['img1.jpg', 'img2.jpg', 'img3.jpg'], batch_size=4)

# Access predictions
predictions = result['predictions'][0]
bboxes = predictions['bboxes']      # [[x1,y1,x2,y2], ...]
scores = predictions['scores']       # [0.95, 0.87, ...]
labels = predictions['labels']       # [0, 2, 1, ...]

for bbox, score, label in zip(bboxes, scores, labels):
    if score > 0.5:
        print(f"Class {label}: {score:.3f} at {bbox}")
```

### 📝 Code: Try Different Models (Same API!)

```python
from mmdet.apis import DetInferencer

# Faster R-CNN
inferencer = DetInferencer(model='faster-rcnn_r50_fpn_2x_coco')
result = inferencer('test.jpg')

# RetinaNet
inferencer = DetInferencer(model='retinanet_r50_fpn_2x_coco')
result = inferencer('test.jpg')

# DINO (transformer-based, high accuracy)
inferencer = DetInferencer(model='dino-5scale_swin-l_8xb2-36e_coco')
result = inferencer('test.jpg')

# RTMDet (real-time, latest)
inferencer = DetInferencer(model='rtmdet-l')
result = inferencer('test.jpg')

# ALL use the SAME API! Just change the model name!
```

### 📝 Code: Video Inference

```python
from mmdet.apis import DetInferencer

inferencer = DetInferencer(model='rtmdet-s', device='cuda:0')

# Process video
result = inferencer(
    'video.mp4',
    out_dir='video_output/',
    show=False,
    no_save_pred=False
)
```

---

## 6. The Config System (MMDetection's Superpower)

### 💡 Config Inheritance — The Key Design

```
MMDetection configs use INHERITANCE:

_base_ configs (shared):
├── models/faster-rcnn_r50_fpn.py      ← Architecture definition
├── datasets/coco_detection.py          ← Dataset config
├── schedules/schedule_1x.py            ← Training schedule
└── default_runtime.py                  ← Logging, hooks, etc.

Your config INHERITS from all four:
faster-rcnn_r50_fpn_1x_coco.py
  └── _base_ = [
        'models/faster-rcnn_r50_fpn.py',      # What model
        'datasets/coco_detection.py',           # What data
        'schedules/schedule_1x.py',             # How to train
        'default_runtime.py'                    # Runtime settings
      ]

Override only what you need to change!
```

### Config File Anatomy

```python
# configs/faster_rcnn/faster-rcnn_r50_fpn_1x_coco.py

_base_ = [
    '../_base_/models/faster-rcnn_r50_fpn.py',
    '../_base_/datasets/coco_detection.py',
    '../_base_/schedules/schedule_1x.py',
    '../_base_/default_runtime.py'
]

# Everything below OVERRIDES the base config values
# If you don't override, base values are used
```

### 📝 Base Model Config (What's Inside)

```python
# _base_/models/faster-rcnn_r50_fpn.py

model = dict(
    type='FasterRCNN',
    
    # BACKBONE
    backbone=dict(
        type='ResNet',
        depth=50,
        num_stages=4,
        out_indices=(0, 1, 2, 3),
        frozen_stages=1,
        norm_cfg=dict(type='BN', requires_grad=True),
        norm_eval=True,
        style='pytorch',
        init_cfg=dict(type='Pretrained', checkpoint='torchvision://resnet50')
    ),
    
    # NECK
    neck=dict(
        type='FPN',
        in_channels=[256, 512, 1024, 2048],
        out_channels=256,
        num_outs=5
    ),
    
    # RPN HEAD
    rpn_head=dict(
        type='RPNHead',
        in_channels=256,
        feat_channels=256,
        anchor_generator=dict(
            type='AnchorGenerator',
            scales=[8],
            ratios=[0.5, 1.0, 2.0],
            strides=[4, 8, 16, 32, 64]
        ),
        bbox_coder=dict(type='DeltaXYWHBBoxCoder'),
        loss_cls=dict(type='CrossEntropyLoss', use_sigmoid=True, loss_weight=1.0),
        loss_bbox=dict(type='L1Loss', loss_weight=1.0)
    ),
    
    # ROI HEAD
    roi_head=dict(
        type='StandardRoIHead',
        bbox_roi_extractor=dict(
            type='SingleRoIExtractor',
            roi_layer=dict(type='RoIAlign', output_size=7, sampling_ratio=0),
            out_channels=256,
            featmap_strides=[4, 8, 16, 32]
        ),
        bbox_head=dict(
            type='Shared2FCBBoxHead',
            in_channels=256,
            fc_out_channels=1024,
            roi_feat_size=7,
            num_classes=80,         # ← CHANGE THIS for custom dataset!
            bbox_coder=dict(type='DeltaXYWHBBoxCoder'),
            reg_class_agnostic=False,
            loss_cls=dict(type='CrossEntropyLoss', use_sigmoid=False, loss_weight=1.0),
            loss_bbox=dict(type='SmoothL1Loss', beta=1.0, loss_weight=1.0)
        )
    ),
    
    # Training config
    train_cfg=dict(
        rpn=dict(
            assigner=dict(type='MaxIoUAssigner', pos_iou_thr=0.7, neg_iou_thr=0.3),
            sampler=dict(type='RandomSampler', num=256, pos_fraction=0.5),
        ),
        rcnn=dict(
            assigner=dict(type='MaxIoUAssigner', pos_iou_thr=0.5, neg_iou_thr=0.5),
            sampler=dict(type='RandomSampler', num=512, pos_fraction=0.25),
        )
    ),
    
    # Test config
    test_cfg=dict(
        rpn=dict(nms_pre=1000, max_per_img=1000, nms=dict(type='nms', iou_threshold=0.7)),
        rcnn=dict(score_thr=0.05, nms=dict(type='nms', iou_threshold=0.5), max_per_img=100)
    )
)
```

### 📝 How to Swap Components (The Magic)

```python
# Switch from Faster R-CNN to Cascade R-CNN:
# Just change roi_head type!
_base_ = '../_base_/models/faster-rcnn_r50_fpn.py'

model = dict(
    roi_head=dict(
        type='CascadeRoIHead',  # ← Changed from StandardRoIHead!
        num_stages=3,
        stage_loss_weights=[1, 0.5, 0.25],
        bbox_head=[
            dict(type='Shared2FCBBoxHead', num_classes=80,
                 loss_cls=dict(type='CrossEntropyLoss'), loss_bbox=dict(type='SmoothL1Loss')),
            dict(type='Shared2FCBBoxHead', num_classes=80,
                 loss_cls=dict(type='CrossEntropyLoss'), loss_bbox=dict(type='SmoothL1Loss')),
            dict(type='Shared2FCBBoxHead', num_classes=80,
                 loss_cls=dict(type='CrossEntropyLoss'), loss_bbox=dict(type='SmoothL1Loss')),
        ]
    )
)
```

```python
# Switch backbone from ResNet-50 to Swin Transformer:
model = dict(
    backbone=dict(
        _delete_=True,  # Remove all old backbone settings
        type='SwinTransformer',
        embed_dims=96,
        depths=[2, 2, 6, 2],
        num_heads=[3, 6, 12, 24],
        window_size=7,
        drop_path_rate=0.2,
        init_cfg=dict(type='Pretrained', checkpoint='swin_tiny_patch4_window7_224.pth')
    ),
    neck=dict(in_channels=[96, 192, 384, 768])  # Match Swin output channels
)
```

```python
# Switch neck from FPN to PANet:
model = dict(
    neck=dict(
        type='PAFPN',  # ← Changed from FPN
        in_channels=[256, 512, 1024, 2048],
        out_channels=256,
        num_outs=5
    )
)
```

### Config Naming Convention

```
{method}_{backbone}_{neck}_{schedule}_{dataset}.py

Examples:
faster-rcnn_r50_fpn_1x_coco.py
│            │    │    │   │
│            │    │    │   └── Dataset: COCO
│            │    │    └────── Schedule: 1x (12 epochs)
│            │    └─────────── Neck: FPN
│            └──────────────── Backbone: ResNet-50
└───────────────────────────── Method: Faster R-CNN

Schedule shortcuts:
  1x = 12 epochs
  2x = 24 epochs
  3x = 36 epochs
  20e = 20 epochs (custom)
```

---

## 7. Training on Custom Dataset (Step-by-Step)

### Step 1: Prepare Dataset (COCO Format)

```
my_dataset/
├── annotations/
│   ├── train.json    ← COCO format JSON
│   └── val.json
├── train/
│   ├── img001.jpg
│   ├── img002.jpg
│   └── ...
└── val/
    ├── img100.jpg
    └── ...
```

### Step 2: Register Dataset

```python
# Option A: Use COCO format (easiest — no code needed, just config!)

# In your config file:
data_root = 'data/my_dataset/'

train_dataloader = dict(
    batch_size=4,
    num_workers=4,
    dataset=dict(
        type='CocoDataset',
        data_root=data_root,
        ann_file='annotations/train.json',
        data_prefix=dict(img='train/'),
        metainfo=dict(classes=('helmet', 'no_helmet', 'person')),
    )
)

val_dataloader = dict(
    batch_size=1,
    num_workers=2,
    dataset=dict(
        type='CocoDataset',
        data_root=data_root,
        ann_file='annotations/val.json',
        data_prefix=dict(img='val/'),
        metainfo=dict(classes=('helmet', 'no_helmet', 'person')),
    )
)
```

```python
# Option B: Register custom dataset class
from mmdet.registry import DATASETS
from mmdet.datasets import BaseDetDataset

@DATASETS.register_module()
class MyCustomDataset(BaseDetDataset):
    METAINFO = {
        'classes': ('helmet', 'no_helmet', 'person'),
        'palette': [(220, 20, 60), (0, 0, 255), (0, 255, 0)]
    }
    
    def load_data_list(self):
        data_list = []
        # Load your annotations and return list of dicts
        # Each dict: {'img_path': ..., 'instances': [...]}
        return data_list
```

### Step 3: Create Training Config

```python
# configs/my_project/faster-rcnn_r50_fpn_1x_my_dataset.py

_base_ = [
    '../_base_/models/faster-rcnn_r50_fpn.py',
    '../_base_/default_runtime.py',
]

# ── Model: change num_classes ──
model = dict(
    roi_head=dict(
        bbox_head=dict(num_classes=3)  # helmet, no_helmet, person
    )
)

# ── Dataset ──
data_root = 'data/my_dataset/'
metainfo = dict(classes=('helmet', 'no_helmet', 'person'))

train_dataloader = dict(
    batch_size=4,
    num_workers=4,
    persistent_workers=True,
    sampler=dict(type='DefaultSampler', shuffle=True),
    dataset=dict(
        type='CocoDataset',
        data_root=data_root,
        ann_file='annotations/train.json',
        data_prefix=dict(img='train/'),
        metainfo=metainfo,
        pipeline=[
            dict(type='LoadImageFromFile'),
            dict(type='LoadAnnotations', with_bbox=True),
            dict(type='Resize', scale=(1333, 800), keep_ratio=True),
            dict(type='RandomFlip', prob=0.5),
            dict(type='PackDetInputs')
        ]
    )
)

val_dataloader = dict(
    batch_size=1,
    num_workers=2,
    dataset=dict(
        type='CocoDataset',
        data_root=data_root,
        ann_file='annotations/val.json',
        data_prefix=dict(img='val/'),
        metainfo=metainfo,
        pipeline=[
            dict(type='LoadImageFromFile'),
            dict(type='Resize', scale=(1333, 800), keep_ratio=True),
            dict(type='LoadAnnotations', with_bbox=True),
            dict(type='PackDetInputs')
        ]
    )
)

val_evaluator = dict(
    type='CocoMetric',
    ann_file=data_root + 'annotations/val.json',
    metric='bbox'
)

# ── Training Schedule ──
train_cfg = dict(type='EpochBasedTrainLoop', max_epochs=24, val_interval=2)
val_cfg = dict(type='ValLoop')

# ── Optimizer ──
optim_wrapper = dict(
    type='OptimWrapper',
    optimizer=dict(type='SGD', lr=0.005, momentum=0.9, weight_decay=0.0001)
)

# ── LR Schedule ──
param_scheduler = [
    dict(type='LinearLR', start_factor=0.001, by_epoch=False, begin=0, end=500),
    dict(type='MultiStepLR', by_epoch=True, milestones=[16, 22], gamma=0.1)
]

# ── Pre-trained weights (transfer learning!) ──
load_from = 'https://download.openmmlab.com/mmdetection/v2.0/faster_rcnn/faster_rcnn_r50_fpn_2x_coco/faster_rcnn_r50_fpn_2x_coco_bbox_mAP-0.384_20200504_210434-a5d8aa15.pth'

# ── Output directory ──
work_dir = './work_dirs/my_detector'
```

### Step 4: Train!

```bash
# Single GPU
python tools/train.py configs/my_project/faster-rcnn_r50_fpn_1x_my_dataset.py

# Multi-GPU (4 GPUs)
bash tools/dist_train.sh configs/my_project/faster-rcnn_r50_fpn_1x_my_dataset.py 4

# Resume from checkpoint
python tools/train.py configs/my_project/faster-rcnn_r50_fpn_1x_my_dataset.py \
    --resume work_dirs/my_detector/epoch_12.pth
```

### 📝 Code: Train with Python API

```python
from mmengine.runner import Runner

# Build runner from config
runner = Runner.from_cfg(cfg)

# Train
runner.train()
```

### Step 5: Test & Evaluate

```bash
# Evaluate on validation set
python tools/test.py \
    configs/my_project/faster-rcnn_r50_fpn_1x_my_dataset.py \
    work_dirs/my_detector/epoch_24.pth \
    --show-dir results/
```

---

## 8. Advanced Features & Techniques

### Multi-Scale Training & Testing (TTA)

```python
# Multi-scale training (in pipeline):
train_pipeline = [
    dict(type='LoadImageFromFile'),
    dict(type='LoadAnnotations', with_bbox=True),
    dict(
        type='RandomChoiceResize',
        scales=[(480, 1333), (512, 1333), (544, 1333),
                (576, 1333), (608, 1333), (640, 1333),
                (672, 1333), (704, 1333), (736, 1333),
                (768, 1333), (800, 1333)],
        keep_ratio=True
    ),
    dict(type='RandomFlip', prob=0.5),
    dict(type='PackDetInputs')
]

# Test-Time Augmentation (TTA):
tta_model = dict(
    type='DetTTAModel',
    tta_cfg=dict(nms=dict(type='nms', iou_threshold=0.5), max_per_img=100)
)
tta_pipeline = [
    dict(type='LoadImageFromFile'),
    dict(
        type='TestTimeAug',
        transforms=[
            [dict(type='Resize', scale=(800, 1333), keep_ratio=True),
             dict(type='Resize', scale=(1200, 1333), keep_ratio=True)],
            [dict(type='RandomFlip', prob=0.0),
             dict(type='RandomFlip', prob=1.0)],
            [dict(type='PackDetInputs')]
        ]
    )
]
```

### Data Augmentation Pipeline

```python
# Rich augmentation options:
train_pipeline = [
    dict(type='LoadImageFromFile'),
    dict(type='LoadAnnotations', with_bbox=True, with_mask=True),
    
    # Geometric
    dict(type='RandomFlip', prob=0.5),
    dict(type='Resize', scale=(1333, 800), keep_ratio=True),
    dict(type='RandomCrop', crop_size=(800, 800)),
    dict(type='RandomAffine', max_rotate_degree=15, max_shear_degree=2),
    
    # Photometric
    dict(type='PhotoMetricDistortion',
         brightness_delta=32, contrast_range=(0.5, 1.5),
         saturation_range=(0.5, 1.5), hue_delta=18),
    dict(type='RandomErasing', n_patches=(1, 5), ratio=(0, 0.2)),
    
    # Mosaic & MixUp (YOLO-style)
    dict(type='Mosaic', img_scale=(640, 640), pad_val=114.0),
    dict(type='MixUp', img_scale=(640, 640), ratio_range=(0.8, 1.6), pad_val=114.0),
    
    # Copy-Paste (for instance segmentation)
    dict(type='CopyPaste', max_num_pasted=100),
    
    dict(type='PackDetInputs')
]
```

### Mixed Precision Training (FP16)

```python
# Enable AMP for ~1.5-2× faster training
optim_wrapper = dict(
    type='AmpOptimWrapper',  # ← Changed from OptimWrapper
    optimizer=dict(type='SGD', lr=0.02, momentum=0.9, weight_decay=0.0001),
    loss_scale='dynamic'  # Auto-scale for stability
)
```

### Knowledge Distillation

```python
# Train a small model using knowledge from a large model
model = dict(
    type='KnowledgeDistillationSingleStageDetector',
    teacher_config='configs/faster_rcnn/faster-rcnn_r101_fpn_2x_coco.py',
    teacher_ckpt='teacher_model.pth',
    distill_cfg=dict(
        student_channels=256,
        teacher_channels=256,
        distill_loss=dict(type='MSELoss', loss_weight=1.0)
    )
)
```

---

## 9. MMDetection 3.x vs 2.x

### Key Differences

| Feature | MMDet 2.x | MMDet 3.x |
|---------|-----------|-----------|
| **Engine** | MMCV Runner | **MMEngine Runner** |
| **Config** | mmcv.Config | **mmengine.Config** |
| **Data** | Custom pipeline | **Unified data transforms** |
| **Registry** | mmcv.Registry | **mmengine.Registry** |
| **Logging** | Custom | **MMEngine logging** |
| **Training** | runner.run() | **runner.train()** |
| **API** | init_detector + inference_detector | **DetInferencer** (simpler) |
| **Hook system** | Basic hooks | **Rich hook system** |

### Migration Quick Guide

```python
# MMDet 2.x:
from mmdet.apis import init_detector, inference_detector
model = init_detector(config, checkpoint, device='cuda')
result = inference_detector(model, image)

# MMDet 3.x (same API still works, plus new one):
from mmdet.apis import DetInferencer
inferencer = DetInferencer(model='faster-rcnn_r50_fpn_2x_coco')
result = inferencer('image.jpg')

# Config changes (2.x → 3.x):
# data = dict(train=dict(...))       → train_dataloader = dict(...)
# optimizer = dict(...)              → optim_wrapper = dict(optimizer=dict(...))
# lr_config = dict(...)              → param_scheduler = [dict(...)]
# runner = dict(max_epochs=12)       → train_cfg = dict(max_epochs=12)
```

---

## 10. Evaluation, Visualization & Analysis

### Evaluation

```bash
# Standard COCO evaluation
python tools/test.py config.py checkpoint.pth

# Output:
# Average Precision (AP) @[ IoU=0.50:0.95 ] = 0.423
# Average Precision (AP) @[ IoU=0.50      ] = 0.654
# Average Precision (AP) @[ IoU=0.75      ] = 0.461
# Average Precision (AP) @[ IoU=small     ] = 0.248
# Average Precision (AP) @[ IoU=medium    ] = 0.467
# Average Precision (AP) @[ IoU=large     ] = 0.568
```

### Analysis Tools

```bash
# Analyze model FLOPs and parameters
python tools/analysis_tools/get_flops.py config.py

# Analyze training logs (plot loss curves)
python tools/analysis_tools/analyze_logs.py plot_curve \
    work_dirs/my_detector/scalars.json \
    --keys loss_cls loss_bbox \
    --out loss_curve.png

# Compare multiple training runs
python tools/analysis_tools/analyze_logs.py plot_curve \
    run1/scalars.json run2/scalars.json \
    --keys bbox_mAP --legend run1 run2

# Visualize predictions
python tools/analysis_tools/browse_dataset.py config.py \
    --output-dir visualizations/ --not-show
```

### 📝 Code: Custom Evaluation

```python
from mmdet.apis import DetInferencer
from mmdet.evaluation import CocoMetric

# Initialize
inferencer = DetInferencer(model='rtmdet-l', device='cuda:0')

# Inference + built-in COCO evaluation
result = inferencer(
    inputs='data/val/',
    out_dir='eval_results/',
    pred_score_thr=0.3
)

# Or programmatic evaluation
from mmengine.evaluator import Evaluator

evaluator = Evaluator(metrics=dict(
    type='CocoMetric',
    ann_file='data/annotations/val.json',
    metric=['bbox', 'segm']
))
```

### Confusion Matrix

```bash
# Generate confusion matrix
python tools/analysis_tools/confusion_matrix.py \
    config.py \
    results.pkl \
    work_dirs/my_detector/ \
    --show
```

---

## 11. Deployment with MMDeploy

### 💡 MMDeploy — The Official Deployment Tool

```
MMDeploy converts MMDetection models to optimized formats:

MMDetection model → MMDeploy → ┌─ ONNX Runtime (CPU/GPU)
                               ├─ TensorRT (NVIDIA GPU)
                               ├─ OpenVINO (Intel)
                               ├─ NCNN (Mobile)
                               ├─ TorchScript
                               └─ CoreML (Apple)
```

### Installation

```bash
pip install mmdeploy
pip install mmdeploy-runtime  # For inference
# Platform-specific backends:
pip install mmdeploy-runtime-gpu  # For TensorRT
```

### 📝 Code: Export to ONNX

```python
from mmdeploy.apis import torch2onnx

torch2onnx(
    img='test.jpg',
    work_dir='mmdeploy_output/',
    save_file='model.onnx',
    deploy_cfg='mmdeploy/configs/mmdet/detection/detection_onnxruntime_dynamic.py',
    model_cfg='configs/faster_rcnn/faster-rcnn_r50_fpn_1x_coco.py',
    model_checkpoint='checkpoint.pth',
    device='cuda:0'
)
```

### 📝 Code: Export to TensorRT

```python
from mmdeploy.apis import torch2onnx, onnx2tensorrt

# Step 1: Convert to ONNX
torch2onnx(
    img='test.jpg',
    work_dir='mmdeploy_output/',
    save_file='model.onnx',
    deploy_cfg='mmdeploy/configs/mmdet/detection/detection_tensorrt_dynamic-320x320-1344x1344.py',
    model_cfg='configs/rtmdet/rtmdet_l_8xb32-300e_coco.py',
    model_checkpoint='rtmdet_l.pth',
    device='cuda:0'
)

# Step 2: Convert ONNX to TensorRT engine
onnx2tensorrt(
    work_dir='mmdeploy_output/',
    save_file='model.engine',
    model_id=0,
    device='cuda:0'
)
```

### 📝 Code: Run Deployed Model

```python
from mmdeploy.apis.utils import build_task_processor
from mmdeploy.utils import get_input_shape, load_config

deploy_cfg = 'mmdeploy/configs/mmdet/detection/detection_onnxruntime_dynamic.py'
model_cfg = 'configs/faster_rcnn/faster-rcnn_r50_fpn_1x_coco.py'
backend_files = ['mmdeploy_output/model.onnx']

# Build processor
task_processor = build_task_processor(model_cfg, deploy_cfg, 'cuda:0')
model = task_processor.build_backend_model(backend_files)

# Run inference (2-5× faster than PyTorch!)
input_shape = get_input_shape(deploy_cfg)
result = task_processor.single_gpu_test(model, data_loader)
```

### Deployment Speed Comparison

| Format | Device | FPS (RTMDet-L) | Speedup |
|--------|--------|---------------|---------|
| PyTorch | V100 | 45 | 1× |
| ONNX Runtime | V100 | 78 | 1.7× |
| TensorRT FP16 | V100 | 142 | 3.2× |
| TensorRT INT8 | V100 | 198 | 4.4× |
| OpenVINO | i7 CPU | 22 | - |
| NCNN | Snapdragon 888 | 15 | - |

---

## 12. MMDetection vs Detectron2 vs YOLO

### Detailed Comparison

| Feature | MMDetection | Detectron2 | YOLO (Ultralytics) |
|---------|-------------|-----------|-------------------|
| **Models** | **300+ configs** | 20+ configs | YOLO only |
| **Architectures** | **80+ methods** | 15+ methods | 1 method (YOLO) |
| **New paper speed** | Weeks | Months | N/A |
| **Config system** | ⭐⭐⭐⭐⭐ (inheritance) | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Ease of use** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Documentation** | ⭐⭐⭐⭐ (CN+EN) | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Deploy tool** | MMDeploy (great) | Limited | Built-in (great) |
| **Community** | Large (Asia-centric) | Large (Western) | Massive (global) |
| **Transformer models** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **Instance seg** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Real-time focus** | RTMDet | No | ⭐⭐⭐⭐⭐ |

### When to Choose MMDetection

```
✅ You need to COMPARE many architectures (biggest model zoo)
✅ You're doing RESEARCH and need SOTA implementations
✅ You want to try TRANSFORMER detectors (DETR, DINO, Co-DETR)
✅ You need the ABSOLUTE best accuracy (DINO, Co-DETR)
✅ You want STRONG augmentation (Mosaic, MixUp, CopyPaste built-in)
✅ You're deploying with MMDeploy (TensorRT, ONNX, NCNN)
✅ You're working in Asia (dominant in Chinese tech companies)
✅ You want a single framework for ALL CV tasks (OpenMMLab ecosystem)

❌ Don't choose if:
  → You need simplest experience → YOLO
  → You only need Mask R-CNN → Detectron2
  → You need TFLite deployment → TF OD API
```

---

## 13. Real-World Use Cases & Companies

### 🏢 Case Study 1: Alibaba — Product Detection

```
Task: Detect products in images for visual search (Taobao/Tmall)
Framework: MMDetection (Cascade R-CNN + Swin backbone)
Scale: Billions of product images
Why MMDetection:
  - Easy to experiment with backbones (tried ResNet, Swin, ConvNeXt)
  - Cascade R-CNN for precise product boxes
  - Integrated deployment with MMDeploy
Result: 96% product detection accuracy
```

### 🏢 Case Study 2: SenseTime — Smart City

```
Task: Vehicle, pedestrian, traffic sign detection for smart city
Framework: MMDetection (DINO + Swin-L for accuracy, RTMDet for real-time)
Pipeline:
  High-res cameras → RTMDet (real-time alerts) 
  Stored footage → DINO (forensic analysis, higher accuracy)
Why: Need BOTH real-time AND high-accuracy models — MMDet has both
```

### 🏢 Case Study 3: DJI — Drone Vision

```
Task: Obstacle detection, tracking for consumer/enterprise drones
Framework: MMDetection + MMDeploy
Model: RTMDet-tiny → ONNX → TensorRT → DJI compute module
Why: RTMDet is optimized for real-time, MMDeploy for edge deployment
Constraint: Must run at 30 FPS on drone's limited compute
```

### 🏢 Case Study 4: Medical Research — Pathology

```
Task: Detect cancer cells in histopathology slides
Framework: MMDetection (Mask R-CNN + FPN)
Dataset: 50,000+ annotated pathology patches
Why MMDetection:
  - Easy to try different backbones (ResNet, HRNet, Swin)
  - Instance segmentation for cell boundaries
  - Reproducible configs for paper publication
Result: Published in top medical journal with MMDetection baselines
```

---

## 14. Tips, Tricks & Best Practices

### ✅ Best Practices

| Tip | Why |
|-----|-----|
| **Start with RTMDet for real-time** | Latest one-stage SOTA from MMDet team |
| **Use config inheritance** | Don't copy paste — inherit from `_base_` |
| **Always load pre-trained** | `load_from = url` gives massive accuracy boost |
| **Use `_delete_=True` to replace** | Cleanly swap backbone/neck/head |
| **Check with `browse_dataset.py`** | Visualize data before training |
| **Use AMP (FP16)** | 1.5-2× faster training, no accuracy loss |
| **Start small, scale up** | Try RTMDet-tiny first, scale if accuracy insufficient |
| **Use MMDeploy for production** | Proper export to TensorRT/ONNX |

### ⚠️ Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `KeyError: 'XXX is not in the registry'` | Module not registered | Import the module or check spelling |
| `RuntimeError: CUDA out of memory` | Batch/image too large | Reduce `batch_size` or `scale` |
| `num_classes mismatch` | Config doesn't match data | Set `num_classes` in bbox_head |
| `FileNotFoundError: config` | Wrong config path | Use relative path from mmdetection root |
| `AssertionError: annotation empty` | Bad annotation file | Check COCO JSON format, verify image paths |
| `Loss is NaN` | LR too high or bad data | Lower learning rate, check annotations |

### Recommended Starting Configs

```
For FAST prototyping:
  → configs/rtmdet/rtmdet_tiny_8xb32-300e_coco.py (fastest)

For BALANCED speed/accuracy:
  → configs/rtmdet/rtmdet_l_8xb32-300e_coco.py

For HIGH accuracy (two-stage):
  → configs/cascade_rcnn/cascade-rcnn_r50_fpn_1x_coco.py

For HIGHEST accuracy (transformer):
  → configs/dino/dino-5scale_swin-l_8xb2-36e_coco.py

For INSTANCE SEGMENTATION:
  → configs/mask_rcnn/mask-rcnn_r50_fpn_2x_coco.py

For MOBILE/EDGE:
  → configs/rtmdet/rtmdet_tiny_8xb32-300e_coco.py + MMDeploy
```

---

## 📋 Chapter Summary

### What You Learned

```
✅ MMDetection = Largest OD model zoo (300+ configs, 80+ methods)
✅ OpenMMLab ecosystem: detection, segmentation, pose, tracking, OCR
✅ Config inheritance: base configs + override only what changes
✅ Modular: swap ANY backbone + neck + head via config
✅ Covers: two-stage, one-stage, anchor-free, transformer detectors
✅ RTMDet: MMDet's real-time SOTA one-stage detector
✅ DINO/Co-DETR: highest accuracy transformer detectors
✅ MMDeploy: official deployment to ONNX, TensorRT, NCNN, OpenVINO
✅ Strong in Asia: Alibaba, SenseTime, Huawei, ByteDance, DJI
```

### 🔥 Interview Quick-Fire

| Question | Answer |
|----------|--------|
| "What is MMDetection?" | OpenMMLab's PyTorch framework with 300+ models, 80+ architectures |
| "MMDetection vs Detectron2?" | MMDet: more models, config inheritance. D2: Meta's standard, Mask R-CNN |
| "What is RTMDet?" | MMDet's real-time detector, SOTA one-stage accuracy+speed |
| "How does MMDet's config work?" | Inheritance from `_base_` configs, override only what changes |
| "How to swap backbone?" | Change `backbone=dict(type='SwinTransformer')` in config |
| "What is DINO in MMDet?" | Transformer detector with denoising training, 63% AP COCO |
| "How to deploy MMDet models?" | Use MMDeploy → export to ONNX/TensorRT/NCNN |
| "Best model for real-time?" | RTMDet-s/m/l (MMDet team's one-stage SOTA) |

---

### ➡️ Next Chapter: [09 - DETR & RT-DETR](./09-DETR-and-RT-DETR.md)

> Now you've seen the biggest model zoo. Next, we'll deep-dive into DETR and RT-DETR — the Transformer-based detectors that represent a completely NEW paradigm: no anchors, no NMS, no hand-designed components. Just attention.

---

*[← Back to Index](./00-INDEX.md) | [← Previous: 07 - Detectron2](./07-Detectron2.md)*
