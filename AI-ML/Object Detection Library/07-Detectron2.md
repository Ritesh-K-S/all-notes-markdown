# 📘 Chapter 07: Detectron2 — Meta's Research Powerhouse

> **Goal**: Master Meta AI's Detectron2 — the industry-standard research framework for object detection, instance segmentation, panoptic segmentation, and keypoint detection. If YOLO is for production, Detectron2 is for pushing boundaries.

---

## 📑 Table of Contents

1. [What is Detectron2 & Why It Matters](#1-what-is-detectron2--why-it-matters)
2. [Installation & Setup](#2-installation--setup)
3. [Detectron2 Architecture & Design Philosophy](#3-detectron2-architecture--design-philosophy)
4. [Model Zoo — Pre-trained Models](#4-model-zoo--pre-trained-models)
5. [Quick Start: Detection in Minutes](#5-quick-start-detection-in-minutes)
6. [Understanding the Config System](#6-understanding-the-config-system)
7. [Supported Tasks & Models](#7-supported-tasks--models)
8. [Training on Custom Dataset (Step-by-Step)](#8-training-on-custom-dataset-step-by-step)
9. [Advanced: Custom Models & Components](#9-advanced-custom-models--components)
10. [Evaluation & Visualization](#10-evaluation--visualization)
11. [Deployment & Production](#11-deployment--production)
12. [Detectron2 vs YOLO vs TF OD API vs MMDetection](#12-detectron2-vs-yolo-vs-tf-od-api-vs-mmdetection)
13. [Real-World Use Cases & Companies](#13-real-world-use-cases--companies)
14. [Tips, Tricks & Best Practices](#14-tips-tricks--best-practices)

---

## 1. What is Detectron2 & Why It Matters

### 💡 The Core Idea

> **Detectron2** is Meta AI's (Facebook's) modular, PyTorch-based framework for state-of-the-art object detection and segmentation. It's the go-to framework for RESEARCH — most new detection papers benchmark against or build upon Detectron2.

### Detectron History

```
2017: Detectron (Caffe2)     → First version, Facebook's internal tool
2019: Detectron2 (PyTorch)   → Complete rewrite, modular, open-source
2020: Detectron2 becomes THE standard research framework
2023: SAM (Segment Anything) built on Detectron2's foundation
2024: Still actively maintained, powers Meta's AI research
```

### 🏢 Who Uses Detectron2

| Company/Org | Use Case |
|-------------|----------|
| **Meta/Facebook** | Content moderation, AR effects, Instagram filters |
| **Meta Reality Labs** | AR/VR object understanding |
| **Academic labs** | 90%+ of detection research papers |
| **FAIR (Meta AI Research)** | Developing DETR, SAM, Mask R-CNN |
| **Waymo** | Research & prototyping |
| **NVIDIA** | Benchmarking, research |
| **Microsoft Research** | Cross-framework comparisons |
| **Medical imaging labs** | Tumor/cell segmentation research |
| **Government** | Satellite imagery analysis |
| **Robotics companies** | Scene understanding research |

### Key Facts

| Fact | Detail |
|------|--------|
| **Creator** | Meta AI (FAIR — Facebook AI Research) |
| **Framework** | PyTorch |
| **License** | Apache 2.0 |
| **GitHub Stars** | 30K+ |
| **Key Models** | Faster R-CNN, Mask R-CNN, RetinaNet, Cascade R-CNN, DETR, Panoptic FPN |
| **Tasks** | Detection, Instance Seg, Semantic Seg, Panoptic Seg, Keypoints |
| **Design** | Modular — swap any component (backbone, neck, head) |

### Why Detectron2 (Not YOLO)?

```
YOLO = Production tool (easy, fast, deploy anywhere)
Detectron2 = Research tool (modular, extensible, all architectures)

Choose Detectron2 when:
✅ Building new architectures / doing research
✅ Need instance segmentation (Mask R-CNN is THE model)
✅ Need panoptic segmentation
✅ Need keypoint detection at highest quality
✅ Want to reproduce papers (most use Detectron2)
✅ Need modular design to swap components
✅ Working with complex multi-task detection
✅ Academic publication (reviewers expect Detectron2 baselines)
```

---

## 2. Installation & Setup

### Prerequisites

```bash
# Required:
# - Python 3.8+
# - PyTorch 1.10+ (with CUDA for GPU)
# - Linux or macOS (Windows has partial support)
```

### Installation

```bash
# Step 1: Install PyTorch (match your CUDA version)
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118

# Step 2: Install Detectron2
pip install 'git+https://github.com/facebookresearch/detectron2.git'

# OR from pre-built wheels (faster):
pip install detectron2 -f https://dl.fbaipublicfiles.com/detectron2/wheels/cu118/torch2.0/index.html

# Step 3: Verify
python -c "import detectron2; print(detectron2.__version__)"
```

### 📝 Verification Script

```python
import detectron2
from detectron2 import model_zoo
from detectron2.engine import DefaultPredictor
from detectron2.config import get_cfg

print(f"Detectron2 version: {detectron2.__version__}")
print(f"PyTorch version: {torch.__version__}")
print(f"CUDA available: {torch.cuda.is_available()}")
print("✅ Detectron2 installed successfully!")
```

### Project Structure

```
detectron2/
├── configs/          # YAML config files for all models
│   ├── COCO-Detection/
│   ├── COCO-InstanceSegmentation/
│   ├── COCO-Keypoints/
│   ├── COCO-PanopticSegmentation/
│   └── Base-*.yaml   # Base configs
├── detectron2/
│   ├── checkpoint/   # Model loading/saving
│   ├── config/       # Config system
│   ├── data/         # Dataset registration & loaders
│   ├── engine/       # Training & inference engines
│   ├── evaluation/   # COCO, VOC evaluators
│   ├── export/       # ONNX, TorchScript export
│   ├── layers/       # Custom layers (ROI Align, etc.)
│   ├── modeling/     # ALL architectures
│   │   ├── backbone/       # ResNet, FPN, etc.
│   │   ├── meta_arch/      # Faster R-CNN, RetinaNet, etc.
│   │   ├── proposal_generator/  # RPN
│   │   └── roi_heads/      # Box head, mask head
│   ├── solver/       # Optimizers, LR schedulers
│   ├── structures/   # Boxes, Instances, ImageList
│   └── utils/        # Visualization, logging
└── tools/
    ├── train_net.py       # Training entry point
    ├── plain_train_net.py # Simple training script
    └── visualize_json_results.py
```

---

## 3. Detectron2 Architecture & Design Philosophy

### 💡 Modular Design: Everything is Swappable

```
┌────────────────────────────────────────────────────────────────┐
│                    DETECTRON2 DESIGN                            │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Every component is a REGISTERED MODULE that can be swapped:   │
│                                                                │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────┐          │
│  │ BACKBONE │──▶│   NECK   │──▶│  META-ARCHITECTURE │         │
│  │          │   │          │   │                    │         │
│  │ Options: │   │ Options: │   │ Options:           │         │
│  │ • ResNet │   │ • FPN    │   │ • GeneralizedRCNN │         │
│  │ • ResNeXt│   │ • None   │   │ • RetinaNet       │         │
│  │ • VGG    │   │          │   │ • Panoptic FPN    │         │
│  │ • Swin   │   │          │   │ • SemanticSeg     │         │
│  │ (custom) │   │ (custom) │   │ (custom)          │         │
│  └──────────┘   └──────────┘   └──────────────────┘          │
│                                         │                      │
│                              ┌──────────┴──────────┐          │
│                              │                     │          │
│                              ▼                     ▼          │
│                     ┌──────────────┐      ┌──────────────┐   │
│                     │PROPOSAL GEN  │      │  ROI HEADS   │   │
│                     │              │      │              │   │
│                     │ • RPN        │      │ • Box Head   │   │
│                     │ • Pre-computed│     │ • Mask Head  │   │
│                     │              │      │ • Keypoint   │   │
│                     └──────────────┘      └──────────────┘   │
│                                                                │
│  Config System: YAML files control which modules are used      │
│  Registry System: Register new modules with @BACKBONE_REGISTRY │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### The Registry Pattern (Key Design)

```python
# Detectron2 uses a REGISTRY pattern:
# Each component type has a registry (backbone, head, etc.)
# You register new implementations, then select via config

from detectron2.modeling import BACKBONE_REGISTRY

@BACKBONE_REGISTRY.register()
class MyCustomBackbone(nn.Module):
    """Your custom backbone, instantly usable in config!"""
    def __init__(self, cfg, input_shape):
        super().__init__()
        # ... your implementation
    
    def forward(self, image):
        # ... return feature dict {"p2": ..., "p3": ..., ...}
        pass

# Now in config:
# MODEL.BACKBONE.NAME: "MyCustomBackbone"  ← Just works!
```

### Data Flow

```
Image (H×W×3)
     │
     ▼
┌─────────────┐
│ Preprocessor│ → Resize, normalize, pad to batch
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Backbone   │ → Extract features {P2, P3, P4, P5, P6}
│  + FPN      │
└──────┬──────┘
       │
       ▼
┌─────────────┐      ┌──────────────────┐
│    RPN      │─────▶│  Proposals       │ (~1000 boxes)
│             │      │  (objectness +   │
│             │      │   rough boxes)   │
└─────────────┘      └────────┬─────────┘
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
            ┌──────────────┐   ┌──────────────┐
            │  Box Head    │   │  Mask Head   │ (optional)
            │ (classify +  │   │ (per-pixel   │
            │  refine box) │   │  mask)       │
            └──────────────┘   └──────────────┘
                    │                   │
                    ▼                   ▼
            ┌──────────────────────────────┐
            │     Final Predictions        │
            │  • Boxes + Classes + Scores  │
            │  • Masks (if Mask R-CNN)     │
            │  • Keypoints (if configured) │
            └──────────────────────────────┘
```

---

## 4. Model Zoo — Pre-trained Models

### 💡 Detectron2's Model Zoo is MASSIVE

> Pre-trained models on COCO, LVIS, Cityscapes, and more — ready to use or fine-tune.

### Object Detection Models

| Model | Backbone | AP (COCO) | Inference (ms) | Config |
|-------|----------|-----------|---------------|--------|
| Faster R-CNN | R-50-FPN | 37.9 | 51 | `COCO-Detection/faster_rcnn_R_50_FPN_3x.yaml` |
| Faster R-CNN | R-101-FPN | 42.0 | 67 | `COCO-Detection/faster_rcnn_R_101_FPN_3x.yaml` |
| Faster R-CNN | X-101-FPN | 43.0 | 104 | `COCO-Detection/faster_rcnn_X_101_32x8d_FPN_3x.yaml` |
| RetinaNet | R-50-FPN | 38.7 | 48 | `COCO-Detection/retinanet_R_50_FPN_3x.yaml` |
| RetinaNet | R-101-FPN | 40.4 | 65 | `COCO-Detection/retinanet_R_101_FPN_3x.yaml` |

### Instance Segmentation Models

| Model | Backbone | Box AP | Mask AP | Config |
|-------|----------|--------|---------|--------|
| Mask R-CNN | R-50-FPN | 38.6 | 35.2 | `COCO-InstanceSegmentation/mask_rcnn_R_50_FPN_3x.yaml` |
| Mask R-CNN | R-101-FPN | 42.9 | 38.6 | `COCO-InstanceSegmentation/mask_rcnn_R_101_FPN_3x.yaml` |
| Mask R-CNN | X-101-FPN | 44.3 | 39.5 | `COCO-InstanceSegmentation/mask_rcnn_X_101_32x8d_FPN_3x.yaml` |
| Cascade Mask R-CNN | R-50-FPN | 42.1 | 36.4 | `Misc/cascade_mask_rcnn_R_50_FPN_3x.yaml` |

### Keypoint Detection Models

| Model | Backbone | Box AP | Keypoint AP | Config |
|-------|----------|--------|------------|--------|
| Keypoint R-CNN | R-50-FPN | 53.6 | 64.0 | `COCO-Keypoints/keypoint_rcnn_R_50_FPN_3x.yaml` |
| Keypoint R-CNN | R-101-FPN | 56.4 | 66.1 | `COCO-Keypoints/keypoint_rcnn_R_101_FPN_3x.yaml` |
| Keypoint R-CNN | X-101-FPN | 57.3 | 66.7 | `COCO-Keypoints/keypoint_rcnn_X_101_32x8d_FPN_3x.yaml` |

### Panoptic Segmentation Models

| Model | Backbone | PQ | Config |
|-------|----------|-----|--------|
| Panoptic FPN | R-50-FPN | 39.4 | `COCO-PanopticSegmentation/panoptic_fpn_R_50_3x.yaml` |
| Panoptic FPN | R-101-FPN | 40.5 | `COCO-PanopticSegmentation/panoptic_fpn_R_101_3x.yaml` |

### 📝 Loading Models from Zoo

```python
from detectron2 import model_zoo

# List all available models
print(model_zoo.get_config_file("COCO-Detection/faster_rcnn_R_50_FPN_3x.yaml"))

# Get config + weights URL
config_file = "COCO-InstanceSegmentation/mask_rcnn_R_50_FPN_3x.yaml"
cfg = model_zoo.get_config(config_file, trained=True)
# trained=True automatically sets weights URL
```

---

## 5. Quick Start: Detection in Minutes

### 📝 Code: Object Detection (5 Lines)

```python
from detectron2 import model_zoo
from detectron2.engine import DefaultPredictor
from detectron2.config import get_cfg
from detectron2.utils.visualizer import Visualizer
from detectron2.data import MetadataCatalog
import cv2

# Step 1: Configure
cfg = get_cfg()
cfg.merge_from_file(model_zoo.get_config_file(
    "COCO-Detection/faster_rcnn_R_50_FPN_3x.yaml"
))
cfg.MODEL.WEIGHTS = model_zoo.get_checkpoint_url(
    "COCO-Detection/faster_rcnn_R_50_FPN_3x.yaml"
)
cfg.MODEL.ROI_HEADS.SCORE_THRESH_TEST = 0.5  # Confidence threshold

# Step 2: Create predictor
predictor = DefaultPredictor(cfg)

# Step 3: Run detection
image = cv2.imread("street.jpg")
outputs = predictor(image)

# Step 4: Visualize
v = Visualizer(image[:, :, ::-1], MetadataCatalog.get(cfg.DATASETS.TRAIN[0]))
out = v.draw_instance_predictions(outputs["instances"].to("cpu"))
cv2.imshow("Detection", out.get_image()[:, :, ::-1])
cv2.waitKey(0)
```

### 📝 Code: Instance Segmentation (Mask R-CNN)

```python
from detectron2 import model_zoo
from detectron2.engine import DefaultPredictor
from detectron2.config import get_cfg
from detectron2.utils.visualizer import Visualizer
from detectron2.data import MetadataCatalog
import cv2

# Load Mask R-CNN (detection + segmentation!)
cfg = get_cfg()
cfg.merge_from_file(model_zoo.get_config_file(
    "COCO-InstanceSegmentation/mask_rcnn_R_50_FPN_3x.yaml"
))
cfg.MODEL.WEIGHTS = model_zoo.get_checkpoint_url(
    "COCO-InstanceSegmentation/mask_rcnn_R_50_FPN_3x.yaml"
)
cfg.MODEL.ROI_HEADS.SCORE_THRESH_TEST = 0.5

predictor = DefaultPredictor(cfg)
image = cv2.imread("people.jpg")
outputs = predictor(image)

# outputs contains:
instances = outputs["instances"]
print(f"Detected {len(instances)} objects")
print(f"Classes: {instances.pred_classes}")   # Class IDs
print(f"Scores:  {instances.scores}")         # Confidence
print(f"Boxes:   {instances.pred_boxes}")     # Bounding boxes
print(f"Masks:   {instances.pred_masks.shape}")  # Binary masks!

# Visualize with masks
v = Visualizer(image[:, :, ::-1], MetadataCatalog.get(cfg.DATASETS.TRAIN[0]))
out = v.draw_instance_predictions(instances.to("cpu"))
cv2.imwrite("segmentation_output.jpg", out.get_image()[:, :, ::-1])
```

### 📝 Code: Keypoint Detection (Pose Estimation)

```python
cfg = get_cfg()
cfg.merge_from_file(model_zoo.get_config_file(
    "COCO-Keypoints/keypoint_rcnn_R_50_FPN_3x.yaml"
))
cfg.MODEL.WEIGHTS = model_zoo.get_checkpoint_url(
    "COCO-Keypoints/keypoint_rcnn_R_50_FPN_3x.yaml"
)
cfg.MODEL.ROI_HEADS.SCORE_THRESH_TEST = 0.7

predictor = DefaultPredictor(cfg)
image = cv2.imread("athlete.jpg")
outputs = predictor(image)

# Access keypoints
keypoints = outputs["instances"].pred_keypoints  # (N, 17, 3) → [x, y, confidence]
# 17 COCO keypoints: nose, eyes, ears, shoulders, elbows, wrists, hips, knees, ankles

# Visualize skeleton
v = Visualizer(image[:, :, ::-1], MetadataCatalog.get(cfg.DATASETS.TRAIN[0]))
out = v.draw_instance_predictions(outputs["instances"].to("cpu"))
cv2.imwrite("pose_output.jpg", out.get_image()[:, :, ::-1])
```

### 📝 Code: Panoptic Segmentation

```python
from detectron2.projects.panoptic_deeplab import add_panoptic_deeplab_config

cfg = get_cfg()
cfg.merge_from_file(model_zoo.get_config_file(
    "COCO-PanopticSegmentation/panoptic_fpn_R_50_3x.yaml"
))
cfg.MODEL.WEIGHTS = model_zoo.get_checkpoint_url(
    "COCO-PanopticSegmentation/panoptic_fpn_R_50_3x.yaml"
)

predictor = DefaultPredictor(cfg)
image = cv2.imread("city_scene.jpg")
outputs = predictor(image)

# Panoptic output: every pixel is labeled
panoptic_seg, segments_info = outputs["panoptic_seg"]
# panoptic_seg: (H, W) tensor — segment ID per pixel
# segments_info: list of dicts — {id, category_id, isthing, area, ...}

# Visualize
v = Visualizer(image[:, :, ::-1], MetadataCatalog.get(cfg.DATASETS.TRAIN[0]))
out = v.draw_panoptic_seg(panoptic_seg.to("cpu"), segments_info)
cv2.imwrite("panoptic_output.jpg", out.get_image()[:, :, ::-1])
```

---

## 6. Understanding the Config System

### 💡 Detectron2's Config = Nested YAML + Python Overrides

> The config system is hierarchical — base configs + task-specific configs + CLI overrides.

### Config Hierarchy

```
Base-RCNN-FPN.yaml          ← Shared settings for all R-CNN+FPN models
     │
     ▼
faster_rcnn_R_50_FPN_3x.yaml  ← Specific model settings (inherits base)
     │
     ▼
Your Python overrides         ← Runtime customization
     │
     ▼
CLI overrides                 ← Quick experiments
```

### 📝 Code: Config System Usage

```python
from detectron2.config import get_cfg

cfg = get_cfg()

# Load from YAML
cfg.merge_from_file("configs/COCO-Detection/faster_rcnn_R_50_FPN_3x.yaml")

# Override in Python
cfg.MODEL.WEIGHTS = "model_final.pth"
cfg.MODEL.ROI_HEADS.NUM_CLASSES = 5        # Your custom classes
cfg.MODEL.ROI_HEADS.SCORE_THRESH_TEST = 0.5
cfg.SOLVER.BASE_LR = 0.001
cfg.SOLVER.MAX_ITER = 5000
cfg.SOLVER.STEPS = (3000, 4000)            # LR decay steps
cfg.SOLVER.IMS_PER_BATCH = 4               # Batch size
cfg.DATALOADER.NUM_WORKERS = 4
cfg.INPUT.MIN_SIZE_TRAIN = (640, 672, 704, 736, 768, 800)  # Multi-scale
cfg.OUTPUT_DIR = "./output"

# Freeze config (no more changes after this)
cfg.freeze()

# Print entire config
print(cfg.dump())
```

### Key Config Options

```yaml
# MODEL settings
MODEL:
  META_ARCHITECTURE: "GeneralizedRCNN"   # or "RetinaNet", "PanopticFPN"
  BACKBONE:
    NAME: "build_resnet_fpn_backbone"
  RESNETS:
    DEPTH: 50                             # ResNet depth (50, 101, 152)
  FPN:
    IN_FEATURES: ["res2", "res3", "res4", "res5"]
  ANCHOR_GENERATOR:
    SIZES: [[32], [64], [128], [256], [512]]
    ASPECT_RATIOS: [[0.5, 1.0, 2.0]]
  ROI_HEADS:
    NAME: "StandardROIHeads"
    NUM_CLASSES: 80                       # CHANGE for custom dataset!
    SCORE_THRESH_TEST: 0.5
  WEIGHTS: "detectron2://model_final.pth" # Pre-trained weights

# SOLVER (training) settings
SOLVER:
  BASE_LR: 0.02                  # Base learning rate
  MAX_ITER: 90000                # Total training iterations
  STEPS: (60000, 80000)          # LR decay schedule
  GAMMA: 0.1                    # LR decay factor
  IMS_PER_BATCH: 16             # Total batch size (across GPUs)
  CHECKPOINT_PERIOD: 5000       # Save every N iterations
  WARMUP_ITERS: 1000            # LR warmup iterations
  WARMUP_FACTOR: 0.001          # Starting warmup LR factor

# INPUT settings
INPUT:
  MIN_SIZE_TRAIN: (640, 672, 704, 736, 768, 800)  # Random resize
  MAX_SIZE_TRAIN: 1333
  MIN_SIZE_TEST: 800
  MAX_SIZE_TEST: 1333
  FORMAT: "BGR"                   # OpenCV default

# DATASETS
DATASETS:
  TRAIN: ("coco_2017_train",)
  TEST: ("coco_2017_val",)
```

### CLI Overrides (Quick Experiments)

```bash
# Override any config value from command line
python tools/train_net.py \
    --config-file configs/COCO-Detection/faster_rcnn_R_50_FPN_3x.yaml \
    SOLVER.BASE_LR 0.001 \
    SOLVER.MAX_ITER 10000 \
    SOLVER.IMS_PER_BATCH 8 \
    MODEL.ROI_HEADS.NUM_CLASSES 3 \
    OUTPUT_DIR ./my_output
```

---

## 7. Supported Tasks & Models

### All Tasks in One Framework

```
┌──────────────────────────────────────────────────────────────┐
│                  DETECTRON2 TASK SUPPORT                      │
├──────────────┬──────────────┬───────────────┬───────────────┤
│  DETECTION   │ INSTANCE SEG │ PANOPTIC SEG  │  KEYPOINTS    │
│              │              │               │               │
│  ┌──────┐   │  ┌──────┐   │ ┌───────────┐ │    ○          │
│  │ BOX  │   │  │██████│   │ │STUFF+THING│ │   /|\         │
│  │      │   │  │██████│   │ │(all pixels)│ │   / \         │
│  └──────┘   │  └──────┘   │ └───────────┘ │  Skeleton     │
│              │              │               │               │
│ Faster R-CNN │ Mask R-CNN  │ Panoptic FPN  │ Keypoint RCNN │
│ RetinaNet   │ Cascade Mask│ PanopticDL    │               │
│ Cascade RCNN│ PointRend   │               │               │
└──────────────┴──────────────┴───────────────┴───────────────┘
```

### Instance Segmentation vs Semantic vs Panoptic

```
INSTANCE SEGMENTATION (Mask R-CNN):
  - Each object gets its OWN mask
  - Can separate car-1 from car-2
  - Only "things" (countable objects)
  
SEMANTIC SEGMENTATION (FCN, DeepLab):
  - Every pixel gets a class label
  - Cannot separate same-class objects
  - Includes "stuff" (sky, road, grass)

PANOPTIC SEGMENTATION (Panoptic FPN):
  - BOTH instance + semantic
  - Every pixel labeled
  - "Things" separated, "stuff" merged
  - The COMPLETE scene understanding

┌─────────────────────────────────────────────┐
│ Image of street scene:                      │
│                                             │
│  Instance: car-1🟦 car-2🟩 person-1🟥      │
│  Semantic: road⬛ sky🟦 building🟫          │
│  Panoptic: ALL of the above TOGETHER       │
└─────────────────────────────────────────────┘
```

### PointRend — Better Mask Edges

```
Standard Mask R-CNN: 28×28 mask (coarse, blurry edges)
PointRend:           Renders uncertain boundary points at high resolution!

Analogy: Like anti-aliasing in video games
  - Don't render everything at high-res (expensive)
  - Only refine the EDGES (where uncertainty is highest)
  - Result: Sharp mask boundaries at minimal extra cost!
```

```python
# Use PointRend for better masks
from detectron2.projects.point_rend import add_pointrend_config

cfg = get_cfg()
add_pointrend_config(cfg)
cfg.merge_from_file("projects/PointRend/configs/InstanceSegmentation/"
                     "pointrend_rcnn_R_50_FPN_3x_coco.yaml")
cfg.MODEL.WEIGHTS = "detectron2://PointRend/..."

predictor = DefaultPredictor(cfg)
# Same API, but masks have sharper edges!
```

---

## 8. Training on Custom Dataset (Step-by-Step)

### Step 1: Register Your Dataset

```python
from detectron2.data import DatasetCatalog, MetadataCatalog
from detectron2.structures import BoxMode
import os
import json

def get_my_dataset_dicts(img_dir, ann_file):
    """
    Load custom dataset in Detectron2 format.
    
    Returns list of dicts, one per image:
    {
        "file_name": str,
        "height": int,
        "width": int,
        "image_id": int,
        "annotations": [
            {
                "bbox": [x1, y1, x2, y2],
                "bbox_mode": BoxMode.XYXY_ABS,
                "category_id": int,
                "segmentation": [[polygon]] or RLE (optional),
            }, ...
        ]
    }
    """
    with open(ann_file) as f:
        coco_data = json.load(f)
    
    # Build image_id → annotations mapping
    img_anns = {}
    for ann in coco_data['annotations']:
        img_id = ann['image_id']
        if img_id not in img_anns:
            img_anns[img_id] = []
        img_anns[img_id].append(ann)
    
    # Build category mapping
    cat_map = {c['id']: i for i, c in enumerate(coco_data['categories'])}
    
    dataset_dicts = []
    for img_info in coco_data['images']:
        record = {
            "file_name": os.path.join(img_dir, img_info['file_name']),
            "height": img_info['height'],
            "width": img_info['width'],
            "image_id": img_info['id'],
        }
        
        annots = []
        for ann in img_anns.get(img_info['id'], []):
            x, y, w, h = ann['bbox']
            obj = {
                "bbox": [x, y, x + w, y + h],
                "bbox_mode": BoxMode.XYXY_ABS,
                "category_id": cat_map[ann['category_id']],
            }
            # Optional: add segmentation mask
            if 'segmentation' in ann:
                obj["segmentation"] = ann['segmentation']
            annots.append(obj)
        
        record["annotations"] = annots
        dataset_dicts.append(record)
    
    return dataset_dicts


# Register dataset
DatasetCatalog.register(
    "my_dataset_train",
    lambda: get_my_dataset_dicts("data/images/train", "data/annotations/train.json")
)
MetadataCatalog.get("my_dataset_train").set(
    thing_classes=["helmet", "no_helmet", "person"]
)

DatasetCatalog.register(
    "my_dataset_val",
    lambda: get_my_dataset_dicts("data/images/val", "data/annotations/val.json")
)
MetadataCatalog.get("my_dataset_val").set(
    thing_classes=["helmet", "no_helmet", "person"]
)
```

### Step 1b: Register COCO-Format Dataset (Even Easier!)

```python
from detectron2.data.datasets import register_coco_instances

# If your data is already in COCO JSON format, just do this:
register_coco_instances(
    "my_dataset_train",
    {},
    "data/annotations/train.json",
    "data/images/train"
)
register_coco_instances(
    "my_dataset_val",
    {},
    "data/annotations/val.json",
    "data/images/val"
)
```

### Step 2: Verify Dataset Registration

```python
from detectron2.data import DatasetCatalog, MetadataCatalog
from detectron2.utils.visualizer import Visualizer
import random
import cv2

# Verify data loads correctly
dataset_dicts = DatasetCatalog.get("my_dataset_train")
print(f"Total training images: {len(dataset_dicts)}")

# Visualize random samples
metadata = MetadataCatalog.get("my_dataset_train")
for d in random.sample(dataset_dicts, 3):
    img = cv2.imread(d["file_name"])
    v = Visualizer(img[:, :, ::-1], metadata=metadata, scale=0.5)
    out = v.draw_dataset_dict(d)
    cv2.imshow("Sample", out.get_image()[:, :, ::-1])
    cv2.waitKey(0)
```

### Step 3: Configure Training

```python
from detectron2.config import get_cfg
from detectron2 import model_zoo

cfg = get_cfg()

# Start from a pre-trained model (transfer learning!)
cfg.merge_from_file(model_zoo.get_config_file(
    "COCO-Detection/faster_rcnn_R_50_FPN_3x.yaml"
))
cfg.MODEL.WEIGHTS = model_zoo.get_checkpoint_url(
    "COCO-Detection/faster_rcnn_R_50_FPN_3x.yaml"
)

# Dataset
cfg.DATASETS.TRAIN = ("my_dataset_train",)
cfg.DATASETS.TEST = ("my_dataset_val",)

# Number of classes
cfg.MODEL.ROI_HEADS.NUM_CLASSES = 3  # helmet, no_helmet, person

# Training settings
cfg.SOLVER.IMS_PER_BATCH = 4       # Batch size (reduce if OOM)
cfg.SOLVER.BASE_LR = 0.001         # Lower LR for fine-tuning
cfg.SOLVER.MAX_ITER = 5000         # Total iterations
cfg.SOLVER.STEPS = (3000, 4000)    # LR decay steps
cfg.SOLVER.GAMMA = 0.1             # Decay factor
cfg.SOLVER.WARMUP_ITERS = 500
cfg.SOLVER.CHECKPOINT_PERIOD = 1000

# Data loader
cfg.DATALOADER.NUM_WORKERS = 4

# Input
cfg.INPUT.MIN_SIZE_TRAIN = (640, 800)
cfg.INPUT.MAX_SIZE_TRAIN = 1333

# Test settings
cfg.MODEL.ROI_HEADS.SCORE_THRESH_TEST = 0.5
cfg.TEST.EVAL_PERIOD = 500         # Evaluate every N iterations

# Output directory
cfg.OUTPUT_DIR = "./output/my_detector"

os.makedirs(cfg.OUTPUT_DIR, exist_ok=True)
```

### Step 4: Train!

```python
from detectron2.engine import DefaultTrainer

# Standard training
trainer = DefaultTrainer(cfg)
trainer.resume_or_load(resume=False)  # False = start fresh from pretrained
trainer.train()

# After training, best model saved at:
# output/my_detector/model_final.pth
```

### Step 5: Train with Validation (Custom Trainer)

```python
from detectron2.engine import DefaultTrainer
from detectron2.evaluation import COCOEvaluator
import os

class MyTrainer(DefaultTrainer):
    """Custom trainer with periodic evaluation."""
    
    @classmethod
    def build_evaluator(cls, cfg, dataset_name, output_folder=None):
        if output_folder is None:
            output_folder = os.path.join(cfg.OUTPUT_DIR, "eval")
        return COCOEvaluator(dataset_name, output_dir=output_folder)

# Use custom trainer
trainer = MyTrainer(cfg)
trainer.resume_or_load(resume=False)
trainer.train()
```

### Step 6: Evaluate & Predict

```python
from detectron2.engine import DefaultPredictor
from detectron2.evaluation import COCOEvaluator, inference_on_dataset
from detectron2.data import build_detection_test_loader

# Load trained model
cfg.MODEL.WEIGHTS = os.path.join(cfg.OUTPUT_DIR, "model_final.pth")
predictor = DefaultPredictor(cfg)

# Run evaluation
evaluator = COCOEvaluator("my_dataset_val", output_dir="./eval_output")
val_loader = build_detection_test_loader(cfg, "my_dataset_val")
results = inference_on_dataset(predictor.model, val_loader, evaluator)
print(results)
# {'bbox': {'AP': 45.2, 'AP50': 72.1, 'AP75': 49.3, ...}}

# Predict on new images
image = cv2.imread("new_test.jpg")
outputs = predictor(image)

# Access results
instances = outputs["instances"]
boxes = instances.pred_boxes.tensor.cpu().numpy()
scores = instances.scores.cpu().numpy()
classes = instances.pred_classes.cpu().numpy()

class_names = MetadataCatalog.get("my_dataset_val").thing_classes
for box, score, cls in zip(boxes, scores, classes):
    print(f"{class_names[cls]}: {score:.3f} at {box}")
```

---

## 9. Advanced: Custom Models & Components

### Build a Custom Backbone

```python
import torch.nn as nn
from detectron2.modeling import BACKBONE_REGISTRY, Backbone, ShapeSpec

@BACKBONE_REGISTRY.register()
class TinyBackbone(Backbone):
    """A minimal custom backbone for demonstration."""
    
    def __init__(self, cfg, input_shape):
        super().__init__()
        
        # Simple feature extractor
        self.conv1 = nn.Sequential(
            nn.Conv2d(3, 64, 7, stride=2, padding=3),
            nn.BatchNorm2d(64), nn.ReLU()
        )
        self.conv2 = nn.Sequential(
            nn.Conv2d(64, 128, 3, stride=2, padding=1),
            nn.BatchNorm2d(128), nn.ReLU()
        )
        self.conv3 = nn.Sequential(
            nn.Conv2d(128, 256, 3, stride=2, padding=1),
            nn.BatchNorm2d(256), nn.ReLU()
        )
        
        # Output feature shapes (needed by FPN)
        self._out_features = ["feat1", "feat2", "feat3"]
        self._out_feature_channels = {"feat1": 64, "feat2": 128, "feat3": 256}
        self._out_feature_strides = {"feat1": 4, "feat2": 8, "feat3": 16}
    
    def forward(self, x):
        f1 = self.conv1(x)
        f2 = self.conv2(f1)
        f3 = self.conv3(f2)
        return {"feat1": f1, "feat2": f2, "feat3": f3}
    
    def output_shape(self):
        return {
            name: ShapeSpec(channels=self._out_feature_channels[name],
                           stride=self._out_feature_strides[name])
            for name in self._out_features
        }

# Use in config:
# cfg.MODEL.BACKBONE.NAME = "TinyBackbone"
```

### Custom Data Augmentation

```python
from detectron2.data import transforms as T
from detectron2.data import DatasetMapper, build_detection_train_loader

def custom_mapper(dataset_dict):
    """Custom data augmentation pipeline."""
    dataset_dict = dataset_dict.copy()
    image = cv2.imread(dataset_dict["file_name"])
    
    # Define augmentation pipeline
    augs = [
        T.RandomFlip(prob=0.5, horizontal=True),
        T.RandomBrightness(0.8, 1.2),
        T.RandomContrast(0.8, 1.2),
        T.RandomSaturation(0.8, 1.2),
        T.ResizeShortestEdge(
            short_edge_length=[640, 672, 704, 736, 768, 800],
            max_size=1333
        ),
        T.RandomCrop("relative_range", (0.5, 0.5)),
    ]
    
    # Apply augmentations
    aug_input = T.AugInput(image)
    transforms = T.AugmentationList(augs)(aug_input)
    image = aug_input.image
    
    # Transform annotations
    annos = [
        transform_instance_annotations(obj, transforms, image.shape[:2])
        for obj in dataset_dict.pop("annotations")
    ]
    
    dataset_dict["image"] = torch.as_tensor(image.transpose(2, 0, 1).astype("float32"))
    dataset_dict["instances"] = annotations_to_instances(annos, image.shape[:2])
    
    return dataset_dict

# Use in custom trainer:
class AugTrainer(DefaultTrainer):
    @classmethod
    def build_train_loader(cls, cfg):
        return build_detection_train_loader(cfg, mapper=custom_mapper)
```

### Custom Loss Function

```python
from detectron2.modeling import ROI_HEADS_REGISTRY
from detectron2.modeling.roi_heads import StandardROIHeads

@ROI_HEADS_REGISTRY.register()
class CustomROIHeads(StandardROIHeads):
    """ROI Heads with custom loss weighting."""
    
    def losses(self, predictions, proposals):
        # Get standard losses
        losses = super().losses(predictions, proposals)
        
        # Custom: weight classification loss higher
        losses["loss_cls"] = losses["loss_cls"] * 2.0
        
        # Custom: add L1 regularization
        for param in self.box_predictor.parameters():
            losses["loss_reg"] = losses.get("loss_reg", 0) + 0.001 * param.abs().sum()
        
        return losses

# cfg.MODEL.ROI_HEADS.NAME = "CustomROIHeads"
```

---

## 10. Evaluation & Visualization

### COCO Evaluation (Standard)

```python
from detectron2.evaluation import COCOEvaluator, inference_on_dataset
from detectron2.data import build_detection_test_loader

evaluator = COCOEvaluator("my_dataset_val", output_dir="./eval")
val_loader = build_detection_test_loader(cfg, "my_dataset_val")
results = inference_on_dataset(predictor.model, val_loader, evaluator)

# Results dict:
# results["bbox"]["AP"]     → mAP @ IoU 0.5:0.95
# results["bbox"]["AP50"]   → mAP @ IoU 0.5
# results["bbox"]["AP75"]   → mAP @ IoU 0.75
# results["bbox"]["APs"]    → mAP for small objects
# results["bbox"]["APm"]    → mAP for medium objects
# results["bbox"]["APl"]    → mAP for large objects
# results["segm"]["AP"]     → Mask mAP (if available)
```

### Visualization Tools

```python
from detectron2.utils.visualizer import Visualizer, ColorMode

image = cv2.imread("test.jpg")
outputs = predictor(image)
metadata = MetadataCatalog.get("my_dataset_val")

# Style 1: Default (colored instances)
v = Visualizer(image[:, :, ::-1], metadata, scale=1.0)
out = v.draw_instance_predictions(outputs["instances"].to("cpu"))

# Style 2: Segmentation overlay
v = Visualizer(image[:, :, ::-1], metadata, scale=1.0,
               instance_mode=ColorMode.SEGMENTATION)
out = v.draw_instance_predictions(outputs["instances"].to("cpu"))

# Style 3: Black & white image with colored masks
v = Visualizer(image[:, :, ::-1], metadata, scale=1.0,
               instance_mode=ColorMode.IMAGE_BW)
out = v.draw_instance_predictions(outputs["instances"].to("cpu"))

# Save result
cv2.imwrite("visualization.jpg", out.get_image()[:, :, ::-1])
```

### 📝 Code: Batch Visualization

```python
import random
from detectron2.utils.visualizer import Visualizer
import matplotlib.pyplot as plt

# Visualize predictions on validation set
dataset_dicts = DatasetCatalog.get("my_dataset_val")
metadata = MetadataCatalog.get("my_dataset_val")

fig, axes = plt.subplots(2, 3, figsize=(18, 12))
for ax, d in zip(axes.flatten(), random.sample(dataset_dicts, 6)):
    img = cv2.imread(d["file_name"])
    outputs = predictor(img)
    
    v = Visualizer(img[:, :, ::-1], metadata, scale=0.5)
    out = v.draw_instance_predictions(outputs["instances"].to("cpu"))
    
    ax.imshow(out.get_image())
    ax.axis('off')
    ax.set_title(f"{len(outputs['instances'])} objects")

plt.tight_layout()
plt.savefig("validation_grid.jpg", dpi=150)
plt.show()
```

---

## 11. Deployment & Production

### Export to ONNX

```python
from detectron2.export import TracingAdapter
import torch

# Create traced model
model = predictor.model
model.eval()

# Prepare sample input
sample_image = torch.randn(1, 3, 800, 800).cuda()

# Trace
traced = torch.jit.trace(model, sample_image)
torch.jit.save(traced, "model_traced.pt")

# Or export to ONNX
from detectron2.export import dump_torchscript_IR, scripting_with_instances

# Use Detectron2's export tools
from detectron2.export import Caffe2Tracer

tracer = Caffe2Tracer(cfg, model, sample_input)
torchscript_model = tracer.export_torchscript()
torchscript_model.save("model.torchscript")
```

### Export to TorchScript (Recommended)

```python
from detectron2.export import scripting_with_instances
from detectron2.structures import Boxes

# Export with proper instance handling
fields = {"pred_boxes": Boxes, "scores": torch.Tensor, "pred_classes": torch.Tensor}
torchscript_model = scripting_with_instances(predictor.model, fields)
torch.jit.save(torchscript_model, "model_scripted.pt")

# Load and use
loaded_model = torch.jit.load("model_scripted.pt")
# ... run inference
```

### Deployment as API (FastAPI)

```python
from fastapi import FastAPI, UploadFile, File
from detectron2.engine import DefaultPredictor
from detectron2.config import get_cfg
from detectron2 import model_zoo
import cv2
import numpy as np

app = FastAPI()

# Initialize predictor once
cfg = get_cfg()
cfg.merge_from_file(model_zoo.get_config_file(
    "COCO-InstanceSegmentation/mask_rcnn_R_50_FPN_3x.yaml"
))
cfg.MODEL.WEIGHTS = "model_final.pth"
cfg.MODEL.ROI_HEADS.NUM_CLASSES = 3
cfg.MODEL.ROI_HEADS.SCORE_THRESH_TEST = 0.5
predictor = DefaultPredictor(cfg)

@app.post("/detect")
async def detect(file: UploadFile = File(...)):
    # Read image
    contents = await file.read()
    nparr = np.frombuffer(contents, np.uint8)
    image = cv2.imdecode(nparr, cv2.IMREAD_COLOR)
    
    # Run detection
    outputs = predictor(image)
    instances = outputs["instances"].to("cpu")
    
    # Format response
    results = []
    for i in range(len(instances)):
        box = instances.pred_boxes[i].tensor.numpy()[0].tolist()
        results.append({
            "class_id": int(instances.pred_classes[i]),
            "score": float(instances.scores[i]),
            "bbox": box
        })
    
    return {"detections": results, "count": len(results)}

# Run: uvicorn app:app --host 0.0.0.0 --port 8000
```

### ⚠️ Deployment Limitations

```
Detectron2 was designed for RESEARCH, not production:

❌ No built-in TFLite/CoreML export
❌ No mobile-optimized models
❌ Heavy dependencies (full PyTorch + detectron2)
❌ Not optimized for edge deployment
❌ No built-in serving solution

For production deployment, consider:
  → Export to ONNX → ONNX Runtime (CPU/GPU)
  → Export to TorchScript → LibTorch (C++)
  → Use YOLO for production (much easier)
  → Use Detectron2 for research → export best model to ONNX
```

---

## 12. Detectron2 vs YOLO vs TF OD API vs MMDetection

### Complete Comparison

| Feature | Detectron2 | YOLO | TF OD API | MMDetection |
|---------|-----------|------|-----------|-------------|
| **Creator** | Meta AI | Ultralytics | Google | OpenMMLab |
| **Framework** | PyTorch | PyTorch | TensorFlow | PyTorch |
| **Ease of use** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **Model variety** | 20+ architectures | YOLO only | 40+ | 300+ |
| **Research use** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Instance seg** | ✅ Best (Mask R-CNN) | ✅ (YOLO-seg) | ⚠️ Limited | ✅ Great |
| **Panoptic seg** | ✅ Native | ❌ | ❌ | ✅ |
| **Keypoints** | ✅ Native | ✅ (YOLO-pose) | ✅ | ✅ |
| **Speed** | Moderate | Fastest | Moderate | Moderate |
| **Mobile deploy** | ❌ Difficult | ✅ Easy | ✅ TFLite | ❌ Difficult |
| **Modularity** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Documentation** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Community** | Large (research) | Massive (industry) | Medium | Large (Asia) |

### When to Choose What

```
"I want to DEPLOY to production quickly"
  → YOLO (easiest, fastest, most deployment options)

"I want the BEST instance segmentation"
  → Detectron2 (Mask R-CNN is the gold standard)

"I want to DO RESEARCH or publish papers"
  → Detectron2 or MMDetection (expected by reviewers)

"I want to COMPARE 100+ architectures"
  → MMDetection (largest model zoo)

"I need MOBILE deployment"
  → TF OD API (TFLite) or YOLO (CoreML/NCNN)

"I need PANOPTIC segmentation"
  → Detectron2 (best panoptic support)
```

---

## 13. Real-World Use Cases & Companies

### 🏢 Case Study 1: Meta/Facebook — Content Moderation

```
Task: Detect harmful content in images/videos
Model: Custom Mask R-CNN (Detectron2)
Scale: Billions of images per day

What they detect:
  - Weapons, violence, nudity
  - Hate symbols
  - Manipulated media
  - Dangerous activities

Why Detectron2:
  - Built in-house, deeply integrated
  - Instance segmentation for precise object boundaries
  - Multi-task: detect, segment, classify simultaneously
  - Easy to add new detection categories
```

### 🏢 Case Study 2: Instagram — AR Effects & Filters

```
Task: Real-time face/body segmentation for AR filters
Model: Optimized Mask R-CNN variant
Pipeline:
  Camera feed → Face detection → Body segmentation → Apply filter → Render
  
Why: Precise mask needed (hair, clothes, body outline)
     PointRend for sharp mask edges
```

### 🏢 Case Study 3: Medical Research — Cell Segmentation

```
Task: Detect and segment individual cells in microscopy images
Model: Mask R-CNN fine-tuned on medical data
Dataset: 10,000 annotated microscopy images
Why Detectron2:
  - Instance segmentation separates touching cells
  - Mask R-CNN handles overlapping objects well
  - Easy to fine-tune from COCO pre-trained
  - Standard for publishing medical imaging papers
Result: 89% AP for cell detection, 83% mask AP
```

### 🏢 Case Study 4: Autonomous Driving Research

```
Task: Scene understanding for self-driving R&D
Model: Panoptic FPN (Detectron2)
Why: Need BOTH instance seg (cars, people) + semantic seg (road, sky)
     Panoptic gives the COMPLETE scene picture

Dataset: Cityscapes (5000 urban images)
  - "Things": car, person, bicycle, bus, truck, motorcycle
  - "Stuff": road, sidewalk, building, sky, vegetation, terrain
```

### 🏢 Case Study 5: Satellite Image Analysis

```
Task: Detect buildings, roads, and vehicles in aerial imagery
Model: Cascade Mask R-CNN with ResNeXt-101 backbone
Why Detectron2:
  - Cascade for precise bounding boxes (important for mapping)
  - Mask R-CNN for building footprint extraction
  - Handles varied object sizes (large buildings, tiny cars)
  - Easy to swap backbones for experimentation
```

---

## 14. Tips, Tricks & Best Practices

### ✅ Training Best Practices

| Tip | Explanation |
|-----|------------|
| **Start from COCO pre-trained** | Transfer learning gives +15-25% mAP over random init |
| **Use smaller LR for fine-tuning** | 0.001-0.005 (not 0.02 default which is for from-scratch) |
| **Reduce batch size if OOM** | Detectron2 uses `IMS_PER_BATCH` (total across GPUs) |
| **Use `SOLVER.STEPS` for LR decay** | Set to 70% and 90% of MAX_ITER |
| **Register datasets properly** | Most errors come from wrong dataset format |
| **Verify with visualization** | Always visualize a few samples before training |
| **Use evaluation during training** | Set `TEST.EVAL_PERIOD` to catch overfitting |
| **Freeze backbone for small datasets** | `cfg.MODEL.BACKBONE.FREEZE_AT = 2` |

### ⚠️ Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `CUDA out of memory` | Batch/image too large | Reduce `IMS_PER_BATCH` or `INPUT.MIN_SIZE_TRAIN` |
| `KeyError: 'my_dataset'` | Dataset not registered | Call `DatasetCatalog.register()` before training |
| `num_classes mismatch` | Config doesn't match data | Set `ROI_HEADS.NUM_CLASSES` correctly |
| `Empty predictions` | Threshold too high | Lower `SCORE_THRESH_TEST` |
| `Loss is NaN` | LR too high or bad data | Lower `BASE_LR`, check annotations |
| `Slow data loading` | Few workers | Increase `DATALOADER.NUM_WORKERS` |
| `Poor accuracy` | Not enough iterations | Increase `MAX_ITER`, check data quality |

### Training Schedule Guide

```
Small dataset (<1000 images):
  MAX_ITER: 3000-5000
  BASE_LR: 0.001
  IMS_PER_BATCH: 2-4
  FREEZE_AT: 2 (freeze early backbone)

Medium dataset (1000-10,000 images):
  MAX_ITER: 10000-30000
  BASE_LR: 0.005
  IMS_PER_BATCH: 4-8
  FREEZE_AT: 0

Large dataset (10,000+ images):
  MAX_ITER: 50000-90000
  BASE_LR: 0.02
  IMS_PER_BATCH: 8-16
  Multi-GPU recommended
```

### Performance Optimization

```python
# Speed up training:
cfg.SOLVER.AMP.ENABLED = True        # Mixed precision (FP16) → 1.5-2× faster
cfg.DATALOADER.NUM_WORKERS = 8       # Parallel data loading
cfg.SOLVER.IMS_PER_BATCH = 16        # Larger batch (if GPU allows)

# Speed up inference:
cfg.MODEL.ROI_HEADS.SCORE_THRESH_TEST = 0.7  # Higher threshold = fewer boxes to process
cfg.INPUT.MIN_SIZE_TEST = 640                  # Smaller input = faster

# Multi-GPU training:
# python tools/train_net.py --num-gpus 4 --config-file ...
```

---

## 📋 Chapter Summary

### What You Learned

```
✅ Detectron2 = Meta's modular PyTorch detection framework
✅ Best for: research, instance segmentation, panoptic segmentation
✅ Registry pattern: register custom backbones, heads, losses
✅ Config system: YAML + Python overrides + CLI overrides
✅ Model Zoo: Faster R-CNN, Mask R-CNN, RetinaNet, Cascade, Panoptic
✅ 4 tasks: detection, instance seg, panoptic seg, keypoints
✅ Training: register dataset → config → DefaultTrainer → evaluate
✅ Custom components: backbone, augmentation, loss — all swappable
✅ Deploy: ONNX, TorchScript, FastAPI (not great for mobile)
✅ Used by: Meta, academic research, medical imaging, satellite
```

### 🔥 Interview Quick-Fire

| Question | Answer |
|----------|--------|
| "What is Detectron2?" | Meta's modular PyTorch framework for detection + segmentation, research standard |
| "Detectron2 vs YOLO?" | D2: research, modular, instance seg. YOLO: production, easy, fast |
| "What is Mask R-CNN?" | Faster R-CNN + mask prediction branch, instance segmentation standard |
| "What is panoptic segmentation?" | Instance seg (things) + semantic seg (stuff) = every pixel labeled |
| "How to fine-tune in Detectron2?" | Register dataset, merge config, set NUM_CLASSES, use DefaultTrainer |
| "What's the registry pattern?" | Register custom modules (backbone, head) to swap via config |
| "What is PointRend?" | Refinement method for sharper mask boundaries |
| "Best for instance segmentation?" | Detectron2 with Mask R-CNN (or Cascade Mask R-CNN) |

---

### ➡️ Next Chapter: [08 - MMDetection](./08-MMDetection.md)

> Now you know Meta's research framework. Next, we'll explore OpenMMLab's MMDetection — the framework with the BIGGEST model zoo (300+ models) and the most comprehensive implementation of every detection architecture ever published.

---

*[← Back to Index](./00-INDEX.md) | [← Previous: 06 - TensorFlow OD API](./06-TensorFlow-OD-API.md)*
