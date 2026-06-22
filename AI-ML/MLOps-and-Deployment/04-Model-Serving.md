# Chapter 04: Model Serving

## Table of Contents
- [What is Model Serving?](#what-is-model-serving)
- [Why Model Serving Matters](#why-model-serving-matters)
- [How Model Serving Works](#how-model-serving-works)
- [Serving Architectures](#serving-architectures)
- [Flask for Model Serving](#flask-for-model-serving)
- [FastAPI for Model Serving](#fastapi-for-model-serving)
- [TensorFlow Serving](#tensorflow-serving)
- [TorchServe](#torchserve)
- [NVIDIA Triton Inference Server](#nvidia-triton-inference-server)
- [Batch vs Real-Time Inference](#batch-vs-real-time-inference)
- [Scaling Model Serving](#scaling-model-serving)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Model Serving?

**Simple Explanation:** Model serving is like turning your ML model into a "waiter" at a restaurant. You (the customer) send in your order (input data), the waiter takes it to the kitchen (the model), and brings back the result (prediction). Model serving is the entire system that makes this happen reliably, fast, and at scale.

When you train a model in a Jupyter notebook, it's like having a recipe. Model serving is when you actually open a restaurant and start cooking for thousands of customers simultaneously.

**Formal Definition:** Model serving is the process of deploying trained machine learning models into production environments where they can receive input data and return predictions via an API or other interfaces, with guarantees around latency, throughput, and availability.

---

## Why Model Serving Matters

### Real-World Impact

| Scenario | Without Serving | With Serving |
|----------|----------------|--------------|
| Fraud Detection | Batch processing, catch fraud next day | Real-time, block fraudulent transaction instantly |
| Recommendation | Static recommendations, same for everyone | Dynamic, personalized in milliseconds |
| Medical Imaging | Doctor waits hours for results | Results within seconds |

### When You'd Use It

- **REST APIs** → When other services need predictions on-demand
- **Streaming** → When predictions are needed on continuous data flows
- **Edge Deployment** → When predictions must happen on device (phones, IoT)
- **Batch Processing** → When you need predictions on large datasets periodically

### Business Value
- A model sitting in a notebook generates **zero** revenue
- Netflix's recommendation system (served model) saves them **$1B/year** in customer retention
- Google serves **billions** of predictions per second across search, ads, and YouTube

---

## How Model Serving Works

### The Big Picture

```
┌──────────────┐     ┌───────────────────────────────────┐     ┌──────────────┐
│   Client     │     │        Model Server                │     │   Model      │
│  (App/User)  │────▶│  ┌─────────┐  ┌──────────────┐   │     │   Storage    │
│              │     │  │ API     │  │ Inference    │   │◀────│  (Registry)  │
│  Send Input  │     │  │ Gateway │─▶│ Engine       │   │     │              │
│  Get Result  │◀────│  │         │  │              │   │     └──────────────┘
│              │     │  └─────────┘  └──────────────┘   │
└──────────────┘     │  ┌─────────┐  ┌──────────────┐   │
                     │  │ Pre-    │  │ Post-        │   │
                     │  │ process │  │ process      │   │
                     │  └─────────┘  └──────────────┘   │
                     └───────────────────────────────────┘
```

### Request Lifecycle

```
1. Client sends HTTP request with input data (JSON/Protobuf)
        │
        ▼
2. API Gateway receives request, authenticates, rate-limits
        │
        ▼
3. Pre-processing: normalize, tokenize, resize, etc.
        │
        ▼
4. Model inference: forward pass through the model
        │
        ▼
5. Post-processing: format output, apply thresholds
        │
        ▼
6. Response sent back to client (JSON/Protobuf)
```

### Key Concepts

| Concept | Definition | Analogy |
|---------|-----------|---------|
| **Latency** | Time from request to response | How long a customer waits for food |
| **Throughput** | Requests handled per second | How many customers served per hour |
| **Availability** | % of time service is operational | Restaurant open 24/7 vs closed on Sundays |
| **Scalability** | Ability to handle more load | Adding more chefs when rush hour hits |

---

## Serving Architectures

### 1. Embedded Model (Simplest)

```
┌─────────────────────────┐
│   Application Code      │
│   ┌─────────────────┐   │
│   │  Model (in-mem) │   │
│   └─────────────────┘   │
└─────────────────────────┘
```

- Model loaded directly in application memory
- No network call for inference
- **Use when:** Low traffic, simple models, edge devices

### 2. Model-as-a-Service (Most Common)

```
┌──────────┐     HTTP/gRPC     ┌──────────────┐
│   App    │──────────────────▶│ Model Server │
│          │◀──────────────────│              │
└──────────┘                   └──────────────┘
```

- Model runs in a separate service
- Communicate via REST/gRPC
- **Use when:** Multiple apps need same model, need independent scaling

### 3. Model Mesh (Enterprise)

```
┌──────────┐        ┌──────────────────────────┐
│   App    │───────▶│    Load Balancer         │
└──────────┘        │         │                │
                    │    ┌────┴────┐           │
                    │    ▼    ▼    ▼           │
                    │  ┌──┐ ┌──┐ ┌──┐         │
                    │  │M1│ │M2│ │M3│         │
                    │  └──┘ └──┘ └──┘         │
                    └──────────────────────────┘
```

- Multiple models behind a single endpoint
- Dynamic loading/unloading based on demand
- **Use when:** Serving hundreds of models, multi-tenant platforms

---

## Flask for Model Serving

### Basic Setup

```python
# app.py - Simple Flask Model Server
from flask import Flask, request, jsonify
import pickle
import numpy as np

# Initialize Flask app
app = Flask(__name__)

# Load model at startup (loaded once, used for every request)
with open('model.pkl', 'rb') as f:
    model = pickle.load(f)

@app.route('/health', methods=['GET'])
def health():
    """Health check endpoint - used by load balancers to verify service is alive"""
    return jsonify({'status': 'healthy'}), 200

@app.route('/predict', methods=['POST'])
def predict():
    """
    Prediction endpoint
    Expects JSON: {"features": [1.0, 2.0, 3.0, ...]}
    Returns JSON: {"prediction": 0, "probability": 0.85}
    """
    try:
        # Parse input data
        data = request.get_json(force=True)
        features = np.array(data['features']).reshape(1, -1)
        
        # Make prediction
        prediction = model.predict(features)[0]
        probability = model.predict_proba(features)[0].max()
        
        return jsonify({
            'prediction': int(prediction),
            'probability': float(probability)
        }), 200
        
    except Exception as e:
        return jsonify({'error': str(e)}), 400

if __name__ == '__main__':
    # Never use debug=True in production!
    app.run(host='0.0.0.0', port=5000)
```

### Production Flask with Gunicorn

```python
# gunicorn_config.py - Production WSGI configuration
import multiprocessing

# Number of worker processes
workers = multiprocessing.cpu_count() * 2 + 1

# Worker class - use 'sync' for CPU-bound ML inference
worker_class = 'sync'

# Bind to all interfaces on port 8000
bind = '0.0.0.0:8000'

# Timeout for worker processes (increase for large models)
timeout = 120

# Max requests before worker restart (prevents memory leaks)
max_requests = 1000
max_requests_jitter = 50

# Logging
accesslog = '-'
errorlog = '-'
loglevel = 'info'
```

```bash
# Run with Gunicorn (production WSGI server)
gunicorn -c gunicorn_config.py app:app
```

### Flask Limitations for ML

| Issue | Description | Impact |
|-------|-------------|--------|
| GIL | Python's Global Interpreter Lock | Can't do true parallel inference |
| Sync | Flask is synchronous by default | One request blocks others |
| No validation | No built-in request validation | Bad inputs can crash server |
| No docs | No auto-generated API docs | Hard for teams to integrate |

> **Warning:** Flask is great for prototyping but has significant limitations for production ML serving. Use FastAPI or dedicated serving frameworks for anything beyond simple demos.

---

## FastAPI for Model Serving

### Why FastAPI over Flask for ML?

| Feature | Flask | FastAPI |
|---------|-------|---------|
| Async Support | No (needs extensions) | Native |
| Input Validation | Manual | Automatic (Pydantic) |
| API Documentation | Manual (Swagger ext) | Auto-generated |
| Performance | ~500 req/s | ~2000+ req/s |
| Type Hints | Optional | Required (catches bugs) |
| WebSocket | Extension needed | Built-in |

### Complete FastAPI Model Server

```python
# main.py - Production-ready FastAPI Model Server
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from typing import List, Optional
import numpy as np
import joblib
import logging
from contextlib import asynccontextmanager

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Global model variable
model = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Load model on startup, cleanup on shutdown"""
    global model
    logger.info("Loading model...")
    model = joblib.load("model.joblib")
    logger.info("Model loaded successfully")
    yield
    # Cleanup on shutdown
    logger.info("Shutting down, releasing resources...")
    model = None

# Initialize FastAPI app
app = FastAPI(
    title="ML Model API",
    description="Production ML model serving endpoint",
    version="1.0.0",
    lifespan=lifespan
)

# --- Request/Response Schemas ---

class PredictionRequest(BaseModel):
    """Input schema with validation"""
    features: List[float] = Field(
        ...,
        min_length=4,  # Enforce feature count
        max_length=4,
        description="List of input features",
        examples=[[5.1, 3.5, 1.4, 0.2]]
    )
    
class PredictionResponse(BaseModel):
    """Output schema"""
    prediction: int
    confidence: float
    class_probabilities: List[float]
    model_version: str = "1.0.0"

class HealthResponse(BaseModel):
    status: str
    model_loaded: bool

# --- Endpoints ---

@app.get("/health", response_model=HealthResponse)
async def health_check():
    """Health check - returns model status"""
    return HealthResponse(
        status="healthy",
        model_loaded=model is not None
    )

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    """
    Make a prediction using the loaded model.
    
    - **features**: List of exactly 4 float values (iris features)
    - Returns prediction class and confidence scores
    """
    if model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")
    
    try:
        # Convert to numpy array
        features = np.array(request.features).reshape(1, -1)
        
        # Get prediction and probabilities
        prediction = model.predict(features)[0]
        probabilities = model.predict_proba(features)[0]
        confidence = float(probabilities.max())
        
        return PredictionResponse(
            prediction=int(prediction),
            confidence=confidence,
            class_probabilities=probabilities.tolist()
        )
    except Exception as e:
        logger.error(f"Prediction failed: {e}")
        raise HTTPException(status_code=500, detail="Prediction failed")

@app.post("/predict/batch")
async def predict_batch(requests: List[PredictionRequest]):
    """Batch prediction endpoint - process multiple inputs at once"""
    if model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")
    
    # Stack all features into single array (more efficient than loop)
    features = np.array([r.features for r in requests])
    
    predictions = model.predict(features)
    probabilities = model.predict_proba(features)
    
    results = []
    for i in range(len(requests)):
        results.append(PredictionResponse(
            prediction=int(predictions[i]),
            confidence=float(probabilities[i].max()),
            class_probabilities=probabilities[i].tolist()
        ))
    
    return results
```

### Running in Production

```python
# run_server.py - Production server with Uvicorn
import uvicorn

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=8000,
        workers=4,           # Multiple worker processes
        log_level="info",
        access_log=True,
        reload=False         # Never use reload in production!
    )
```

### Adding Middleware (Authentication, CORS, Rate Limiting)

```python
# middleware.py - Production middleware setup
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from starlette.middleware.base import BaseHTTPMiddleware
import time
import logging

logger = logging.getLogger(__name__)

# CORS middleware - controls which domains can call your API
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://yourdomain.com"],  # Never use "*" in production
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
)

# Custom middleware for request timing
class TimingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start_time = time.time()
        response = await call_next(request)
        process_time = time.time() - start_time
        
        # Add timing header to response
        response.headers["X-Process-Time"] = str(process_time)
        
        # Log slow requests
        if process_time > 1.0:
            logger.warning(
                f"Slow request: {request.url.path} took {process_time:.2f}s"
            )
        
        return response

app.add_middleware(TimingMiddleware)
```

### Client Example

```python
# client.py - How to call the model API
import requests

# Single prediction
response = requests.post(
    "http://localhost:8000/predict",
    json={"features": [5.1, 3.5, 1.4, 0.2]}
)
print(response.json())
# Output: {"prediction": 0, "confidence": 0.97, "class_probabilities": [0.97, 0.02, 0.01], "model_version": "1.0.0"}

# Batch prediction
batch_data = [
    {"features": [5.1, 3.5, 1.4, 0.2]},
    {"features": [6.7, 3.0, 5.2, 2.3]},
    {"features": [5.8, 2.7, 4.1, 1.0]}
]
response = requests.post("http://localhost:8000/predict/batch", json=batch_data)
print(response.json())
```

---

## TensorFlow Serving

### What is TF Serving?

TensorFlow Serving is Google's production-grade serving system specifically designed for ML models. Think of it as a "super-optimized waiter" that only works with TensorFlow models but does it incredibly fast.

### Key Advantages

- **C++ backend** → Much faster than Python-based servers
- **Model versioning** → Seamlessly switch between model versions
- **Batching** → Automatically groups requests for GPU efficiency
- **gRPC + REST** → Supports both protocols
- **Zero downtime updates** → Load new model without dropping requests

### Saving Model for TF Serving

```python
# save_model.py - Export model in SavedModel format
import tensorflow as tf
from tensorflow import keras
import numpy as np

# Train a simple model (or load your existing one)
model = keras.Sequential([
    keras.layers.Dense(64, activation='relu', input_shape=(4,)),
    keras.layers.Dense(32, activation='relu'),
    keras.layers.Dense(3, activation='softmax')
])
model.compile(optimizer='adam', loss='categorical_crossentropy')

# Train (simplified)
X_train = np.random.randn(1000, 4)
y_train = keras.utils.to_categorical(np.random.randint(0, 3, 1000))
model.fit(X_train, y_train, epochs=10)

# Save in SavedModel format (required for TF Serving)
# Version number in path is CRITICAL - TF Serving uses it
export_path = "models/iris_classifier/1"  # "1" is the version number
model.save(export_path)

print(f"Model saved to: {export_path}")
print(f"Model signature: {model.signatures if hasattr(model, 'signatures') else 'default'}")
```

### Directory Structure for TF Serving

```
models/
└── iris_classifier/          # Model name
    ├── 1/                    # Version 1
    │   ├── saved_model.pb
    │   └── variables/
    │       ├── variables.data-00000-of-00001
    │       └── variables.index
    └── 2/                    # Version 2 (latest served automatically)
        ├── saved_model.pb
        └── variables/
```

### Running TF Serving with Docker

```bash
# Pull TF Serving image
docker pull tensorflow/serving:latest

# Run with your model
docker run -p 8501:8501 \
    -v "$(pwd)/models/iris_classifier:/models/iris_classifier" \
    -e MODEL_NAME=iris_classifier \
    tensorflow/serving:latest

# With GPU support
docker run --gpus all -p 8501:8501 \
    -v "$(pwd)/models/iris_classifier:/models/iris_classifier" \
    -e MODEL_NAME=iris_classifier \
    tensorflow/serving:latest-gpu
```

### TF Serving Configuration (Advanced)

```protobuf
# model_config.list - Serve multiple models
model_config_list {
  config {
    name: 'iris_classifier'
    base_path: '/models/iris_classifier'
    model_platform: 'tensorflow'
    model_version_policy {
      specific {
        versions: 1
        versions: 2
      }
    }
  }
  config {
    name: 'sentiment_model'
    base_path: '/models/sentiment'
    model_platform: 'tensorflow'
  }
}
```

```bash
# Run with config file
docker run -p 8501:8501 \
    -v "$(pwd)/models:/models" \
    -v "$(pwd)/model_config.list:/models/model_config.list" \
    tensorflow/serving \
    --model_config_file=/models/model_config.list
```

### Calling TF Serving

```python
# tf_serving_client.py - REST and gRPC clients
import requests
import json
import numpy as np

# --- REST API ---
def predict_rest(instances):
    """Call TF Serving via REST API"""
    url = "http://localhost:8501/v1/models/iris_classifier:predict"
    
    payload = {
        "instances": instances  # List of input arrays
    }
    
    response = requests.post(url, json=payload)
    return response.json()

# Single prediction
result = predict_rest([[5.1, 3.5, 1.4, 0.2]])
print(result)
# Output: {"predictions": [[0.97, 0.02, 0.01]]}

# Batch prediction
batch = [[5.1, 3.5, 1.4, 0.2], [6.7, 3.0, 5.2, 2.3]]
result = predict_rest(batch)
print(result)

# --- gRPC Client (faster, binary protocol) ---
import grpc
import tensorflow as tf
from tensorflow_serving.apis import predict_pb2
from tensorflow_serving.apis import prediction_service_pb2_grpc

def predict_grpc(features):
    """Call TF Serving via gRPC (2-3x faster than REST)"""
    # Create gRPC channel
    channel = grpc.insecure_channel('localhost:8500')
    stub = prediction_service_pb2_grpc.PredictionServiceStub(channel)
    
    # Build request
    request = predict_pb2.PredictRequest()
    request.model_spec.name = 'iris_classifier'
    request.model_spec.signature_name = 'serving_default'
    
    # Convert numpy array to tensor proto
    request.inputs['input_1'].CopyFrom(
        tf.make_tensor_proto(features, dtype=tf.float32)
    )
    
    # Make prediction
    result = stub.Predict(request, timeout=10.0)
    
    # Parse response
    output = tf.make_ndarray(result.outputs['output_1'])
    return output

result = predict_grpc(np.array([[5.1, 3.5, 1.4, 0.2]]))
print(f"Prediction: {result}")
```

### Batching Configuration

```
# batching_config.txt - Auto-batching for GPU efficiency
max_batch_size { value: 32 }
batch_timeout_micros { value: 5000 }    # Wait 5ms to fill batch
max_enqueued_batches { value: 100 }
num_batch_threads { value: 4 }
```

```bash
docker run -p 8501:8501 \
    -v "$(pwd)/models:/models" \
    tensorflow/serving \
    --model_name=iris_classifier \
    --enable_batching=true \
    --batching_parameters_file=/models/batching_config.txt
```

---

## TorchServe

### What is TorchServe?

TorchServe is PyTorch's official model serving framework. It's like TF Serving but for PyTorch models, with added flexibility for custom pre/post-processing.

### Setting Up TorchServe

```python
# Step 1: Create a custom handler
# handler.py - Defines how to load model and process requests

import torch
import torch.nn as nn
import json
import logging
from ts.torch_handler.base_handler import BaseHandler

logger = logging.getLogger(__name__)

class IrisHandler(BaseHandler):
    """Custom handler for Iris classification model"""
    
    def initialize(self, context):
        """Load model and any other initialization"""
        super().initialize(context)
        self.model.eval()  # Set to inference mode
        logger.info("Model loaded successfully")
    
    def preprocess(self, data):
        """
        Transform raw input into model input
        Input: [{"body": {"features": [5.1, 3.5, 1.4, 0.2]}}]
        Output: torch.Tensor
        """
        inputs = []
        for row in data:
            # Handle different input formats
            if isinstance(row, dict) and 'body' in row:
                body = row['body']
                if isinstance(body, (bytes, bytearray)):
                    body = json.loads(body)
                features = body['features']
            else:
                features = row.get('features', row)
            
            inputs.append(features)
        
        # Convert to tensor
        tensor = torch.FloatTensor(inputs)
        return tensor
    
    def inference(self, data):
        """Run model inference"""
        with torch.no_grad():
            output = self.model(data)
            probabilities = torch.softmax(output, dim=1)
        return probabilities
    
    def postprocess(self, inference_output):
        """
        Transform model output to API response
        Output: List of predictions (one per input)
        """
        results = []
        for probs in inference_output:
            prediction = torch.argmax(probs).item()
            confidence = probs.max().item()
            results.append({
                'prediction': prediction,
                'confidence': round(confidence, 4),
                'probabilities': probs.tolist()
            })
        return results
```

### Packaging the Model

```python
# save_torchserve_model.py
import torch
import torch.nn as nn

# Define model architecture
class IrisNet(nn.Module):
    def __init__(self):
        super().__init__()
        self.layers = nn.Sequential(
            nn.Linear(4, 64),
            nn.ReLU(),
            nn.Linear(64, 32),
            nn.ReLU(),
            nn.Linear(32, 3)
        )
    
    def forward(self, x):
        return self.layers(x)

# Create and save model
model = IrisNet()
# ... train model ...

# Save model state dict
torch.save(model.state_dict(), "iris_model.pth")

# Alternative: Save with TorchScript (recommended for serving)
scripted_model = torch.jit.script(model)
scripted_model.save("iris_model_scripted.pt")
```

```bash
# Create Model Archive (.mar file) using torch-model-archiver
torch-model-archiver \
    --model-name iris_classifier \
    --version 1.0 \
    --model-file model.py \
    --serialized-file iris_model.pth \
    --handler handler.py \
    --export-path model_store/

# For TorchScript model (no need for model-file)
torch-model-archiver \
    --model-name iris_classifier \
    --version 1.0 \
    --serialized-file iris_model_scripted.pt \
    --handler handler.py \
    --export-path model_store/
```

### Running TorchServe

```bash
# Start TorchServe
torchserve --start \
    --model-store model_store \
    --models iris_classifier=iris_classifier.mar \
    --ncs  # No config snapshots

# Check status
curl http://localhost:8080/ping
# Output: {"status": "Healthy"}

# Make prediction
curl -X POST http://localhost:8080/predictions/iris_classifier \
    -H "Content-Type: application/json" \
    -d '{"features": [5.1, 3.5, 1.4, 0.2]}'

# Stop TorchServe
torchserve --stop
```

### TorchServe Configuration

```properties
# config.properties - TorchServe configuration

# Inference settings
inference_address=http://0.0.0.0:8080
management_address=http://0.0.0.0:8081
metrics_address=http://0.0.0.0:8082

# Performance tuning
default_workers_per_model=4
job_queue_size=100
batch_size=8
max_batch_delay=100

# GPU settings
number_of_gpu=1

# Model settings
default_response_timeout=120
load_models=all
```

---

## NVIDIA Triton Inference Server

### What is Triton?

Triton is NVIDIA's enterprise-grade inference server. Think of it as the "Formula 1 race car" of model serving — incredibly fast but requires more setup. It supports multiple frameworks (TensorFlow, PyTorch, ONNX, TensorRT) simultaneously.

### Key Features

```
┌───────────────────────────────────────────────────┐
│              NVIDIA Triton Server                   │
├───────────────────────────────────────────────────┤
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────┐│
│  │TensorFlow│ │ PyTorch │ │  ONNX   │ │TensorRT││
│  │ Backend  │ │ Backend │ │ Backend │ │Backend ││
│  └─────────┘ └─────────┘ └─────────┘ └────────┘│
├───────────────────────────────────────────────────┤
│  • Dynamic Batching        • Model Ensemble       │
│  • Concurrent Model Exec   • GPU/CPU Scheduling   │
│  • Model Versioning        • Metrics (Prometheus) │
│  • gRPC & HTTP             • Shared Memory        │
└───────────────────────────────────────────────────┘
```

### Model Repository Structure

```
model_repository/
├── iris_tensorflow/
│   ├── config.pbtxt
│   ├── 1/
│   │   └── model.savedmodel/
│   └── 2/
│       └── model.savedmodel/
├── iris_pytorch/
│   ├── config.pbtxt
│   └── 1/
│       └── model.pt
└── iris_onnx/
    ├── config.pbtxt
    └── 1/
        └── model.onnx
```

### Model Configuration

```protobuf
# config.pbtxt for a PyTorch model
name: "iris_pytorch"
platform: "pytorch_libtorch"
max_batch_size: 32

input [
  {
    name: "INPUT__0"
    data_type: TYPE_FP32
    dims: [4]
  }
]

output [
  {
    name: "OUTPUT__0"
    data_type: TYPE_FP32
    dims: [3]
  }
]

# Dynamic batching - groups requests for efficiency
dynamic_batching {
  preferred_batch_size: [8, 16, 32]
  max_queue_delay_microseconds: 5000
}

# Instance group - how many model copies to run
instance_group [
  {
    count: 2
    kind: KIND_GPU
    gpus: [0]
  }
]
```

### Running Triton

```bash
# Pull Triton container
docker pull nvcr.io/nvidia/tritonserver:23.10-py3

# Run Triton with model repository
docker run --gpus all -p 8000:8000 -p 8001:8001 -p 8002:8002 \
    -v $(pwd)/model_repository:/models \
    nvcr.io/nvidia/tritonserver:23.10-py3 \
    tritonserver --model-repository=/models

# Ports:
# 8000 - HTTP/REST
# 8001 - gRPC
# 8002 - Metrics (Prometheus)
```

### Triton Client

```python
# triton_client.py - Using Triton's Python client
import tritonclient.http as httpclient
import numpy as np

# Create client
client = httpclient.InferenceServerClient(url="localhost:8000")

# Check server health
print(f"Server ready: {client.is_server_ready()}")
print(f"Model ready: {client.is_model_ready('iris_pytorch')}")

# Prepare input
input_data = np.array([[5.1, 3.5, 1.4, 0.2]], dtype=np.float32)

# Create input tensor
inputs = [
    httpclient.InferInput("INPUT__0", input_data.shape, "FP32")
]
inputs[0].set_data_from_numpy(input_data)

# Create output request
outputs = [
    httpclient.InferRequestedOutput("OUTPUT__0")
]

# Make inference request
result = client.infer(
    model_name="iris_pytorch",
    inputs=inputs,
    outputs=outputs
)

# Get result
output = result.as_numpy("OUTPUT__0")
print(f"Prediction probabilities: {output}")
print(f"Predicted class: {np.argmax(output)}")
```

### Model Ensemble (Pipeline)

```protobuf
# config.pbtxt - Ensemble model (preprocessing → model → postprocessing)
name: "iris_pipeline"
platform: "ensemble"
max_batch_size: 32

input [
  {
    name: "RAW_INPUT"
    data_type: TYPE_FP32
    dims: [4]
  }
]
output [
  {
    name: "FINAL_OUTPUT"
    data_type: TYPE_FP32
    dims: [3]
  }
]

ensemble_scheduling {
  step [
    {
      model_name: "preprocessor"
      model_version: -1
      input_map {
        key: "INPUT"
        value: "RAW_INPUT"
      }
      output_map {
        key: "OUTPUT"
        value: "preprocessed"
      }
    },
    {
      model_name: "iris_pytorch"
      model_version: -1
      input_map {
        key: "INPUT__0"
        value: "preprocessed"
      }
      output_map {
        key: "OUTPUT__0"
        value: "FINAL_OUTPUT"
      }
    }
  ]
}
```

---

## Batch vs Real-Time Inference

### Comparison

```
Real-Time (Online) Inference          Batch (Offline) Inference
┌──────────────────────────┐          ┌──────────────────────────┐
│  Request → Predict → Respond │      │  Dataset → Predict All   │
│  Latency: < 100ms        │          │  Latency: Hours OK       │
│  One-at-a-time            │          │  Millions at once         │
│  Always running           │          │  Scheduled (cron/airflow) │
└──────────────────────────┘          └──────────────────────────┘
```

| Aspect | Real-Time | Batch |
|--------|-----------|-------|
| **Latency** | < 100ms | Hours acceptable |
| **Throughput** | Hundreds/sec | Millions/run |
| **Cost** | High (always on) | Low (run & shut down) |
| **Freshness** | Instant | Stale (last batch run) |
| **Use Cases** | Fraud detection, search ranking | Recommendations, reports |
| **Infrastructure** | Kubernetes, auto-scaling | Spark, Airflow |

### Batch Inference Implementation

```python
# batch_inference.py - Efficient batch prediction
import pandas as pd
import numpy as np
import joblib
from pathlib import Path
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def run_batch_inference(
    input_path: str,
    output_path: str,
    model_path: str,
    batch_size: int = 10000
):
    """
    Run batch inference on a large dataset.
    
    Processes data in chunks to avoid memory issues.
    """
    # Load model
    model = joblib.load(model_path)
    logger.info(f"Model loaded from {model_path}")
    
    # Process in chunks (memory efficient)
    results = []
    total_processed = 0
    
    for chunk in pd.read_csv(input_path, chunksize=batch_size):
        # Preprocess chunk
        features = chunk.drop(columns=['id'], errors='ignore')
        
        # Predict
        predictions = model.predict(features)
        probabilities = model.predict_proba(features)
        
        # Append results
        chunk['prediction'] = predictions
        chunk['confidence'] = probabilities.max(axis=1)
        results.append(chunk)
        
        total_processed += len(chunk)
        logger.info(f"Processed {total_processed} records")
    
    # Save results
    output_df = pd.concat(results)
    output_df.to_csv(output_path, index=False)
    logger.info(f"Results saved to {output_path}")
    
    return total_processed

# Run batch inference
if __name__ == "__main__":
    run_batch_inference(
        input_path="data/new_customers.csv",
        output_path="data/predictions.csv",
        model_path="models/classifier.joblib",
        batch_size=10000
    )
```

### Hybrid Pattern (Pre-compute + Real-Time Fallback)

```python
# hybrid_serving.py - Best of both worlds
import redis
import joblib
import numpy as np
from fastapi import FastAPI

app = FastAPI()
model = joblib.load("model.joblib")
cache = redis.Redis(host='localhost', port=6379)

@app.post("/predict/{user_id}")
async def predict(user_id: str, features: dict):
    """
    Hybrid pattern:
    1. Check cache for pre-computed prediction (from batch job)
    2. If not found, compute in real-time
    """
    # Try cache first (pre-computed by batch job)
    cached = cache.get(f"prediction:{user_id}")
    
    if cached:
        return {"prediction": int(cached), "source": "batch_cache"}
    
    # Fallback to real-time inference
    X = np.array(features['values']).reshape(1, -1)
    prediction = model.predict(X)[0]
    
    # Cache for future requests (TTL: 1 hour)
    cache.setex(f"prediction:{user_id}", 3600, int(prediction))
    
    return {"prediction": int(prediction), "source": "real_time"}
```

---

## Scaling Model Serving

### Horizontal Scaling Architecture

```
                    ┌──────────────────┐
                    │  Load Balancer   │
                    │  (Nginx/HAProxy) │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
       ┌──────▼──────┐ ┌────▼────┐ ┌──────▼──────┐
       │  Instance 1 │ │Instance2│ │  Instance 3 │
       │  (GPU)      │ │ (GPU)   │ │  (GPU)      │
       └─────────────┘ └─────────┘ └─────────────┘
```

### Auto-Scaling with Kubernetes

```yaml
# deployment.yaml - K8s deployment for model server
apiVersion: apps/v1
kind: Deployment
metadata:
  name: model-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: model-server
  template:
    metadata:
      labels:
        app: model-server
    spec:
      containers:
      - name: model-server
        image: my-model-server:v1.0
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
            nvidia.com/gpu: 1
          limits:
            memory: "4Gi"
            cpu: "2000m"
            nvidia.com/gpu: 1
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 60
          periodSeconds: 30
---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: model-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: model-server
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: inference_latency_p95
      target:
        type: AverageValue
        averageValue: "100m"  # 100ms
```

### Performance Optimization Techniques

```python
# optimization.py - Techniques to speed up inference

import numpy as np
import onnxruntime as ort
import torch

# --- 1. ONNX Runtime (2-5x speedup over native PyTorch/TF) ---
def optimize_with_onnx(model_path: str):
    """Load model with ONNX Runtime for faster inference"""
    # Create session with optimization
    session_options = ort.SessionOptions()
    session_options.graph_optimization_level = ort.GraphOptimizationLevel.ORT_ENABLE_ALL
    session_options.intra_op_num_threads = 4
    
    session = ort.InferenceSession(
        model_path,
        sess_options=session_options,
        providers=['CUDAExecutionProvider', 'CPUExecutionProvider']
    )
    
    # Run inference
    input_name = session.get_inputs()[0].name
    input_data = np.random.randn(1, 4).astype(np.float32)
    
    result = session.run(None, {input_name: input_data})
    return result

# --- 2. Model Warmup (avoid cold start) ---
def warmup_model(model, input_shape, n_warmup=10):
    """
    Run dummy predictions to warm up the model.
    First inference is always slow (JIT compilation, memory allocation).
    """
    dummy_input = torch.randn(1, *input_shape)
    
    with torch.no_grad():
        for _ in range(n_warmup):
            _ = model(dummy_input)
    
    print("Model warmed up!")

# --- 3. Input Batching for Throughput ---
class DynamicBatcher:
    """
    Collects individual requests and batches them for efficient GPU usage.
    
    Why: GPU is most efficient processing many inputs at once.
    A single input uses ~5% of GPU, batching 32 uses ~80%.
    """
    
    def __init__(self, model, max_batch_size=32, max_wait_ms=50):
        self.model = model
        self.max_batch_size = max_batch_size
        self.max_wait_ms = max_wait_ms
        self.queue = []
    
    async def predict(self, features):
        """Add to batch queue and wait for result"""
        import asyncio
        
        future = asyncio.Future()
        self.queue.append((features, future))
        
        # If batch is full, process immediately
        if len(self.queue) >= self.max_batch_size:
            await self._process_batch()
        
        return await future
    
    async def _process_batch(self):
        """Process all queued requests as a single batch"""
        if not self.queue:
            return
        
        # Extract features and futures
        batch_features = [item[0] for item in self.queue]
        futures = [item[1] for item in self.queue]
        self.queue = []
        
        # Batch inference (single GPU call)
        batch_tensor = torch.FloatTensor(batch_features)
        with torch.no_grad():
            results = self.model(batch_tensor)
        
        # Distribute results back to individual requests
        for i, future in enumerate(futures):
            future.set_result(results[i].tolist())
```

---

## Common Mistakes

### 1. Loading Model on Every Request

```python
# ❌ WRONG - Model loaded fresh for every single request (SLOW!)
@app.post("/predict")
def predict(data: dict):
    model = joblib.load("model.pkl")  # 500ms+ every request!
    return model.predict(data['features'])

# ✅ CORRECT - Load once at startup, reuse for all requests
model = joblib.load("model.pkl")  # Loaded once

@app.post("/predict")
def predict(data: dict):
    return model.predict(data['features'])  # <1ms inference
```

### 2. Not Handling Model Version Mismatches

```python
# ❌ WRONG - No version tracking
@app.post("/predict")
def predict(data: dict):
    return {"prediction": model.predict(data)}

# ✅ CORRECT - Include model version in response
MODEL_VERSION = "2.1.0"
MODEL_FEATURES = ["age", "income", "score", "tenure"]

@app.post("/predict")
def predict(data: dict):
    # Validate input matches expected features
    if set(data.keys()) != set(MODEL_FEATURES):
        raise HTTPException(400, f"Expected features: {MODEL_FEATURES}")
    
    return {
        "prediction": model.predict(data),
        "model_version": MODEL_VERSION
    }
```

### 3. No Input Validation

```python
# ❌ WRONG - Accepts any garbage input
@app.post("/predict")
def predict(data: dict):
    return model.predict(data['features'])  # Will crash on bad input

# ✅ CORRECT - Validate everything
from pydantic import BaseModel, validator

class PredictRequest(BaseModel):
    features: List[float]
    
    @validator('features')
    def validate_features(cls, v):
        if len(v) != 4:
            raise ValueError(f"Expected 4 features, got {len(v)}")
        if any(np.isnan(x) or np.isinf(x) for x in v):
            raise ValueError("Features cannot contain NaN or Inf")
        return v
```

### 4. Ignoring Memory Leaks

```python
# ❌ WRONG - Accumulates data in memory forever
predictions_log = []  # Grows unbounded!

@app.post("/predict")
def predict(data: dict):
    result = model.predict(data)
    predictions_log.append(result)  # Memory leak!
    return result

# ✅ CORRECT - Use bounded collections or external storage
from collections import deque

predictions_log = deque(maxlen=10000)  # Bounded

@app.post("/predict")
def predict(data: dict):
    result = model.predict(data)
    # Log to external system (not in memory)
    logger.info(f"Prediction: {result}")
    return result
```

### 5. No Health Checks or Graceful Shutdown

```python
# ✅ CORRECT - Proper health check with model validation
@app.get("/health")
async def health():
    """Deep health check - verifies model can actually predict"""
    try:
        # Test with known input/output pair
        test_input = np.array([[5.1, 3.5, 1.4, 0.2]])
        result = model.predict(test_input)
        
        if result is None:
            return JSONResponse(
                status_code=503,
                content={"status": "unhealthy", "reason": "model returned None"}
            )
        
        return {"status": "healthy", "model_loaded": True}
    except Exception as e:
        return JSONResponse(
            status_code=503,
            content={"status": "unhealthy", "reason": str(e)}
        )
```

---

## Interview Questions

### Conceptual Questions

**Q1: What's the difference between model serving and model deployment?**
> **A:** Deployment is the broader process of taking a model to production (infrastructure, CI/CD, monitoring). Serving is specifically about making predictions available via an API or interface. Serving is a subset of deployment.

**Q2: When would you choose gRPC over REST for model serving?**
> **A:** gRPC when: (1) internal service-to-service communication, (2) high throughput needed (2-3x faster), (3) streaming predictions, (4) strong typing required. REST when: (1) external/public APIs, (2) browser clients, (3) simpler debugging, (4) wider ecosystem support.

**Q3: How do you handle model updates without downtime?**
> **A:** Blue-green deployment (run old and new simultaneously, switch traffic), canary deployment (route 5% traffic to new model, monitor, gradually increase), or shadow deployment (send traffic to both, only serve old model's response, compare results).

**Q4: What's the cold start problem in model serving?**
> **A:** First request takes much longer because: model needs to be loaded into memory/GPU, frameworks perform JIT compilation, caches are empty. Solutions: model warmup on startup, keep minimum replicas always running, use smaller models for first response.

### System Design Questions

**Q5: Design a model serving system that handles 10,000 requests per second.**
> **A:** 
> - Load balancer (Nginx/ALB) distributing across multiple instances
> - Auto-scaling based on CPU/latency metrics (K8s HPA)
> - Dynamic batching to maximize GPU utilization
> - Model optimization (ONNX/TensorRT for 2-5x speedup)
> - Response caching (Redis) for repeated inputs
> - Async processing for non-critical predictions
> - Consider: gRPC over REST, model sharding for very large models

**Q6: How do you serve a 10GB model with < 50ms latency?**
> **A:** 
> - Quantize to INT8 (4x smaller, 2x faster)
> - Use TensorRT for GPU optimization
> - Distill into smaller student model
> - Split model across GPUs (model parallelism)
> - Pre-compute embeddings where possible
> - Use shared memory to avoid data copying

### Coding Questions

**Q7: Write a model serving endpoint with proper error handling, input validation, and response caching.**

```python
# See the FastAPI section above for a complete implementation
# Key points interviewers look for:
# 1. Pydantic models for validation
# 2. Try/except with proper HTTP error codes
# 3. Health check endpoint
# 4. Model loaded at startup (not per-request)
# 5. Logging for debugging
# 6. Async support for throughput
```

---

## Quick Reference

### Serving Framework Comparison

| Framework | Best For | Language | GPU | Multi-Framework | Complexity |
|-----------|----------|----------|-----|-----------------|------------|
| Flask | Prototyping | Python | ❌ | ✅ | Low |
| FastAPI | Production Python | Python | ❌ | ✅ | Low-Med |
| TF Serving | TensorFlow models | C++ | ✅ | ❌ (TF only) | Medium |
| TorchServe | PyTorch models | Java/Python | ✅ | ❌ (PyTorch) | Medium |
| Triton | Enterprise/Multi-model | C++ | ✅ | ✅ | High |
| BentoML | End-to-end | Python | ✅ | ✅ | Low-Med |
| Ray Serve | Distributed | Python | ✅ | ✅ | Medium |

### Decision Matrix

```
Need simple prototype?           → Flask/FastAPI
Need production Python API?      → FastAPI + Uvicorn
Only TensorFlow models?          → TF Serving
Only PyTorch models?             → TorchServe
Multiple frameworks + GPU?       → Triton
Need auto-scaling + batching?    → Triton or Ray Serve
Fastest time-to-production?      → BentoML
```

### Performance Benchmarks (Approximate)

| Setup | Latency (p50) | Throughput | Notes |
|-------|---------------|------------|-------|
| Flask + Gunicorn | 15-50ms | 200-500 rps | CPU only |
| FastAPI + Uvicorn | 5-20ms | 500-2000 rps | CPU, async |
| TF Serving (GPU) | 2-10ms | 1000-5000 rps | Batching helps |
| TorchServe (GPU) | 3-15ms | 800-3000 rps | Custom handlers |
| Triton (GPU) | 1-5ms | 2000-10000 rps | Full optimization |

### Essential Commands Cheatsheet

```bash
# FastAPI
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4

# TF Serving
docker run -p 8501:8501 -v ./models:/models -e MODEL_NAME=my_model tensorflow/serving

# TorchServe
torch-model-archiver --model-name m --handler h.py --serialized-file model.pt
torchserve --start --model-store store --models m=m.mar

# Triton
docker run --gpus all -p 8000:8000 -v ./models:/models nvcr.io/nvidia/tritonserver tritonserver --model-repository=/models

# Testing
curl -X POST http://localhost:8000/predict -H "Content-Type: application/json" -d '{"features":[1,2,3,4]}'
```

### Key Metrics to Monitor

| Metric | Target | Alert If |
|--------|--------|----------|
| Latency (p99) | < 100ms | > 500ms |
| Error Rate | < 0.1% | > 1% |
| Throughput | Varies | Drop > 20% |
| GPU Utilization | > 60% | < 20% (waste) or > 95% (bottleneck) |
| Memory Usage | < 80% | > 90% |
| Model Load Time | < 30s | > 60s |

---

> **Pro Tip:** Start with FastAPI for rapid iteration, then graduate to Triton/TF Serving when you need raw performance. Most companies don't need Triton until they're serving millions of requests per day.
