# 📘 Chapter 12: Deployment & Optimization — Making Models Production-Ready

> **Goal**: Learn how to take a trained detection model and make it FAST, SMALL, and DEPLOYABLE — on servers, edge devices, mobile phones, and browsers. This is the skill that separates prototypes from products.

---

## 📑 Table of Contents

1. [Why Deployment Matters](#1-why-deployment-matters)
2. [Model Export Formats (ONNX, TensorRT, TFLite, CoreML)](#2-model-export-formats-onnx-tensorrt-tflite-coreml)
3. [ONNX — The Universal Format](#3-onnx--the-universal-format)
4. [TensorRT — Maximum GPU Speed (NVIDIA)](#4-tensorrt--maximum-gpu-speed-nvidia)
5. [TFLite — Mobile & Edge (Android, Coral, RPi)](#5-tflite--mobile--edge-android-coral-rpi)
6. [CoreML — Apple Devices (iOS, macOS)](#6-coreml--apple-devices-ios-macos)
7. [OpenVINO — Intel Hardware Optimization](#7-openvino--intel-hardware-optimization)
8. [Quantization — Making Models Smaller & Faster](#8-quantization--making-models-smaller--faster)
9. [Pruning & Knowledge Distillation](#9-pruning--knowledge-distillation)
10. [Edge Devices (Jetson, Coral, RPi, Phones)](#10-edge-devices-jetson-coral-rpi-phones)
11. [Serving at Scale (Triton, TF Serving, FastAPI)](#11-serving-at-scale-triton-tf-serving-fastapi)
12. [Optimization Checklist & Benchmarks](#12-optimization-checklist--benchmarks)

---

## 1. Why Deployment Matters

### The Reality Gap

```
IN THE LAB:                          IN PRODUCTION:
• PyTorch on A100 GPU               • Jetson Nano (20× slower GPU)
• 1 image at a time                  • 30 FPS continuous video
• "It works on my machine"           • Must work 24/7 for years
• mAP is 52%!                        • Latency must be <30ms
• 68MB model                         • Device has 2GB RAM total
• Python script                      • No Python, C++ embedded

THE GAP:
  Training model = 20% of the work
  Deploying model = 80% of the work
```

### The Deployment Pipeline

```
TRAINED MODEL (.pt / .pth)
       │
       ▼
┌──────────────┐
│ OPTIMIZE     │  Quantize, prune, distill
│              │  → Smaller + faster
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ EXPORT       │  Convert to target format
│              │  ONNX, TensorRT, TFLite, CoreML
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ DEPLOY       │  Serve on target hardware
│              │  Server, edge, mobile, browser
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ MONITOR      │  Track latency, accuracy, drift
│              │  Alert on degradation
└──────────────┘
```

---

## 2. Model Export Formats (ONNX, TensorRT, TFLite, CoreML)

### Format Comparison

| Format | Target | Speed Boost | Creator | Ecosystem |
|--------|--------|-------------|---------|-----------|
| **ONNX** | Universal | 1.5-2× | Microsoft + Partners | Cross-platform |
| **TensorRT** | NVIDIA GPUs | 3-5× | NVIDIA | NVIDIA only |
| **TFLite** | Mobile/Edge | 2-4× | Google | Android, Coral, RPi |
| **CoreML** | Apple devices | 2-3× | Apple | iOS, macOS |
| **OpenVINO** | Intel HW | 2-3× | Intel | Intel CPUs/GPUs/VPUs |
| **NCNN** | Mobile (C++) | 2-3× | Tencent | Android, iOS |
| **TorchScript** | PyTorch | 1.2× | Meta | PyTorch ecosystem |

### Which Format for Which Target?

```
┌──────────────────────────────────────────────────────────┐
│         WHERE ARE YOU DEPLOYING?                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  NVIDIA GPU (Server/Cloud)    → TensorRT (.engine)       │
│  NVIDIA Jetson (Edge)         → TensorRT (.engine)       │
│  Any GPU/CPU (Flexible)       → ONNX (.onnx)            │
│  Android phone                → TFLite (.tflite)         │
│  Google Coral / Edge TPU      → TFLite INT8 (.tflite)    │
│  iPhone / iPad                → CoreML (.mlpackage)      │
│  Intel CPU/iGPU               → OpenVINO (.xml/.bin)     │
│  Raspberry Pi                 → TFLite or NCNN           │
│  Web Browser                  → ONNX.js or TF.js         │
│  C++ embedded                 → NCNN or TensorRT         │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 3. ONNX — The Universal Format

### 💡 "Convert Once, Run Anywhere"

> **ONNX** (Open Neural Network Exchange) is the universal model format. Export from PyTorch/TF → ONNX → run on any platform with ONNX Runtime.

### 📝 Code: Export to ONNX

```python
# ── Method 1: Ultralytics (Easiest) ──
from ultralytics import YOLO

model = YOLO("yolov8m.pt")
model.export(format="onnx", dynamic=True, simplify=True)
# Creates: yolov8m.onnx

# ── Method 2: PyTorch Direct ──
import torch

model = load_your_model()
model.eval()

dummy_input = torch.randn(1, 3, 640, 640).cuda()
torch.onnx.export(
    model,
    dummy_input,
    "model.onnx",
    opset_version=17,
    input_names=['images'],
    output_names=['output'],
    dynamic_axes={
        'images': {0: 'batch', 2: 'height', 3: 'width'},
        'output': {0: 'batch'}
    }
)

# ── Verify ONNX model ──
import onnx
model = onnx.load("model.onnx")
onnx.checker.check_model(model)
print("✅ ONNX model is valid!")

# ── Simplify (reduce redundant ops) ──
import onnxsim
model_simplified, check = onnxsim.simplify(model)
onnx.save(model_simplified, "model_simplified.onnx")
```

### 📝 Code: Run with ONNX Runtime

```python
import onnxruntime as ort
import numpy as np
import cv2

# ── Setup ──
# CPU:
session = ort.InferenceSession("yolov8m.onnx", providers=['CPUExecutionProvider'])
# GPU:
session = ort.InferenceSession("yolov8m.onnx", providers=['CUDAExecutionProvider'])
# TensorRT (via ONNX Runtime):
session = ort.InferenceSession("yolov8m.onnx", providers=['TensorrtExecutionProvider'])

# ── Inference ──
image = cv2.imread("test.jpg")
blob = cv2.dnn.blobFromImage(image, 1/255.0, (640, 640), swapRB=True)

outputs = session.run(None, {"images": blob})

# ── Post-process ──
# outputs[0] shape depends on model format
predictions = outputs[0]  # (1, num_boxes, 85) for YOLO

# ── Benchmark ──
import time
times = []
for _ in range(100):
    start = time.time()
    session.run(None, {"images": blob})
    times.append(time.time() - start)

print(f"ONNX Runtime: {np.mean(times)*1000:.1f}ms avg ({1/np.mean(times):.0f} FPS)")
```

### ONNX Runtime Speed Comparison

| Backend | Device | YOLOv8m Latency | FPS |
|---------|--------|----------------|-----|
| PyTorch | V100 GPU | 8.5ms | 117 |
| ONNX Runtime (CUDA) | V100 GPU | 5.2ms | 192 |
| ONNX Runtime (TRT) | V100 GPU | 3.1ms | 322 |
| ONNX Runtime (CPU) | i7-12700 | 45ms | 22 |
| ONNX Runtime (CPU) | Apple M2 | 28ms | 35 |

---

## 4. TensorRT — Maximum GPU Speed (NVIDIA)

### 💡 "The Fastest Inference on NVIDIA GPUs"

> **TensorRT** is NVIDIA's inference optimizer. It analyzes your model, applies GPU-specific optimizations, and generates an ultra-fast engine. Typical speedup: 3-5× over PyTorch.

### What TensorRT Does Under the Hood

```
YOUR MODEL                          TENSORRT ENGINE
┌────────────────┐                  ┌────────────────┐
│ Conv → BN → ReLU│                 │ FusedConvBNReLU │  ← Layer fusion!
│ Conv → BN → ReLU│  ──────────▶   │ FusedConvBNReLU │
│ Concat          │                 │ OptConcat       │  ← Optimized ops
│ Upsample        │                 │ FP16/INT8 ops  │  ← Precision reduction
│ ...             │                 │ Custom kernels │  ← GPU-specific kernels
└────────────────┘                  └────────────────┘

Optimizations applied:
  1. LAYER FUSION: Conv + BatchNorm + Activation → single kernel
  2. PRECISION: FP32 → FP16 (2× faster, <1% accuracy loss)
  3. KERNEL AUTO-TUNING: Tests multiple GPU kernels, picks fastest
  4. MEMORY: Optimizes tensor memory layout for GPU architecture
  5. GRAPH: Removes redundant operations
```

### 📝 Code: Export to TensorRT

```python
# ── Method 1: Ultralytics (Easiest!) ──
from ultralytics import YOLO

model = YOLO("yolov8m.pt")

# FP16 (recommended — fast + accurate)
model.export(format="engine", half=True, imgsz=640)
# Creates: yolov8m.engine

# INT8 (fastest, slight accuracy drop — needs calibration data)
model.export(format="engine", int8=True, data="coco128.yaml", imgsz=640)

# ── Use TensorRT model ──
trt_model = YOLO("yolov8m.engine")
results = trt_model("image.jpg")  # Same API, 3-5× faster!
```

```python
# ── Method 2: Direct TensorRT API ──
import tensorrt as trt

def build_engine(onnx_path, engine_path, fp16=True):
    logger = trt.Logger(trt.Logger.WARNING)
    builder = trt.Builder(logger)
    network = builder.create_network(
        1 << int(trt.NetworkDefinitionCreationFlag.EXPLICIT_BATCH)
    )
    parser = trt.OnnxParser(network, logger)
    
    # Parse ONNX
    with open(onnx_path, 'rb') as f:
        parser.parse(f.read())
    
    # Configure
    config = builder.create_builder_config()
    config.set_memory_pool_limit(trt.MemoryPoolType.WORKSPACE, 1 << 30)  # 1GB
    
    if fp16:
        config.set_flag(trt.BuilderFlag.FP16)
    
    # Build engine (takes 2-10 minutes!)
    engine = builder.build_serialized_network(network, config)
    
    # Save
    with open(engine_path, 'wb') as f:
        f.write(engine)
    
    print(f"✅ TensorRT engine saved: {engine_path}")

build_engine("yolov8m.onnx", "yolov8m.engine", fp16=True)
```

### TensorRT Speed Benchmarks

| Model | Precision | GPU | Latency | FPS | Speedup vs PyTorch |
|-------|-----------|-----|---------|-----|-------------------|
| YOLOv8n | FP16 | T4 | 1.0ms | 1000 | 4.2× |
| YOLOv8s | FP16 | T4 | 1.4ms | 714 | 3.8× |
| YOLOv8m | FP16 | T4 | 2.8ms | 357 | 3.5× |
| YOLOv8l | FP16 | T4 | 4.1ms | 243 | 3.2× |
| YOLOv8m | INT8 | T4 | 1.9ms | 526 | 5.1× |
| YOLOv8m | FP16 | A100 | 1.2ms | 833 | 4.0× |
| YOLOv8m | FP16 | Jetson Orin | 8.5ms | 117 | 3.0× |

> ⚠️ TensorRT engines are GPU-SPECIFIC. An engine built on T4 won't work on A100! Rebuild for each target GPU.

---

## 5. TFLite — Mobile & Edge (Android, Coral, RPi)

### 💡 For Android, Raspberry Pi, Coral, and Microcontrollers

### 📝 Code: Export to TFLite

```python
# ── Method 1: From Ultralytics ──
from ultralytics import YOLO

model = YOLO("yolov8n.pt")  # Use nano for mobile!
model.export(format="tflite", int8=True, data="coco128.yaml")
# Creates: yolov8n_saved_model/yolov8n_int8.tflite

# ── Method 2: From TensorFlow SavedModel ──
import tensorflow as tf

converter = tf.lite.TFLiteConverter.from_saved_model("saved_model/")

# FP16 quantization (good balance)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.target_spec.supported_types = [tf.float16]

# Full INT8 quantization (smallest + fastest)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
def representative_dataset():
    """Calibration dataset for INT8."""
    for _ in range(100):
        data = np.random.rand(1, 320, 320, 3).astype(np.float32)
        yield [data]
converter.representative_dataset = representative_dataset
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type = tf.uint8
converter.inference_output_type = tf.uint8

tflite_model = converter.convert()
with open("model_int8.tflite", "wb") as f:
    f.write(tflite_model)

print(f"Model size: {len(tflite_model) / 1024 / 1024:.1f} MB")
```

### 📝 Code: Run TFLite on Android (Python equivalent)

```python
import numpy as np
import tflite_runtime.interpreter as tflite

# Load model
interpreter = tflite.Interpreter(model_path="model.tflite")
interpreter.allocate_tensors()

# Get I/O details
input_details = interpreter.get_input_details()
output_details = interpreter.get_output_details()

# Prepare input
image = preprocess_image("test.jpg", size=(320, 320))
input_data = np.expand_dims(image, axis=0).astype(np.float32)

# Run inference
interpreter.set_tensor(input_details[0]['index'], input_data)
interpreter.invoke()

# Get output
output = interpreter.get_tensor(output_details[0]['index'])
print(f"Output shape: {output.shape}")
```

### Model Size Comparison

| Model | FP32 | FP16 | INT8 | Reduction |
|-------|------|------|------|-----------|
| YOLOv8n | 6.2MB | 3.3MB | 1.7MB | 3.6× smaller |
| YOLOv8s | 22.5MB | 11.5MB | 6.0MB | 3.7× smaller |
| YOLOv8m | 52MB | 26MB | 13.5MB | 3.8× smaller |
| EfficientDet-D0 | 15.6MB | 8MB | 4.2MB | 3.7× smaller |

---

## 6. CoreML — Apple Devices (iOS, macOS)

### 📝 Code: Export to CoreML

```python
# ── Ultralytics ──
from ultralytics import YOLO

model = YOLO("yolov8n.pt")
model.export(format="coreml", nms=True)  # Includes NMS in model!
# Creates: yolov8n.mlpackage

# ── From PyTorch (coremltools) ──
import coremltools as ct
import torch

model = load_model()
model.eval()
traced = torch.jit.trace(model, torch.randn(1, 3, 640, 640))

mlmodel = ct.convert(
    traced,
    inputs=[ct.ImageType(shape=(1, 3, 640, 640), scale=1/255.0)],
    minimum_deployment_target=ct.target.iOS16
)
mlmodel.save("model.mlpackage")
```

### iOS Performance

| Model | iPhone 15 Pro (A17) | iPhone 13 (A15) | iPad M2 |
|-------|-------------------|----------------|---------|
| YOLOv8n | 4ms (250 FPS) | 6ms (166 FPS) | 3ms |
| YOLOv8s | 8ms (125 FPS) | 12ms (83 FPS) | 6ms |
| YOLOv8m | 18ms (55 FPS) | 28ms (35 FPS) | 14ms |

---

## 7. OpenVINO — Intel Hardware Optimization

### 💡 For Intel CPUs, iGPUs, and VPUs

```python
# ── Export ──
from ultralytics import YOLO
model = YOLO("yolov8n.pt")
model.export(format="openvino", half=True)

# ── Run with OpenVINO ──
from openvino.runtime import Core
import numpy as np

core = Core()
model = core.read_model("yolov8n_openvino_model/yolov8n.xml")
compiled = core.compile_model(model, "CPU")  # or "GPU" for Intel iGPU

input_data = np.random.rand(1, 3, 640, 640).astype(np.float32)
result = compiled([input_data])

# Speed on Intel i7-12700: ~18ms (55 FPS) for YOLOv8n
# Speed on Intel iGPU: ~12ms (83 FPS)
```

---

## 8. Quantization — Making Models Smaller & Faster

### 💡 "Reduce Precision = Massive Speedup"

```
WHAT IS QUANTIZATION?

FP32 (Standard):  32 bits per weight  → Full precision, baseline
FP16 (Half):      16 bits per weight  → 2× smaller, ~0% accuracy loss
INT8 (Integer):    8 bits per weight  → 4× smaller, 1-3% accuracy loss
INT4 (Extreme):    4 bits per weight  → 8× smaller, 3-5% accuracy loss

Visual:
FP32: ████████████████████████████████ (32 bits per number)
FP16: ████████████████                 (16 bits — 2× smaller)
INT8: ████████                         (8 bits — 4× smaller!)
INT4: ████                             (4 bits — 8× smaller!)
```

### Types of Quantization

```
POST-TRAINING QUANTIZATION (PTQ):
  Train normally (FP32) → Quantize after training
  ✅ Easy (no retraining)
  ⚠️ Some accuracy loss (1-3%)
  Best for: Most cases

QUANTIZATION-AWARE TRAINING (QAT):
  Simulate quantization DURING training
  ✅ Minimal accuracy loss (<1%)
  ⚠️ Requires retraining
  Best for: When PTQ accuracy is insufficient
```

### 📝 Code: Quantization Examples

```python
# ── FP16 (Simplest, almost free) ──
from ultralytics import YOLO
model = YOLO("yolov8m.pt")
model.export(format="engine", half=True)  # FP16 TensorRT
# Speed: ~2× faster, accuracy: ~0% loss

# ── INT8 Post-Training (Needs calibration data) ──
model.export(format="engine", int8=True, data="coco128.yaml")
# Speed: ~3-4× faster, accuracy: 1-2% loss
# data="coco128.yaml" provides calibration images

# ── INT8 TFLite ──
model.export(format="tflite", int8=True, data="coco128.yaml")
```

```python
# ── PyTorch Dynamic Quantization ──
import torch

model = load_model()
model.eval()

# Dynamic quantization (weights quantized, activations at runtime)
quantized_model = torch.quantization.quantize_dynamic(
    model,
    {torch.nn.Linear, torch.nn.Conv2d},
    dtype=torch.qint8
)

# Compare sizes
original_size = os.path.getsize("model.pt") / 1e6
torch.save(quantized_model.state_dict(), "model_quant.pt")
quant_size = os.path.getsize("model_quant.pt") / 1e6
print(f"Original: {original_size:.1f}MB → Quantized: {quant_size:.1f}MB")
```

### Quantization Impact (YOLOv8m on T4 GPU)

| Precision | Model Size | Latency | FPS | mAP (COCO) | Accuracy Drop |
|-----------|-----------|---------|-----|-----------|---------------|
| FP32 | 52MB | 9.8ms | 102 | 50.2% | Baseline |
| FP16 | 26MB | 2.8ms | 357 | 50.1% | -0.1% |
| INT8 | 13.5MB | 1.9ms | 526 | 49.0% | -1.2% |
| INT4 | 7MB | 1.4ms | 714 | 47.5% | -2.7% |

> 💡 **Recommendation**: Always use FP16 (free speedup). Use INT8 when you need maximum throughput and can tolerate ~1% accuracy drop.

---

## 9. Pruning & Knowledge Distillation

### Pruning: Remove Unimportant Weights

```
CONCEPT:
  Most neural network weights are near-zero (unimportant)
  Remove them → smaller model, similar accuracy!

BEFORE PRUNING:                 AFTER PRUNING (50%):
  ████████████████████           ██░░████░░██████░░██
  ████████████████████           ░░██████████░░██░░██
  ████████████████████    →      ████░░██████████░░░░
  ████████████████████           ██████░░████░░██████
  
  100% of weights                50% of weights (zeros removed)
  100% compute                   ~60% compute (with sparse ops)
```

### 📝 Code: Pruning with PyTorch

```python
import torch
import torch.nn.utils.prune as prune

model = load_model()

# Prune 30% of weights in all Conv2d layers
for name, module in model.named_modules():
    if isinstance(module, torch.nn.Conv2d):
        prune.l1_unstructured(module, name='weight', amount=0.3)

# Check sparsity
total = 0
pruned = 0
for name, module in model.named_modules():
    if isinstance(module, torch.nn.Conv2d):
        total += module.weight.nelement()
        pruned += (module.weight == 0).sum().item()

print(f"Sparsity: {pruned/total*100:.1f}%")  # ~30%

# Make pruning permanent (actually remove zeros)
for name, module in model.named_modules():
    if isinstance(module, torch.nn.Conv2d):
        prune.remove(module, 'weight')

# Fine-tune for a few epochs to recover accuracy
# optimizer = torch.optim.SGD(model.parameters(), lr=0.001)
# ... training loop ...
```

### Knowledge Distillation: Learn from a Big Model

```
CONCEPT:
  Large model (teacher) → teaches → Small model (student)
  Student learns to MIMIC the teacher's outputs
  → Student achieves teacher-like accuracy at 5-10× less compute!

┌────────────────┐
│ TEACHER MODEL  │  Large, slow, accurate (YOLOv8x)
│ (frozen)       │
└───────┬────────┘
        │ "soft labels" (probability distributions)
        ▼
┌────────────────┐
│ STUDENT MODEL  │  Small, fast (YOLOv8n)
│ (training)     │  Learns from BOTH:
│                │  • Ground truth labels (hard)
│                │  • Teacher's outputs (soft)
└────────────────┘

Why soft labels help:
  Hard label: "cat" → [0, 0, 1, 0, 0, ...]  (just the answer)
  Soft label: [0.01, 0.05, 0.85, 0.05, 0.02, ...]  (relationships!)
  
  Soft labels contain MORE information:
  "This cat looks a bit like a tiger (0.05)"
  "Definitely not a car (0.001)"
  → Student learns richer representations!
```

### 📝 Code: Knowledge Distillation

```python
import torch
import torch.nn.functional as F

def distillation_loss(student_logits, teacher_logits, labels, 
                      temperature=4.0, alpha=0.5):
    """
    Combined loss for knowledge distillation.
    
    Args:
        student_logits: Student model predictions
        teacher_logits: Teacher model predictions (no grad!)
        labels: Ground truth labels
        temperature: Softens probability distributions (higher = softer)
        alpha: Balance between soft and hard loss
    """
    # Soft loss: student matches teacher's soft predictions
    soft_student = F.log_softmax(student_logits / temperature, dim=1)
    soft_teacher = F.softmax(teacher_logits / temperature, dim=1)
    soft_loss = F.kl_div(soft_student, soft_teacher, reduction='batchmean')
    soft_loss *= temperature ** 2  # Scale by T²
    
    # Hard loss: student matches ground truth
    hard_loss = F.cross_entropy(student_logits, labels)
    
    # Combined
    return alpha * soft_loss + (1 - alpha) * hard_loss

# Training loop
teacher_model.eval()  # Frozen!
student_model.train()

for images, labels in dataloader:
    # Teacher forward (no gradients)
    with torch.no_grad():
        teacher_out = teacher_model(images)
    
    # Student forward
    student_out = student_model(images)
    
    # Distillation loss
    loss = distillation_loss(student_out, teacher_out, labels)
    
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
```

---

## 10. Edge Devices (Jetson, Coral, RPi, Phones)

### Device Comparison

| Device | GPU/NPU | Power | Price | Best Model | FPS (YOLOv8n) |
|--------|---------|-------|-------|-----------|---------------|
| **Jetson Orin Nano** | 1024 CUDA | 7-15W | $199 | TensorRT FP16 | 60+ |
| **Jetson Orin NX** | 1024 CUDA | 10-25W | $399 | TensorRT FP16 | 100+ |
| **Jetson AGX Orin** | 2048 CUDA | 15-60W | $999 | TensorRT FP16 | 200+ |
| **Google Coral** | Edge TPU | 2W | $60 | TFLite INT8 | 30-40 |
| **Coral Dev Board** | Edge TPU | 5W | $130 | TFLite INT8 | 40-50 |
| **Raspberry Pi 5** | CPU only | 5W | $80 | TFLite/NCNN | 5-8 |
| **RPi + Coral USB** | Edge TPU | 7W | $140 | TFLite INT8 | 25-35 |
| **iPhone 15 Pro** | ANE + GPU | - | $999 | CoreML | 200+ |
| **Samsung S24** | NPU + GPU | - | $799 | TFLite/NCNN | 80+ |
| **Hailo-8** | AI Accelerator | 2.5W | $100 | HailoRT | 100+ |

### 📝 Code: Deploy on Jetson

```python
# On NVIDIA Jetson (Orin, Xavier, Nano)
from ultralytics import YOLO

# Step 1: Export with TensorRT (do this ON the Jetson!)
model = YOLO("yolov8n.pt")
model.export(format="engine", half=True, imgsz=640)

# Step 2: Run real-time detection
model = YOLO("yolov8n.engine")

import cv2
cap = cv2.VideoCapture(0)  # Camera

while True:
    ret, frame = cap.read()
    results = model(frame, conf=0.5)
    
    annotated = results[0].plot()
    cv2.imshow("Jetson Detection", annotated)
    
    if cv2.waitKey(1) == ord('q'):
        break
```

### 📝 Code: Deploy on Raspberry Pi

```bash
# On Raspberry Pi:
pip install ultralytics
pip install tflite-runtime  # Lighter than full TF

# Export on desktop (or Pi if patient):
# yolo export model=yolov8n.pt format=tflite int8=True
```

```python
# Run on RPi
from ultralytics import YOLO

model = YOLO("yolov8n_int8.tflite")
results = model("test.jpg")

# With PiCamera2
from picamera2 import Picamera2
camera = Picamera2()
camera.start()

while True:
    frame = camera.capture_array()
    results = model(frame, conf=0.5)
    # Process results...
```

### Choosing the Right Edge Device

```
Need real-time (30+ FPS) + reasonable cost?
  → Jetson Orin Nano ($199) or Coral Dev Board ($130)

Need ultra-low power (<5W)?
  → Google Coral USB ($60) or Hailo-8

Need to process 4K video?
  → Jetson AGX Orin ($999)

Building a consumer product (mobile app)?
  → CoreML (iOS) or TFLite (Android) — use phone's own NPU

Very limited budget?
  → Raspberry Pi 5 + Coral USB ($140 total)

Industrial deployment (ruggedized)?
  → Jetson Orin (industrial variants) or Hailo
```

---

## 11. Serving at Scale (Triton, TF Serving, FastAPI)

### 💡 Serving = Making Your Model Available as an API

### Option 1: NVIDIA Triton Inference Server (Production-Grade)

```bash
# Docker-based deployment (supports ALL formats!)
docker run --gpus all -p 8000:8000 -p 8001:8001 \
    -v /models:/models \
    nvcr.io/nvidia/tritonserver:24.01-py3 \
    tritonserver --model-repository=/models

# Model repository structure:
models/
└── yolov8/
    ├── config.pbtxt
    └── 1/
        └── model.onnx  # or model.plan (TensorRT)
```

```python
# config.pbtxt
"""
name: "yolov8"
platform: "onnxruntime_onnx"
max_batch_size: 32
input [
  { name: "images" data_type: TYPE_FP32 dims: [3, 640, 640] }
]
output [
  { name: "output" data_type: TYPE_FP32 dims: [-1, 85] }
]
instance_group [
  { count: 2  kind: KIND_GPU  gpus: [0] }
]
dynamic_batching { max_queue_delay_microseconds: 100 }
"""
```

```python
# Client code
import tritonclient.http as httpclient
import numpy as np

client = httpclient.InferenceServerClient(url="localhost:8000")

# Prepare input
image = preprocess("test.jpg")
inputs = [httpclient.InferInput("images", image.shape, "FP32")]
inputs[0].set_data_from_numpy(image)

# Infer
results = client.infer("yolov8", inputs)
output = results.as_numpy("output")
```

### Triton Features

```
✅ Supports: ONNX, TensorRT, TF, PyTorch, OpenVINO, custom
✅ Dynamic batching (automatically groups requests)
✅ Model versioning (A/B testing, rollback)
✅ Multi-model serving (multiple models on one server)
✅ Multi-GPU load balancing
✅ Health checks, metrics (Prometheus)
✅ gRPC + HTTP/REST APIs
✅ Concurrent model execution
✅ Model ensembling (chain models)
```

### Option 2: FastAPI (Simple, Flexible)

```python
from fastapi import FastAPI, UploadFile, File
from ultralytics import YOLO
import cv2
import numpy as np

app = FastAPI()
model = YOLO("yolov8m.engine")  # TensorRT for speed

@app.post("/detect")
async def detect(file: UploadFile = File(...)):
    contents = await file.read()
    nparr = np.frombuffer(contents, np.uint8)
    image = cv2.imdecode(nparr, cv2.IMREAD_COLOR)
    
    results = model(image, conf=0.5)
    
    detections = []
    for box in results[0].boxes:
        detections.append({
            "class": model.names[int(box.cls)],
            "confidence": round(float(box.conf), 3),
            "bbox": box.xyxy[0].tolist()
        })
    
    return {"detections": detections, "count": len(detections)}

# Run: uvicorn app:app --host 0.0.0.0 --port 8000 --workers 4
```

### Option 3: TF Serving (Google Ecosystem)

```bash
# For TensorFlow models
docker run -p 8501:8501 \
    --mount type=bind,source=/models/detector,target=/models/detector \
    -e MODEL_NAME=detector \
    tensorflow/serving
```

### Serving Comparison

| Feature | Triton | FastAPI | TF Serving |
|---------|--------|---------|------------|
| **Complexity** | High | Low | Medium |
| **Performance** | Best | Good | Good |
| **Dynamic batching** | ✅ | Manual | ✅ |
| **Multi-format** | ✅ All | PyTorch/ONNX | TF only |
| **GPU support** | ✅ | ✅ | ✅ |
| **Multi-model** | ✅ | Manual | ✅ |
| **Best for** | Production at scale | Prototypes, small scale | TF ecosystem |

---

## 12. Optimization Checklist & Benchmarks

### 🔥 The Complete Optimization Checklist

```
□ STEP 1: Choose right MODEL SIZE
   YOLOv8n (mobile) → s (edge) → m (server) → l/x (accuracy)

□ STEP 2: Choose right INPUT SIZE
   320 (fast) → 640 (standard) → 1280 (high-accuracy)
   Smaller input = faster inference (quadratic scaling!)

□ STEP 3: Export to OPTIMIZED FORMAT
   NVIDIA GPU → TensorRT FP16 (3-5× speedup)
   CPU → ONNX Runtime (1.5-2× speedup)
   Mobile → TFLite INT8 or CoreML
   Intel → OpenVINO

□ STEP 4: QUANTIZE
   Always use FP16 (free speedup)
   Use INT8 if need more speed (1-2% accuracy cost)

□ STEP 5: OPTIMIZE POST-PROCESSING
   Tune confidence threshold (higher = fewer boxes = faster NMS)
   Tune NMS IoU threshold
   Limit max detections per image

□ STEP 6: BATCH where possible
   Process multiple images at once (GPU utilization)
   Dynamic batching in Triton

□ STEP 7: PROFILE & MEASURE
   Always benchmark on TARGET hardware
   Measure end-to-end (not just model inference)
   Include pre-processing + post-processing time
```

### Speed Impact of Each Optimization

```
Starting point: YOLOv8m, PyTorch, FP32, 640×640, V100 GPU

Optimization                    Latency    Cumulative Speedup
─────────────────────────────────────────────────────────────
Baseline (PyTorch FP32)         9.8ms      1.0×
+ Export to ONNX                6.2ms      1.6×
+ TensorRT conversion           3.5ms      2.8×
+ FP16 precision                2.8ms      3.5×
+ INT8 quantization             1.9ms      5.2×
+ Reduce input 640→480          1.2ms      8.2×
+ Use YOLOv8s instead of m     0.9ms      10.9×
+ Use YOLOv8n                   0.5ms      19.6×

From 9.8ms → 0.5ms = 20× speedup!
From 102 FPS → 2000 FPS!
```

### End-to-End Latency Breakdown

```
TYPICAL DETECTION PIPELINE LATENCY:

┌──────────────────────────────────────────────────┐
│ Component          │ Time    │ % of Total        │
├────────────────────┼─────────┼───────────────────┤
│ Image capture      │ 1-5ms   │ 5-15%            │
│ Pre-processing     │ 1-3ms   │ 5-10%            │
│   (resize, norm)   │         │                   │
│ MODEL INFERENCE    │ 2-50ms  │ 40-70%  ← Focus! │
│ Post-processing    │ 1-5ms   │ 5-15%            │
│   (NMS, format)    │         │                   │
│ Business logic     │ 0-5ms   │ 0-10%            │
│ Network (if API)   │ 5-50ms  │ 10-30%           │
│                    │         │                   │
│ TOTAL              │ 10-120ms│ 100%              │
└──────────────────────────────────────────────────┘

KEY INSIGHT: Model inference is only 40-70% of total latency!
Don't just optimize the model — optimize the WHOLE pipeline.
```

### 🔥 Interview Quick-Fire

| Question | Answer |
|----------|--------|
| "How to speed up OD inference?" | Export to TensorRT (NVIDIA), quantize FP16/INT8, reduce input size |
| "What is quantization?" | Reduce weight precision (FP32→FP16→INT8) for smaller+faster models |
| "ONNX vs TensorRT?" | ONNX: universal, portable. TensorRT: NVIDIA-specific, 2-3× faster |
| "How to deploy on mobile?" | TFLite (Android), CoreML (iOS), NCNN (cross-platform C++) |
| "What is knowledge distillation?" | Train small model to mimic large model's outputs (soft labels) |
| "Best edge device for OD?" | Jetson Orin (best NVIDIA), Coral (cheapest), phone NPU (consumer) |
| "How to serve at scale?" | Triton (production), FastAPI (simple), TF Serving (TF ecosystem) |
| "TensorRT engine is portable?" | NO — must rebuild for each GPU architecture (T4 ≠ A100 ≠ Jetson) |

---

### ➡️ Next Chapter: [13 - Comparison & Selection Guide](./13-Comparison-and-Selection-Guide.md)

> Now you can deploy and optimize. The final chapter brings everything together — a comprehensive decision matrix for choosing the right library, model, and approach for ANY use case.

---

*[← Back to Index](./00-INDEX.md) | [← Previous: 11 - Real-World Use Cases](./11-Real-World-UseCases.md)*
