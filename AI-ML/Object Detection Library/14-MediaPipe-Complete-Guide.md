# 📘 Chapter 14: MediaPipe — Google's Real-Time ML Framework

> **Goal**: Master Google's MediaPipe — the most popular framework for real-time, on-device object detection, face detection, hand tracking, pose estimation, and more. From understanding its architecture to deploying production apps on mobile, web, and edge devices.

---

## 📑 Table of Contents

1. [What is MediaPipe & Why It Matters](#1-what-is-mediapipe--why-it-matters)
2. [MediaPipe Architecture & Core Concepts](#2-mediapipe-architecture--core-concepts)
3. [MediaPipe Solutions Overview](#3-mediapipe-solutions-overview)
4. [Object Detection with MediaPipe](#4-object-detection-with-mediapipe)
5. [Face Detection — BlazeFace](#5-face-detection--blazeface)
6. [Face Mesh — 468 Landmark Points](#6-face-mesh--468-landmark-points)
7. [Hand Tracking — 21 Landmarks per Hand](#7-hand-tracking--21-landmarks-per-hand)
8. [Pose Estimation — BlazePose (33 Landmarks)](#8-pose-estimation--blazepose-33-landmarks)
9. [Holistic — Full Body + Hands + Face Together](#9-holistic--full-body--hands--face-together)
10. [Image Segmentation & Selfie Segmentation](#10-image-segmentation--selfie-segmentation)
11. [MediaPipe Model Maker — Custom Training](#11-mediapipe-model-maker--custom-training)
12. [MediaPipe on Web (JavaScript)](#12-mediapipe-on-web-javascript)
13. [MediaPipe on Mobile (Android & iOS)](#13-mediapipe-on-mobile-android--ios)
14. [MediaPipe vs Other Libraries (YOLO, OpenCV, Detectron2)](#14-mediapipe-vs-other-libraries-yolo-opencv-detectron2)
15. [Real-World Use Cases & Company Deployments](#15-real-world-use-cases--company-deployments)
16. [Performance Optimization & Deployment](#16-performance-optimization--deployment)
17. [Interview Questions & Quick Reference](#17-interview-questions--quick-reference)

---

## 1. What is MediaPipe & Why It Matters

### 💡 The Core Idea

> **MediaPipe** is Google's open-source framework for building **real-time**, **cross-platform** ML pipelines for processing video, audio, and sensor data. It provides pre-built, production-ready solutions for object detection, face detection, hand tracking, pose estimation, and more — all optimized to run on **mobile, web, and edge devices** without a GPU.

### 🏢 Who Uses MediaPipe

| Company | Use Case |
|---------|----------|
| **Google** | Google Lens, Google Meet backgrounds, ARCore, Google Photos |
| **Snapchat** | AR filters, face effects (inspired by MediaPipe approach) |
| **TikTok** | Real-time face effects, gesture recognition |
| **Zoom** | Virtual background segmentation |
| **Samsung** | Galaxy AR features, camera AI |
| **Meta (Quest)** | Hand tracking for VR headsets |
| **Nike** | Foot scanning app (pose/landmark detection) |
| **Peloton** | Exercise form detection |
| **YouTube** | Content moderation, creator tools |

### 📊 MediaPipe by the Numbers

```
✅ 25,000+ GitHub stars
✅ Runs on: Android, iOS, Web (JS), Python, C++
✅ Pre-built solutions: 15+ ready-to-use ML tasks
✅ Speed: 30+ FPS on mobile phones (no GPU needed!)
✅ Models as small as 2-5 MB (fit on any device)
✅ Powers billions of devices through Google products
✅ Free and open-source (Apache 2.0 license)
```

### The Problem MediaPipe Solves

```
WITHOUT MediaPipe:
  ├── Want face detection on mobile?
  │     → Train model → Convert to TFLite → Handle camera → 
  │       Manage threading → Optimize for each device → Test
  │     → 3-6 months of work
  │
  ├── Want hand tracking in browser?
  │     → Find model → Convert to TFJS → Handle WebGL →
  │       Manage performance → Cross-browser testing
  │     → 2-4 months of work
  │
  └── Want pose estimation on edge?
        → Similar pain...

WITH MediaPipe:
  ├── Face detection on mobile?    → 5 lines of code, works today
  ├── Hand tracking in browser?    → 5 lines of JavaScript, 30 FPS
  └── Pose estimation on edge?     → Pre-optimized, production-ready
  
  Total time: Hours, not months!
```

### Paper & Background Info

```
Title: "MediaPipe: A Framework for Building Perception Pipelines"
Authors: Google Research (Camillo Lugaresi et al.)
Published: 2019 (CVPR Workshop on CV for AR/VR)
Key Papers:
  - BlazeFace (2019): Sub-millisecond face detection
  - BlazePose (2020): Real-time body pose on mobile
  - BlazePalm + Hand Landmarks (2020): 21-point hand tracking
  - MediaPipe Holistic (2020): Unified face+hands+pose
Repository: https://github.com/google-ai-edge/mediapipe
```

### MediaPipe Evolution Timeline

```
2019: MediaPipe open-sourced (graph-based pipeline framework)
2020: MediaPipe Solutions released (pre-built tasks)
2021: Web/JS support added
2022: Model Maker introduced (custom training)
2023: MediaPipe Tasks API (unified, simplified API)
2024: MediaPipe LLM Inference, GenAI tasks added
2025: Expanded model zoo, improved accuracy
```

---

## 2. MediaPipe Architecture & Core Concepts

### 💡 How MediaPipe Works Internally

> MediaPipe is built on a **graph-based architecture** where data flows through a pipeline of processing nodes (called Calculators). This allows complex ML pipelines to be built by connecting simple, reusable components.

### The Graph-Based Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    MediaPipe Graph Pipeline                       │
│                                                                   │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌────┐│
│  │  Camera  │───▶│  Preprocess  │───▶│  ML Model    │───▶│Post││
│  │  Input   │    │  (Resize,    │    │  (TFLite/    │    │Proc││
│  │          │    │   Normalize) │    │   ONNX)      │    │    ││
│  └──────────┘    └──────────────┘    └──────────────┘    └────┘│
│       │                                                     │    │
│       │              ┌──────────────┐                      │    │
│       └─────────────▶│  Tracking    │◀─────────────────────┘    │
│                      │  (Smoothing, │                            │
│                      │   Temporal)  │                            │
│                      └──────────────┘                            │
│                             │                                    │
│                             ▼                                    │
│                      ┌──────────────┐                            │
│                      │   Output     │                            │
│                      │  (Landmarks, │                            │
│                      │   Boxes, etc)│                            │
│                      └──────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

### Key Concepts

| Concept | Description | Analogy |
|---------|-------------|---------|
| **Graph** | The complete pipeline of operations | Assembly line in a factory |
| **Calculator** | A single processing node | One worker station on the line |
| **Packet** | A timestamped unit of data flowing through | A part moving on the conveyor |
| **Stream** | A sequence of packets (like video frames) | The conveyor belt itself |
| **Task** | A high-level pre-built solution | A complete product (e.g., "face detector") |

### The Two Levels of MediaPipe

```
LEVEL 1: MediaPipe Tasks (HIGH-LEVEL) — USE THIS!
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  → Pre-built solutions for common ML tasks
  → 5 lines of code to detect objects/faces/hands
  → Available for Python, Android, iOS, Web
  → Models included, no ML knowledge needed
  
  Example:
    detector = ObjectDetector.create_from_options(options)
    result = detector.detect(image)

LEVEL 2: MediaPipe Framework (LOW-LEVEL) — For Custom Pipelines
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  → Build custom graph-based pipelines
  → Define your own Calculators
  → Wire together arbitrary ML models
  → Used internally at Google for complex systems
  
  Example:
    Define .pbtxt graph → Add calculators → Run pipeline
```

### MediaPipe Tasks API Architecture (The Modern Way)

```
┌─────────────────────────────────────────────────────┐
│              MediaPipe Tasks API                      │
├─────────────────────────────────────────────────────┤
│                                                      │
│  ┌─────────────┐  ┌───────────────┐  ┌──────────┐ │
│  │  Vision     │  │  Text         │  │  Audio   │ │
│  │  Tasks      │  │  Tasks        │  │  Tasks   │ │
│  ├─────────────┤  ├───────────────┤  ├──────────┤ │
│  │ • Object    │  │ • Text        │  │ • Audio  │ │
│  │   Detection │  │   Classifier  │  │   Class. │ │
│  │ • Face Det. │  │ • Text        │  │ • Speech │ │
│  │ • Hand Land.│  │   Embedding   │  │          │ │
│  │ • Pose Land.│  │ • Language    │  │          │ │
│  │ • Image Seg.│  │   Detector    │  │          │ │
│  │ • Image     │  │               │  │          │ │
│  │   Classify  │  │               │  │          │ │
│  └─────────────┘  └───────────────┘  └──────────┘ │
│                                                      │
├─────────────────────────────────────────────────────┤
│  Runtime: TFLite + GPU Delegate + NNAPI + CoreML     │
└─────────────────────────────────────────────────────┘
```

### Running Modes (How to Process Input)

```python
# MediaPipe supports 3 running modes:

# 1. IMAGE MODE — Process a single image
#    Use case: Photo processing, batch analysis
result = detector.detect(image)

# 2. VIDEO MODE — Process video frames sequentially
#    Use case: Video file analysis, offline processing
result = detector.detect_for_video(frame, timestamp_ms)

# 3. LIVE_STREAM MODE — Process camera feed in real-time
#    Use case: Real-time apps, AR, live analysis
#    (Asynchronous — provides results via callback)
detector.detect_async(frame, timestamp_ms)
```

---

## 3. MediaPipe Solutions Overview

### Complete List of Available Solutions (2025)

```
┌─────────────────────────────────────────────────────────────────┐
│                 MediaPipe Vision Tasks                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  DETECTION & RECOGNITION:                                        │
│  ├── Object Detection        → Detect 80+ object categories     │
│  ├── Face Detection          → Detect faces (BlazeFace)         │
│  ├── Face Landmarker         → 478 face landmarks (mesh)        │
│  ├── Hand Landmarker         → 21 hand landmarks per hand       │
│  ├── Pose Landmarker         → 33 body landmarks (BlazePose)    │
│  ├── Holistic Landmarker     → Face + Hands + Pose combined     │
│  └── Gesture Recognition     → Recognize hand gestures          │
│                                                                   │
│  CLASSIFICATION & SEGMENTATION:                                  │
│  ├── Image Classification    → Classify image content           │
│  ├── Image Segmentation      → Pixel-level segmentation         │
│  ├── Interactive Segmentation → Click-to-segment                │
│  └── Image Embedding         → Feature vectors for similarity   │
│                                                                   │
│  GENERATION & OTHER:                                             │
│  ├── Face Stylizer           → Style transfer for faces         │
│  ├── LLM Inference           → Run LLMs on-device              │
│  └── Object Segmentation     → Segment specific objects         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Model Sizes — Why MediaPipe is Special

```
┌────────────────────┬────────────┬─────────────┬────────────────┐
│ Task               │ Model Size │ Latency     │ Platform       │
├────────────────────┼────────────┼─────────────┼────────────────┤
│ Object Detection   │ 4-25 MB    │ 10-50ms     │ All            │
│ Face Detection     │ 0.2 MB     │ <5ms        │ All            │
│ Face Mesh          │ 2.6 MB     │ 5-15ms      │ All            │
│ Hand Landmarks     │ 5.7 MB     │ 8-20ms      │ All            │
│ Pose Landmarks     │ 3-6 MB     │ 8-25ms      │ All            │
│ Image Segmentation │ 2-10 MB    │ 10-40ms     │ All            │
│ Gesture Recognizer │ 5.7 MB     │ 10-25ms     │ All            │
├────────────────────┼────────────┼─────────────┼────────────────┤
│ Compare: YOLOv8n   │ 6.2 MB     │ 15-30ms     │ GPU preferred  │
│ Compare: YOLOv8s   │ 22.5 MB    │ 30-80ms     │ GPU preferred  │
│ Compare: DETR      │ 160+ MB    │ 60-100ms    │ GPU required   │
└────────────────────┴────────────┴─────────────┴────────────────┘

💡 Key Insight: MediaPipe models are TINY and run on CPU!
   This is why they dominate mobile and web deployment.
```

---

## 4. Object Detection with MediaPipe

### 💡 What It Does

> MediaPipe Object Detector identifies and localizes multiple objects in images or video, returning bounding boxes with class labels and confidence scores. It uses models based on **MobileNet + SSD** architecture, optimized for real-time performance on mobile/edge devices.

### Available Models

```
┌──────────────────────────────────────────────────────────────────┐
│               MediaPipe Object Detection Models                   │
├────────────────────┬────────┬────────┬────────┬─────────────────┤
│ Model              │ Size   │ mAP    │ Latency│ Best For        │
├────────────────────┼────────┼────────┼────────┼─────────────────┤
│ EfficientDet-Lite0 │ 4.4 MB │ 25.7%  │ 12ms   │ Ultra-fast edge │
│ EfficientDet-Lite1 │ 5.8 MB │ 30.3%  │ 18ms   │ Mobile apps     │
│ EfficientDet-Lite2 │ 7.2 MB │ 33.8%  │ 25ms   │ Balanced        │
│ SSD MobileNetV2    │ 6.6 MB │ 22.0%  │ 10ms   │ Fastest         │
│ Custom (trained)   │ Varies │ Custom │ Varies │ Your use case   │
└────────────────────┴────────┴────────┴────────┴─────────────────┘

Note: mAP measured on COCO 2017 validation set
Latency: Measured on Pixel 6 CPU
```

### Installation

```bash
# Install MediaPipe for Python
pip install mediapipe

# Or with specific version
pip install mediapipe==0.10.14

# For model downloads (automatic with Tasks API)
# Models are auto-downloaded on first use, or manually:
# https://developers.google.com/mediapipe/solutions/vision/object_detector#models
```

### Code Example: Object Detection on Image

```python
import mediapipe as mp
from mediapipe.tasks import python
from mediapipe.tasks.python import vision
import cv2
import numpy as np

# ─── Step 1: Download model (or provide path) ───
# Model: EfficientDet-Lite0 (auto-downloaded or manual download)
model_path = "efficientdet_lite0.tflite"

# ─── Step 2: Configure detector options ───
BaseOptions = mp.tasks.BaseOptions
ObjectDetector = mp.tasks.vision.ObjectDetector
ObjectDetectorOptions = mp.tasks.vision.ObjectDetectorOptions
VisionRunningMode = mp.tasks.vision.RunningMode

options = ObjectDetectorOptions(
    base_options=BaseOptions(model_asset_path=model_path),
    running_mode=VisionRunningMode.IMAGE,
    max_results=5,                    # Max detections to return
    score_threshold=0.5,              # Minimum confidence
    category_allowlist=[],            # Empty = all categories
    category_denylist=[]              # Categories to exclude
)

# ─── Step 3: Create detector and detect ───
with ObjectDetector.create_from_options(options) as detector:
    # Load image
    mp_image = mp.Image.create_from_file("street.jpg")
    
    # Detect objects
    detection_result = detector.detect(mp_image)
    
    # ─── Step 4: Process results ───
    for detection in detection_result.detections:
        # Get bounding box
        bbox = detection.bounding_box
        x = bbox.origin_x
        y = bbox.origin_y
        w = bbox.width
        h = bbox.height
        
        # Get category and confidence
        category = detection.categories[0]
        label = category.category_name
        score = category.score
        
        print(f"Detected: {label} ({score:.2f}) at [{x},{y},{w},{h}]")

# ─── OUTPUT ───
# Detected: car (0.92) at [120, 340, 200, 150]
# Detected: person (0.88) at [50, 100, 80, 250]
# Detected: dog (0.75) at [400, 350, 100, 90]
# Detected: bicycle (0.71) at [280, 300, 130, 120]
# Detected: traffic light (0.65) at [350, 20, 30, 80]
```

### Code Example: Object Detection on Video (Real-Time)

```python
import mediapipe as mp
from mediapipe.tasks import python
from mediapipe.tasks.python import vision
import cv2
import time

# ─── Configuration ───
model_path = "efficientdet_lite0.tflite"

BaseOptions = mp.tasks.BaseOptions
ObjectDetector = mp.tasks.vision.ObjectDetector
ObjectDetectorOptions = mp.tasks.vision.ObjectDetectorOptions
VisionRunningMode = mp.tasks.vision.RunningMode

# ─── For LIVE_STREAM mode, we need a callback ───
detection_results = []

def result_callback(result, output_image, timestamp_ms):
    """Called when detection is complete for a frame."""
    detection_results.append(result)

# ─── Configure for live stream ───
options = ObjectDetectorOptions(
    base_options=BaseOptions(model_asset_path=model_path),
    running_mode=VisionRunningMode.LIVE_STREAM,
    max_results=5,
    score_threshold=0.5,
    result_callback=result_callback
)

# ─── Real-time detection from webcam ───
with ObjectDetector.create_from_options(options) as detector:
    cap = cv2.VideoCapture(0)
    
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        
        # Convert BGR to RGB (MediaPipe uses RGB)
        rgb_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        mp_image = mp.Image(image_format=mp.ImageFormat.SRGB, data=rgb_frame)
        
        # Get timestamp
        timestamp_ms = int(time.time() * 1000)
        
        # Detect asynchronously
        detector.detect_async(mp_image, timestamp_ms)
        
        # Draw results from last detection
        if detection_results:
            latest_result = detection_results[-1]
            for detection in latest_result.detections:
                bbox = detection.bounding_box
                category = detection.categories[0]
                
                # Draw bounding box
                cv2.rectangle(
                    frame,
                    (bbox.origin_x, bbox.origin_y),
                    (bbox.origin_x + bbox.width, bbox.origin_y + bbox.height),
                    (0, 255, 0), 2
                )
                
                # Draw label
                label = f"{category.category_name}: {category.score:.2f}"
                cv2.putText(
                    frame, label,
                    (bbox.origin_x, bbox.origin_y - 10),
                    cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 255, 0), 2
                )
        
        # Show frame
        cv2.imshow("MediaPipe Object Detection", frame)
        
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    
    cap.release()
    cv2.destroyAllWindows()

# ─── OUTPUT (Console + Visual) ───
# Opens webcam window showing real-time bounding boxes
# FPS: 25-30 on laptop CPU (no GPU needed!)
# Detects: persons, cars, bottles, phones, etc.
```

### Code Example: Object Detection on Video File

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
import cv2

model_path = "efficientdet_lite0.tflite"

# ─── Configure for VIDEO mode ───
options = vision.ObjectDetectorOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.VIDEO,
    max_results=10,
    score_threshold=0.4
)

with vision.ObjectDetector.create_from_options(options) as detector:
    cap = cv2.VideoCapture("traffic_video.mp4")
    fps = cap.get(cv2.CAP_PROP_FPS)
    frame_count = 0
    
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        
        # Calculate timestamp
        timestamp_ms = int(frame_count * 1000 / fps)
        
        # Convert to MediaPipe Image
        rgb_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        mp_image = mp.Image(image_format=mp.ImageFormat.SRGB, data=rgb_frame)
        
        # Detect for video (synchronous, sequential frames)
        result = detector.detect_for_video(mp_image, timestamp_ms)
        
        # Process detections
        print(f"Frame {frame_count}: {len(result.detections)} objects detected")
        for det in result.detections:
            cat = det.categories[0]
            print(f"  → {cat.category_name}: {cat.score:.2f}")
        
        frame_count += 1
    
    cap.release()

# ─── OUTPUT ───
# Frame 0: 3 objects detected
#   → car: 0.91
#   → car: 0.87
#   → person: 0.72
# Frame 1: 4 objects detected
#   → car: 0.93
#   → car: 0.85
#   → person: 0.74
#   → bicycle: 0.61
# ...
```

### Use Cases for MediaPipe Object Detection

| Use Case | Example | Why MediaPipe? |
|----------|---------|----------------|
| **Retail shelf monitoring** | Count products on shelf | Runs on cheap edge cameras |
| **Smart doorbell** | Detect people, packages | Real-time on low-power devices |
| **Wildlife camera trap** | Detect animals | Battery-powered, no internet |
| **Warehouse automation** | Count boxes, detect items | Works on Raspberry Pi |
| **Assistive technology** | Describe surroundings for visually impaired | Real-time on mobile phone |
| **Parking management** | Detect cars in spaces | Low-cost IP camera processing |

---

## 5. Face Detection — BlazeFace

### 💡 What It Does

> **BlazeFace** is MediaPipe's ultra-fast face detector. It can detect multiple faces in an image and return bounding boxes + 6 facial keypoints (eyes, ears, nose, mouth) in **sub-millisecond** time on mobile devices. This is what powers Google Duo's face tracking and Android's face unlock.

### Architecture: How BlazeFace Works

```
┌──────────────────────────────────────────────────────────────────┐
│                    BlazeFace Architecture                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Input Image (128×128 for short-range, 256×256 for full-range)   │
│       │                                                            │
│       ▼                                                            │
│  ┌─────────────────────────────────────────┐                      │
│  │  Feature Extraction                      │                      │
│  │  • MobileNet-like backbone               │                      │
│  │  • Depthwise Separable Convolutions      │                      │
│  │  • BlazeBlock (custom efficient block)   │                      │
│  └─────────────────────────────────────────┘                      │
│       │                                                            │
│       ▼                                                            │
│  ┌─────────────────────────────────────────┐                      │
│  │  SSD-style Detection Head                │                      │
│  │  • Anchor-based (predefined boxes)       │                      │
│  │  • Multi-scale feature maps              │                      │
│  │  • Predict: box + confidence + keypoints │                      │
│  └─────────────────────────────────────────┘                      │
│       │                                                            │
│       ▼                                                            │
│  ┌─────────────────────────────────────────┐                      │
│  │  Post-processing                         │                      │
│  │  • Non-Maximum Suppression (NMS)         │                      │
│  │  • Weighted blending of overlapping dets │                      │
│  │  • Keypoint refinement                   │                      │
│  └─────────────────────────────────────────┘                      │
│       │                                                            │
│       ▼                                                            │
│  OUTPUT: Bounding box + 6 keypoints per face                      │
│  (right eye, left eye, nose tip, mouth center, right ear, left ear)│
│                                                                    │
└──────────────────────────────────────────────────────────────────┘

Key innovation: BlazeBlock
  → Modified residual block with depthwise convolutions
  → 5× faster than standard residual blocks
  → Achieves same accuracy with fewer FLOPs
```

### Two Model Variants

```
SHORT-RANGE MODEL (Front camera):
  • Input: 128×128
  • Distance: Up to 2 meters
  • Model size: 98 KB (!)
  • Latency: 0.6ms on Pixel 3
  • Use: Video calls, selfies, face unlock

FULL-RANGE MODEL (Back camera):
  • Input: 256×256
  • Distance: Up to 5 meters
  • Model size: 196 KB
  • Latency: 1.2ms on Pixel 3
  • Use: Group photos, surveillance, general use
```

### Code Example: Face Detection

```python
import mediapipe as mp
from mediapipe.tasks import python
from mediapipe.tasks.python import vision
import cv2
import numpy as np

# ─── Configure Face Detector ───
model_path = "blaze_face_short_range.tflite"

options = vision.FaceDetectorOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.IMAGE,
    min_detection_confidence=0.5,
    min_suppression_threshold=0.3    # NMS threshold
)

# ─── Detect faces in image ───
with vision.FaceDetector.create_from_options(options) as detector:
    image = mp.Image.create_from_file("group_photo.jpg")
    result = detector.detect(image)
    
    print(f"Detected {len(result.detections)} faces")
    
    for i, detection in enumerate(result.detections):
        # Bounding box
        bbox = detection.bounding_box
        print(f"\nFace {i+1}:")
        print(f"  Box: x={bbox.origin_x}, y={bbox.origin_y}, "
              f"w={bbox.width}, h={bbox.height}")
        print(f"  Confidence: {detection.categories[0].score:.3f}")
        
        # 6 Keypoints
        for j, keypoint in enumerate(detection.keypoints):
            kp_names = ["Right Eye", "Left Eye", "Nose Tip", 
                       "Mouth Center", "Right Ear", "Left Ear"]
            print(f"  {kp_names[j]}: ({keypoint.x:.3f}, {keypoint.y:.3f})")

# ─── OUTPUT ───
# Detected 3 faces
#
# Face 1:
#   Box: x=120, y=80, w=150, h=180
#   Confidence: 0.987
#   Right Eye: (0.352, 0.284)
#   Left Eye: (0.428, 0.279)
#   Nose Tip: (0.389, 0.368)
#   Mouth Center: (0.391, 0.432)
#   Right Ear: (0.298, 0.321)
#   Left Ear: (0.472, 0.318)
#
# Face 2:
#   Box: x=380, y=95, w=140, h=165
#   Confidence: 0.964
#   ...
#
# Face 3:
#   Box: x=600, y=110, w=130, h=155
#   Confidence: 0.891
#   ...
```

### Code Example: Real-Time Face Detection with Visualization

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
import cv2
import time

# ─── Real-time face detection from webcam ───
model_path = "blaze_face_short_range.tflite"

latest_result = None

def callback(result, image, timestamp_ms):
    global latest_result
    latest_result = result

options = vision.FaceDetectorOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.LIVE_STREAM,
    min_detection_confidence=0.5,
    result_callback=callback
)

with vision.FaceDetector.create_from_options(options) as detector:
    cap = cv2.VideoCapture(0)
    prev_time = time.time()
    
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        
        # Convert and detect
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        mp_image = mp.Image(image_format=mp.ImageFormat.SRGB, data=rgb)
        detector.detect_async(mp_image, int(time.time() * 1000))
        
        # Draw detections
        if latest_result:
            h, w = frame.shape[:2]
            for detection in latest_result.detections:
                bbox = detection.bounding_box
                
                # Draw bounding box
                cv2.rectangle(
                    frame,
                    (bbox.origin_x, bbox.origin_y),
                    (bbox.origin_x + bbox.width, bbox.origin_y + bbox.height),
                    (0, 255, 0), 2
                )
                
                # Draw keypoints
                for kp in detection.keypoints:
                    cx = int(kp.x * w)
                    cy = int(kp.y * h)
                    cv2.circle(frame, (cx, cy), 4, (255, 0, 0), -1)
                
                # Draw confidence
                score = detection.categories[0].score
                cv2.putText(frame, f"{score:.2f}",
                          (bbox.origin_x, bbox.origin_y - 10),
                          cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2)
        
        # Calculate FPS
        curr_time = time.time()
        fps = 1 / (curr_time - prev_time)
        prev_time = curr_time
        cv2.putText(frame, f"FPS: {fps:.1f}", (10, 30),
                   cv2.FONT_HERSHEY_SIMPLEX, 1, (255, 255, 0), 2)
        
        cv2.imshow("Face Detection", frame)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    
    cap.release()
    cv2.destroyAllWindows()

# ─── OUTPUT ───
# Opens webcam showing:
# • Green bounding boxes around each face
# • Blue dots on keypoints (eyes, nose, mouth, ears)
# • Confidence score above each box
# • FPS counter: typically 60-120+ FPS on modern laptop
```

### Use Cases for Face Detection

| Use Case | How It Works | Real Example |
|----------|-------------|--------------|
| **Face unlock** | Detect face → Verify with face recognition | Android phones |
| **Video call effects** | Detect face → Apply filters/blur background | Google Meet, Zoom |
| **Photo tagging** | Detect faces → Group by similarity | Google Photos |
| **Attendance system** | Detect faces → Match with database | Office/school |
| **Driver drowsiness** | Detect face → Monitor eye keypoints | Fleet management |
| **Age/emotion estimation** | Detect face → Crop → Classify | Retail analytics |
| **AR filters** | Detect face → Track keypoints → Overlay graphics | Snapchat, Instagram |

---

## 6. Face Mesh — 468 Landmark Points

### 💡 What It Does

> **MediaPipe Face Mesh** (now called Face Landmarker) estimates **478 3D facial landmarks** in real-time. It creates a dense mesh of your face that tracks expressions, head rotation, eye movement, and lip movement — all running at 30+ FPS on mobile devices. This powers AR makeup, virtual try-on, and face filters.

### The 478 Landmarks Explained

```
┌────────────────────────────────────────────────────────────────┐
│              478 Face Landmarks - Key Regions                    │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  FACE OVAL (36 points):          Jawline and face boundary      │
│  LEFT EYE (16 points):           Eye contour                    │
│  RIGHT EYE (16 points):          Eye contour                    │
│  LEFT EYEBROW (10 points):       Eyebrow shape                  │
│  RIGHT EYEBROW (10 points):      Eyebrow shape                  │
│  LIPS (40 points):               Upper + lower lip contours     │
│  NOSE (24 points):               Nose bridge + nostrils         │
│  LEFT IRIS (5 points):           Iris center + boundary         │
│  RIGHT IRIS (5 points):          Iris center + boundary         │
│  FOREHEAD, CHEEKS, CHIN:         Remaining points               │
│                                                                  │
│  Total: 478 3D landmarks (x, y, z coordinates)                  │
│  + Blendshapes: 52 expression coefficients                      │
│  + Face transform: 3D rotation + translation matrix             │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

### Blendshapes — Expression Tracking

```
MediaPipe provides 52 blendshape coefficients (0.0 to 1.0):

EYES:
  • eyeBlinkLeft, eyeBlinkRight          → Eye open/close
  • eyeLookUpLeft, eyeLookDownRight       → Gaze direction
  • eyeSquintLeft, eyeSquintRight         → Squinting
  • eyeWideLeft, eyeWideRight             → Surprise

MOUTH:
  • mouthSmileLeft, mouthSmileRight       → Smiling
  • mouthOpen, jawOpen                     → Mouth/jaw opening
  • mouthPucker, mouthFunnel              → Lip shapes
  • mouthLeft, mouthRight                  → Mouth shift

BROWS:
  • browDownLeft, browDownRight            → Frowning
  • browInnerUp, browOuterUpLeft          → Raising eyebrows

OTHER:
  • cheekPuff, cheekSquintLeft            → Cheek expressions
  • noseSneerLeft, noseSneerRight         → Nose wrinkling

Use Case: Drive 3D avatar expressions (Apple Memoji style)!
```

### Code Example: Face Mesh with Landmark Visualization

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
from mediapipe import solutions
import cv2
import numpy as np

# ─── Configure Face Landmarker ───
model_path = "face_landmarker_v2_with_blendshapes.task"

options = vision.FaceLandmarkerOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.IMAGE,
    num_faces=2,                          # Max faces to track
    min_face_detection_confidence=0.5,
    min_face_presence_confidence=0.5,
    min_tracking_confidence=0.5,
    output_face_blendshapes=True,         # Get expression data
    output_facial_transformation_matrixes=True  # Get 3D pose
)

# ─── Process image ───
with vision.FaceLandmarker.create_from_options(options) as landmarker:
    image = mp.Image.create_from_file("face_photo.jpg")
    result = landmarker.detect(image)
    
    print(f"Detected {len(result.face_landmarks)} face(s)")
    
    for face_idx, landmarks in enumerate(result.face_landmarks):
        print(f"\nFace {face_idx + 1}: {len(landmarks)} landmarks")
        
        # Print some key landmarks
        nose_tip = landmarks[1]  # Index 1 = nose tip
        left_eye = landmarks[33]  # Index 33 = left eye center
        right_eye = landmarks[263]  # Index 263 = right eye center
        
        print(f"  Nose tip: ({nose_tip.x:.3f}, {nose_tip.y:.3f}, {nose_tip.z:.3f})")
        print(f"  Left eye: ({left_eye.x:.3f}, {left_eye.y:.3f}, {left_eye.z:.3f})")
        print(f"  Right eye: ({right_eye.x:.3f}, {right_eye.y:.3f}, {right_eye.z:.3f})")
    
    # Blendshapes (expressions)
    if result.face_blendshapes:
        print("\n--- Blendshapes (Expressions) ---")
        for blendshape in result.face_blendshapes[0][:10]:
            print(f"  {blendshape.category_name}: {blendshape.score:.3f}")
    
    # Transformation matrix (head pose)
    if result.facial_transformation_matrixes:
        matrix = result.facial_transformation_matrixes[0]
        print(f"\n--- Head Pose Matrix (4x4) ---")
        print(f"  Shape: {np.array(matrix).shape}")

# ─── OUTPUT ───
# Detected 1 face(s)
#
# Face 1: 478 landmarks
#   Nose tip: (0.498, 0.532, -0.042)
#   Left eye: (0.572, 0.387, -0.028)
#   Right eye: (0.412, 0.391, -0.031)
#
# --- Blendshapes (Expressions) ---
#   _neutral: 0.012
#   browDownLeft: 0.034
#   browDownRight: 0.028
#   browInnerUp: 0.156
#   browOuterUpLeft: 0.089
#   browOuterUpRight: 0.072
#   cheekPuff: 0.008
#   cheekSquintLeft: 0.245
#   cheekSquintRight: 0.231
#   eyeBlinkLeft: 0.021
#
# --- Head Pose Matrix (4x4) ---
#   Shape: (4, 4)
```

### Code Example: Real-Time Face Mesh with Drawing

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
from mediapipe.framework.formats import landmark_pb2
import cv2
import numpy as np
import time

# ─── Helper: Draw face mesh on frame ───
def draw_face_mesh(frame, face_landmarks):
    """Draw all face mesh connections on the frame."""
    h, w = frame.shape[:2]
    
    # MediaPipe face mesh connections
    mp_face_mesh = mp.solutions.face_mesh
    
    # Draw each landmark as a small dot
    for landmark in face_landmarks:
        x = int(landmark.x * w)
        y = int(landmark.y * h)
        cv2.circle(frame, (x, y), 1, (0, 255, 0), -1)
    
    # Draw specific connections for key features
    # Eyes
    LEFT_EYE = [33, 160, 158, 133, 153, 144]
    RIGHT_EYE = [362, 385, 387, 263, 373, 380]
    LIPS = [61, 146, 91, 181, 84, 17, 314, 405, 321, 375, 291, 
            308, 324, 318, 402, 317, 14, 87, 178, 88, 95, 78]
    
    for indices in [LEFT_EYE, RIGHT_EYE, LIPS]:
        points = []
        for idx in indices:
            lm = face_landmarks[idx]
            points.append((int(lm.x * w), int(lm.y * h)))
        points = np.array(points, dtype=np.int32)
        cv2.polylines(frame, [points], True, (0, 255, 255), 1)
    
    return frame

# ─── Real-time face mesh ───
model_path = "face_landmarker_v2_with_blendshapes.task"
latest_result = None

def callback(result, image, timestamp_ms):
    global latest_result
    latest_result = result

options = vision.FaceLandmarkerOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.LIVE_STREAM,
    num_faces=1,
    output_face_blendshapes=True,
    result_callback=callback
)

with vision.FaceLandmarker.create_from_options(options) as landmarker:
    cap = cv2.VideoCapture(0)
    
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        mp_image = mp.Image(image_format=mp.ImageFormat.SRGB, data=rgb)
        landmarker.detect_async(mp_image, int(time.time() * 1000))
        
        if latest_result and latest_result.face_landmarks:
            frame = draw_face_mesh(frame, latest_result.face_landmarks[0])
            
            # Show expression info
            if latest_result.face_blendshapes:
                blendshapes = latest_result.face_blendshapes[0]
                # Find dominant expression
                top_expr = max(blendshapes[1:], key=lambda b: b.score)
                cv2.putText(frame, f"Expression: {top_expr.category_name} ({top_expr.score:.2f})",
                          (10, 60), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (255, 255, 0), 2)
        
        cv2.imshow("Face Mesh", frame)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    
    cap.release()
    cv2.destroyAllWindows()

# ─── OUTPUT ───
# Opens webcam showing:
# • 478 green dots forming face mesh
# • Yellow lines around eyes and lips
# • Expression label: "mouthSmileLeft (0.82)" etc.
# • Real-time at 30+ FPS
```

### Use Cases for Face Mesh

| Use Case | How It Works | Real Example |
|----------|-------------|--------------|
| **AR makeup try-on** | Map landmarks → Overlay lipstick/eyeshadow | L'Oréal, Sephora apps |
| **Virtual glasses try-on** | Use eye/ear landmarks → Position 3D glasses | Warby Parker, Lenskart |
| **Avatar animation** | Blendshapes → Drive 3D character face | Apple Memoji, VTubers |
| **Emotion detection** | Blendshapes → Classify emotion | Customer service analytics |
| **Head pose estimation** | Transform matrix → Get rotation angles | Driver monitoring |
| **Sign language** | Face expressions + hand landmarks → Translate | Accessibility tools |
| **Lip reading** | Lip landmarks → Sequence model → Text | Hearing assistance |
| **Anti-spoofing** | Depth (z-coordinate) → Detect flat images | Face authentication |

---

## 7. Hand Tracking — 21 Landmarks per Hand

### 💡 What It Does

> **MediaPipe Hand Landmarker** detects hands in images/video and returns **21 3D landmarks** per hand, along with handedness (left/right) classification. It uses a two-stage pipeline: palm detection → hand landmark estimation. This powers gesture control, sign language recognition, and AR hand effects.

### The 21 Hand Landmarks

```
┌────────────────────────────────────────────────────────────────┐
│                    21 Hand Landmarks                             │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  FINGER TIP INDICES:                                            │
│    4  = Thumb tip                                                │
│    8  = Index finger tip                                         │
│    12 = Middle finger tip                                        │
│    16 = Ring finger tip                                          │
│    20 = Pinky finger tip                                         │
│                                                                  │
│  LANDMARK MAP:                                                   │
│                                                                  │
│         8                                                        │
│         |    12                                                   │
│     7   |   |    16                                              │
│     |   |   |   |    20                                          │
│     6   |   11  |   |                                            │
│     |   |   |   15  |                                            │
│  4  5   |   10  |   19                                           │
│  |  |   7   |   14  |                                            │
│  3  |   |   9   |   18                                           │
│  |  |   6   |   13  |                                            │
│  2  |   |   |   |   17                                           │
│  |  5   |   |   |   |                                            │
│  1  |   |   |   |   |                                            │
│  |  |   |   |   |   |                                            │
│  └──┴───┴───┴───┴───┘                                           │
│         0 (WRIST)                                                │
│                                                                  │
│  0  = Wrist                                                      │
│  1-4   = Thumb (CMC, MCP, IP, TIP)                              │
│  5-8   = Index finger (MCP, PIP, DIP, TIP)                      │
│  9-12  = Middle finger (MCP, PIP, DIP, TIP)                     │
│  13-16 = Ring finger (MCP, PIP, DIP, TIP)                       │
│  17-20 = Pinky (MCP, PIP, DIP, TIP)                             │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

### Two-Stage Pipeline

```
Stage 1: PALM DETECTION (BlazePalm)
  ├── Input: Full image (256×256)
  ├── Output: Palm bounding box + orientation
  ├── Model: SSD-based, ~1.3 MB
  ├── Why palm? → Rigid object, easier to detect than hand shape
  └── Runs ONLY when hand not already tracked

Stage 2: HAND LANDMARK MODEL
  ├── Input: Cropped hand region (224×224)
  ├── Output: 21 3D landmarks (x, y, z) + handedness
  ├── Model: ~5.7 MB
  ├── z = depth relative to wrist (closer to camera = negative)
  └── Runs every frame on tracked region

TRACKING OPTIMIZATION:
  Frame 1: Detect palm → Predict landmarks → Get bounding box
  Frame 2-N: Use previous bounding box → Skip palm detection!
  Result: 3-4× faster because palm detection is expensive!
  Re-detect only when tracking confidence drops below threshold.
```

### Code Example: Hand Landmark Detection

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
import cv2
import numpy as np

# ─── Configure Hand Landmarker ───
model_path = "hand_landmarker.task"

options = vision.HandLandmarkerOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.IMAGE,
    num_hands=2,                          # Detect up to 2 hands
    min_hand_detection_confidence=0.5,
    min_hand_presence_confidence=0.5,
    min_tracking_confidence=0.5
)

# ─── Detect hand landmarks ───
with vision.HandLandmarker.create_from_options(options) as landmarker:
    image = mp.Image.create_from_file("hand_gesture.jpg")
    result = landmarker.detect(image)
    
    print(f"Detected {len(result.hand_landmarks)} hand(s)")
    
    for hand_idx in range(len(result.hand_landmarks)):
        landmarks = result.hand_landmarks[hand_idx]
        handedness = result.handedness[hand_idx][0]
        
        print(f"\nHand {hand_idx + 1}: {handedness.category_name} "
              f"(confidence: {handedness.score:.3f})")
        
        # Key landmark positions
        wrist = landmarks[0]
        thumb_tip = landmarks[4]
        index_tip = landmarks[8]
        middle_tip = landmarks[12]
        ring_tip = landmarks[16]
        pinky_tip = landmarks[20]
        
        print(f"  Wrist:      ({wrist.x:.3f}, {wrist.y:.3f}, {wrist.z:.3f})")
        print(f"  Thumb tip:  ({thumb_tip.x:.3f}, {thumb_tip.y:.3f}, {thumb_tip.z:.3f})")
        print(f"  Index tip:  ({index_tip.x:.3f}, {index_tip.y:.3f}, {index_tip.z:.3f})")
        print(f"  Middle tip: ({middle_tip.x:.3f}, {middle_tip.y:.3f}, {middle_tip.z:.3f})")
        print(f"  Ring tip:   ({ring_tip.x:.3f}, {ring_tip.y:.3f}, {ring_tip.z:.3f})")
        print(f"  Pinky tip:  ({pinky_tip.x:.3f}, {pinky_tip.y:.3f}, {pinky_tip.z:.3f})")

# ─── OUTPUT ───
# Detected 2 hand(s)
#
# Hand 1: Right (confidence: 0.987)
#   Wrist:      (0.523, 0.812, 0.000)
#   Thumb tip:  (0.612, 0.634, -0.078)
#   Index tip:  (0.578, 0.489, -0.092)
#   Middle tip: (0.545, 0.467, -0.088)
#   Ring tip:   (0.512, 0.501, -0.075)
#   Pinky tip:  (0.478, 0.556, -0.061)
#
# Hand 2: Left (confidence: 0.943)
#   Wrist:      (0.234, 0.798, 0.000)
#   Thumb tip:  (0.145, 0.621, -0.065)
#   Index tip:  (0.189, 0.478, -0.081)
#   Middle tip: (0.221, 0.456, -0.079)
#   Ring tip:   (0.256, 0.489, -0.068)
#   Pinky tip:  (0.289, 0.534, -0.052)
```

### Code Example: Gesture Recognition (Count Fingers)

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
import cv2
import time
import math

# ─── Finger counting logic ───
def count_fingers(landmarks):
    """
    Count raised fingers using landmark positions.
    A finger is "up" if its tip is above its PIP joint (for fingers)
    or if thumb tip is left of thumb IP (for thumb).
    """
    finger_tips = [4, 8, 12, 16, 20]    # Tip landmarks
    finger_pips = [3, 6, 10, 14, 18]    # PIP/IP landmarks
    
    count = 0
    
    # Thumb: Check if tip is to the LEFT of IP joint (for right hand)
    # Use x-coordinate comparison
    if landmarks[4].x < landmarks[3].x:  # Right hand
        count += 1
    
    # Other fingers: tip above PIP (lower y = higher in image)
    for tip, pip in zip(finger_tips[1:], finger_pips[1:]):
        if landmarks[tip].y < landmarks[pip].y:
            count += 1
    
    return count

def recognize_gesture(landmarks):
    """Recognize common hand gestures."""
    fingers_up = []
    
    # Thumb
    if landmarks[4].x < landmarks[3].x:
        fingers_up.append("thumb")
    
    # Index
    if landmarks[8].y < landmarks[6].y:
        fingers_up.append("index")
    
    # Middle
    if landmarks[12].y < landmarks[10].y:
        fingers_up.append("middle")
    
    # Ring
    if landmarks[16].y < landmarks[14].y:
        fingers_up.append("ring")
    
    # Pinky
    if landmarks[20].y < landmarks[18].y:
        fingers_up.append("pinky")
    
    # Gesture classification
    if len(fingers_up) == 0:
        return "FIST ✊"
    elif fingers_up == ["index"]:
        return "POINTING ☝️"
    elif fingers_up == ["index", "middle"]:
        return "PEACE ✌️"
    elif fingers_up == ["thumb"]:
        return "THUMBS UP 👍"
    elif len(fingers_up) == 5:
        return "OPEN HAND 🖐️"
    elif fingers_up == ["index", "pinky"]:
        return "ROCK 🤘"
    elif fingers_up == ["thumb", "pinky"]:
        return "CALL ME 🤙"
    else:
        return f"FINGERS: {len(fingers_up)}"

# ─── Real-time gesture recognition ───
model_path = "hand_landmarker.task"
latest_result = None

def callback(result, image, timestamp_ms):
    global latest_result
    latest_result = result

options = vision.HandLandmarkerOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.LIVE_STREAM,
    num_hands=2,
    min_hand_detection_confidence=0.5,
    result_callback=callback
)

with vision.HandLandmarker.create_from_options(options) as landmarker:
    cap = cv2.VideoCapture(0)
    
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        
        # Flip for mirror effect
        frame = cv2.flip(frame, 1)
        h, w = frame.shape[:2]
        
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        mp_image = mp.Image(image_format=mp.ImageFormat.SRGB, data=rgb)
        landmarker.detect_async(mp_image, int(time.time() * 1000))
        
        if latest_result and latest_result.hand_landmarks:
            for hand_idx, landmarks in enumerate(latest_result.hand_landmarks):
                # Draw landmarks
                for lm in landmarks:
                    cx, cy = int(lm.x * w), int(lm.y * h)
                    cv2.circle(frame, (cx, cy), 5, (0, 255, 0), -1)
                
                # Draw connections
                connections = [(0,1),(1,2),(2,3),(3,4),
                             (0,5),(5,6),(6,7),(7,8),
                             (0,9),(9,10),(10,11),(11,12),
                             (0,13),(13,14),(14,15),(15,16),
                             (0,17),(17,18),(18,19),(19,20)]
                for start, end in connections:
                    x1, y1 = int(landmarks[start].x * w), int(landmarks[start].y * h)
                    x2, y2 = int(landmarks[end].x * w), int(landmarks[end].y * h)
                    cv2.line(frame, (x1, y1), (x2, y2), (255, 0, 0), 2)
                
                # Recognize gesture
                gesture = recognize_gesture(landmarks)
                fingers = count_fingers(landmarks)
                
                # Display
                handedness = latest_result.handedness[hand_idx][0].category_name
                cv2.putText(frame, f"{handedness}: {gesture} ({fingers} fingers)",
                          (10, 40 + hand_idx * 40),
                          cv2.FONT_HERSHEY_SIMPLEX, 0.8, (255, 255, 0), 2)
        
        cv2.imshow("Hand Gesture Recognition", frame)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    
    cap.release()
    cv2.destroyAllWindows()

# ─── OUTPUT ───
# Opens webcam showing:
# • Green dots on all 21 hand landmarks
# • Blue lines connecting the joints
# • Text showing: "Right: PEACE ✌️ (2 fingers)"
# • Updates in real-time at 25-30 FPS
# • Try different gestures: fist, thumbs up, peace sign, etc.
```

### Code Example: Distance Between Fingers (Pinch Detection)

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
import cv2
import math
import time

def calculate_distance(lm1, lm2):
    """Calculate Euclidean distance between two landmarks."""
    return math.sqrt(
        (lm1.x - lm2.x)**2 + 
        (lm1.y - lm2.y)**2 + 
        (lm1.z - lm2.z)**2
    )

def detect_pinch(landmarks, threshold=0.05):
    """Detect if thumb and index finger are pinching."""
    thumb_tip = landmarks[4]
    index_tip = landmarks[8]
    distance = calculate_distance(thumb_tip, index_tip)
    return distance < threshold, distance

# ─── In real-time loop: ───
# if is_pinching:
#     # Trigger action (e.g., select object, zoom in, click)
#     cv2.putText(frame, "PINCH DETECTED!", (10, 80),
#               cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 0, 255), 3)

# ─── OUTPUT ───
# When thumb and index finger come close together:
#   "PINCH DETECTED!" appears on screen
# Use for: gesture-based UI control, virtual object manipulation
```

### Use Cases for Hand Tracking

| Use Case | How It Works | Real Example |
|----------|-------------|--------------|
| **Sign language** | Track 21 landmarks → Sequence model → Text/speech | Google's sign language translator |
| **Gesture control** | Recognize gestures → Map to commands | Smart TV, presentations |
| **Virtual piano** | Track finger tips → Detect key presses | Music education apps |
| **AR try-on** | Track hand → Place virtual ring/watch | Jewelry shopping apps |
| **Surgical training** | Track hand movements → Score precision | Medical VR simulators |
| **Gaming** | Hand poses → Game controls | VR/AR games |
| **Accessibility** | Hand gestures → Computer control | Disability assistive tech |
| **Drawing in air** | Track index tip path → Draw on screen | Creative/art apps |

---

## 8. Pose Estimation — BlazePose (33 Landmarks)

### 💡 What It Does

> **MediaPipe Pose Landmarker** (powered by BlazePose) detects **33 body pose landmarks** in real-time, covering the full body from head to feet. It outputs 3D coordinates (x, y, z) plus visibility scores. This enables fitness tracking, dance analysis, sports analytics, and full-body AR effects.

### The 33 Pose Landmarks

```
┌────────────────────────────────────────────────────────────────┐
│                    33 Pose Landmarks                             │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│              0 (nose)                                            │
│           1 / \ 4                                               │
│    (L eye) 2   5 (R eye)                                        │
│          3      6                                                │
│     (L ear)     (R ear)                                          │
│           7     8                                                │
│    (mouth)       (mouth)                                         │
│                                                                  │
│          11 ─── 12                                               │
│   (L shoulder)   (R shoulder)                                    │
│         |          |                                             │
│        13         14                                             │
│    (L elbow)   (R elbow)                                        │
│         |          |                                             │
│        15         16                                             │
│    (L wrist)   (R wrist)                                        │
│       / \        / \                                             │
│     17  19    18  20                                             │
│  (pinky)(index) (pinky)(index)                                   │
│     21          22                                               │
│   (thumb)     (thumb)                                            │
│                                                                  │
│        23 ─── 24                                                │
│     (L hip)   (R hip)                                            │
│        |        |                                                │
│       25       26                                                │
│   (L knee)  (R knee)                                            │
│        |        |                                                │
│       27       28                                                │
│  (L ankle) (R ankle)                                            │
│      / \      / \                                                │
│    29  31   30  32                                               │
│  (heel)(toe)(heel)(toe)                                          │
│                                                                  │
└────────────────────────────────────────────────────────────────┘

Landmark Index Reference:
  0: nose              11: left shoulder    23: left hip
  1: left eye inner    12: right shoulder   24: right hip
  2: left eye          13: left elbow       25: left knee
  3: left eye outer    14: right elbow      26: right knee
  4: right eye inner   15: left wrist       27: left ankle
  5: right eye         16: right wrist      28: right ankle
  6: right eye outer   17: left pinky       29: left heel
  7: left ear          18: right pinky      30: right heel
  8: right ear         19: left index       31: left foot index
  9: mouth left        20: right index      32: right foot index
  10: mouth right      21: left thumb
                       22: right thumb
```

### BlazePose Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    BlazePose Pipeline                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Stage 1: Person Detection (first frame only)                     │
│  ├── Detect person bounding box in full image                     │
│  ├── Output: person region of interest (ROI)                      │
│  └── Skip on subsequent frames (use tracking)                     │
│                                                                    │
│  Stage 2: Pose Landmark Model                                     │
│  ├── Input: Cropped person region (256×256)                       │
│  ├── Backbone: Custom CNN (MobileNet-like)                        │
│  ├── Heads:                                                       │
│  │   ├── Heatmap head → Coarse landmark locations                │
│  │   ├── Regression head → Fine 3D coordinates (x, y, z)        │
│  │   └── Segmentation head → Person mask (optional)              │
│  └── Output: 33 landmarks + visibility + presence                │
│                                                                    │
│  Stage 3: Tracking (frames 2+)                                    │
│  ├── Use previous landmarks → Predict ROI for next frame         │
│  ├── Skip person detection (3-4× faster!)                        │
│  └── Re-detect if tracking confidence drops                      │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘

Model Variants:
  • Lite:  3 MB,  fastest, good for mobile gaming
  • Full:  6 MB,  balanced accuracy/speed
  • Heavy: 26 MB, highest accuracy, research use
```

### Code Example: Pose Estimation

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
import cv2
import numpy as np

# ─── Configure Pose Landmarker ───
model_path = "pose_landmarker_full.task"

options = vision.PoseLandmarkerOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.IMAGE,
    num_poses=1,                          # Max people to detect
    min_pose_detection_confidence=0.5,
    min_pose_presence_confidence=0.5,
    min_tracking_confidence=0.5,
    output_segmentation_masks=True        # Optional body mask
)

# ─── Detect pose ───
with vision.PoseLandmarker.create_from_options(options) as landmarker:
    image = mp.Image.create_from_file("person_standing.jpg")
    result = landmarker.detect(image)
    
    print(f"Detected {len(result.pose_landmarks)} person(s)")
    
    if result.pose_landmarks:
        landmarks = result.pose_landmarks[0]
        
        # Key joint positions
        joints = {
            "Nose": landmarks[0],
            "Left Shoulder": landmarks[11],
            "Right Shoulder": landmarks[12],
            "Left Elbow": landmarks[13],
            "Right Elbow": landmarks[14],
            "Left Wrist": landmarks[15],
            "Right Wrist": landmarks[16],
            "Left Hip": landmarks[23],
            "Right Hip": landmarks[24],
            "Left Knee": landmarks[25],
            "Right Knee": landmarks[26],
            "Left Ankle": landmarks[27],
            "Right Ankle": landmarks[28],
        }
        
        print("\nKey Joint Positions:")
        for name, lm in joints.items():
            print(f"  {name:16s}: x={lm.x:.3f}, y={lm.y:.3f}, z={lm.z:.3f}, "
                  f"visibility={lm.visibility:.3f}")
    
    # Segmentation mask
    if result.segmentation_masks:
        mask = result.segmentation_masks[0].numpy_view()
        print(f"\nSegmentation mask shape: {mask.shape}")
        print(f"Person pixels: {(mask > 0.5).sum()}")

# ─── OUTPUT ───
# Detected 1 person(s)
#
# Key Joint Positions:
#   Nose            : x=0.501, y=0.152, z=-0.145, visibility=0.998
#   Left Shoulder   : x=0.582, y=0.302, z=-0.098, visibility=0.997
#   Right Shoulder  : x=0.418, y=0.298, z=-0.101, visibility=0.996
#   Left Elbow      : x=0.621, y=0.458, z=-0.052, visibility=0.991
#   Right Elbow     : x=0.378, y=0.462, z=-0.048, visibility=0.988
#   Left Wrist      : x=0.598, y=0.612, z=0.012, visibility=0.982
#   Right Wrist     : x=0.401, y=0.608, z=0.015, visibility=0.979
#   Left Hip        : x=0.548, y=0.598, z=-0.025, visibility=0.994
#   Right Hip       : x=0.452, y=0.595, z=-0.028, visibility=0.993
#   Left Knee       : x=0.562, y=0.768, z=0.035, visibility=0.987
#   Right Knee      : x=0.438, y=0.772, z=0.032, visibility=0.985
#   Left Ankle      : x=0.571, y=0.938, z=0.012, visibility=0.971
#   Right Ankle     : x=0.429, y=0.942, z=0.015, visibility=0.968
#
# Segmentation mask shape: (480, 640)
# Person pixels: 89234
```

### Code Example: Exercise Rep Counter (Bicep Curls)

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
import cv2
import numpy as np
import math
import time

def calculate_angle(a, b, c):
    """
    Calculate angle at point b, formed by points a-b-c.
    Returns angle in degrees.
    """
    a = np.array([a.x, a.y])
    b = np.array([b.x, b.y])
    c = np.array([c.x, c.y])
    
    ba = a - b
    bc = c - b
    
    cosine_angle = np.dot(ba, bc) / (np.linalg.norm(ba) * np.linalg.norm(bc))
    cosine_angle = np.clip(cosine_angle, -1.0, 1.0)
    angle = np.degrees(np.arccos(cosine_angle))
    
    return angle

# ─── Exercise tracker state ───
rep_count = 0
stage = "down"  # "down" or "up"
angles_history = []

# ─── Real-time pose tracking ───
model_path = "pose_landmarker_full.task"
latest_result = None

def callback(result, image, timestamp_ms):
    global latest_result
    latest_result = result

options = vision.PoseLandmarkerOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.LIVE_STREAM,
    num_poses=1,
    result_callback=callback
)

with vision.PoseLandmarker.create_from_options(options) as landmarker:
    cap = cv2.VideoCapture(0)
    
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        
        h, w = frame.shape[:2]
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        mp_image = mp.Image(image_format=mp.ImageFormat.SRGB, data=rgb)
        landmarker.detect_async(mp_image, int(time.time() * 1000))
        
        if latest_result and latest_result.pose_landmarks:
            landmarks = latest_result.pose_landmarks[0]
            
            # Get shoulder, elbow, wrist for left arm
            shoulder = landmarks[11]  # Left shoulder
            elbow = landmarks[13]     # Left elbow
            wrist = landmarks[15]     # Left wrist
            
            # Calculate elbow angle
            angle = calculate_angle(shoulder, elbow, wrist)
            angles_history.append(angle)
            
            # Count reps
            if angle > 160:  # Arm extended
                stage = "down"
            if angle < 40 and stage == "down":  # Arm curled
                stage = "up"
                rep_count += 1
            
            # Draw skeleton
            joints = [(11,13), (13,15), (12,14), (14,16),  # Arms
                     (11,12), (11,23), (12,24), (23,24),   # Torso
                     (23,25), (25,27), (24,26), (26,28)]   # Legs
            
            for start_idx, end_idx in joints:
                start = landmarks[start_idx]
                end = landmarks[end_idx]
                x1, y1 = int(start.x * w), int(start.y * h)
                x2, y2 = int(end.x * w), int(end.y * h)
                cv2.line(frame, (x1, y1), (x2, y2), (0, 255, 0), 3)
            
            # Draw landmarks
            for lm in landmarks:
                cx, cy = int(lm.x * w), int(lm.y * h)
                cv2.circle(frame, (cx, cy), 5, (255, 0, 0), -1)
            
            # Display info
            cv2.putText(frame, f"Angle: {angle:.1f} deg", (10, 40),
                       cv2.FONT_HERSHEY_SIMPLEX, 1, (255, 255, 0), 2)
            cv2.putText(frame, f"Reps: {rep_count}", (10, 80),
                       cv2.FONT_HERSHEY_SIMPLEX, 1.5, (0, 0, 255), 3)
            cv2.putText(frame, f"Stage: {stage}", (10, 120),
                       cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 255, 255), 2)
        
        cv2.imshow("Bicep Curl Counter", frame)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break
    
    cap.release()
    cv2.destroyAllWindows()

# ─── OUTPUT ───
# Opens webcam showing:
# • Full body skeleton overlay (green lines + blue dots)
# • "Angle: 145.3 deg" — current elbow angle
# • "Reps: 7" — total bicep curls counted
# • "Stage: up/down" — current phase of movement
# • Counts automatically when arm goes from extended to curled
```

### Code Example: Yoga Pose Classification

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
import numpy as np
import math

def landmarks_to_angles(landmarks):
    """Extract key body angles from pose landmarks."""
    
    def angle_3points(a, b, c):
        a = np.array([a.x, a.y])
        b = np.array([b.x, b.y])
        c = np.array([c.x, c.y])
        ba = a - b
        bc = c - b
        cos_angle = np.dot(ba, bc) / (np.linalg.norm(ba) * np.linalg.norm(bc) + 1e-6)
        return np.degrees(np.arccos(np.clip(cos_angle, -1, 1)))
    
    angles = {
        "left_elbow": angle_3points(landmarks[11], landmarks[13], landmarks[15]),
        "right_elbow": angle_3points(landmarks[12], landmarks[14], landmarks[16]),
        "left_shoulder": angle_3points(landmarks[13], landmarks[11], landmarks[23]),
        "right_shoulder": angle_3points(landmarks[14], landmarks[12], landmarks[24]),
        "left_hip": angle_3points(landmarks[11], landmarks[23], landmarks[25]),
        "right_hip": angle_3points(landmarks[12], landmarks[24], landmarks[26]),
        "left_knee": angle_3points(landmarks[23], landmarks[25], landmarks[27]),
        "right_knee": angle_3points(landmarks[24], landmarks[26], landmarks[28]),
    }
    return angles

def classify_yoga_pose(angles):
    """Simple rule-based yoga pose classifier."""
    
    # Warrior II: Arms extended, one knee bent ~90°, other straight
    if (angles["left_shoulder"] > 150 and angles["right_shoulder"] > 150 and
        (angles["left_knee"] < 120 or angles["right_knee"] < 120)):
        return "Warrior II (Virabhadrasana II)"
    
    # Tree Pose: One leg straight, arms up
    if (angles["left_knee"] > 160 and angles["right_knee"] < 90) or \
       (angles["right_knee"] > 160 and angles["left_knee"] < 90):
        return "Tree Pose (Vrikshasana)"
    
    # T-Pose: Both arms extended horizontally
    if (angles["left_shoulder"] > 150 and angles["right_shoulder"] > 150 and
        angles["left_elbow"] > 150 and angles["right_elbow"] > 150 and
        angles["left_knee"] > 160 and angles["right_knee"] > 160):
        return "T-Pose / Star Pose"
    
    # Mountain Pose: Standing straight, arms down
    if (angles["left_knee"] > 160 and angles["right_knee"] > 160 and
        angles["left_shoulder"] < 40 and angles["right_shoulder"] < 40):
        return "Mountain Pose (Tadasana)"
    
    return "Unknown Pose"

# ─── Usage ───
model_path = "pose_landmarker_full.task"
options = vision.PoseLandmarkerOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.IMAGE,
    num_poses=1
)

with vision.PoseLandmarker.create_from_options(options) as landmarker:
    image = mp.Image.create_from_file("yoga_warrior.jpg")
    result = landmarker.detect(image)
    
    if result.pose_landmarks:
        landmarks = result.pose_landmarks[0]
        angles = landmarks_to_angles(landmarks)
        pose_name = classify_yoga_pose(angles)
        
        print(f"Detected Pose: {pose_name}")
        print(f"\nJoint Angles:")
        for joint, angle in angles.items():
            print(f"  {joint:18s}: {angle:.1f}°")

# ─── OUTPUT ───
# Detected Pose: Warrior II (Virabhadrasana II)
#
# Joint Angles:
#   left_elbow        : 172.3°
#   right_elbow       : 168.7°
#   left_shoulder     : 162.4°
#   right_shoulder    : 158.9°
#   left_hip          : 98.2°
#   right_hip         : 165.1°
#   left_knee         : 92.5°
#   right_knee        : 171.8°
```

### Use Cases for Pose Estimation

| Use Case | How It Works | Real Example |
|----------|-------------|--------------|
| **Fitness tracking** | Count reps by tracking joint angles | Peloton, Nike Training Club |
| **Physical therapy** | Monitor exercise form, provide feedback | Kaia Health, Hinge Health |
| **Dance scoring** | Compare poses to reference choreography | Just Dance (game) |
| **Sports analytics** | Analyze athlete biomechanics | NBA, FIFA broadcast |
| **Safety monitoring** | Detect falls, unusual postures | Elderly care systems |
| **Ergonomics** | Analyze sitting/standing posture | Office wellness apps |
| **Animation/Gaming** | Drive 3D character from video | Motion capture alternative |
| **Martial arts** | Score form, count moves | Training apps |
| **Yoga instruction** | Check pose alignment, provide corrections | Down Dog, Yoga Studio |
| **Retail analytics** | Analyze shopping behavior patterns | Store layout optimization |

---

## 9. Holistic — Full Body + Hands + Face Together

### 💡 What It Does

> **MediaPipe Holistic** combines **pose (33) + face mesh (478) + both hands (21×2 = 42)** landmarks into a single pipeline, totaling **553 landmarks** tracked simultaneously in real-time. This is the most comprehensive body tracking solution available for real-time applications.

### Architecture: How Holistic Combines Everything

```
┌──────────────────────────────────────────────────────────────────┐
│                  MediaPipe Holistic Pipeline                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Input Frame                                                       │
│       │                                                            │
│       ▼                                                            │
│  ┌──────────────┐                                                 │
│  │ Pose Model   │ → 33 body landmarks                            │
│  │ (BlazePose)  │   + hand ROIs + face ROI                       │
│  └──────────────┘                                                 │
│       │                                                            │
│       ├── Hand ROI (from wrist landmarks) ─┐                     │
│       │                                     ▼                      │
│       │                        ┌──────────────────┐               │
│       │                        │ Hand Landmarker  │               │
│       │                        │ (×2: left+right) │               │
│       │                        │ → 21 landmarks/  │               │
│       │                        │    hand           │               │
│       │                        └──────────────────┘               │
│       │                                                            │
│       ├── Face ROI (from face landmarks) ──┐                     │
│       │                                     ▼                      │
│       │                        ┌──────────────────┐               │
│       │                        │ Face Mesh Model  │               │
│       │                        │ → 478 face       │               │
│       │                        │   landmarks      │               │
│       │                        └──────────────────┘               │
│       │                                                            │
│       ▼                                                            │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ OUTPUT: 33 pose + 478 face + 21 left hand + 21 right hand│    │
│  │ Total: 553 landmarks per frame                            │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                    │
│  KEY INSIGHT: Pose model provides ROIs for face & hands,          │
│  so we don't need to run separate detection for each!             │
│  This makes it faster than running 3 models independently.        │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Code Example: Holistic Tracking

```python
import mediapipe as mp
import cv2
import time

# ─── Using the legacy API (Holistic is unified in legacy solutions) ───
mp_holistic = mp.solutions.holistic
mp_drawing = mp.solutions.drawing_utils
mp_drawing_styles = mp.solutions.drawing_styles

# ─── Real-time holistic tracking ───
cap = cv2.VideoCapture(0)

with mp_holistic.Holistic(
    static_image_mode=False,
    model_complexity=1,          # 0=Lite, 1=Full, 2=Heavy
    smooth_landmarks=True,       # Temporal smoothing
    enable_segmentation=True,    # Body segmentation mask
    min_detection_confidence=0.5,
    min_tracking_confidence=0.5
) as holistic:
    
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        
        # Convert to RGB
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        rgb.flags.writeable = False
        
        # Process frame
        results = holistic.process(rgb)
        
        rgb.flags.writeable = True
        frame = cv2.cvtColor(rgb, cv2.COLOR_RGB2BGR)
        
        # Draw face mesh
        if results.face_landmarks:
            mp_drawing.draw_landmarks(
                frame, results.face_landmarks,
                mp_holistic.FACEMESH_TESSELATION,
                landmark_drawing_spec=None,
                connection_drawing_spec=mp_drawing_styles.get_default_face_mesh_tesselation_style()
            )
            face_count = len(results.face_landmarks.landmark)
            cv2.putText(frame, f"Face: {face_count} landmarks", (10, 30),
                       cv2.FONT_HERSHEY_SIMPLEX, 0.6, (0, 255, 0), 2)
        
        # Draw pose
        if results.pose_landmarks:
            mp_drawing.draw_landmarks(
                frame, results.pose_landmarks,
                mp_holistic.POSE_CONNECTIONS,
                landmark_drawing_spec=mp_drawing_styles.get_default_pose_landmarks_style()
            )
            cv2.putText(frame, "Pose: 33 landmarks", (10, 60),
                       cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 0), 2)
        
        # Draw left hand
        if results.left_hand_landmarks:
            mp_drawing.draw_landmarks(
                frame, results.left_hand_landmarks,
                mp_holistic.HAND_CONNECTIONS,
                mp_drawing_styles.get_default_hand_landmarks_style(),
                mp_drawing_styles.get_default_hand_connections_style()
            )
            cv2.putText(frame, "Left hand: 21 landmarks", (10, 90),
                       cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 0, 255), 2)
        
        # Draw right hand
        if results.right_hand_landmarks:
            mp_drawing.draw_landmarks(
                frame, results.right_hand_landmarks,
                mp_holistic.HAND_CONNECTIONS,
                mp_drawing_styles.get_default_hand_landmarks_style(),
                mp_drawing_styles.get_default_hand_connections_style()
            )
            cv2.putText(frame, "Right hand: 21 landmarks", (10, 120),
                       cv2.FONT_HERSHEY_SIMPLEX, 0.6, (0, 255, 255), 2)
        
        # Segmentation mask overlay
        if results.segmentation_mask is not None:
            mask = results.segmentation_mask
            condition = mask > 0.5
            # Create purple background
            bg = np.zeros_like(frame)
            bg[:] = (128, 0, 128)
            frame = np.where(condition[:, :, None], frame, bg)
        
        cv2.imshow("MediaPipe Holistic", frame)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

cap.release()
cv2.destroyAllWindows()

# ─── OUTPUT ───
# Opens webcam showing ALL simultaneously:
# • Dense face mesh (green wireframe)
# • Body skeleton (yellow connections)
# • Left hand landmarks (magenta)
# • Right hand landmarks (cyan)
# • Purple background replacement (segmentation)
# • Info text showing detected landmark counts
# • Runs at 15-25 FPS depending on hardware
```

### Use Cases for Holistic Tracking

| Use Case | Why Holistic? | Real Example |
|----------|---------------|--------------|
| **Sign language** | Needs face expressions + hand shapes + body position together | Complete sign translation |
| **Full-body avatars** | Drive entire avatar from single camera | VTubers, Metaverse |
| **Dance instruction** | Track every body part simultaneously | Online dance classes |
| **Telehealth** | Doctor observes patient's full body + face expressions | Remote diagnosis |
| **Theater/Performance** | Capture full performance with one camera | Digital theater |
| **Virtual presentations** | Full body + gestures + expressions in VR | Virtual meeting rooms |

---

## 10. Image Segmentation & Selfie Segmentation

### 💡 What It Does

> **MediaPipe Image Segmenter** performs pixel-level classification of images, assigning each pixel to a category. **Selfie Segmentation** is a specialized variant that separates people from backgrounds in real-time — powering virtual backgrounds in video calls.

### Types of Segmentation Available

```
┌────────────────────────────────────────────────────────────────┐
│           MediaPipe Segmentation Options                         │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. SELFIE SEGMENTATION (Person/Background)                     │
│     • Output: Binary mask (person = 1, background = 0)          │
│     • Models: General (256×256) or Landscape (144×256)          │
│     • Size: 200 KB (!)                                           │
│     • Speed: < 5ms on mobile                                     │
│     • Use: Video calls, photo editing                            │
│                                                                  │
│  2. IMAGE SEGMENTATION (Multi-class)                            │
│     • Output: Per-pixel category (21 categories)                │
│     • Categories: person, car, road, building, sky, etc.        │
│     • Model: DeepLab V3 based                                   │
│     • Size: 2-10 MB                                              │
│     • Use: Scene understanding, autonomous driving              │
│                                                                  │
│  3. INTERACTIVE SEGMENTATION                                    │
│     • Output: Mask for clicked object                           │
│     • Input: Image + click point                                │
│     • Similar to SAM but lighter                                │
│     • Use: Photo editing, object selection                      │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

### Code Example: Selfie Segmentation (Virtual Background)

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
import cv2
import numpy as np
import time

# ─── Method 1: Using Tasks API (Recommended) ───
model_path = "selfie_segmenter.tflite"

options = vision.ImageSegmenterOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.VIDEO,
    output_category_mask=True
)

# ─── Virtual Background Replacement ───
# Load background image
background = cv2.imread("beach_background.jpg")

cap = cv2.VideoCapture(0)
ret, frame = cap.read()
h, w = frame.shape[:2]
background = cv2.resize(background, (w, h))

frame_count = 0

with vision.ImageSegmenter.create_from_options(options) as segmenter:
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        
        # Convert to MediaPipe format
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        mp_image = mp.Image(image_format=mp.ImageFormat.SRGB, data=rgb)
        
        timestamp_ms = int(frame_count * 1000 / 30)
        result = segmenter.segment_for_video(mp_image, timestamp_ms)
        
        # Get category mask (0 = background, 1-15 = person parts)
        category_mask = result.category_mask.numpy_view()
        
        # Create binary person mask
        person_mask = (category_mask > 0).astype(np.uint8)
        
        # Smooth the mask edges (reduce jaggies)
        person_mask = cv2.GaussianBlur(person_mask.astype(np.float32), (7, 7), 3)
        person_mask = np.stack([person_mask] * 3, axis=-1)
        
        # Blend: person from camera + background from image
        output = (frame * person_mask + background * (1 - person_mask)).astype(np.uint8)
        
        cv2.imshow("Virtual Background", output)
        frame_count += 1
        
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

cap.release()
cv2.destroyAllWindows()

# ─── OUTPUT ───
# Opens webcam showing:
# • You (the person) rendered on top of a beach background
# • Clean separation between person and background
# • Smooth edges (Gaussian blur on mask)
# • Runs at 25-30 FPS on laptop CPU
# • Similar to Zoom/Google Meet virtual background feature
```

### Code Example: Background Blur (Bokeh Effect)

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
import cv2
import numpy as np

# ─── Selfie segmentation for background blur ───
model_path = "selfie_segmenter.tflite"

options = vision.ImageSegmenterOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.VIDEO,
    output_category_mask=True
)

cap = cv2.VideoCapture(0)
frame_count = 0

with vision.ImageSegmenter.create_from_options(options) as segmenter:
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        mp_image = mp.Image(image_format=mp.ImageFormat.SRGB, data=rgb)
        
        timestamp_ms = int(frame_count * 1000 / 30)
        result = segmenter.segment_for_video(mp_image, timestamp_ms)
        
        category_mask = result.category_mask.numpy_view()
        person_mask = (category_mask > 0).astype(np.float32)
        
        # Smooth mask
        person_mask = cv2.GaussianBlur(person_mask, (15, 15), 5)
        mask_3ch = np.stack([person_mask] * 3, axis=-1)
        
        # Create heavily blurred background
        blurred = cv2.GaussianBlur(frame, (55, 55), 0)
        
        # Composite: sharp person + blurred background
        output = (frame * mask_3ch + blurred * (1 - mask_3ch)).astype(np.uint8)
        
        cv2.imshow("Background Blur", output)
        frame_count += 1
        
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

cap.release()
cv2.destroyAllWindows()

# ─── OUTPUT ───
# Opens webcam showing:
# • Person in sharp focus
# • Background with strong Gaussian blur (bokeh effect)
# • Like "Portrait Mode" on smartphones
# • Real-time at 25-30 FPS on CPU
```

### Code Example: Multi-Class Segmentation

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
import cv2
import numpy as np

# ─── Multi-class image segmentation ───
model_path = "deeplabv3.tflite"  # 21-class segmentation model

options = vision.ImageSegmenterOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.IMAGE,
    output_category_mask=True
)

# Category labels (Pascal VOC classes)
CATEGORIES = [
    "background", "aeroplane", "bicycle", "bird", "boat",
    "bottle", "bus", "car", "cat", "chair",
    "cow", "dining table", "dog", "horse", "motorbike",
    "person", "potted plant", "sheep", "sofa", "train", "tv/monitor"
]

# Colors for each category
COLORS = np.random.randint(0, 255, size=(21, 3), dtype=np.uint8)
COLORS[0] = [0, 0, 0]  # Background = black

with vision.ImageSegmenter.create_from_options(options) as segmenter:
    image = mp.Image.create_from_file("street_scene.jpg")
    result = segmenter.segment(image)
    
    category_mask = result.category_mask.numpy_view()
    
    # Create colored segmentation map
    colored_mask = COLORS[category_mask]
    
    # Print detected categories
    unique_categories = np.unique(category_mask)
    print("Detected categories:")
    for cat_id in unique_categories:
        if cat_id > 0:  # Skip background
            pixel_count = (category_mask == cat_id).sum()
            percentage = pixel_count / category_mask.size * 100
            print(f"  {CATEGORIES[cat_id]:15s}: {percentage:.1f}% of image ({pixel_count} pixels)")
    
    # Save colored segmentation
    cv2.imwrite("segmentation_output.png", colored_mask)

# ─── OUTPUT ───
# Detected categories:
#   person         : 12.3% of image (47234 pixels)
#   car            : 18.7% of image (71892 pixels)
#   bicycle        : 3.2% of image (12288 pixels)
#   potted plant   : 1.8% of image (6912 pixels)
#
# Saves colored segmentation image where each category has unique color
```

### Use Cases for Segmentation

| Use Case | How It Works | Real Example |
|----------|-------------|--------------|
| **Video call backgrounds** | Separate person → Replace/blur background | Google Meet, Zoom, Teams |
| **Photo editing** | Segment subject → Apply effects separately | Snapseed, Lightroom |
| **Portrait mode** | Segment person → Blur background | Smartphone cameras |
| **AR effects** | Segment body parts → Overlay graphics | Instagram, Snapchat |
| **Green screen replacement** | Segment without physical green screen | OBS Studio, streaming |
| **Autonomous driving** | Segment road, cars, pedestrians, sky | Waymo, Tesla |
| **Medical imaging** | Segment organs, tumors | Hospital systems |
| **Retail** | Segment product from background | E-commerce photo cleanup |

---

## 11. MediaPipe Model Maker — Custom Training

### 💡 What It Does

> **MediaPipe Model Maker** lets you fine-tune MediaPipe's pre-trained models on YOUR custom data with minimal code. You can create custom object detectors, image classifiers, and text classifiers that are optimized for on-device deployment (TFLite format).

### What You Can Customize

```
┌────────────────────────────────────────────────────────────────┐
│           MediaPipe Model Maker Tasks                            │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ✅ Object Detection    → Custom objects (your classes)         │
│  ✅ Image Classification → Custom image categories              │
│  ✅ Text Classification  → Custom text categories               │
│  ✅ Gesture Recognition  → Custom hand gestures                 │
│                                                                  │
│  Process:                                                        │
│  1. Prepare labeled data (images + annotations)                 │
│  2. Choose base model (EfficientDet-Lite, MobileNet, etc.)     │
│  3. Fine-tune with Model Maker (few lines of code)             │
│  4. Export as TFLite → Deploy on mobile/edge                    │
│                                                                  │
│  Minimum data needed: ~50-100 images per class (with augment.)  │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

### Code Example: Train Custom Object Detector

```python
# ─── Install Model Maker ───
# pip install mediapipe-model-maker

from mediapipe_model_maker import object_detector
import os

# ─── Step 1: Prepare Data ───
# Data format: COCO JSON or Pascal VOC
# Directory structure:
# dataset/
# ├── train/
# │   ├── images/
# │   │   ├── img001.jpg
# │   │   ├── img002.jpg
# │   │   └── ...
# │   └── labels.json  (COCO format)
# └── val/
#     ├── images/
#     └── labels.json

# ─── Step 2: Load Dataset ───
train_data = object_detector.Dataset.from_coco_folder(
    dirname="dataset/train",
    cache_dir="/tmp/od_cache"
)
val_data = object_detector.Dataset.from_coco_folder(
    dirname="dataset/val",
    cache_dir="/tmp/od_cache"
)

print(f"Training samples: {train_data.size}")
print(f"Validation samples: {val_data.size}")
print(f"Categories: {train_data.label_names}")

# ─── Step 3: Configure Training ───
spec = object_detector.SupportedModels.MOBILENET_V2  
# Options: MOBILENET_V2, MOBILENET_MULTI_AVG, EFFICIENTDET_LITE0-4

hparams = object_detector.HParams(
    learning_rate=0.01,
    batch_size=8,
    epochs=50,
    export_dir="exported_model"
)

options = object_detector.ObjectDetectorOptions(
    supported_model=spec,
    hparams=hparams
)

# ─── Step 4: Train ───
model = object_detector.ObjectDetector.create(
    train_data=train_data,
    validation_data=val_data,
    options=options
)

# ─── Step 5: Evaluate ───
metrics = model.evaluate(val_data)
print(f"\nEvaluation Results:")
print(f"  mAP: {metrics['AP']:.3f}")
print(f"  mAP@50: {metrics['AP50']:.3f}")
print(f"  mAP@75: {metrics['AP75']:.3f}")

# ─── Step 6: Export ───
model.export_model("custom_detector.tflite")
print("\nModel exported as TFLite!")
print(f"Model size: {os.path.getsize('exported_model/custom_detector.tflite') / 1024:.1f} KB")

# ─── OUTPUT ───
# Training samples: 500
# Validation samples: 100
# Categories: ['hardhat', 'vest', 'no_hardhat', 'no_vest']
#
# Epoch 1/50: loss=2.345, val_loss=2.123
# Epoch 10/50: loss=0.892, val_loss=0.945
# Epoch 25/50: loss=0.423, val_loss=0.512
# Epoch 50/50: loss=0.198, val_loss=0.389
#
# Evaluation Results:
#   mAP: 0.721
#   mAP@50: 0.892
#   mAP@75: 0.634
#
# Model exported as TFLite!
# Model size: 4832.5 KB
```

### Code Example: Use Custom Model

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
import cv2

# ─── Use your custom-trained model ───
model_path = "exported_model/custom_detector.tflite"

options = vision.ObjectDetectorOptions(
    base_options=mp.tasks.BaseOptions(model_asset_path=model_path),
    running_mode=vision.RunningMode.IMAGE,
    max_results=10,
    score_threshold=0.4
)

with vision.ObjectDetector.create_from_options(options) as detector:
    image = mp.Image.create_from_file("construction_site.jpg")
    result = detector.detect(image)
    
    print(f"Detected {len(result.detections)} objects:")
    for det in result.detections:
        cat = det.categories[0]
        bbox = det.bounding_box
        print(f"  {cat.category_name}: {cat.score:.2f} "
              f"at [{bbox.origin_x},{bbox.origin_y},{bbox.width},{bbox.height}]")

# ─── OUTPUT ───
# Detected 5 objects:
#   hardhat: 0.94 at [120,45,80,75]
#   vest: 0.91 at [95,200,150,180]
#   hardhat: 0.87 at [350,52,72,68]
#   no_vest: 0.82 at [400,190,140,175]
#   no_hardhat: 0.76 at [520,48,65,60]
```

### Code Example: Train Custom Image Classifier

```python
from mediapipe_model_maker import image_classifier
import os

# ─── Data structure for classification ───
# dataset/
# ├── cat/
# │   ├── img1.jpg
# │   ├── img2.jpg
# │   └── ...
# ├── dog/
# │   ├── img1.jpg
# │   └── ...
# └── bird/
#     └── ...

# ─── Load data ───
data = image_classifier.Dataset.from_folder("dataset/")
train_data, rest_data = data.split(0.8)
val_data, test_data = rest_data.split(0.5)

print(f"Train: {train_data.size}, Val: {val_data.size}, Test: {test_data.size}")

# ─── Train classifier ───
options = image_classifier.ImageClassifierOptions(
    supported_model=image_classifier.SupportedModels.EFFICIENTNET_LITE0,
    hparams=image_classifier.HParams(
        epochs=20,
        batch_size=32,
        learning_rate=0.005
    )
)

model = image_classifier.ImageClassifier.create(
    train_data=train_data,
    validation_data=val_data,
    options=options
)

# ─── Evaluate ───
metrics = model.evaluate(test_data)
print(f"Test accuracy: {metrics['accuracy']:.3f}")

# ─── Export ───
model.export_model("custom_classifier.tflite")

# ─── OUTPUT ───
# Train: 800, Val: 100, Test: 100
# Epoch 1/20: accuracy=0.452, val_accuracy=0.480
# Epoch 10/20: accuracy=0.891, val_accuracy=0.870
# Epoch 20/20: accuracy=0.967, val_accuracy=0.940
# Test accuracy: 0.930
```

---

## 12. MediaPipe on Web (JavaScript)

### 💡 Why Web?

> MediaPipe runs directly in the browser using **WebAssembly (WASM)** and **WebGL** for GPU acceleration. No server needed — all ML inference happens on the user's device. This enables privacy-preserving, zero-latency ML applications accessible from any browser.

### Code Example: Object Detection in Browser

```html
<!DOCTYPE html>
<html>
<head>
    <title>MediaPipe Object Detection - Web</title>
    <style>
        #output { position: relative; display: inline-block; }
        canvas { position: absolute; top: 0; left: 0; }
        video { display: block; }
    </style>
</head>
<body>
    <h1>MediaPipe Object Detection (Browser)</h1>
    <div id="output">
        <video id="webcam" autoplay playsinline></video>
        <canvas id="canvas"></canvas>
    </div>
    <p id="fps">FPS: --</p>

    <!-- MediaPipe Tasks Vision CDN -->
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest/wasm/vision_bundle.js"></script>
    
    <script type="module">
        import { ObjectDetector, FilesetResolver } from 
            "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest";

        let objectDetector;
        let video, canvas, ctx;
        let lastTime = performance.now();

        // ─── Initialize ───
        async function init() {
            // Load WASM files
            const vision = await FilesetResolver.forVisionTasks(
                "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest/wasm"
            );
            
            // Create detector
            objectDetector = await ObjectDetector.createFromOptions(vision, {
                baseOptions: {
                    modelAssetPath: "https://storage.googleapis.com/mediapipe-models/object_detector/efficientdet_lite0/float16/1/efficientdet_lite0.tflite",
                    delegate: "GPU"  // Use WebGL for acceleration
                },
                runningMode: "VIDEO",
                maxResults: 5,
                scoreThreshold: 0.5
            });

            // Setup webcam
            video = document.getElementById("webcam");
            canvas = document.getElementById("canvas");
            ctx = canvas.getContext("2d");
            
            const stream = await navigator.mediaDevices.getUserMedia({ 
                video: { width: 640, height: 480 } 
            });
            video.srcObject = stream;
            video.addEventListener("loadeddata", () => {
                canvas.width = video.videoWidth;
                canvas.height = video.videoHeight;
                detectFrame();
            });
        }

        // ─── Detection loop ───
        function detectFrame() {
            const now = performance.now();
            const results = objectDetector.detectForVideo(video, now);
            
            // Clear canvas
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // Draw detections
            for (const detection of results.detections) {
                const bbox = detection.boundingBox;
                const category = detection.categories[0];
                
                // Draw box
                ctx.strokeStyle = "#00FF00";
                ctx.lineWidth = 2;
                ctx.strokeRect(bbox.originX, bbox.originY, bbox.width, bbox.height);
                
                // Draw label
                ctx.fillStyle = "#00FF00";
                ctx.font = "16px Arial";
                ctx.fillText(
                    `${category.categoryName} (${(category.score * 100).toFixed(0)}%)`,
                    bbox.originX, bbox.originY - 5
                );
            }
            
            // FPS counter
            const fps = 1000 / (now - lastTime);
            lastTime = now;
            document.getElementById("fps").textContent = `FPS: ${fps.toFixed(1)}`;
            
            requestAnimationFrame(detectFrame);
        }

        init();
    </script>
</body>
</html>

<!-- OUTPUT (in browser):
  • Webcam feed with green bounding boxes
  • Labels showing detected objects + confidence
  • FPS counter: typically 20-30 FPS in browser
  • No server needed — runs entirely client-side!
  • Works on desktop Chrome, Firefox, Edge, Safari
  • Works on mobile browsers too
-->
```

### Code Example: Hand Tracking in Browser

```html
<!DOCTYPE html>
<html>
<head>
    <title>MediaPipe Hand Tracking - Web</title>
</head>
<body>
    <video id="webcam" autoplay playsinline width="640" height="480"></video>
    <canvas id="canvas" width="640" height="480"></canvas>
    <p id="gesture">Gesture: --</p>

    <script type="module">
        import { HandLandmarker, FilesetResolver, DrawingUtils } from 
            "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest";

        let handLandmarker;
        const video = document.getElementById("webcam");
        const canvas = document.getElementById("canvas");
        const ctx = canvas.getContext("2d");

        async function init() {
            const vision = await FilesetResolver.forVisionTasks(
                "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest/wasm"
            );
            
            handLandmarker = await HandLandmarker.createFromOptions(vision, {
                baseOptions: {
                    modelAssetPath: "https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task",
                    delegate: "GPU"
                },
                runningMode: "VIDEO",
                numHands: 2,
                minHandDetectionConfidence: 0.5,
                minTrackingConfidence: 0.5
            });

            const stream = await navigator.mediaDevices.getUserMedia({ video: true });
            video.srcObject = stream;
            video.addEventListener("loadeddata", detect);
        }

        function detect() {
            const results = handLandmarker.detectForVideo(video, performance.now());
            
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            if (results.landmarks) {
                const drawingUtils = new DrawingUtils(ctx);
                
                for (const landmarks of results.landmarks) {
                    // Draw connections
                    drawingUtils.drawConnectors(
                        landmarks, 
                        HandLandmarker.HAND_CONNECTIONS,
                        { color: "#00FF00", lineWidth: 3 }
                    );
                    // Draw landmarks
                    drawingUtils.drawLandmarks(landmarks, {
                        color: "#FF0000",
                        lineWidth: 1,
                        radius: 4
                    });
                }
                
                // Simple gesture detection
                if (results.landmarks.length > 0) {
                    const lm = results.landmarks[0];
                    const fingersUp = countFingers(lm);
                    document.getElementById("gesture").textContent = 
                        `Gesture: ${fingersUp} fingers up`;
                }
            }
            
            requestAnimationFrame(detect);
        }

        function countFingers(landmarks) {
            let count = 0;
            // Index, Middle, Ring, Pinky
            const tips = [8, 12, 16, 20];
            const pips = [6, 10, 14, 18];
            for (let i = 0; i < 4; i++) {
                if (landmarks[tips[i]].y < landmarks[pips[i]].y) count++;
            }
            // Thumb (x comparison)
            if (landmarks[4].x < landmarks[3].x) count++;
            return count;
        }

        init();
    </script>
</body>
</html>

<!-- OUTPUT (in browser):
  • Webcam with hand skeleton overlay
  • Green lines connecting joints
  • Red dots on each landmark
  • Text showing "Gesture: 3 fingers up" (updates real-time)
  • 25-30 FPS in browser
  • Fully client-side, no data sent to server
-->
```

### Web Deployment Considerations

```
ADVANTAGES of Web MediaPipe:
  ✅ No installation needed (just open URL)
  ✅ Privacy: all processing on user's device
  ✅ Cross-platform: any modern browser
  ✅ Free hosting (GitHub Pages, Vercel)
  ✅ WebGL GPU acceleration

LIMITATIONS:
  ⚠️ Slower than native (WASM overhead)
  ⚠️ Safari WebGL support varies
  ⚠️ Large model files = slow first load
  ⚠️ No multi-threading in all browsers
  ⚠️ Battery drain on mobile

OPTIMIZATION TIPS:
  • Use float16 models (half the size, same accuracy)
  • Enable GPU delegate for 2-3× speedup
  • Cache models with Service Workers
  • Reduce input resolution for speed
  • Use requestAnimationFrame (not setInterval)
```

---

## 13. MediaPipe on Mobile (Android & iOS)

### 💡 Mobile: MediaPipe's Home Turf

> MediaPipe was originally built FOR mobile. It achieves its best performance on phones via hardware acceleration (NNAPI on Android, CoreML/Metal on iOS). Models run at 30-60+ FPS on modern phones.

### Android Integration (Kotlin)

```kotlin
// ─── build.gradle (app level) ───
dependencies {
    implementation("com.google.mediapipe:tasks-vision:0.10.14")
}

// ─── ObjectDetectionActivity.kt ───
import com.google.mediapipe.tasks.vision.objectdetector.ObjectDetector
import com.google.mediapipe.tasks.vision.objectdetector.ObjectDetectorOptions
import com.google.mediapipe.tasks.core.BaseOptions
import com.google.mediapipe.framework.image.BitmapImageBuilder

class ObjectDetectionActivity : AppCompatActivity() {
    
    private lateinit var objectDetector: ObjectDetector
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setupDetector()
    }
    
    private fun setupDetector() {
        val baseOptions = BaseOptions.builder()
            .setModelAssetPath("efficientdet_lite0.tflite")  // In assets/
            .setDelegate(BaseOptions.Delegate.GPU)  // Use GPU
            .build()
        
        val options = ObjectDetectorOptions.builder()
            .setBaseOptions(baseOptions)
            .setRunningMode(RunningMode.LIVE_STREAM)
            .setMaxResults(5)
            .setScoreThreshold(0.5f)
            .setResultListener { result, inputImage ->
                // Process detections on UI thread
                runOnUiThread {
                    displayResults(result)
                }
            }
            .build()
        
        objectDetector = ObjectDetector.createFromOptions(this, options)
    }
    
    // Called for each camera frame
    fun processFrame(bitmap: Bitmap, timestamp: Long) {
        val mpImage = BitmapImageBuilder(bitmap).build()
        objectDetector.detectAsync(mpImage, timestamp)
    }
    
    private fun displayResults(result: ObjectDetectionResult) {
        for (detection in result.detections()) {
            val bbox = detection.boundingBox()
            val category = detection.categories()[0]
            
            Log.d("Detection", "${category.categoryName()}: " +
                "${category.score()} at [${bbox.left}, ${bbox.top}, " +
                "${bbox.right}, ${bbox.bottom}]")
        }
    }
    
    override fun onDestroy() {
        super.onDestroy()
        objectDetector.close()
    }
}

// ─── OUTPUT (Logcat) ───
// Detection: person: 0.92 at [120, 80, 320, 450]
// Detection: car: 0.87 at [400, 200, 600, 380]
// Detection: dog: 0.74 at [50, 350, 200, 480]
// Frame rate: 30-60 FPS on modern Android phones
```

### iOS Integration (Swift)

```swift
// ─── Podfile ───
// pod 'MediaPipeTasksVision'

// ─── ObjectDetectionViewController.swift ───
import MediaPipeTasksVision
import AVFoundation
import UIKit

class ObjectDetectionViewController: UIViewController {
    
    private var objectDetector: ObjectDetector?
    private var cameraSession: AVCaptureSession?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupDetector()
        setupCamera()
    }
    
    private func setupDetector() {
        let baseOptions = BaseOptions()
        baseOptions.modelAssetPath = Bundle.main.path(
            forResource: "efficientdet_lite0", ofType: "tflite"
        )!
        baseOptions.delegate = .GPU  // Use Metal for acceleration
        
        let options = ObjectDetectorOptions()
        options.baseOptions = baseOptions
        options.runningMode = .liveStream
        options.maxResults = 5
        options.scoreThreshold = 0.5
        options.objectDetectorLiveStreamDelegate = self
        
        objectDetector = try? ObjectDetector(options: options)
    }
    
    // Camera frame callback
    func processFrame(_ pixelBuffer: CVPixelBuffer, timestamp: Int) {
        let mpImage = try? MPImage(pixelBuffer: pixelBuffer)
        try? objectDetector?.detectAsync(image: mpImage!, timestampInMilliseconds: timestamp)
    }
}

// Delegate for live stream results
extension ObjectDetectionViewController: ObjectDetectorLiveStreamDelegate {
    func objectDetector(
        _ objectDetector: ObjectDetector,
        didFinishDetection result: ObjectDetectorResult?,
        timestampInMilliseconds: Int,
        error: Error?
    ) {
        guard let result = result else { return }
        
        DispatchQueue.main.async {
            for detection in result.detections {
                let bbox = detection.boundingBox
                let category = detection.categories[0]
                print("\(category.categoryName ?? ""): \(category.score)")
            }
        }
    }
}

// ─── OUTPUT ───
// person: 0.94
// car: 0.89
// Runs at 30-60 FPS on iPhone 12+
// Uses Metal GPU for acceleration
```

### Mobile Performance Comparison

```
┌────────────────────────────────┬───────────────────┬───────────────────┐
│ Task                           │ Android (Pixel 7) │ iOS (iPhone 14)   │
├────────────────────────────────┼───────────────────┼───────────────────┤
│ Object Detection (Lite0)       │ 12ms (83 FPS)     │ 10ms (100 FPS)   │
│ Face Detection                 │ 3ms (333 FPS)     │ 2ms (500 FPS)    │
│ Hand Landmarks                 │ 15ms (66 FPS)     │ 12ms (83 FPS)    │
│ Pose Landmarks                 │ 18ms (55 FPS)     │ 15ms (66 FPS)    │
│ Selfie Segmentation            │ 8ms (125 FPS)     │ 6ms (166 FPS)    │
├────────────────────────────────┼───────────────────┼───────────────────┤
│ Hardware Acceleration          │ NNAPI / GPU       │ CoreML / Metal    │
│ Model format                   │ TFLite            │ TFLite (via MP)   │
└────────────────────────────────┴───────────────────┴───────────────────┘
```

---

## 14. MediaPipe vs Other Libraries (YOLO, OpenCV, Detectron2)

### Comprehensive Comparison

```
┌──────────────────┬───────────────┬──────────────┬──────────────┬─────────────┐
│ Feature          │ MediaPipe     │ YOLOv8       │ OpenCV DNN   │ Detectron2  │
├──────────────────┼───────────────┼──────────────┼──────────────┼─────────────┤
│ Primary Use      │ Mobile/Web/   │ General OD   │ Inference    │ Research    │
│                  │ Edge          │              │ engine       │             │
│ Model Size       │ 0.2-25 MB     │ 6-100+ MB   │ Varies       │ 170+ MB    │
│ CPU Performance  │ ⚡⚡⚡⚡⚡      │ ⚡⚡⚡        │ ⚡⚡⚡⚡       │ ⚡          │
│ GPU Performance  │ ⚡⚡⚡         │ ⚡⚡⚡⚡⚡      │ ⚡⚡⚡        │ ⚡⚡⚡⚡⚡     │
│ Accuracy (COCO)  │ 25-34% mAP   │ 37-55% mAP  │ Varies       │ 45-60% mAP │
│ Mobile Support   │ ✅ Native     │ ⚠️ Export    │ ⚠️ Limited   │ ❌          │
│ Web Support      │ ✅ Native     │ ❌           │ ❌           │ ❌          │
│ Face Detection   │ ✅ Built-in   │ ❌           │ ✅ Haar/DNN  │ ❌          │
│ Hand Tracking    │ ✅ Built-in   │ ❌           │ ❌           │ ❌          │
│ Pose Estimation  │ ✅ Built-in   │ ✅ (v8-pose) │ ❌           │ ❌          │
│ Segmentation     │ ✅ Built-in   │ ✅ (v8-seg)  │ ❌           │ ✅ Mask RCNN│
│ Custom Training  │ ✅ Model Maker│ ✅ Easy      │ ❌           │ ✅ Complex  │
│ Ease of Use      │ ⭐⭐⭐⭐⭐      │ ⭐⭐⭐⭐⭐      │ ⭐⭐⭐⭐       │ ⭐⭐⭐       │
│ Languages        │ Python, JS,   │ Python       │ Python, C++, │ Python     │
│                  │ Java, Swift,  │              │ Java         │            │
│                  │ C++           │              │              │            │
│ Best For         │ Real-time on  │ High-acc OD  │ Simple CV    │ Research,  │
│                  │ device        │ on GPU       │ tasks        │ SOTA       │
└──────────────────┴───────────────┴──────────────┴──────────────┴─────────────┘
```

### When to Choose MediaPipe

```
✅ CHOOSE MediaPipe WHEN:
  • Deploying to mobile (Android/iOS)
  • Running in web browser (JavaScript)
  • Need real-time on CPU (no GPU available)
  • Need face/hand/pose tracking specifically
  • Model size must be < 25 MB
  • Privacy matters (all processing on-device)
  • Quick prototyping with pre-built solutions
  • Building AR/VR experiences
  • Need cross-platform from single codebase

❌ DON'T CHOOSE MediaPipe WHEN:
  • Need highest possible accuracy (use Detectron2/DINO)
  • Processing on server with powerful GPU (use YOLO/DETR)
  • Need to detect 100+ custom classes (use YOLO + custom training)
  • Need instance segmentation with masks (use Mask R-CNN)
  • Need oriented bounding boxes (use YOLOv8-OBB)
  • Building research prototypes (use MMDetection)
```

### Decision Matrix

```
┌──────────────────────────────────────────────────────────────────┐
│                    WHICH LIBRARY TO USE?                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  "I need face/hand/pose tracking" ─────────────── MediaPipe      │
│  "I need to run on mobile phones" ─────────────── MediaPipe      │
│  "I need to run in a web browser" ─────────────── MediaPipe      │
│  "I need highest accuracy on GPU" ─────────────── YOLOv8/DETR    │
│  "I need real-time OD (any device)" ──────────── MediaPipe/YOLO  │
│  "I need instance segmentation"  ──────────────── Detectron2     │
│  "I need open-vocabulary detection" ──────────── GroundingDINO   │
│  "I need to deploy on edge (RPi)" ────────────── MediaPipe       │
│  "I'm doing research/ablations" ──────────────── MMDetection     │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 15. Real-World Use Cases & Company Deployments

### 1. Google Meet — Virtual Background

```
PROBLEM: Replace user's background in video calls
SOLUTION: MediaPipe Selfie Segmentation

Pipeline:
  Camera frame → Selfie Segmenter → Person mask → Composite with background
  
Specs:
  • Model: 200 KB selfie segmentation
  • Latency: < 5ms per frame
  • Resolution: 256×256 mask (upsampled)
  • Works on: Chrome, Android, iOS, Chromebook
  • No GPU needed — runs on any device
```

### 2. Snapchat/TikTok — AR Face Filters

```
PROBLEM: Track face to apply AR effects (dog ears, makeup, etc.)
SOLUTION: Face Mesh (478 landmarks) + Segmentation

Pipeline:
  Camera → Face Detection → Face Mesh → Track 478 points → 
  Overlay 3D effects aligned to landmarks → Render

Specs:
  • 478 landmarks tracked at 30+ FPS
  • 3D depth for proper occlusion
  • Expression tracking for reactive filters
  • Sub-millisecond face detection (BlazeFace)
```

### 3. Peloton / Nike Training — Exercise Form Detection

```
PROBLEM: Analyze exercise form and count reps without wearables
SOLUTION: MediaPipe Pose (33 landmarks)

Pipeline:
  Camera → Pose Detection → Calculate joint angles → 
  Classify exercise → Count reps → Score form quality

Specs:
  • 33 body landmarks in 3D
  • Angle calculation at joints
  • Real-time feedback on form
  • Works with phone camera (no special hardware)
  • 30+ FPS on mobile
  
Exercises tracked: squats, pushups, bicep curls, lunges, planks
```

### 4. Sign Language Translation

```
PROBLEM: Translate sign language to text/speech in real-time
SOLUTION: MediaPipe Holistic (face + hands + pose)

Pipeline:
  Camera → Holistic model → Extract 553 landmarks →
  Sequence model (LSTM/Transformer) → Sign classification → Text/Speech

Why MediaPipe?
  • Needs face (expressions matter in sign language)
  • Needs both hands (finger spelling, gestures)
  • Needs body (some signs use body movement)
  • Must be real-time for conversation
  • Must work on mobile (accessibility)
```

### 5. Smart Retail — Shelf Monitoring

```
PROBLEM: Monitor store shelves for out-of-stock items
SOLUTION: MediaPipe Object Detection (custom trained)

Pipeline:
  Edge camera → Object Detection (custom model) →
  Count products per shelf → Alert if below threshold

Why MediaPipe?
  • Runs on cheap edge devices (Raspberry Pi + camera)
  • Custom model trained with Model Maker (50 product images)
  • No cloud needed (privacy + no internet required)
  • Low power consumption (battery-operated cameras)
  • Small model size (< 5 MB)
```

### 6. Assistive Technology — Navigation for Visually Impaired

```
PROBLEM: Help blind users understand their surroundings
SOLUTION: MediaPipe Object Detection + Spatial Audio

Pipeline:
  Phone camera → Object Detection → 
  Determine object positions → Convert to spatial audio cues
  "Person ahead, 3 meters" / "Car approaching from left"

Why MediaPipe?
  • Runs entirely on phone (no internet needed)
  • Real-time (< 50ms latency for safety)
  • Low battery consumption
  • Works offline
  • Privacy (no video sent to cloud)
```

### 7. Surgical Training & Assistance

```
PROBLEM: Track surgeon's hands for training assessment
SOLUTION: MediaPipe Hand Landmarks + Pose

Pipeline:
  Overhead camera → Hand tracking (21 landmarks) →
  Track tool usage → Score precision → Provide feedback

Metrics tracked:
  • Hand steadiness (landmark jitter)
  • Tool grip type (finger positions)
  • Movement efficiency (path length)
  • Speed and timing
```

### 8. Wildlife Monitoring

```
PROBLEM: Count/identify animals in remote cameras
SOLUTION: MediaPipe Object Detection (custom model on edge)

Pipeline:
  Motion trigger → Capture image → Object Detection →
  Classify animal → Log count → Send alert if needed

Why MediaPipe?
  • Solar-powered edge device (minimal compute)
  • Model < 5 MB fits on microcontroller
  • No internet needed in remote areas
  • Battery-efficient inference
  • Custom model trained on local species
```

---

## 16. Performance Optimization & Deployment

### Optimization Techniques

```
┌──────────────────────────────────────────────────────────────────┐
│              MediaPipe Optimization Strategies                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. HARDWARE ACCELERATION                                         │
│     • Android: NNAPI delegate (uses NPU/DSP)                     │
│     • iOS: CoreML delegate (uses Neural Engine)                   │
│     • Desktop: GPU delegate (OpenGL/Metal)                        │
│     • Web: WebGL delegate                                         │
│                                                                    │
│  2. MODEL SELECTION                                               │
│     • Choose smallest model that meets accuracy needs             │
│     • EfficientDet-Lite0 (4.4 MB) for speed                     │
│     • EfficientDet-Lite2 (7.2 MB) for accuracy                  │
│     • Use float16 models (half size, ~same accuracy)             │
│                                                                    │
│  3. INPUT OPTIMIZATION                                            │
│     • Reduce input resolution (640→320 for 4× speedup)          │
│     • Skip frames (process every 2nd frame, track between)       │
│     • Use ROI cropping (don't process full frame)                │
│                                                                    │
│  4. PIPELINE OPTIMIZATION                                         │
│     • Use LIVE_STREAM mode (async, non-blocking)                 │
│     • Enable tracking (skip detection on tracked frames)          │
│     • Batch processing for offline workloads                     │
│                                                                    │
│  5. DEPLOYMENT FORMATS                                            │
│     • TFLite (default, mobile-optimized)                         │
│     • WASM (web browsers)                                         │
│     • AAR (Android Archive)                                       │
│     • Framework/XCFramework (iOS)                                 │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Code Example: Optimized Deployment Pipeline

```python
import mediapipe as mp
from mediapipe.tasks.python import vision
import cv2
import time
import threading
from collections import deque

class OptimizedDetector:
    """Production-ready MediaPipe detector with optimizations."""
    
    def __init__(self, model_path, max_results=5, score_threshold=0.5):
        self.latest_result = None
        self.fps_history = deque(maxlen=30)
        self.prev_time = time.time()
        
        options = vision.ObjectDetectorOptions(
            base_options=mp.tasks.BaseOptions(
                model_asset_path=model_path,
                # Use GPU delegate for acceleration (if available)
                # delegate=mp.tasks.BaseOptions.Delegate.GPU
            ),
            running_mode=vision.RunningMode.LIVE_STREAM,
            max_results=max_results,
            score_threshold=score_threshold,
            result_callback=self._result_callback
        )
        
        self.detector = vision.ObjectDetector.create_from_options(options)
    
    def _result_callback(self, result, image, timestamp_ms):
        """Async callback for detection results."""
        self.latest_result = result
    
    def process_frame(self, frame):
        """Process a single frame (non-blocking)."""
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        mp_image = mp.Image(image_format=mp.ImageFormat.SRGB, data=rgb)
        self.detector.detect_async(mp_image, int(time.time() * 1000))
        
        # Calculate FPS
        curr_time = time.time()
        self.fps_history.append(1 / (curr_time - self.prev_time))
        self.prev_time = curr_time
    
    @property
    def fps(self):
        """Average FPS over last 30 frames."""
        return sum(self.fps_history) / len(self.fps_history) if self.fps_history else 0
    
    def get_detections(self):
        """Get latest detection results."""
        if self.latest_result is None:
            return []
        return self.latest_result.detections
    
    def close(self):
        """Clean up resources."""
        self.detector.close()


# ─── Usage ───
detector = OptimizedDetector(
    model_path="efficientdet_lite0.tflite",
    max_results=5,
    score_threshold=0.5
)

cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)   # Lower resolution = faster
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 480)

frame_skip = 0  # Process every frame (set to 1 to skip every other)
frame_count = 0

try:
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
        
        # Frame skipping for performance
        frame_count += 1
        if frame_count % (frame_skip + 1) != 0:
            continue
        
        detector.process_frame(frame)
        
        # Draw results
        for det in detector.get_detections():
            bbox = det.bounding_box
            cat = det.categories[0]
            cv2.rectangle(frame, 
                         (bbox.origin_x, bbox.origin_y),
                         (bbox.origin_x + bbox.width, bbox.origin_y + bbox.height),
                         (0, 255, 0), 2)
            cv2.putText(frame, f"{cat.category_name}: {cat.score:.2f}",
                       (bbox.origin_x, bbox.origin_y - 10),
                       cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 255, 0), 2)
        
        cv2.putText(frame, f"FPS: {detector.fps:.1f}", (10, 30),
                   cv2.FONT_HERSHEY_SIMPLEX, 1, (0, 0, 255), 2)
        
        cv2.imshow("Optimized Detection", frame)
        if cv2.waitKey(1) & 0xFF == ord('q'):
            break

finally:
    detector.close()
    cap.release()
    cv2.destroyAllWindows()

# ─── OUTPUT ───
# Smooth real-time detection at 25-30+ FPS
# Clean bounding boxes with labels
# FPS counter showing actual performance
# Resource-efficient (async processing, frame skipping)
```

### Deployment Checklist

```
┌────────────────────────────────────────────────────────────────┐
│           MediaPipe Production Deployment Checklist              │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  □ Choose appropriate model size for target device              │
│  □ Enable hardware acceleration (GPU/NNAPI/CoreML)             │
│  □ Set appropriate score_threshold (avoid false positives)      │
│  □ Limit max_results to what UI can display                    │
│  □ Use LIVE_STREAM mode for camera input                       │
│  □ Handle model loading asynchronously (don't block UI)        │
│  □ Add error handling for camera permissions                   │
│  □ Test on lowest-spec target device                           │
│  □ Profile memory usage (especially on mobile)                 │
│  □ Implement graceful fallback if model fails to load          │
│  □ Consider frame skipping for very slow devices               │
│  □ Cache model files (don't re-download on each launch)        │
│  □ Handle lifecycle events (pause/resume on mobile)            │
│  □ Test in low-light conditions                                │
│  □ Validate with diverse user demographics                     │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

### Edge Device Deployment

```python
# ─── Running on Raspberry Pi ───

# Installation on Raspberry Pi (ARM):
# pip install mediapipe-rpi4  (or mediapipe for newer versions)

# Performance expectations (Raspberry Pi 4):
# • Object Detection: 5-8 FPS (EfficientDet-Lite0)
# • Face Detection: 15-20 FPS (BlazeFace)
# • Hand Landmarks: 4-6 FPS
# • Selfie Segmentation: 8-12 FPS

# Optimization for Raspberry Pi:
# 1. Use smallest model variant
# 2. Reduce input resolution to 320x240
# 3. Use frame skipping (process every 3rd frame)
# 4. Disable visualization for headless operation
# 5. Use threading for camera capture
```

---

## 17. Interview Questions & Quick Reference

### Common Interview Questions

| Question | Answer |
|----------|--------|
| "What is MediaPipe?" | Google's open-source framework for real-time, cross-platform ML pipelines for vision, audio, and text tasks. Known for on-device inference. |
| "How does BlazeFace achieve sub-ms detection?" | Uses tiny model (98KB), low-res input (128×128), depthwise convolutions, SSD architecture, and BlazeBlock (custom efficient residual block). |
| "How does MediaPipe track hands efficiently?" | Two-stage: Palm detection (first frame) → Hand landmark estimation (all frames). Reuses previous bbox for tracking, only re-detects when confidence drops. |
| "MediaPipe vs YOLO for mobile?" | MediaPipe: built for mobile (tiny models, CPU-optimized, native SDKs). YOLO: better accuracy, needs GPU/export. MediaPipe wins on mobile, YOLO wins on server. |
| "What are blendshapes?" | 52 coefficients (0-1) representing facial expressions (blink, smile, etc.). Used to drive 3D avatars or detect emotions. |
| "How does Holistic work?" | Runs Pose model first → Uses wrist landmarks for hand ROI, face landmarks for face ROI → Runs Hand + Face models on cropped regions. |
| "Explain MediaPipe's graph architecture" | DAG of Calculator nodes connected by Streams. Packets (timestamped data) flow through. Enables modular, reusable, parallel pipelines. |
| "How is MediaPipe deployed on web?" | Via WebAssembly (WASM) for compute + WebGL for GPU acceleration. Models loaded as .tflite, executed client-side. |
| "What's Model Maker?" | MediaPipe's transfer learning tool. Fine-tunes pre-trained models on custom data with minimal code. Exports TFLite for deployment. |
| "MediaPipe vs OpenCV for face detection?" | MediaPipe: ML-based (BlazeFace), 478 landmarks, tracks expressions. OpenCV: Haar cascades (old), limited keypoints. MediaPipe is far superior for modern apps. |

### Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│                 MEDIAPIPE CHEAT SHEET                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  INSTALLATION: pip install mediapipe                               │
│                                                                    │
│  RUNNING MODES:                                                    │
│    • IMAGE        → detector.detect(image)                        │
│    • VIDEO        → detector.detect_for_video(frame, ts)          │
│    • LIVE_STREAM  → detector.detect_async(frame, ts) + callback   │
│                                                                    │
│  KEY MODELS & SIZES:                                              │
│    • Face Detection:    98-196 KB  (BlazeFace)                    │
│    • Face Mesh:         2.6 MB     (478 landmarks)                │
│    • Hand Landmarks:    5.7 MB     (21 per hand)                  │
│    • Pose Landmarks:    3-6 MB     (33 body points)               │
│    • Object Detection:  4-25 MB    (EfficientDet-Lite)            │
│    • Selfie Segment:    200 KB     (person/background)            │
│                                                                    │
│  LANDMARKS COUNT:                                                  │
│    • Face:    478 (3D)                                             │
│    • Hand:    21 per hand (3D)                                    │
│    • Pose:    33 body (3D)                                        │
│    • Holistic: 553 total (face+2hands+pose)                       │
│                                                                    │
│  PLATFORMS: Python, JavaScript, Android, iOS, C++                  │
│                                                                    │
│  HARDWARE: CPU (default), GPU delegate, NNAPI, CoreML             │
│                                                                    │
│  TYPICAL PERFORMANCE:                                             │
│    • Mobile (CPU):  30-60+ FPS                                    │
│    • Desktop (CPU): 25-40 FPS                                     │
│    • Browser (WebGL): 20-30 FPS                                   │
│    • Raspberry Pi:  5-15 FPS                                      │
│                                                                    │
│  PAPERS:                                                           │
│    • BlazeFace (2019) — Sub-ms face detection                     │
│    • BlazePose (2020) — Real-time body pose                       │
│    • MediaPipe Hands (2020) — 21-point hand tracking             │
│    • MediaPipe Holistic (2020) — Unified tracking                │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### MediaPipe Model Download Links

```
All models available at:
https://developers.google.com/mediapipe/solutions/guide

Python model download:
  Object Detection: efficientdet_lite0.tflite (or lite1, lite2)
  Face Detection:   blaze_face_short_range.tflite
  Face Mesh:        face_landmarker_v2_with_blendshapes.task
  Hand Landmarks:   hand_landmarker.task
  Pose Landmarks:   pose_landmarker_full.task (or lite, heavy)
  Segmentation:     selfie_segmenter.tflite
  
Models auto-download with:
  wget https://storage.googleapis.com/mediapipe-models/{task}/{model}/float16/latest/{filename}
```

### Complete Comparison: When to Use What

```
SCENARIO → BEST CHOICE:

"Detect faces on iPhone"                  → MediaPipe Face Detection
"Count cars in parking lot (edge device)" → MediaPipe Object Detection
"Hand gesture control for smart TV"       → MediaPipe Hand Landmarks
"Fitness app with rep counting"           → MediaPipe Pose Estimation
"AR face filter in browser"               → MediaPipe Face Mesh (JS)
"Video call background blur"              → MediaPipe Selfie Segmentation
"High-accuracy detection on server"       → YOLOv8 / DETR (NOT MediaPipe)
"Research with 300+ models"               → MMDetection (NOT MediaPipe)
"Detect any object from text prompt"      → GroundingDINO (NOT MediaPipe)
"Instance segmentation masks"             → Detectron2 / SAM (NOT MediaPipe)
```

---

## Summary & Key Takeaways

```
┌──────────────────────────────────────────────────────────────────┐
│              KEY TAKEAWAYS FROM THIS CHAPTER                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. MediaPipe = Google's framework for REAL-TIME, ON-DEVICE ML    │
│     (Not for highest accuracy — for fastest deployment)           │
│                                                                    │
│  2. Strengths: Tiny models, cross-platform, pre-built solutions  │
│     CPU-optimized, privacy-preserving, production-ready           │
│                                                                    │
│  3. Best for: Mobile apps, web apps, edge devices, AR/VR         │
│     face/hand/pose tracking, background segmentation              │
│                                                                    │
│  4. NOT for: Server-side high-accuracy detection, research,      │
│     open-vocabulary detection, large-scale instance segmentation  │
│                                                                    │
│  5. Key models: BlazeFace (face), BlazePose (body),             │
│     Hand Landmarker (hands), EfficientDet-Lite (objects)         │
│                                                                    │
│  6. Model Maker: Fine-tune on custom data → Export TFLite        │
│     Minimum ~50-100 images per class                             │
│                                                                    │
│  7. Performance: 30-60+ FPS on mobile CPU without GPU!           │
│     Face detection in < 1ms, models as small as 98 KB            │
│                                                                    │
│  8. Ecosystem: Python + JS + Android + iOS + C++                 │
│     All from one framework, consistent API                       │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## Next Steps

```
After mastering MediaPipe:
  → Compare with YOLO for your use case (Chapter 04)
  → Learn deployment optimizations (Chapter 12)  
  → See the full comparison guide (Chapter 13)
  → Try GroundingDINO for open-set detection (Chapter 10)
```
