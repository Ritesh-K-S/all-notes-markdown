# 📘 Chapter 06: TensorFlow Object Detection API

> **Goal**: Master Google's unified framework for training and deploying object detection models. It hosts dozens of pre-trained architectures (SSD, Faster R-CNN, EfficientDet, CenterNet) in one ecosystem with consistent APIs.

---

## 📑 Table of Contents

1. [What is the TF Object Detection API?](#1-what-is-the-tf-object-detection-api)
2. [Installation & Setup](#2-installation--setup)
3. [Model Zoo — Pre-trained Models](#3-model-zoo--pre-trained-models)
4. [Architecture: How the API is Organized](#4-architecture-how-the-api-is-organized)
5. [Quick Start: Detection in 10 Lines](#5-quick-start-detection-in-10-lines)
6. [Understanding Pipeline Config Files](#6-understanding-pipeline-config-files)
7. [Dataset Preparation (TFRecord Format)](#7-dataset-preparation-tfrecord-format)
8. [Training a Custom Model (Step-by-Step)](#8-training-a-custom-model-step-by-step)
9. [Evaluation & Metrics](#9-evaluation--metrics)
10. [Exporting & Deploying Models](#10-exporting--deploying-models)
11. [TF OD API vs YOLO vs Detectron2](#11-tf-od-api-vs-yolo-vs-detectron2)
12. [Real-World Use Cases & Companies](#12-real-world-use-cases--companies)
13. [Tips, Troubleshooting & Best Practices](#13-tips-troubleshooting--best-practices)

---

## 1. What is the TF Object Detection API?

### 💡 The Core Idea

> A **single framework** where you can pick ANY architecture (SSD, Faster R-CNN, EfficientDet, CenterNet), configure it via a `.config` file, train it on your data, and deploy it — all using the same pipeline.

### Think of It Like a Restaurant Menu

```
┌────────────────────────────────────────────────────────┐
│           TF OBJECT DETECTION API                       │
│           "Pick Your Architecture" Menu                 │
├────────────────────────────────────────────────────────┤
│                                                        │
│  🏗️ ARCHITECTURE (pick one):                          │
│     □ SSD (fast, simple)                              │
│     □ Faster R-CNN (accurate, two-stage)              │
│     □ EfficientDet (efficient)                        │
│     □ CenterNet (anchor-free)                         │
│     □ SSD MobileNet (mobile-optimized)               │
│                                                        │
│  🦴 BACKBONE (pick one):                              │
│     □ MobileNet V2                                    │
│     □ ResNet 50/101/152                               │
│     □ EfficientNet B0-B7                              │
│     □ Inception V2                                    │
│     □ Hourglass                                       │
│                                                        │
│  📊 DATASET: Your custom data (TFRecord format)       │
│                                                        │
│  🎯 OUTPUT: Trained model (.pb / SavedModel / TFLite) │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 🏢 Who Uses It

| Company | Use Case |
|---------|----------|
| **Google** | Google Lens, Google Photos, Street View |
| **Waymo** | Self-driving car perception R&D |
| **Uber ATG** | Autonomous vehicle detection pipeline |
| **Airbus** | Satellite object detection |
| **Bosch** | Industrial quality inspection |
| **Siemens** | Manufacturing defect detection |
| **Various hospitals** | Medical imaging pipelines |
| **Government agencies** | Surveillance, satellite analysis |

### Key Facts

| Fact | Detail |
|------|--------|
| **Creator** | Google Research |
| **Framework** | TensorFlow 2.x |
| **Repository** | github.com/tensorflow/models/tree/master/research/object_detection |
| **License** | Apache 2.0 (free commercial use) |
| **Models** | 40+ pre-trained models in Model Zoo |
| **Tasks** | Detection, Keypoint, Segmentation |
| **Deployment** | TF Serving, TFLite, TF.js, Coral Edge TPU |

### Why Use TF OD API?

```
✅ Google's "official" detection framework
✅ Most mature (since 2017, battle-tested)
✅ 40+ pre-trained models ready to use
✅ Config-file driven (no code changes to switch architectures!)
✅ TFLite/Coral deployment path (Google hardware)
✅ TF Serving for production (gRPC, REST)
✅ Integration with Google Cloud (Vertex AI)
✅ Supports TPU training (fast!)
```

---

## 2. Installation & Setup

### Prerequisites

```bash
# Python 3.8-3.11
# TensorFlow 2.x (GPU recommended)
pip install tensorflow[and-cuda]  # GPU version
# OR
pip install tensorflow  # CPU only
```

### Installation (Step-by-Step)

```bash
# Step 1: Clone the TensorFlow Models repository
git clone https://github.com/tensorflow/models.git
cd models/research

# Step 2: Compile Protobufs (required for config parsing)
# Windows:
protoc object_detection/protos/*.proto --python_out=.
# Linux/Mac:
protoc object_detection/protos/*.proto --python_out=.

# Step 3: Install the Object Detection API
cp object_detection/packages/tf2/setup.py .
pip install .

# Step 4: Install COCO API (for evaluation)
pip install pycocotools

# Step 5: Verify installation
python object_detection/builders/model_builder_tf2_test.py
# Should print: "OK" if everything works
```

### 📝 Quick Verification Script

```python
import tensorflow as tf
from object_detection.utils import label_map_util
from object_detection.utils import visualization_utils as viz_utils
from object_detection.builders import model_builder

print(f"TensorFlow version: {tf.__version__}")
print(f"GPU available: {tf.config.list_physical_devices('GPU')}")
print("✅ TF Object Detection API installed successfully!")
```

### Directory Structure After Setup

```
models/
└── research/
    └── object_detection/
        ├── builders/          # Model builders from config
        ├── configs/           # Example .config files
        ├── core/              # Core detection logic
        ├── data/              # Label maps, dataset tools
        ├── exporter_lib_v2/   # Export utilities
        ├── models/            # Architecture implementations
        ├── protos/            # Protocol buffer definitions
        ├── utils/             # Visualization, metrics
        ├── model_main_tf2.py  # Training entry point
        └── export_tflite_graph_tf2.py  # TFLite export
```

---

## 3. Model Zoo — Pre-trained Models

### 💡 The Model Zoo is the BIGGEST advantage of TF OD API

> 40+ models pre-trained on COCO, ready to download and use (or fine-tune).

### Popular Models (Speed vs Accuracy)

| Model | Speed (ms) | COCO AP | Use Case |
|-------|-----------|---------|----------|
| **SSD MobileNet V2** | 19 | 20.2% | Mobile, real-time |
| **SSD MobileNet V2 FPNLite** | 22 | 26.5% | Mobile, better accuracy |
| **EfficientDet D0** | 39 | 33.6% | Edge, balanced |
| **EfficientDet D1** | 54 | 38.4% | Edge, better accuracy |
| **CenterNet MobileNet V2** | 23 | 23.4% | Mobile, anchor-free |
| **CenterNet Hourglass-104** | 70 | 41.9% | Anchor-free, good accuracy |
| **SSD ResNet50 V1 FPN** | 46 | 34.3% | Server, balanced |
| **Faster R-CNN ResNet50 V1** | 89 | 31.0% | Server, two-stage |
| **Faster R-CNN ResNet101 V1** | 104 | 37.1% | Server, high accuracy |
| **Faster R-CNN ResNet152** | 148 | 37.4% | Research, max accuracy |
| **EfficientDet D4** | 133 | 48.5% | High accuracy |
| **EfficientDet D7** | 325 | 51.2% | Maximum accuracy |
| **Faster R-CNN Inception ResNet V2** | 236 | 37.7% | Research |

### Download a Pre-trained Model

```bash
# Download SSD MobileNet V2 (fastest)
wget http://download.tensorflow.org/models/object_detection/tf2/20200711/ssd_mobilenet_v2_320x320_coco17_tpu-8.tar.gz
tar -xvf ssd_mobilenet_v2_320x320_coco17_tpu-8.tar.gz

# Download Faster R-CNN ResNet50
wget http://download.tensorflow.org/models/object_detection/tf2/20200711/faster_rcnn_resnet50_v1_640x640_coco17_tpu-8.tar.gz

# Download EfficientDet D0
wget http://download.tensorflow.org/models/object_detection/tf2/20200711/efficientdet_d0_coco17_tpu-32.tar.gz
```

### Model Naming Convention

```
ssd_mobilenet_v2_320x320_coco17_tpu-8
│    │           │       │      │
│    │           │       │      └─ Training hardware (TPU pods)
│    │           │       └──────── Dataset (COCO 2017)
│    │           └──────────────── Input resolution
│    └──────────────────────────── Backbone
└───────────────────────────────── Architecture (SSD, Faster R-CNN, etc.)
```

---

## 4. Architecture: How the API is Organized

### The Config-Driven Design

```
The ENTIRE model is defined by a single .config file!
No code changes needed to switch architectures.

Switch from SSD to Faster R-CNN?
  → Just change the .config file!
  → Same training script, same data pipeline, same evaluation

This is the key design philosophy.
```

### Pipeline Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                   TF OD API PIPELINE                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────┐                                         │
│  │ pipeline.config │ ← Defines EVERYTHING                   │
│  └───────┬────────┘                                         │
│          │                                                   │
│    ┌─────┼─────────────────────────────────┐                │
│    │     │                                 │                │
│    ▼     ▼              ▼                  ▼                │
│ ┌──────┐ ┌───────────┐ ┌──────────┐ ┌──────────────┐      │
│ │model │ │train_config│ │eval_config│ │train_input_  │      │
│ │config│ │            │ │          │ │reader        │      │
│ └──┬───┘ └─────┬─────┘ └────┬─────┘ └──────┬───────┘      │
│    │            │            │              │               │
│    ▼            ▼            ▼              ▼               │
│ Architecture  Optimizer    Metrics      TFRecord           │
│ + Backbone    + LR Sched   + Eval       Data Pipeline      │
│ + Neck/Head   + Augment    Settings                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Supported Architectures (Meta-Architectures)

| Architecture | Type | Speed | Accuracy | Best For |
|-------------|------|-------|----------|----------|
| **SSD** | One-stage | ⚡⚡⚡⚡⚡ | ⭐⭐⭐ | Mobile, real-time |
| **Faster R-CNN** | Two-stage | ⚡⚡ | ⭐⭐⭐⭐⭐ | Max accuracy |
| **EfficientDet** | One-stage | ⚡⚡⚡⚡ | ⭐⭐⭐⭐ | Efficiency |
| **CenterNet** | Anchor-free | ⚡⚡⚡⚡ | ⭐⭐⭐⭐ | Keypoints, simple |

---

## 5. Quick Start: Detection in 10 Lines

### 📝 Code: Detect Objects Using Pre-trained Model

```python
import tensorflow as tf
import numpy as np
from PIL import Image
from object_detection.utils import label_map_util
from object_detection.utils import visualization_utils as viz_utils

# Step 1: Load pre-trained model
model = tf.saved_model.load('ssd_mobilenet_v2_320x320_coco17_tpu-8/saved_model')
detect_fn = model.signatures['serving_default']

# Step 2: Load label map (class names)
label_map = label_map_util.create_category_index_from_labelmap(
    'models/research/object_detection/data/mscoco_label_map.pbtxt',
    use_display_name=True
)

# Step 3: Load and prepare image
image = Image.open('test.jpg')
image_np = np.array(image)
input_tensor = tf.convert_to_tensor(image_np)[tf.newaxis, ...]

# Step 4: Run detection
detections = detect_fn(input_tensor)

# Step 5: Process results
num_detections = int(detections.pop('num_detections'))
detections = {key: value[0, :num_detections].numpy()
              for key, value in detections.items()}
detections['num_detections'] = num_detections
detections['detection_classes'] = detections['detection_classes'].astype(np.int64)

# Step 6: Visualize
image_with_boxes = image_np.copy()
viz_utils.visualize_boxes_and_labels_on_image_array(
    image_with_boxes,
    detections['detection_boxes'],
    detections['detection_classes'],
    detections['detection_scores'],
    label_map,
    use_normalized_coordinates=True,
    max_boxes_to_draw=20,
    min_score_thresh=0.5,
    line_thickness=2
)

# Save or display
Image.fromarray(image_with_boxes).save('output.jpg')
```

### 📝 Code: Even Simpler with TF Hub

```python
import tensorflow_hub as hub
import tensorflow as tf
import numpy as np
from PIL import Image

# Load ANY model from TF Hub in ONE line
detector = hub.load("https://tfhub.dev/tensorflow/ssd_mobilenet_v2/2")

# Prepare image
image = np.array(Image.open("test.jpg"))
input_tensor = tf.convert_to_tensor(image)[tf.newaxis, ...]

# Detect!
results = detector(input_tensor)

# Results
boxes = results['detection_boxes'][0].numpy()      # [N, 4] (y1,x1,y2,x2 normalized)
scores = results['detection_scores'][0].numpy()    # [N] confidence
classes = results['detection_classes'][0].numpy()   # [N] class IDs

# Filter
for i in range(len(scores)):
    if scores[i] > 0.5:
        print(f"Class {int(classes[i])}: {scores[i]:.2f} at {boxes[i]}")
```

---

## 6. Understanding Pipeline Config Files

### 💡 The .config File is the HEART of TF OD API

> Every model, every training run, every configuration is defined in a single `.config` protocol buffer text file. Change the config = change the entire behavior.

### Config File Structure (Sections)

```protobuf
# pipeline.config has 5 main sections:

model {
  # Architecture definition (SSD, Faster R-CNN, etc.)
  # Backbone, neck, head, anchor settings
}

train_config {
  # Training settings
  # Optimizer, learning rate, batch size
  # Data augmentation
}

train_input_reader {
  # Training data path and format
  # TFRecord path, label map path
}

eval_config {
  # Evaluation settings
  # Metrics, number of examples
}

eval_input_reader {
  # Validation data path
}
```

### 📝 Example: Complete SSD MobileNet Config

```protobuf
model {
  ssd {
    num_classes: 3  # YOUR number of classes
    
    image_resizer {
      fixed_shape_resizer {
        height: 320
        width: 320
      }
    }
    
    feature_extractor {
      type: "ssd_mobilenet_v2_fpn_keras"
      depth_multiplier: 1.0
      min_depth: 16
      conv_hyperparams {
        regularizer { l2_regularizer { weight: 0.00004 } }
        initializer { truncated_normal_initializer { stddev: 0.03 } }
        activation: RELU_6
        batch_norm { decay: 0.97 epsilon: 0.001 }
      }
    }
    
    box_coder {
      faster_rcnn_box_coder {
        y_scale: 10.0
        x_scale: 10.0
        height_scale: 5.0
        width_scale: 5.0
      }
    }
    
    anchor_generator {
      ssd_anchor_generator {
        num_layers: 6
        min_scale: 0.2
        max_scale: 0.95
        aspect_ratios: 1.0
        aspect_ratios: 2.0
        aspect_ratios: 0.5
        aspect_ratios: 3.0
        aspect_ratios: 0.333
      }
    }
    
    post_processing {
      batch_non_max_suppression {
        score_threshold: 0.3
        iou_threshold: 0.6
        max_detections_per_class: 100
        max_total_detections: 100
      }
      score_converter: SIGMOID
    }
    
    loss {
      classification_loss { weighted_sigmoid_focal { gamma: 2.0 alpha: 0.25 } }
      localization_loss { weighted_smooth_l1 {} }
      classification_weight: 1.0
      localization_weight: 1.0
    }
  }
}

train_config {
  batch_size: 16
  num_steps: 50000
  
  optimizer {
    momentum_optimizer {
      learning_rate {
        cosine_decay_learning_rate {
          learning_rate_base: 0.08
          total_steps: 50000
          warmup_learning_rate: 0.001
          warmup_steps: 1000
        }
      }
      momentum_optimizer_value: 0.9
    }
  }
  
  data_augmentation_options { random_horizontal_flip {} }
  data_augmentation_options { random_crop_image { min_area: 0.75 } }
  data_augmentation_options { random_adjust_brightness {} }
  data_augmentation_options { random_adjust_contrast {} }
  
  fine_tune_checkpoint: "ssd_mobilenet_v2/checkpoint/ckpt-0"
  fine_tune_checkpoint_type: "detection"
  
  startup_delay_steps: 0
  use_bfloat16: false
}

train_input_reader {
  label_map_path: "data/label_map.pbtxt"
  tf_record_input_reader {
    input_path: "data/train.tfrecord"
  }
}

eval_config {
  metrics_set: "coco_detection_metrics"
  use_moving_averages: false
  batch_size: 1
}

eval_input_reader {
  label_map_path: "data/label_map.pbtxt"
  tf_record_input_reader {
    input_path: "data/val.tfrecord"
  }
  shuffle: false
  num_epochs: 1
}
```

### Key Config Modifications (Common Changes)

| What to Change | Where in Config | Example |
|---------------|----------------|---------|
| Number of classes | `model.ssd.num_classes` | `num_classes: 5` |
| Input size | `model.ssd.image_resizer` | `height: 640, width: 640` |
| Batch size | `train_config.batch_size` | `batch_size: 8` |
| Learning rate | `train_config.optimizer` | `learning_rate_base: 0.04` |
| Total steps | `train_config.num_steps` | `num_steps: 100000` |
| Fine-tune checkpoint | `train_config.fine_tune_checkpoint` | Path to pre-trained model |
| Training data | `train_input_reader.tf_record_input_reader.input_path` | Your TFRecord |
| Label map | `train_input_reader.label_map_path` | Your label map |

---

## 7. Dataset Preparation (TFRecord Format)

### 💡 TF OD API requires data in TFRecord format

> TFRecord is TensorFlow's binary data format. It's efficient for streaming large datasets during training.

### Step 1: Create Label Map (.pbtxt)

```protobuf
# label_map.pbtxt
item {
  id: 1
  name: 'helmet'
}
item {
  id: 2
  name: 'no_helmet'
}
item {
  id: 3
  name: 'person'
}
```

> ⚠️ **Important**: IDs start at 1, NOT 0! (0 is reserved for background)

### Step 2: Convert Annotations to TFRecord

```python
"""
Convert your dataset to TFRecord format.
Supports: Pascal VOC XML, COCO JSON, CSV, or custom annotations.
"""
import tensorflow as tf
import os
import io
from PIL import Image
from object_detection.utils import dataset_util
import json

def create_tf_example(image_path, annotations, label_map):
    """
    Create a single TFRecord example from image + annotations.
    
    Args:
        image_path: Path to image file
        annotations: List of dicts with keys: 'bbox', 'class'
                    bbox format: [xmin, ymin, xmax, ymax] (absolute pixels)
        label_map: Dict mapping class name → ID
    """
    # Read image
    with tf.io.gfile.GFile(image_path, 'rb') as fid:
        encoded_image = fid.read()
    
    image = Image.open(io.BytesIO(encoded_image))
    width, height = image.size
    
    # Get filename and format
    filename = os.path.basename(image_path).encode('utf8')
    image_format = b'jpeg' if image_path.endswith('.jpg') else b'png'
    
    # Prepare annotation lists
    xmins, xmaxs, ymins, ymaxs = [], [], [], []
    classes_text, classes = [], []
    
    for ann in annotations:
        xmin, ymin, xmax, ymax = ann['bbox']
        
        # Normalize to [0, 1]
        xmins.append(xmin / width)
        xmaxs.append(xmax / width)
        ymins.append(ymin / height)
        ymaxs.append(ymax / height)
        
        class_name = ann['class']
        classes_text.append(class_name.encode('utf8'))
        classes.append(label_map[class_name])
    
    # Create TFRecord example
    tf_example = tf.train.Example(features=tf.train.Features(feature={
        'image/height': dataset_util.int64_feature(height),
        'image/width': dataset_util.int64_feature(width),
        'image/filename': dataset_util.bytes_feature(filename),
        'image/source_id': dataset_util.bytes_feature(filename),
        'image/encoded': dataset_util.bytes_feature(encoded_image),
        'image/format': dataset_util.bytes_feature(image_format),
        'image/object/bbox/xmin': dataset_util.float_list_feature(xmins),
        'image/object/bbox/xmax': dataset_util.float_list_feature(xmaxs),
        'image/object/bbox/ymin': dataset_util.float_list_feature(ymins),
        'image/object/bbox/ymax': dataset_util.float_list_feature(ymaxs),
        'image/object/class/text': dataset_util.bytes_list_feature(classes_text),
        'image/object/class/label': dataset_util.int64_list_feature(classes),
    }))
    
    return tf_example


def create_tfrecord(image_dir, annotations_file, output_path, label_map):
    """Convert entire dataset to TFRecord."""
    writer = tf.io.TFRecordWriter(output_path)
    
    # Load annotations (example: COCO format)
    with open(annotations_file) as f:
        coco_data = json.load(f)
    
    # Build image → annotations mapping
    img_anns = {}
    for ann in coco_data['annotations']:
        img_id = ann['image_id']
        if img_id not in img_anns:
            img_anns[img_id] = []
        
        # COCO format: [x, y, width, height] → [xmin, ymin, xmax, ymax]
        x, y, w, h = ann['bbox']
        cat_name = next(c['name'] for c in coco_data['categories'] 
                       if c['id'] == ann['category_id'])
        img_anns[img_id].append({
            'bbox': [x, y, x+w, y+h],
            'class': cat_name
        })
    
    # Create TFRecords
    count = 0
    for img_info in coco_data['images']:
        image_path = os.path.join(image_dir, img_info['file_name'])
        annotations = img_anns.get(img_info['id'], [])
        
        tf_example = create_tf_example(image_path, annotations, label_map)
        writer.write(tf_example.SerializeToString())
        count += 1
    
    writer.close()
    print(f"✅ Created {output_path} with {count} examples")


# Usage
label_map = {'helmet': 1, 'no_helmet': 2, 'person': 3}

create_tfrecord(
    image_dir='dataset/images/train/',
    annotations_file='dataset/annotations/train.json',
    output_path='data/train.tfrecord',
    label_map=label_map
)

create_tfrecord(
    image_dir='dataset/images/val/',
    annotations_file='dataset/annotations/val.json',
    output_path='data/val.tfrecord',
    label_map=label_map
)
```

### 📝 Convert from Pascal VOC XML

```python
import xml.etree.ElementTree as ET
import glob

def parse_voc_xml(xml_path):
    """Parse Pascal VOC XML annotation."""
    tree = ET.parse(xml_path)
    root = tree.getroot()
    
    size = root.find('size')
    width = int(size.find('width').text)
    height = int(size.find('height').text)
    
    annotations = []
    for obj in root.findall('object'):
        name = obj.find('name').text
        bbox = obj.find('bndbox')
        xmin = int(bbox.find('xmin').text)
        ymin = int(bbox.find('ymin').text)
        xmax = int(bbox.find('xmax').text)
        ymax = int(bbox.find('ymax').text)
        
        annotations.append({
            'class': name,
            'bbox': [xmin, ymin, xmax, ymax]
        })
    
    return annotations, width, height

# Convert all XMLs
for xml_file in glob.glob('annotations/*.xml'):
    annotations, w, h = parse_voc_xml(xml_file)
    # ... create tf_example and write to TFRecord
```

### Final Dataset Directory Structure

```
project/
├── data/
│   ├── train.tfrecord       # Training data
│   ├── val.tfrecord         # Validation data
│   └── label_map.pbtxt     # Class definitions
├── models/
│   ├── ssd_mobilenet_v2/    # Downloaded pre-trained model
│   │   ├── checkpoint/
│   │   └── pipeline.config
│   └── my_model/            # Your training output
│       └── pipeline.config  # Modified config
└── exported_model/          # Exported for deployment
```

---

## 8. Training a Custom Model (Step-by-Step)

### The Complete Training Workflow

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────┐
│ 1. Prepare  │───▶│ 2. Configure │───▶│ 3. Train    │───▶│4. Export │
│    Data     │    │    Pipeline  │    │             │    │          │
│ (TFRecord)  │    │  (.config)   │    │(model_main) │    │(SavedModel)│
└─────────────┘    └──────────────┘    └─────────────┘    └──────────┘
```

### Step 1: Choose & Download Base Model

```bash
# Pick a model from Model Zoo based on your needs:
# Fast + Mobile → SSD MobileNet V2 FPNLite 320x320
# Balanced     → SSD ResNet50 V1 FPN 640x640
# High accuracy → Faster R-CNN ResNet101 V1 640x640

# Download (example: SSD MobileNet)
wget http://download.tensorflow.org/models/object_detection/tf2/20200711/ssd_mobilenet_v2_fpnlite_320x320_coco17_tpu-8.tar.gz
tar -xvf ssd_mobilenet_v2_fpnlite_320x320_coco17_tpu-8.tar.gz
```

### Step 2: Modify pipeline.config

```python
# Script to auto-modify config file
import re

def modify_config(config_path, output_path, num_classes, batch_size, 
                  num_steps, fine_tune_checkpoint, train_record, 
                  val_record, label_map_path):
    """Modify pipeline.config for custom training."""
    
    with open(config_path, 'r') as f:
        config = f.read()
    
    # Modify number of classes
    config = re.sub(r'num_classes: \d+', f'num_classes: {num_classes}', config)
    
    # Modify batch size
    config = re.sub(r'batch_size: \d+', f'batch_size: {batch_size}', config)
    
    # Modify number of training steps
    config = re.sub(r'num_steps: \d+', f'num_steps: {num_steps}', config)
    
    # Modify fine-tune checkpoint
    config = re.sub(
        r'fine_tune_checkpoint: ".*?"',
        f'fine_tune_checkpoint: "{fine_tune_checkpoint}"',
        config
    )
    
    # Set fine_tune_checkpoint_type to "detection"
    config = re.sub(
        r'fine_tune_checkpoint_type: ".*?"',
        'fine_tune_checkpoint_type: "detection"',
        config
    )
    
    # Modify train TFRecord path
    config = re.sub(
        r'(train_input_reader[\s\S]*?input_path: )".*?"',
        f'\\1"{train_record}"',
        config
    )
    
    # Modify val TFRecord path
    config = re.sub(
        r'(eval_input_reader[\s\S]*?input_path: )".*?"',
        f'\\1"{val_record}"',
        config
    )
    
    # Modify label map paths
    config = re.sub(
        r'label_map_path: ".*?"',
        f'label_map_path: "{label_map_path}"',
        config
    )
    
    with open(output_path, 'w') as f:
        f.write(config)
    
    print(f"✅ Config saved to {output_path}")

# Usage
modify_config(
    config_path='ssd_mobilenet_v2/pipeline.config',
    output_path='models/my_model/pipeline.config',
    num_classes=3,
    batch_size=8,
    num_steps=50000,
    fine_tune_checkpoint='ssd_mobilenet_v2/checkpoint/ckpt-0',
    train_record='data/train.tfrecord',
    val_record='data/val.tfrecord',
    label_map_path='data/label_map.pbtxt'
)
```

### Step 3: Train!

```bash
# Training command
python model_main_tf2.py \
    --model_dir=models/my_model \
    --pipeline_config_path=models/my_model/pipeline.config

# This will:
# 1. Load the config
# 2. Build the model
# 3. Load pre-trained weights (transfer learning)
# 4. Train for num_steps
# 5. Save checkpoints every 1000 steps
# 6. Run evaluation periodically
```

### 📝 Code: Training with Python API (Alternative)

```python
import tensorflow as tf
from object_detection import model_lib_v2

# Set paths
pipeline_config = 'models/my_model/pipeline.config'
model_dir = 'models/my_model'

# Train
model_lib_v2.train_loop(
    pipeline_config_path=pipeline_config,
    model_dir=model_dir,
    train_steps=50000,
    use_tpu=False,
    checkpoint_every_n=1000,
    record_summaries=True
)
```

### Step 4: Monitor with TensorBoard

```bash
# Launch TensorBoard to monitor training
tensorboard --logdir=models/my_model

# Open browser: http://localhost:6006
# See: loss curves, mAP, learning rate, sample detections
```

### Step 5: Evaluate

```bash
# Run evaluation
python model_main_tf2.py \
    --model_dir=models/my_model \
    --pipeline_config_path=models/my_model/pipeline.config \
    --checkpoint_dir=models/my_model
```

---

## 9. Evaluation & Metrics

### COCO Metrics (Default)

```python
from object_detection.metrics import coco_evaluation

# After training, evaluation produces:
# DetectionBoxes_Precision/mAP          → Overall mAP (0.5:0.95)
# DetectionBoxes_Precision/mAP@.50IOU   → mAP at IoU=0.5
# DetectionBoxes_Precision/mAP@.75IOU   → mAP at IoU=0.75
# DetectionBoxes_Precision/mAP (small)  → Small objects
# DetectionBoxes_Precision/mAP (medium) → Medium objects
# DetectionBoxes_Precision/mAP (large)  → Large objects
# DetectionBoxes_Recall/AR@1            → Recall with 1 detection
# DetectionBoxes_Recall/AR@10           → Recall with 10 detections
# DetectionBoxes_Recall/AR@100          → Recall with 100 detections
```

### 📝 Code: Custom Evaluation Script

```python
import tensorflow as tf
import numpy as np
from object_detection.utils import label_map_util
from object_detection.metrics import coco_tools

def evaluate_model(model_path, test_tfrecord, label_map_path):
    """Evaluate a trained model on test data."""
    
    # Load model
    model = tf.saved_model.load(model_path)
    detect_fn = model.signatures['serving_default']
    
    # Load test data
    dataset = tf.data.TFRecordDataset(test_tfrecord)
    
    all_detections = []
    all_groundtruths = []
    
    for raw_record in dataset:
        # Parse TFRecord
        example = tf.train.Example()
        example.ParseFromString(raw_record.numpy())
        
        # Extract image and run detection
        encoded = example.features.feature['image/encoded'].bytes_list.value[0]
        image = tf.image.decode_image(encoded)
        input_tensor = tf.expand_dims(image, 0)
        
        detections = detect_fn(input_tensor)
        
        # Collect for COCO evaluation
        # ... (aggregate predictions and ground truths)
    
    # Compute COCO metrics
    # ... 
    return metrics

metrics = evaluate_model(
    'exported_model/saved_model',
    'data/test.tfrecord',
    'data/label_map.pbtxt'
)
print(f"mAP@0.5: {metrics['mAP@.50']:.3f}")
```

---

## 10. Exporting & Deploying Models

### Export to SavedModel (Standard Deployment)

```bash
# Export trained model to SavedModel format
python exporter_main_v2.py \
    --input_type=image_tensor \
    --pipeline_config_path=models/my_model/pipeline.config \
    --trained_checkpoint_dir=models/my_model \
    --output_directory=exported_model
```

### 📝 Code: Export Programmatically

```python
from object_detection import export_tflite_graph_lib_tf2 as export_lib

# Export to SavedModel
export_lib.export_tflite_model(
    pipeline_config='models/my_model/pipeline.config',
    trained_checkpoint_dir='models/my_model',
    output_directory='exported_model',
    max_detections=10
)
```

### Export to TFLite (Mobile/Edge)

```python
import tensorflow as tf

# Step 1: Export to TFLite-compatible SavedModel
# (Use export_tflite_graph_tf2.py)

# Step 2: Convert to TFLite
converter = tf.lite.TFLiteConverter.from_saved_model(
    'exported_model/saved_model'
)

# Optional: Quantize for smaller/faster model
converter.optimizations = [tf.lite.Optimize.DEFAULT]

# Optional: Full integer quantization (for Edge TPU)
def representative_dataset():
    for _ in range(100):
        yield [np.random.rand(1, 320, 320, 3).astype(np.float32)]

converter.representative_dataset = representative_dataset
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type = tf.uint8
converter.inference_output_type = tf.uint8

# Convert
tflite_model = converter.convert()

# Save
with open('model.tflite', 'wb') as f:
    f.write(tflite_model)

print(f"TFLite model size: {len(tflite_model) / 1024 / 1024:.1f} MB")
```

### Deploy with TF Serving (Production API)

```bash
# Step 1: Install TF Serving (Docker is easiest)
docker pull tensorflow/serving

# Step 2: Start serving your model
docker run -p 8501:8501 \
    --mount type=bind,source=/path/to/exported_model,target=/models/detector \
    -e MODEL_NAME=detector \
    tensorflow/serving

# Step 3: Send requests (REST API)
curl -X POST http://localhost:8501/v1/models/detector:predict \
    -d '{"instances": [{"input_tensor": [...]}]}'
```

### 📝 Code: Client for TF Serving

```python
import requests
import numpy as np
from PIL import Image
import json

def detect_via_serving(image_path, server_url="http://localhost:8501"):
    """Send image to TF Serving and get detections."""
    
    # Prepare image
    image = Image.open(image_path)
    image_np = np.array(image).tolist()
    
    # Build request
    data = json.dumps({
        "signature_name": "serving_default",
        "instances": [{"input_tensor": image_np}]
    })
    
    # Send request
    headers = {"content-type": "application/json"}
    response = requests.post(
        f"{server_url}/v1/models/detector:predict",
        data=data,
        headers=headers
    )
    
    # Parse response
    predictions = response.json()['predictions'][0]
    
    boxes = predictions['detection_boxes']
    scores = predictions['detection_scores']
    classes = predictions['detection_classes']
    
    # Filter by confidence
    results = []
    for box, score, cls in zip(boxes, scores, classes):
        if score > 0.5:
            results.append({
                'class': int(cls),
                'score': score,
                'box': box  # [y1, x1, y2, x2] normalized
            })
    
    return results

# Usage
detections = detect_via_serving("test.jpg")
for det in detections:
    print(f"Class {det['class']}: {det['score']:.2f}")
```

### Deployment Options Summary

| Target | Format | Tool | Speed |
|--------|--------|------|-------|
| Server (GPU) | SavedModel | TF Serving | Fastest |
| Server (CPU) | SavedModel | TF Serving | Fast |
| Android | TFLite | TF Lite Runtime | 20-60ms |
| iOS | TFLite / CoreML | TF Lite | 20-50ms |
| Edge (Coral) | TFLite (INT8) | Edge TPU Runtime | 10-30ms |
| Browser | TF.js | TensorFlow.js | 50-200ms |
| Raspberry Pi | TFLite | TF Lite Runtime | 100-500ms |

---

## 11. TF OD API vs YOLO vs Detectron2

### Feature Comparison

| Feature | TF OD API | YOLO (Ultralytics) | Detectron2 |
|---------|-----------|-------------------|-----------|
| **Framework** | TensorFlow | PyTorch | PyTorch |
| **Config system** | .config (protobuf) | .yaml | .yaml (Python) |
| **Model variety** | 40+ architectures | YOLO variants only | 70+ architectures |
| **Ease of use** | ⭐⭐ (complex setup) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Documentation** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Mobile deploy** | ✅ TFLite (best) | ✅ NCNN, CoreML | ❌ Hard |
| **Google Cloud** | ✅ Native | ⚠️ Manual | ⚠️ Manual |
| **TPU training** | ✅ Native | ❌ | ❌ |
| **Real-time** | ⚠️ (depends on model) | ✅ Best | ❌ Too slow |
| **Research** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Community** | Medium (declining) | Massive (growing) | Large (Meta) |

### When to Choose TF OD API

```
✅ You're in the TensorFlow ecosystem
✅ Need TFLite deployment (Android, Coral, Raspberry Pi)
✅ Using Google Cloud / Vertex AI
✅ Need TPU training
✅ Want to try many architectures without code changes
✅ Enterprise TF Serving deployment
✅ Working with existing TF infrastructure

❌ DON'T choose if:
  - You want easiest experience → Use YOLO
  - You need cutting-edge research models → Use Detectron2
  - You want PyTorch → Use YOLO or Detectron2
  - You need real-time → Use YOLO
```

---

## 12. Real-World Use Cases & Companies

### 🏢 Case Study 1: Google Lens

```
Task: Real-time object recognition on mobile
Architecture: SSD MobileNet V2 (optimized for mobile)
Deployment: TFLite on Android/iOS
Scale: 1B+ devices
Pipeline:
  Camera frame → TFLite model → Identify objects → Show info
  
Why TF OD API: 
  - Native TFLite export path
  - MobileNet optimized for Qualcomm/MediaTek
  - Google's own ecosystem
```

### 🏢 Case Study 2: Smart City Traffic

```
Task: Count vehicles, detect violations at intersections
Architecture: EfficientDet-D2 (balanced speed/accuracy)
Deployment: TF Serving on edge servers at intersections
Hardware: NVIDIA Jetson (with TensorRT via ONNX)

Pipeline:
  16 cameras → Edge server → EfficientDet → Count + Classify
  → Upload stats to cloud → Traffic optimization

Scale: 500+ intersections, processing 24/7
```

### 🏢 Case Study 3: Retail Analytics

```
Task: Customer behavior analysis, product interaction
Architecture: CenterNet with Hourglass backbone
Deployment: TF Serving cluster in store backend

Detects:
  - People (tracking across cameras)
  - Hands reaching for products
  - Cart items
  - Queue length

Why TF OD API: Multiple architectures tested easily via config swap
```

### 🏢 Case Study 4: Agricultural Drone

```
Task: Detect crop diseases from aerial images
Architecture: Faster R-CNN ResNet50 (needs accuracy over speed)
Training: Transfer learning from COCO → fine-tune on crop dataset

Setup:
  - 15,000 drone images labeled
  - 5 disease classes + healthy
  - Trained on Google Cloud TPU (fast!)
  - Deployed as TFLite on drone companion device
  
Result: 93% disease detection accuracy
```

---

## 13. Tips, Troubleshooting & Best Practices

### ✅ Best Practices

| Tip | Why |
|-----|-----|
| Always start from pre-trained | Transfer learning gives +10-20% mAP |
| Use SSD for real-time, Faster R-CNN for accuracy | Match architecture to use case |
| Start with small input resolution, increase if needed | 320→640→1024 |
| Monitor TensorBoard during training | Catch issues early |
| Train for at least 20K steps | Models need time to converge |
| Use data augmentation | Prevents overfitting |
| Include background images (5-10%) | Reduces false positives |
| Validate on held-out set | Never evaluate on training data |

### ⚠️ Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `ValueError: num_classes mismatch` | Config classes ≠ label map | Ensure num_classes matches label_map.pbtxt |
| `ResourceExhaustedError (OOM)` | Batch size too large | Reduce batch_size in config |
| `Loss is NaN` | Learning rate too high | Reduce learning_rate_base |
| `No detections output` | Threshold too high, or bad training | Lower score_threshold, check data |
| `Protobuf compilation error` | Missing protoc step | Run `protoc` compilation |
| `Module not found: object_detection` | Not installed properly | Run `pip install .` in research/ |
| `TFRecord parsing error` | Wrong feature format | Check TFRecord schema matches expected |

### Hyperparameter Guidelines

```
┌───────────────────────────────────────────────────────┐
│  DATASET SIZE → RECOMMENDED SETTINGS                  │
├───────────────────────────────────────────────────────┤
│                                                       │
│  Small (< 500 images):                               │
│    • batch_size: 4-8                                  │
│    • num_steps: 10,000-20,000                        │
│    • learning_rate: 0.01-0.04                        │
│    • Heavy augmentation                              │
│    • Fine-tune from COCO checkpoint                  │
│                                                       │
│  Medium (500-5,000 images):                          │
│    • batch_size: 8-16                                │
│    • num_steps: 30,000-50,000                        │
│    • learning_rate: 0.04-0.08                        │
│    • Standard augmentation                           │
│    • Fine-tune from COCO checkpoint                  │
│                                                       │
│  Large (5,000+ images):                              │
│    • batch_size: 16-64                               │
│    • num_steps: 50,000-200,000                       │
│    • learning_rate: 0.08                             │
│    • Standard augmentation                           │
│    • Can train from scratch (but fine-tune better)   │
│                                                       │
└───────────────────────────────────────────────────────┘
```

### Architecture Selection Guide

```
"I need FAST inference (< 30ms):"
  → SSD MobileNet V2 (320×320)
  
"I need MOBILE deployment:"
  → SSD MobileNet V2 FPNLite (320×320)
  → EfficientDet-Lite0

"I need BALANCED speed/accuracy:"
  → EfficientDet D1-D3
  → SSD ResNet50 FPN (640×640)

"I need MAXIMUM accuracy:"
  → Faster R-CNN ResNet101
  → EfficientDet D6-D7

"I need ANCHOR-FREE:"
  → CenterNet (Hourglass or MobileNet)

"I need INSTANCE SEGMENTATION:"
  → Mask R-CNN (in TF OD API)
```

---

## 📋 Chapter Summary

### What You Learned

```
✅ TF OD API = Google's unified detection framework (40+ models)
✅ Config-driven: change .config file = change entire architecture
✅ TFRecord format for data (binary, efficient streaming)
✅ Model Zoo: pre-trained models ready to fine-tune
✅ Pipeline: Data prep → Config → Train → Export → Deploy
✅ Deployment: TF Serving (server), TFLite (mobile), Coral (edge)
✅ Best for: TF ecosystem, Google Cloud, TFLite mobile, TPU training
✅ Not best for: PyTorch users, real-time (use YOLO), cutting-edge research
```

### 🔥 Interview Quick-Fire

| Question | Answer |
|----------|--------|
| "What is TF OD API?" | Google's framework hosting 40+ detection models, config-driven training |
| "How do you switch architectures?" | Change the pipeline.config file — no code changes needed |
| "What format does it need?" | TFRecord (binary) for data, .pbtxt for label map |
| "How to deploy to mobile?" | Export to TFLite format, use TF Lite interpreter |
| "TF OD API vs YOLO?" | TF: more architectures, TFLite deploy. YOLO: easier, faster, PyTorch |
| "What's the Model Zoo?" | Collection of 40+ pre-trained COCO models ready to download |
| "SSD vs Faster R-CNN?" | SSD: fast (one-stage). Faster R-CNN: accurate (two-stage) |
| "How to fine-tune?" | Download model + modify config (num_classes, paths) + run training |

---

### ➡️ Next Chapter: [07 - Detectron2](./07-Detectron2.md)

> Now you know Google's framework. Next, we'll explore Meta's Detectron2 — the research powerhouse built on PyTorch that dominates academic papers and offers the most comprehensive set of state-of-the-art detection models.

---

*[← Back to Index](./00-INDEX.md) | [← Previous: 05 - EfficientDet](./05-EfficientDet.md)*
