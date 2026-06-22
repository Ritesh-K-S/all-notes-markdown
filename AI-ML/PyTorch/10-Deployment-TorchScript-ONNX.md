# Chapter 10: Deployment — TorchScript, ONNX & Production Serving

## Table of Contents
- [10.1 The Deployment Problem](#101-the-deployment-problem)
- [10.2 TorchScript — JIT Compilation](#102-torchscript--jit-compilation)
- [10.3 ONNX Export](#103-onnx-export)
- [10.4 TorchServe — Production Model Serving](#104-torchserve--production-model-serving)
- [10.5 Mobile Deployment with PyTorch Mobile](#105-mobile-deployment-with-pytorch-mobile)
- [10.6 Optimization for Inference](#106-optimization-for-inference)
- [10.7 Complete Deployment Pipeline](#107-complete-deployment-pipeline)
- [10.8 Common Mistakes](#108-common-mistakes)
- [10.9 Interview Questions](#109-interview-questions)
- [10.10 Quick Reference](#1010-quick-reference)

---

## 10.1 The Deployment Problem

### What It Is
Deployment means taking your trained PyTorch model and making it available for real users — in a web app, mobile phone, embedded device, or cloud service. The challenge is that PyTorch is designed for **research flexibility** (dynamic graphs, Python control flow), but production needs **speed, reliability, and no Python dependency**.

Think of it like this: you build a prototype car in your workshop (training in Python), but to sell it to millions of people, you need a factory-optimized assembly line (deployment runtime).

### Why It Matters
- **Python is slow** — The GIL, interpreter overhead, and dynamic typing make raw Python 10-100x slower than optimized C++
- **Dependency hell** — Production servers don't want to install PyTorch (1.5+ GB), CUDA, cuDNN just to serve predictions
- **Portability** — You need to run models on phones (iOS/Android), browsers (WASM), edge devices (Jetson, Raspberry Pi)
- **Scalability** — Handling thousands of requests per second requires specialized serving infrastructure

### The Deployment Landscape

```
┌─────────────────────────────────────────────────────────────┐
│                   TRAINING (Python + PyTorch)                │
│    model.train() → model.eval() → Save Weights             │
└──────────────────────────────┬──────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                 ▼
     ┌────────────┐   ┌─────────────┐   ┌───────────────┐
     │ TorchScript│   │    ONNX     │   │  Pure Python  │
     │   (.pt)    │   │   (.onnx)   │   │  (Flask/Fast) │
     └─────┬──────┘   └──────┬──────┘   └───────┬───────┘
           │                  │                   │
     ┌─────┴──────┐   ┌──────┴──────┐   ┌───────┴───────┐
     │ TorchServe │   │ ONNX Runtime│   │  REST API     │
     │ C++ Runtime│   │ TensorRT    │   │  (Dev/Small)  │
     │ Mobile     │   │ OpenVINO    │   │               │
     └────────────┘   └─────────────┘   └───────────────┘
```

---

## 10.2 TorchScript — JIT Compilation

### What It Is
TorchScript is a way to serialize PyTorch models into a format that can be loaded and run **without Python**. It's like compiling your Python model into a standalone executable. The JIT (Just-In-Time) compiler converts your Python + PyTorch code into an intermediate representation (IR) that a C++ runtime can execute.

### Why It Matters
- Run models without Python interpreter (C++, mobile, embedded)
- Optimize computation graphs (operator fusion, constant folding)
- Static type checking catches bugs before deployment
- Models become self-contained — ship a single `.pt` file

### How It Works — Two Approaches

#### Approach 1: Tracing (`torch.jit.trace`)

Tracing runs your model with sample input and **records** every operation. Like recording someone cooking a recipe — you get exact steps but miss any "if-else" decisions they might make with different ingredients.

```python
import torch
import torch.nn as nn

# Define a simple model
class SimpleNet(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(784, 256)
        self.fc2 = nn.Linear(256, 10)
        self.relu = nn.ReLU()
    
    def forward(self, x):
        x = self.relu(self.fc1(x))
        x = self.fc2(x)
        return x

# Create model and set to eval mode (CRITICAL for deployment)
model = SimpleNet()
model.eval()  # Disables dropout, batchnorm running stats, etc.

# Create example input (batch_size=1, features=784)
example_input = torch.randn(1, 784)

# Trace the model — records all operations with this input
traced_model = torch.jit.trace(model, example_input)

# Save the traced model — self-contained, no Python needed
traced_model.save("simple_net_traced.pt")

# Load and run without original model class definition
loaded_model = torch.jit.load("simple_net_traced.pt")
output = loaded_model(example_input)
print(f"Output shape: {output.shape}")  # torch.Size([1, 10])
```

**When to use tracing:**
- Model has NO data-dependent control flow (no if/else based on input)
- Simple feedforward architectures
- Most CNN models

#### Approach 2: Scripting (`torch.jit.script`)

Scripting **analyzes your Python code** and converts it to TorchScript IR. It understands loops, conditions, and control flow — like having a translator convert your recipe from English to French word by word.

```python
import torch
import torch.nn as nn

class DynamicNet(nn.Module):
    """Model with data-dependent control flow — MUST use scripting"""
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(784, 256)
        self.fc2 = nn.Linear(256, 128)
        self.fc3 = nn.Linear(128, 10)
        self.fc_shortcut = nn.Linear(784, 10)
    
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        # Data-dependent branch — tracing would only capture ONE path
        if x.sum() > 0:
            x = torch.relu(self.fc1(x))
            x = torch.relu(self.fc2(x))
            x = self.fc3(x)
        else:
            # Shortcut for "easy" inputs
            x = self.fc_shortcut(x)
        return x

model = DynamicNet()
model.eval()

# Script the model — analyzes code, captures ALL branches
scripted_model = torch.jit.script(model)

# Inspect the generated TorchScript code
print(scripted_model.code)
# Shows the TorchScript IR — useful for debugging

# Save and load
scripted_model.save("dynamic_net_scripted.pt")
loaded = torch.jit.load("dynamic_net_scripted.pt")

# Test both branches
positive_input = torch.abs(torch.randn(1, 784))  # sum > 0
negative_input = -torch.abs(torch.randn(1, 784))  # sum < 0
print(loaded(positive_input).shape)   # Goes through full path
print(loaded(negative_input).shape)   # Goes through shortcut
```

### Hybrid Approach — Mixing Trace and Script

```python
import torch
import torch.nn as nn

class HybridModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.backbone = nn.Sequential(
            nn.Linear(784, 512),
            nn.ReLU(),
            nn.Linear(512, 256),
            nn.ReLU()
        )
        self.head_easy = nn.Linear(256, 10)
        self.head_hard = nn.Sequential(
            nn.Linear(256, 128),
            nn.ReLU(),
            nn.Linear(128, 10)
        )
    
    @torch.jit.export  # Makes this method accessible from TorchScript
    def classify(self, x: torch.Tensor, confidence_threshold: float = 0.8) -> torch.Tensor:
        features = self.backbone(x)
        easy_pred = self.head_easy(features)
        
        # Use easy head if confident, else use harder head
        max_prob = torch.softmax(easy_pred, dim=-1).max()
        if max_prob > confidence_threshold:
            return easy_pred
        else:
            return self.head_hard(features)
    
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        return self.classify(x)

model = HybridModel()
model.eval()

# Script handles the control flow
scripted = torch.jit.script(model)
scripted.save("hybrid_model.pt")

# Load and use the exported method
loaded = torch.jit.load("hybrid_model.pt")
result = loaded.classify(torch.randn(1, 784), 0.5)
print(f"Result: {result.shape}")
```

### TorchScript Type Annotations

TorchScript requires explicit types for non-trivial code:

```python
import torch
from typing import Tuple, List, Dict, Optional

@torch.jit.script
def process_detections(
    boxes: torch.Tensor,           # Shape [N, 4]
    scores: torch.Tensor,          # Shape [N]
    threshold: float = 0.5
) -> Tuple[torch.Tensor, torch.Tensor]:
    """Filter detections by confidence score"""
    mask = scores > threshold
    filtered_boxes = boxes[mask]
    filtered_scores = scores[mask]
    return filtered_boxes, filtered_scores

# Using Optional types
@torch.jit.script
def normalize(
    x: torch.Tensor, 
    mean: Optional[torch.Tensor] = None
) -> torch.Tensor:
    if mean is None:
        mean = x.mean()
    return x - mean

# Using List and Dict
@torch.jit.script
def multi_scale_features(
    x: torch.Tensor,
    scales: List[int]
) -> Dict[str, torch.Tensor]:
    results: Dict[str, torch.Tensor] = {}
    for s in scales:
        pooled = torch.nn.functional.adaptive_avg_pool1d(
            x.unsqueeze(0), s
        ).squeeze(0)
        results[str(s)] = pooled
    return results
```

### Tracing vs Scripting — Decision Matrix

| Criterion | `torch.jit.trace` | `torch.jit.script` |
|-----------|-------------------|---------------------|
| Control flow | ❌ Not captured | ✅ Fully captured |
| Dynamic shapes | ✅ Works naturally | ✅ Works with annotations |
| Dictionary outputs | ⚠️ Flattened | ✅ Preserved |
| Third-party libs | ❌ Not supported | ❌ Not supported |
| Debugging ease | ✅ Simple | ⚠️ Cryptic errors |
| Performance | ⚡ Slightly faster | ⚡ Good |
| Use when... | Simple forward pass | Complex logic |

---

## 10.3 ONNX Export

### What It Is
ONNX (Open Neural Network Exchange) is a **universal format** for neural networks. Think of it as PDF for documents — any tool can read it regardless of what created it. You can export from PyTorch and run in TensorRT (NVIDIA), OpenVINO (Intel), CoreML (Apple), or ONNX Runtime (Microsoft).

### Why It Matters
- **Hardware optimization** — TensorRT on NVIDIA GPUs gives 2-5x speedup over raw PyTorch
- **Cross-platform** — One model runs everywhere (cloud, edge, mobile)
- **Vendor independence** — Not locked into one framework
- **Production runtimes** — ONNX Runtime is 2-3x faster than PyTorch for inference

### How It Works

ONNX represents neural networks as a **directed acyclic graph (DAG)** of operators. Each operator (Conv, MatMul, Relu) has a well-defined specification. PyTorch's export traces your model and maps PyTorch ops → ONNX ops.

```
PyTorch Model                    ONNX Graph
┌──────────────┐                ┌──────────────┐
│ nn.Linear    │  ──export──▶  │ MatMul + Add │
│ nn.ReLU      │                │ Relu          │
│ nn.Conv2d    │                │ Conv          │
│ nn.BatchNorm │                │ BN (fused)    │
└──────────────┘                └──────────────┘
```

### Basic ONNX Export

```python
import torch
import torch.nn as nn

class ConvClassifier(nn.Module):
    def __init__(self, num_classes=10):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(3, 32, 3, padding=1),
            nn.BatchNorm2d(32),
            nn.ReLU(inplace=True),
            nn.MaxPool2d(2),
            nn.Conv2d(32, 64, 3, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(inplace=True),
            nn.AdaptiveAvgPool2d(1)
        )
        self.classifier = nn.Linear(64, num_classes)
    
    def forward(self, x):
        x = self.features(x)
        x = x.flatten(1)
        x = self.classifier(x)
        return x

# Create and prepare model
model = ConvClassifier(num_classes=10)
model.eval()  # CRITICAL: eval mode fuses BN, disables dropout

# Dummy input matching expected shape
dummy_input = torch.randn(1, 3, 224, 224)

# Export to ONNX
torch.onnx.export(
    model,                          # Model to export
    dummy_input,                    # Example input for tracing
    "classifier.onnx",             # Output file path
    export_params=True,             # Store trained weights in file
    opset_version=17,               # ONNX operator set version (use latest stable)
    do_constant_folding=True,       # Optimize: fold constants
    input_names=['image'],          # Name inputs for serving
    output_names=['logits'],        # Name outputs for serving
    dynamic_axes={                  # Allow variable batch size
        'image': {0: 'batch_size'},
        'logits': {0: 'batch_size'}
    }
)
print("✓ Model exported to classifier.onnx")
```

### Dynamic Axes — Variable Input Shapes

```python
import torch
import torch.nn as nn

class FlexibleModel(nn.Module):
    """Model that handles variable sequence length"""
    def __init__(self, vocab_size=30000, embed_dim=256, num_classes=5):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.lstm = nn.LSTM(embed_dim, 128, batch_first=True, bidirectional=True)
        self.classifier = nn.Linear(256, num_classes)
    
    def forward(self, input_ids: torch.Tensor) -> torch.Tensor:
        # input_ids: [batch_size, seq_len]
        embedded = self.embedding(input_ids)
        lstm_out, _ = self.lstm(embedded)
        # Use last hidden state
        last_hidden = lstm_out[:, -1, :]
        return self.classifier(last_hidden)

model = FlexibleModel()
model.eval()

# Export with dynamic batch AND sequence length
dummy = torch.randint(0, 30000, (1, 50))  # batch=1, seq=50

torch.onnx.export(
    model, dummy, "text_classifier.onnx",
    opset_version=17,
    input_names=['input_ids'],
    output_names=['predictions'],
    dynamic_axes={
        'input_ids': {
            0: 'batch_size',      # Variable batch
            1: 'sequence_length'  # Variable sequence
        },
        'predictions': {
            0: 'batch_size'
        }
    }
)
```

### Running ONNX Models with ONNX Runtime

```python
import numpy as np
# pip install onnxruntime-gpu  (or onnxruntime for CPU)
import onnxruntime as ort

# Create inference session
# Providers specify execution priority (first available is used)
session = ort.InferenceSession(
    "classifier.onnx",
    providers=['CUDAExecutionProvider', 'CPUExecutionProvider']
)

# Inspect model inputs/outputs
for inp in session.get_inputs():
    print(f"Input: {inp.name}, Shape: {inp.shape}, Type: {inp.type}")
for out in session.get_outputs():
    print(f"Output: {out.name}, Shape: {out.shape}, Type: {out.type}")

# Run inference — note: input must be numpy array
input_data = np.random.randn(1, 3, 224, 224).astype(np.float32)
outputs = session.run(
    ['logits'],                    # Output names to fetch
    {'image': input_data}          # Input name → numpy array
)
logits = outputs[0]
predicted_class = np.argmax(logits, axis=1)
print(f"Predicted class: {predicted_class[0]}")
```

### Validating ONNX Export

```python
import torch
import numpy as np
import onnxruntime as ort
import onnx

# Step 1: Check ONNX model is well-formed
onnx_model = onnx.load("classifier.onnx")
onnx.checker.check_model(onnx_model)
print("✓ ONNX model is valid")

# Step 2: Compare PyTorch vs ONNX outputs
model = ConvClassifier(num_classes=10)
model.eval()

test_input = torch.randn(4, 3, 224, 224)  # Batch of 4

# PyTorch inference
with torch.no_grad():
    pytorch_output = model(test_input).numpy()

# ONNX Runtime inference
session = ort.InferenceSession("classifier.onnx")
onnx_output = session.run(None, {'image': test_input.numpy()})[0]

# Compare — should be very close (floating point differences are OK)
np.testing.assert_allclose(pytorch_output, onnx_output, rtol=1e-5, atol=1e-6)
print("✓ Outputs match between PyTorch and ONNX")

# Step 3: Benchmark speed
import time

# PyTorch speed
start = time.perf_counter()
for _ in range(100):
    with torch.no_grad():
        _ = model(test_input)
pytorch_time = (time.perf_counter() - start) / 100

# ONNX Runtime speed
start = time.perf_counter()
for _ in range(100):
    _ = session.run(None, {'image': test_input.numpy()})
onnx_time = (time.perf_counter() - start) / 100

print(f"PyTorch: {pytorch_time*1000:.2f} ms/batch")
print(f"ONNX Runtime: {onnx_time*1000:.2f} ms/batch")
print(f"Speedup: {pytorch_time/onnx_time:.2f}x")
```

### ONNX with Quantization

```python
import onnxruntime as ort
from onnxruntime.quantization import quantize_dynamic, QuantType

# Dynamic quantization — easiest, good for CPU
quantize_dynamic(
    model_input="classifier.onnx",
    model_output="classifier_int8.onnx",
    weight_type=QuantType.QInt8  # Quantize weights to INT8
)

# Compare sizes
import os
original_size = os.path.getsize("classifier.onnx") / 1024 / 1024
quantized_size = os.path.getsize("classifier_int8.onnx") / 1024 / 1024
print(f"Original: {original_size:.2f} MB")
print(f"Quantized: {quantized_size:.2f} MB")
print(f"Compression: {original_size/quantized_size:.1f}x")

# Run quantized model — same API
session_q = ort.InferenceSession("classifier_int8.onnx")
output_q = session_q.run(None, {'image': np.random.randn(1, 3, 224, 224).astype(np.float32)})
```

### ONNX Opset Versions — What to Use

| Opset | Key Additions | Recommendation |
|-------|---------------|----------------|
| 11 | Clip, Resize improvements | Minimum for most models |
| 13 | Squeeze/Unsqueeze changes | Good baseline |
| 14 | Reshape, BatchNorm improvements | Recommended minimum |
| 17 | LayerNorm, GroupNorm native | **Recommended for transformers** |
| 18+ | Latest ops | If you need cutting-edge ops |

> **Pro Tip:** Always use the highest opset your deployment runtime supports. Higher opsets = more operations supported natively = fewer custom op workarounds.

---

## 10.4 TorchServe — Production Model Serving

### What It Is
TorchServe is PyTorch's official production serving solution. Think of it as a **restaurant kitchen for ML models** — it handles taking orders (HTTP requests), routing them to the right chef (model worker), preparing the food (inference), and serving it back to customers (responses). It handles batching, scaling, monitoring, and versioning out of the box.

### Why It Matters
- **Production-grade** — Handles load balancing, health checks, metrics
- **Dynamic batching** — Combines multiple requests into one GPU batch for efficiency
- **Multi-model** — Serve many models from one server
- **A/B testing** — Easy model versioning and traffic splitting
- **Monitoring** — Prometheus metrics, logging, and health endpoints

### Architecture

```
                    ┌──────────────────────────────────┐
                    │          TorchServe               │
 HTTP Request      │                                    │
 ──────────────▶   │  ┌───────────┐    ┌────────────┐ │
                    │  │ Frontend  │───▶│  Backend    │ │
 POST /predictions │  │ (Netty)   │    │  Workers    │ │
 ──────────────▶   │  │           │    │             │ │
                    │  │ - REST API│    │ Worker 1 ───│─│──▶ GPU 0
 GET /metrics      │  │ - gRPC    │    │ Worker 2 ───│─│──▶ GPU 1
 ──────────────▶   │  │ - Batching│    │ Worker 3 ───│─│──▶ CPU
                    │  └───────────┘    └────────────┘ │
                    │                                    │
                    │  Management API (port 8081)        │
                    │  Inference API  (port 8080)        │
                    │  Metrics API    (port 8082)        │
                    └──────────────────────────────────┘
```

### Step 1: Create a Model Handler

```python
# handler.py — Custom handler for image classification
import torch
import torch.nn.functional as F
from torchvision import transforms
from ts.torch_handler.base_handler import BaseHandler
import io
from PIL import Image
import json
import logging

logger = logging.getLogger(__name__)

class ImageClassifierHandler(BaseHandler):
    """
    Custom handler for image classification.
    TorchServe calls these methods in order:
    1. initialize() — Load model once at startup
    2. preprocess() — Transform raw input to tensor
    3. inference() — Run model forward pass
    4. postprocess() — Format output for response
    """
    
    def initialize(self, context):
        """Load model and class labels"""
        super().initialize(context)
        
        # Load class labels
        mapping_file = context.manifest['model'].get('labelFile', 'index_to_name.json')
        if mapping_file:
            with open(mapping_file) as f:
                self.class_names = json.load(f)
        
        # Define preprocessing (must match training!)
        self.transform = transforms.Compose([
            transforms.Resize(256),
            transforms.CenterCrop(224),
            transforms.ToTensor(),
            transforms.Normalize(
                mean=[0.485, 0.456, 0.406],
                std=[0.229, 0.224, 0.225]
            )
        ])
        
        logger.info("Model initialized successfully")
    
    def preprocess(self, requests):
        """Convert raw bytes to tensors"""
        images = []
        for request in requests:
            # Each request contains raw image bytes
            image_data = request.get('data') or request.get('body')
            image = Image.open(io.BytesIO(image_data)).convert('RGB')
            tensor = self.transform(image)
            images.append(tensor)
        
        # Stack into batch [N, C, H, W]
        return torch.stack(images).to(self.device)
    
    def inference(self, input_batch):
        """Run model prediction"""
        with torch.no_grad():
            logits = self.model(input_batch)
            probabilities = F.softmax(logits, dim=1)
        return probabilities
    
    def postprocess(self, inference_output):
        """Format results for response"""
        results = []
        for probs in inference_output:
            top5_prob, top5_idx = torch.topk(probs, 5)
            result = {}
            for i in range(5):
                class_idx = str(top5_idx[i].item())
                class_name = self.class_names.get(class_idx, f"class_{class_idx}")
                result[class_name] = round(top5_prob[i].item(), 4)
            results.append(result)
        return results
```

### Step 2: Package the Model Archive (.mar)

```bash
# Install torch-model-archiver
pip install torch-model-archiver torchserve

# Package model into .mar file
torch-model-archiver \
    --model-name image_classifier \
    --version 1.0 \
    --serialized-file model_weights.pth \
    --model-file model.py \
    --handler handler.py \
    --extra-files "index_to_name.json" \
    --export-path model_store/

# For TorchScript models (no model-file needed):
torch-model-archiver \
    --model-name image_classifier_ts \
    --version 1.0 \
    --serialized-file model_scripted.pt \
    --handler handler.py \
    --extra-files "index_to_name.json" \
    --export-path model_store/
```

### Step 3: Configure and Start TorchServe

```properties
# config.properties
inference_address=http://0.0.0.0:8080
management_address=http://0.0.0.0:8081
metrics_address=http://0.0.0.0:8082

# Worker configuration
default_workers_per_model=4
job_queue_size=1000

# Batching — KEY for GPU efficiency
batch_size=32
max_batch_delay=100

# Memory management
vmargs=-Xmx4g -XX:+UseG1GC
```

```bash
# Start TorchServe
torchserve --start \
    --model-store model_store \
    --models image_classifier=image_classifier.mar \
    --ts-config config.properties

# Register additional models dynamically
curl -X POST "http://localhost:8081/models?url=image_classifier_v2.mar&initial_workers=2"

# Scale workers
curl -X PUT "http://localhost:8081/models/image_classifier?min_worker=4&max_worker=8"
```

### Step 4: Make Predictions

```python
import requests

# Single prediction
with open("test_image.jpg", "rb") as f:
    response = requests.post(
        "http://localhost:8080/predictions/image_classifier",
        data=f.read(),
        headers={"Content-Type": "application/octet-stream"}
    )
print(response.json())
# [{"golden_retriever": 0.9234, "labrador": 0.0521, ...}]

# Batch prediction (send multiple images)
files = [
    ('data', open('img1.jpg', 'rb')),
    ('data', open('img2.jpg', 'rb')),
]
response = requests.post(
    "http://localhost:8080/predictions/image_classifier",
    files=files
)

# Check model status
status = requests.get("http://localhost:8081/models/image_classifier")
print(status.json())

# Get metrics (Prometheus format)
metrics = requests.get("http://localhost:8082/metrics")
print(metrics.text)
```

### TorchServe with Docker

```dockerfile
# Dockerfile for TorchServe
FROM pytorch/torchserve:latest-gpu

# Copy model archives
COPY model_store/ /home/model-server/model-store/
COPY config.properties /home/model-server/config.properties

# Expose ports
EXPOSE 8080 8081 8082

# Start TorchServe
CMD ["torchserve", \
     "--start", \
     "--model-store", "/home/model-server/model-store", \
     "--models", "all", \
     "--ts-config", "/home/model-server/config.properties", \
     "--foreground"]
```

```bash
# Build and run
docker build -t my-model-server .
docker run -p 8080:8080 -p 8081:8081 --gpus all my-model-server
```

---

## 10.5 Mobile Deployment with PyTorch Mobile

### What It Is
PyTorch Mobile (now evolving into **ExecuTorch**) lets you run models on iOS and Android devices. It's like shrinking a powerful desktop computer into a phone-sized package — you sacrifice some power for portability and privacy (data never leaves the device).

### Why It Matters
- **Privacy** — No data sent to servers (medical, financial apps)
- **Latency** — No network round-trip, instant predictions
- **Offline** — Works without internet connection
- **Cost** — No server costs for inference

### Optimizing for Mobile

```python
import torch
from torch.utils.mobile_optimizer import optimize_for_mobile

# Load your scripted/traced model
model = torch.jit.load("model_scripted.pt")

# Optimize for mobile — fuses operations, removes dropout, etc.
optimized_model = optimize_for_mobile(model)

# Save with mobile-optimized format
optimized_model._save_for_lite_interpreter("model_mobile.ptl")

# Check size reduction
import os
original = os.path.getsize("model_scripted.pt") / 1024 / 1024
mobile = os.path.getsize("model_mobile.ptl") / 1024 / 1024
print(f"Original: {original:.2f} MB → Mobile: {mobile:.2f} MB")
```

### Quantization for Mobile (INT8)

```python
import torch
import torch.quantization as quant

class MobileModel(torch.nn.Module):
    def __init__(self):
        super().__init__()
        self.quant = quant.QuantStub()        # Marks start of quantized region
        self.features = torch.nn.Sequential(
            torch.nn.Conv2d(3, 16, 3, padding=1),
            torch.nn.ReLU(inplace=True),
            torch.nn.Conv2d(16, 32, 3, padding=1),
            torch.nn.ReLU(inplace=True),
            torch.nn.AdaptiveAvgPool2d(1)
        )
        self.classifier = torch.nn.Linear(32, 10)
        self.dequant = quant.DeQuantStub()    # Marks end of quantized region
    
    def forward(self, x):
        x = self.quant(x)
        x = self.features(x)
        x = x.flatten(1)
        x = self.classifier(x)
        x = self.dequant(x)
        return x

# Step 1: Prepare model for quantization
model = MobileModel()
model.eval()
model.qconfig = quant.get_default_qconfig('qnnpack')  # Mobile-optimized
quant.prepare(model, inplace=True)

# Step 2: Calibrate with representative data
# Run ~100-1000 samples through the model
calibration_data = torch.randn(100, 3, 32, 32)
with torch.no_grad():
    for i in range(0, 100, 10):
        model(calibration_data[i:i+10])

# Step 3: Convert to quantized model
quantized_model = quant.convert(model)

# Step 4: Script and optimize for mobile
scripted = torch.jit.script(quantized_model)
optimized = torch.utils.mobile_optimizer.optimize_for_mobile(scripted)
optimized._save_for_lite_interpreter("model_quantized_mobile.ptl")

# Compare model sizes
print(f"FP32: {os.path.getsize('model_scripted.pt')/1024:.0f} KB")
print(f"INT8 Mobile: {os.path.getsize('model_quantized_mobile.ptl')/1024:.0f} KB")
# Expect ~4x size reduction
```

### ExecuTorch (Next-Gen Mobile)

```python
# ExecuTorch — PyTorch's new edge deployment framework (2024+)
# pip install executorch

import torch
from executorch.exir import to_edge
from torch.export import export

class SimpleModel(torch.nn.Module):
    def __init__(self):
        super().__init__()
        self.linear = torch.nn.Linear(10, 5)
    
    def forward(self, x):
        return self.linear(x)

model = SimpleModel()
model.eval()

# Step 1: Export with torch.export (new standard)
example_input = (torch.randn(1, 10),)
exported = export(model, example_input)

# Step 2: Lower to edge format
edge_program = to_edge(exported)

# Step 3: Generate executable
executorch_program = edge_program.to_executorch()

# Step 4: Save .pte file (runs on iOS/Android/embedded)
with open("model.pte", "wb") as f:
    f.write(executorch_program.buffer)
```

---

## 10.6 Optimization for Inference

### What It Is
Inference optimization means making your model predict faster without sacrificing (much) accuracy. It's like tuning a car engine — same car, more speed, better fuel efficiency.

### Quantization Strategies

```python
import torch
import torch.nn as nn

# ═══════════════════════════════════════════
# Strategy 1: Dynamic Quantization (Easiest)
# ═══════════════════════════════════════════
# Quantizes weights statically, activations dynamically at runtime
# Best for: LSTM, Transformer models (weight-bound)
model = torch.nn.Sequential(
    nn.Linear(512, 256),
    nn.ReLU(),
    nn.Linear(256, 10)
)
model.eval()

quantized_dynamic = torch.quantization.quantize_dynamic(
    model,
    {nn.Linear},           # Which layers to quantize
    dtype=torch.qint8      # Target dtype
)
# Size: ~4x smaller, Speed: ~2x faster on CPU

# ═══════════════════════════════════════════
# Strategy 2: Static Quantization (Best accuracy)
# ═══════════════════════════════════════════
# Calibrates activation ranges with real data
# Best for: CNN models (compute-bound)
# (See mobile section above for full example)

# ═══════════════════════════════════════════
# Strategy 3: Quantization-Aware Training (QAT)
# ═══════════════════════════════════════════
# Simulates quantization during training — highest accuracy
# Best for: When post-training quantization loses too much accuracy

class QATModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.quant = torch.quantization.QuantStub()
        self.conv1 = nn.Conv2d(3, 32, 3, padding=1)
        self.bn1 = nn.BatchNorm2d(32)
        self.relu = nn.ReLU(inplace=True)
        self.fc = nn.Linear(32, 10)
        self.dequant = torch.quantization.DeQuantStub()
    
    def forward(self, x):
        x = self.quant(x)
        x = self.relu(self.bn1(self.conv1(x)))
        x = x.mean([2, 3])  # Global average pool
        x = self.fc(x)
        x = self.dequant(x)
        return x

# Prepare for QAT
qat_model = QATModel()
qat_model.train()
qat_model.qconfig = torch.quantization.get_default_qat_qconfig('fbgemm')
torch.quantization.prepare_qat(qat_model, inplace=True)

# Train normally (fake quantization is inserted)
optimizer = torch.optim.Adam(qat_model.parameters(), lr=1e-4)
for epoch in range(10):
    for batch_x, batch_y in train_loader:
        output = qat_model(batch_x)
        loss = nn.CrossEntropyLoss()(output, batch_y)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

# Convert to actual quantized model
qat_model.eval()
quantized_qat = torch.quantization.convert(qat_model)
```

### torch.compile() — The Modern Way (PyTorch 2.0+)

```python
import torch

model = get_my_model()
model.eval()

# torch.compile — JIT compiles the model graph
# Default mode — good balance of compile time and speedup
compiled_model = torch.compile(model)

# Modes:
# "default"     — Good balance (2-3x speedup typical)
# "reduce-overhead" — Minimizes Python overhead (good for small models)
# "max-autotune"    — Tries all optimizations (slow compile, fastest run)
compiled_fast = torch.compile(model, mode="max-autotune")

# First call is slow (compilation), subsequent calls are fast
with torch.no_grad():
    # Warm-up (triggers compilation)
    _ = compiled_model(torch.randn(1, 3, 224, 224))
    
    # Fast inference
    import time
    start = time.perf_counter()
    for _ in range(100):
        output = compiled_model(torch.randn(1, 3, 224, 224))
    print(f"Compiled: {(time.perf_counter()-start)*10:.2f} ms/sample")
```

### Mixed Precision Inference

```python
import torch

model = get_my_model().cuda().eval()

# FP16 inference — 2x memory savings, faster on modern GPUs
with torch.no_grad(), torch.cuda.amp.autocast():
    input_tensor = torch.randn(32, 3, 224, 224, device='cuda')
    output = model(input_tensor)  # Runs in FP16 where safe, FP32 where needed

# BFloat16 — better numerical stability than FP16
with torch.no_grad(), torch.cuda.amp.autocast(dtype=torch.bfloat16):
    output = model(input_tensor)

# Permanently convert model to FP16 (if accuracy is acceptable)
model_fp16 = model.half()
input_fp16 = input_tensor.half()
output = model_fp16(input_fp16)
```

### Pruning — Remove Unnecessary Weights

```python
import torch
import torch.nn.utils.prune as prune

model = get_my_model()

# Unstructured pruning — zero out individual weights
# Removes 30% of weights with smallest magnitude
prune.l1_unstructured(model.fc1, name='weight', amount=0.3)

# Structured pruning — remove entire channels/filters
# More hardware-friendly (actual speedup vs just sparsity)
prune.ln_structured(model.conv1, name='weight', amount=0.2, n=2, dim=0)

# Global pruning — prune across all layers
parameters_to_prune = [
    (model.fc1, 'weight'),
    (model.fc2, 'weight'),
    (model.fc3, 'weight'),
]
prune.global_unstructured(
    parameters_to_prune,
    pruning_method=prune.L1Unstructured,
    amount=0.4  # Remove 40% of all weights globally
)

# Make pruning permanent (remove the mask)
for module, param in parameters_to_prune:
    prune.remove(module, param)

# Check sparsity
def get_sparsity(model):
    total = 0
    zeros = 0
    for p in model.parameters():
        total += p.numel()
        zeros += (p == 0).sum().item()
    return zeros / total * 100

print(f"Model sparsity: {get_sparsity(model):.1f}%")
```

### Knowledge Distillation for Smaller Models

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DistillationLoss(nn.Module):
    """
    Combines hard label loss with soft teacher knowledge.
    Temperature > 1 softens probabilities, revealing dark knowledge.
    """
    def __init__(self, temperature=4.0, alpha=0.7):
        super().__init__()
        self.temperature = temperature
        self.alpha = alpha  # Weight for distillation vs hard labels
    
    def forward(self, student_logits, teacher_logits, targets):
        # Hard loss — standard cross-entropy with true labels
        hard_loss = F.cross_entropy(student_logits, targets)
        
        # Soft loss — KL divergence between teacher and student
        soft_student = F.log_softmax(student_logits / self.temperature, dim=1)
        soft_teacher = F.softmax(teacher_logits / self.temperature, dim=1)
        soft_loss = F.kl_div(soft_student, soft_teacher, reduction='batchmean')
        soft_loss *= (self.temperature ** 2)  # Scale gradient magnitude
        
        # Combined loss
        return self.alpha * soft_loss + (1 - self.alpha) * hard_loss

# Training loop
teacher_model = load_big_model()  # Large, accurate model
teacher_model.eval()

student_model = SmallModel()      # Small, fast model
optimizer = torch.optim.Adam(student_model.parameters(), lr=1e-3)
distill_loss = DistillationLoss(temperature=4.0, alpha=0.7)

for epoch in range(50):
    for batch_x, batch_y in train_loader:
        # Get teacher predictions (no gradient needed)
        with torch.no_grad():
            teacher_logits = teacher_model(batch_x)
        
        # Get student predictions
        student_logits = student_model(batch_x)
        
        # Compute distillation loss
        loss = distill_loss(student_logits, teacher_logits, batch_y)
        
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
```

---

## 10.7 Complete Deployment Pipeline

### End-to-End Example: Image Classification API

```python
"""
Complete deployment pipeline:
Train → Optimize → Export → Serve → Monitor
"""

import torch
import torch.nn as nn
from torchvision import models, transforms
import time
import os

# ═══════════════════════════════════════════
# STEP 1: Prepare the trained model
# ═══════════════════════════════════════════
# Load pre-trained ResNet-18 (in practice, this is your fine-tuned model)
model = models.resnet18(weights=models.ResNet18_Weights.DEFAULT)
model.eval()

# ═══════════════════════════════════════════
# STEP 2: Benchmark baseline
# ═══════════════════════════════════════════
dummy = torch.randn(1, 3, 224, 224)

start = time.perf_counter()
with torch.no_grad():
    for _ in range(100):
        _ = model(dummy)
baseline_ms = (time.perf_counter() - start) / 100 * 1000
print(f"Baseline PyTorch: {baseline_ms:.2f} ms/sample")

# ═══════════════════════════════════════════
# STEP 3: Export to TorchScript
# ═══════════════════════════════════════════
traced = torch.jit.trace(model, dummy)
traced = torch.jit.freeze(traced)  # Freeze — inlines parameters as constants
traced.save("resnet18_frozen.pt")

# Benchmark TorchScript
start = time.perf_counter()
for _ in range(100):
    _ = traced(dummy)
ts_ms = (time.perf_counter() - start) / 100 * 1000
print(f"TorchScript: {ts_ms:.2f} ms/sample ({baseline_ms/ts_ms:.1f}x)")

# ═══════════════════════════════════════════
# STEP 4: Export to ONNX
# ═══════════════════════════════════════════
torch.onnx.export(
    model, dummy, "resnet18.onnx",
    opset_version=17,
    input_names=['image'],
    output_names=['predictions'],
    dynamic_axes={'image': {0: 'batch'}, 'predictions': {0: 'batch'}}
)

# ═══════════════════════════════════════════
# STEP 5: Quantize for CPU deployment
# ═══════════════════════════════════════════
quantized = torch.quantization.quantize_dynamic(
    model, {nn.Linear}, dtype=torch.qint8
)

# ═══════════════════════════════════════════
# STEP 6: Create FastAPI serving endpoint
# ═══════════════════════════════════════════
# (Alternative to TorchServe for simpler deployments)
"""
# serve.py
from fastapi import FastAPI, UploadFile, File
from PIL import Image
import torch
import io
import torchvision.transforms as T

app = FastAPI()

# Load model once at startup
model = torch.jit.load("resnet18_frozen.pt")
model.eval()

transform = T.Compose([
    T.Resize(256),
    T.CenterCrop(224),
    T.ToTensor(),
    T.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225])
])

@app.post("/predict")
async def predict(file: UploadFile = File(...)):
    image_bytes = await file.read()
    image = Image.open(io.BytesIO(image_bytes)).convert("RGB")
    tensor = transform(image).unsqueeze(0)
    
    with torch.no_grad():
        output = model(tensor)
        probs = torch.softmax(output, dim=1)
        top5_prob, top5_idx = probs.topk(5)
    
    return {
        "predictions": [
            {"class_id": idx.item(), "confidence": prob.item()}
            for prob, idx in zip(top5_prob[0], top5_idx[0])
        ]
    }

# Run with: uvicorn serve:app --host 0.0.0.0 --port 8000 --workers 4
"""

print("\n✓ Deployment pipeline complete!")
print(f"  Model sizes:")
print(f"    PyTorch: {os.path.getsize('resnet18_frozen.pt')/1024/1024:.1f} MB")
print(f"    ONNX:    {os.path.getsize('resnet18.onnx')/1024/1024:.1f} MB")
```

---

## 10.8 Common Mistakes

### Mistake 1: Forgetting `model.eval()`
```python
# ❌ WRONG — BatchNorm and Dropout still active
model.train()  # or just not calling eval()
traced = torch.jit.trace(model, dummy)

# ✅ CORRECT — Always eval() before export
model.eval()
traced = torch.jit.trace(model, dummy)
```

### Mistake 2: Not Testing with Different Input Shapes
```python
# ❌ Exported with fixed batch=1, breaks with batch=32
torch.onnx.export(model, torch.randn(1, 3, 224, 224), "model.onnx")

# ✅ Use dynamic_axes for variable dimensions
torch.onnx.export(model, torch.randn(1, 3, 224, 224), "model.onnx",
                  dynamic_axes={'input': {0: 'batch'}})
```

### Mistake 3: Tracing Models with Control Flow
```python
# ❌ Tracing only captures ONE branch
class Model(nn.Module):
    def forward(self, x):
        if x.shape[0] > 1:  # Data-dependent!
            return self.batch_path(x)
        return self.single_path(x)

# Tracing with batch=1 will ALWAYS use single_path
traced = torch.jit.trace(model, torch.randn(1, 10))  # Bug!

# ✅ Use scripting for control flow
scripted = torch.jit.script(model)
```

### Mistake 4: Mismatched Preprocessing
```python
# ❌ Training used one normalization, serving uses different
# Training: normalize with ImageNet stats
# Serving: forgot to normalize, or used different values

# ✅ Always embed preprocessing in the model or document it
class DeployableModel(nn.Module):
    def __init__(self, backbone):
        super().__init__()
        self.backbone = backbone
        self.register_buffer('mean', torch.tensor([0.485, 0.456, 0.406]).view(1, 3, 1, 1))
        self.register_buffer('std', torch.tensor([0.229, 0.224, 0.225]).view(1, 3, 1, 1))
    
    def forward(self, x):
        # x comes in as [0, 1] range from uint8/255
        x = (x - self.mean) / self.std
        return self.backbone(x)
```

### Mistake 5: Ignoring Warm-up in Benchmarks
```python
# ❌ First run includes compilation/memory allocation overhead
start = time.time()
output = compiled_model(input_data)  # Includes compilation!
print(f"Time: {time.time()-start}")  # Misleadingly slow

# ✅ Always warm up, then benchmark
# Warm-up runs
for _ in range(10):
    _ = compiled_model(input_data)

# Actual benchmark
torch.cuda.synchronize()  # Wait for GPU ops to finish
start = time.perf_counter()
for _ in range(100):
    _ = compiled_model(input_data)
torch.cuda.synchronize()
elapsed = (time.perf_counter() - start) / 100
print(f"Time: {elapsed*1000:.2f} ms")
```

---

## 10.9 Interview Questions

### Q1: What's the difference between `torch.jit.trace` and `torch.jit.script`?
**A:** Tracing records operations during one execution — fast and simple but misses control flow. Scripting analyzes the Python AST and converts to TorchScript IR — handles if/else, loops, but requires type annotations and has more restrictions. Use tracing for simple models, scripting for dynamic ones.

### Q2: When would you choose ONNX over TorchScript?
**A:** Choose ONNX when: (1) deploying to non-PyTorch runtimes (TensorRT, OpenVINO, CoreML), (2) need cross-framework interop, (3) targeting specific hardware optimizations (Intel CPUs with OpenVINO, NVIDIA GPUs with TensorRT). Choose TorchScript when: staying in PyTorch ecosystem, need dynamic control flow preservation, or using TorchServe.

### Q3: How does dynamic batching work in TorchServe?
**A:** TorchServe collects incoming requests into a queue and groups them into batches up to `batch_size`. It waits up to `max_batch_delay` ms for the batch to fill. This amortizes GPU kernel launch overhead across multiple requests, dramatically improving throughput. The trade-off is added latency for individual requests.

### Q4: What's quantization and what are the three approaches?
**A:** Quantization converts FP32 weights/activations to INT8 (4x smaller, 2-4x faster on CPU). Three approaches: (1) **Dynamic** — quantize weights statically, activations at runtime; easiest, good for linear layers. (2) **Static** — calibrate activation ranges with data; best for CNNs. (3) **QAT** — simulate quantization during training; highest accuracy, most effort.

### Q5: How do you ensure model correctness after export?
**A:** Run the same inputs through both the original and exported model, compare outputs with `np.testing.assert_allclose(rtol=1e-5, atol=1e-6)`. Test edge cases: min/max values, different batch sizes, boundary conditions. Also validate with production-like data distribution.

### Q6: Explain `torch.compile()` and when to use it.
**A:** `torch.compile()` (PyTorch 2.0+) uses TorchDynamo to capture the computation graph and TorchInductor to generate optimized kernels (Triton for GPU, C++ for CPU). It's a one-line optimization that gives 2-3x speedup. Use it for inference when you don't need to export — it's simpler than TorchScript/ONNX but requires Python.

### Q7: What's the deployment strategy for a model that needs <10ms latency?
**A:** (1) Use TensorRT via ONNX export for GPU, or OpenVINO for Intel CPU. (2) Quantize to INT8/FP16. (3) Use `torch.compile(mode="max-autotune")`. (4) Batch requests for throughput. (5) Use CUDA graphs to eliminate kernel launch overhead. (6) Consider model pruning/distillation to reduce compute.

---

## 10.10 Quick Reference

| Task | Tool/Method | Command |
|------|------------|---------|
| Export (no control flow) | `torch.jit.trace` | `traced = torch.jit.trace(model, example)` |
| Export (with control flow) | `torch.jit.script` | `scripted = torch.jit.script(model)` |
| Freeze parameters | `torch.jit.freeze` | `frozen = torch.jit.freeze(traced)` |
| Export to ONNX | `torch.onnx.export` | `torch.onnx.export(model, x, "m.onnx")` |
| Run ONNX | ONNX Runtime | `ort.InferenceSession("m.onnx")` |
| Dynamic quantization | `quantize_dynamic` | `torch.quantization.quantize_dynamic(m, {nn.Linear})` |
| Compile (PyTorch 2.0) | `torch.compile` | `compiled = torch.compile(model)` |
| Serve model | TorchServe | `torchserve --start --models m=m.mar` |
| Mobile optimize | `optimize_for_mobile` | `optimize_for_mobile(scripted)` |
| Prune weights | `torch.nn.utils.prune` | `prune.l1_unstructured(layer, 'weight', 0.3)` |

### Deployment Decision Flowchart

```
Need to deploy model?
│
├── Target: Cloud GPU → ONNX + TensorRT (fastest)
├── Target: Cloud CPU → ONNX Runtime or torch.compile
├── Target: Mobile    → TorchScript + quantize + optimize_for_mobile
├── Target: Edge/IoT  → ExecuTorch or ONNX + TFLite
├── Target: Browser   → ONNX.js or torch.export → WASM
└── Target: Simple API → FastAPI + torch.jit.load (quick & easy)
```

### Performance Comparison (ResNet-50, CPU)

| Method | Latency (ms) | Model Size | Setup Effort |
|--------|-------------|-----------|--------------|
| PyTorch (raw) | 120 | 98 MB | None |
| torch.compile | 55 | 98 MB | 1 line |
| TorchScript (frozen) | 80 | 98 MB | Low |
| ONNX Runtime | 45 | 95 MB | Medium |
| ONNX + INT8 | 25 | 25 MB | Medium |
| TensorRT (GPU) | 5 | 50 MB | High |

> **Key Takeaway:** For most deployments, start with `torch.compile()` or ONNX Runtime. Only invest in TensorRT/custom optimization when latency requirements demand it.
