# Chapter 09: TF Serving and Deployment

## Table of Contents
- [What is Model Deployment?](#what-is-model-deployment)
- [Why Deployment Matters](#why-deployment-matters)
- [SavedModel Format](#savedmodel-format)
- [TensorFlow Serving](#tensorflow-serving)
- [TensorFlow Lite (TFLite)](#tensorflow-lite-tflite)
- [TensorFlow.js](#tensorflowjs)
- [Model Optimization](#model-optimization)
- [End-to-End Deployment Patterns](#end-to-end-deployment-patterns)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Model Deployment?

### Simple Explanation

Building a machine learning model is like writing a song in your bedroom. **Deployment** is putting that song on Spotify so millions of people can listen to it. Your model is useless sitting in a Jupyter notebook — deployment makes it available to users, apps, and systems in the real world.

```
The ML Lifecycle:

Data → Train → Evaluate → DEPLOY → Monitor → Retrain
                            ↑
                     YOU ARE HERE
                     
Without deployment:
┌─────────────────────────────┐
│  Jupyter Notebook           │
│  "My model gets 99% acc!"  │
│  But nobody can use it...   │
└─────────────────────────────┘

With deployment:
┌─────────────────────────────┐
│  REST API / Mobile App      │
│  10,000 predictions/second  │
│  Available 24/7 worldwide   │
└─────────────────────────────┘
```

### Deployment Targets

```
Where can TensorFlow models run?

┌────────────────────────────────────────────────────────┐
│                    TensorFlow Model                     │
├──────────┬───────────┬──────────┬──────────┬───────────┤
│  Server  │  Mobile   │ Browser  │  Edge    │  Cloud    │
│          │           │          │          │           │
│ TF       │ TF Lite   │ TF.js    │ TF Lite  │ Vertex AI │
│ Serving  │ Android   │ WebGL    │ Coral    │ SageMaker │
│          │ iOS       │ WebGPU   │ RPi      │ Azure ML  │
│          │ Flutter   │ WASM     │ Arduino  │           │
└──────────┴───────────┴──────────┴──────────┴───────────┘
```

---

## Why Deployment Matters

### The Gap Between Training and Production

| Aspect | Training | Production |
|--------|----------|------------|
| Batch size | Large (32-512) | Single or small (1-8) |
| Latency | Doesn't matter | Critical (< 100ms) |
| Throughput | Not priority | 1000s of req/sec |
| Hardware | GPU clusters | CPU, mobile, edge |
| Data | Clean, preprocessed | Raw, messy, diverse |
| Errors | Retry, debug | Must handle gracefully |
| Versioning | "latest checkpoint" | A/B testing, rollbacks |
| Memory | 32GB+ GPU | 2GB mobile phone |

### Real-World Deployment Scenarios

- **E-commerce**: Product recommendation in < 50ms per request
- **Self-driving cars**: Object detection at 30 FPS on embedded GPU
- **Healthcare**: X-ray analysis on hospital servers (HIPAA compliant)
- **Mobile**: Photo filters running on-device (no internet needed)
- **Web**: Sentiment analysis running in the browser (zero server cost)

---

## SavedModel Format

### What is SavedModel?

SavedModel is TensorFlow's **universal serialization format**. It contains everything needed to run a model without the original Python code:

```
saved_model/
├── saved_model.pb          ← Computation graph (the model's architecture + ops)
├── fingerprint.pb          ← Model fingerprint for integrity
├── variables/
│   ├── variables.data-00000-of-00001  ← Actual weight values
│   └── variables.index                 ← Index for weight lookup
└── assets/                 ← Extra files (vocabularies, etc.)
    └── vocab.txt
```

### Saving a Model

```python
import tensorflow as tf

# ============================================
# Method 1: Save entire model (recommended)
# ============================================
model = tf.keras.Sequential([
    tf.keras.layers.Dense(128, activation='relu', input_shape=(784,)),
    tf.keras.layers.Dense(10, activation='softmax')
])
model.compile(optimizer='adam', loss='sparse_categorical_crossentropy')

# Train the model...
# model.fit(x_train, y_train, epochs=5)

# Save as SavedModel
model.save('my_model')  # Creates a directory 'my_model/'

# Or explicitly specify format
model.save('my_model', save_format='tf')  # SavedModel format (default)
# model.save('my_model.keras')  # Keras native format (TF 2.16+)

# ============================================
# Method 2: Save with custom signatures
# ============================================
# Signatures define the input/output interface for serving

class MyModule(tf.Module):
    def __init__(self):
        super().__init__()
        self.model = tf.keras.Sequential([
            tf.keras.layers.Dense(128, activation='relu', input_shape=(784,)),
            tf.keras.layers.Dense(10, activation='softmax')
        ])
    
    @tf.function(input_signature=[tf.TensorSpec(shape=[None, 784], dtype=tf.float32)])
    def predict(self, x):
        """Explicit signature: tells TF Serving exactly what inputs to expect."""
        return self.model(x, training=False)
    
    @tf.function(input_signature=[tf.TensorSpec(shape=[None, 784], dtype=tf.float32)])
    def predict_with_confidence(self, x):
        """Another signature for a different serving endpoint."""
        probs = self.model(x, training=False)
        classes = tf.argmax(probs, axis=-1)
        confidence = tf.reduce_max(probs, axis=-1)
        return {'class': classes, 'confidence': confidence}

module = MyModule()

# Save with named signatures
tf.saved_model.save(module, 'my_module', signatures={
    'serving_default': module.predict,
    'predict_with_confidence': module.predict_with_confidence
})
```

### Loading a Model

```python
# ============================================
# Method 1: Load as Keras model (full functionality)
# ============================================
loaded_model = tf.keras.models.load_model('my_model')
predictions = loaded_model.predict(x_test)

# Can continue training!
loaded_model.fit(x_train, y_train, epochs=5)

# ============================================
# Method 2: Load as SavedModel (for serving)
# ============================================
loaded = tf.saved_model.load('my_module')

# Use the default signature
result = loaded.predict(tf.constant(x_test[:5], dtype=tf.float32))
print(result.shape)  # (5, 10)

# Use the custom signature
result = loaded.predict_with_confidence(tf.constant(x_test[:5], dtype=tf.float32))
print(f"Classes: {result['class'].numpy()}")
print(f"Confidence: {result['confidence'].numpy()}")
```

### Inspecting a SavedModel

```python
# Command line tool to inspect SavedModel
# saved_model_cli show --dir my_model --all

# Python inspection
loaded = tf.saved_model.load('my_model')

# List all signatures
if hasattr(loaded, 'signatures'):
    print("Available signatures:", list(loaded.signatures.keys()))

# Check input/output specs
concrete_fn = loaded.signatures['serving_default']
print("Inputs:", concrete_fn.structured_input_signature)
print("Outputs:", concrete_fn.structured_outputs)
```

### SavedModel vs Other Formats

| Format | Extension | Contains | Use Case |
|--------|-----------|----------|----------|
| SavedModel | Directory | Graph + Weights + Signatures | Production serving |
| Keras Native | `.keras` | Architecture + Weights + Optimizer | Continue training |
| HDF5 (Legacy) | `.h5` | Architecture + Weights | Legacy compatibility |
| Weights Only | `.weights.h5` | Just weights | Transfer learning |
| ONNX | `.onnx` | Cross-framework graph | Framework interop |

> **Pro Tip**: Always use SavedModel for production deployment. Use `.keras` for saving/resuming training. Avoid HDF5 `.h5` for new projects (it's legacy).

---

## TensorFlow Serving

### What is TF Serving?

TensorFlow Serving is a high-performance serving system designed for production ML models. It handles:
- **Model versioning**: Serve multiple versions simultaneously
- **Hot-swapping**: Update models without downtime
- **Batching**: Group requests for GPU efficiency
- **REST & gRPC APIs**: Two protocols for different needs
- **Monitoring**: Health checks and metrics

```
Client Request Flow:

Client App                    TF Serving                    Model
┌─────────┐   HTTP/gRPC    ┌───────────────┐           ┌─────────┐
│  Send    │ ──────────── > │  Deserialize  │           │         │
│  Image   │                │  Input        │ ────────> │ Predict │
│          │                │               │           │         │
│  Receive │ < ──────────── │  Serialize    │ <──────── │ Return  │
│  Result  │   JSON/Proto   │  Output       │           │ Probs   │
└─────────┘                 └───────────────┘           └─────────┘
```

### Setting Up TF Serving

```bash
# ============================================
# Method 1: Docker (Recommended)
# ============================================

# Pull the TF Serving Docker image
docker pull tensorflow/serving

# Serve your model
# Directory structure must be:
# models/
#   my_model/
#     1/          ← Version 1
#       saved_model.pb
#       variables/
#     2/          ← Version 2 (optional)
#       saved_model.pb
#       variables/

docker run -p 8501:8501 \
  -v "$(pwd)/models/my_model:/models/my_model" \
  -e MODEL_NAME=my_model \
  tensorflow/serving

# The server is now running!
# REST API: http://localhost:8501/v1/models/my_model:predict
# gRPC:     localhost:8500
```

### Preparing a Model for Serving

```python
import tensorflow as tf
import os

# Build and train your model
model = tf.keras.Sequential([
    tf.keras.layers.Dense(128, activation='relu', input_shape=(784,)),
    tf.keras.layers.Dropout(0.3),
    tf.keras.layers.Dense(10, activation='softmax')
])
model.compile(optimizer='adam', loss='sparse_categorical_crossentropy', metrics=['accuracy'])
# model.fit(x_train, y_train, epochs=10)

# ============================================
# Save with version directory structure
# ============================================
version = 1
export_path = os.path.join('models', 'mnist_classifier', str(version))
model.save(export_path)

print(f"Model saved to: {export_path}")
# models/mnist_classifier/1/saved_model.pb
# models/mnist_classifier/1/variables/...

# To deploy a new version, save as version 2
# TF Serving will automatically detect and load it!
version = 2
export_path_v2 = os.path.join('models', 'mnist_classifier', str(version))
# improved_model.save(export_path_v2)
```

### Making Predictions via REST API

```python
import requests
import json
import numpy as np

# ============================================
# REST API (HTTP/JSON) — Simple but slower
# ============================================

# Prepare input data
image = np.random.rand(1, 784).tolist()  # Must be a Python list for JSON

# Make prediction request
url = "http://localhost:8501/v1/models/mnist_classifier:predict"
payload = {
    "signature_name": "serving_default",
    "instances": image  # Can send multiple instances as a list
}

response = requests.post(url, json=payload)
result = response.json()

predictions = result['predictions'][0]
predicted_class = np.argmax(predictions)
confidence = max(predictions)

print(f"Predicted class: {predicted_class}")
print(f"Confidence: {confidence:.4f}")

# ============================================
# Check model status
# ============================================
status_url = "http://localhost:8501/v1/models/mnist_classifier"
status = requests.get(status_url).json()
print(json.dumps(status, indent=2))
# {
#   "model_version_status": [
#     {
#       "version": "1",
#       "state": "AVAILABLE",
#       "status": { "error_code": "OK" }
#     }
#   ]
# }

# ============================================
# Get model metadata (input/output specs)
# ============================================
metadata_url = "http://localhost:8501/v1/models/mnist_classifier/metadata"
metadata = requests.get(metadata_url).json()
print(json.dumps(metadata, indent=2))
```

### Making Predictions via gRPC (Faster)

```python
# pip install grpcio tensorflow-serving-api

import grpc
import numpy as np
import tensorflow as tf
from tensorflow_serving.apis import predict_pb2
from tensorflow_serving.apis import prediction_service_pb2_grpc

# ============================================
# gRPC API — Faster (binary protocol, no JSON overhead)
# ============================================

# Create gRPC channel and stub
channel = grpc.insecure_channel('localhost:8500')
stub = prediction_service_pb2_grpc.PredictionServiceStub(channel)

# Create prediction request
request = predict_pb2.PredictRequest()
request.model_spec.name = 'mnist_classifier'
request.model_spec.signature_name = 'serving_default'

# Set input data
input_data = np.random.rand(1, 784).astype(np.float32)
request.inputs['flatten_input'].CopyFrom(
    tf.make_tensor_proto(input_data, shape=[1, 784])
)

# Make prediction
response = stub.Predict(request, timeout=10.0)

# Parse response
output_tensor = response.outputs['dense_1']
predictions = tf.make_ndarray(output_tensor)
print(f"Predicted class: {np.argmax(predictions)}")

# ============================================
# Batch predictions via gRPC
# ============================================
batch_data = np.random.rand(100, 784).astype(np.float32)
request.inputs['flatten_input'].CopyFrom(
    tf.make_tensor_proto(batch_data, shape=[100, 784])
)
response = stub.Predict(request, timeout=30.0)
```

### REST vs gRPC

| Feature | REST (HTTP/JSON) | gRPC (Protocol Buffers) |
|---------|-----------------|------------------------|
| Speed | Slower (~2-5x) | Faster |
| Payload format | JSON (text, larger) | Protobuf (binary, compact) |
| Ease of use | Very easy | Requires protobuf setup |
| Browser support | Yes | Limited (needs grpc-web) |
| Streaming | No | Yes |
| Best for | Simple apps, testing | Production, high throughput |

### TF Serving Configuration

```python
# model_config.config — Serve multiple models with one TF Serving instance

"""
model_config_list {
  config {
    name: "mnist_classifier"
    base_path: "/models/mnist_classifier"
    model_platform: "tensorflow"
    model_version_policy {
      specific {
        versions: 1
        versions: 2
      }
    }
  }
  config {
    name: "image_classifier"
    base_path: "/models/image_classifier"
    model_platform: "tensorflow"
  }
}
"""

# Docker command with config file:
# docker run -p 8501:8501 \
#   -v "$(pwd)/models:/models" \
#   -v "$(pwd)/model_config.config:/models/model_config.config" \
#   tensorflow/serving \
#   --model_config_file=/models/model_config.config

# ============================================
# Batching configuration for better GPU utilization
# ============================================
"""
# batching_config.txt
max_batch_size { value: 128 }
batch_timeout_micros { value: 10000 }
max_enqueued_batches { value: 1000 }
num_batch_threads { value: 8 }
"""

# Docker command with batching:
# docker run -p 8501:8501 \
#   -v "$(pwd)/models:/models" \
#   -e MODEL_NAME=my_model \
#   tensorflow/serving \
#   --enable_batching=true \
#   --batching_parameters_file=/models/batching_config.txt
```

---

## TensorFlow Lite (TFLite)

### What is TFLite?

TensorFlow Lite is a lightweight version of TensorFlow for **mobile and edge devices**. It converts full TensorFlow models into a compact, optimized format (.tflite).

```
Full Model vs TFLite:

Full TensorFlow Model:              TFLite Model:
┌──────────────────────┐            ┌──────────────────────┐
│  Size: 100 MB        │    ──>     │  Size: 25 MB         │
│  Ops: 1000+          │  Convert   │  Ops: ~130 optimized │
│  Needs: TF runtime   │            │  Needs: Lite runtime │
│  Target: Server/GPU  │            │  Target: Phone/Edge  │
│  Latency: 50ms (GPU) │            │  Latency: 20ms (CPU) │
└──────────────────────┘            └──────────────────────┘
```

### Converting a Model to TFLite

```python
import tensorflow as tf

# ============================================
# Step 1: Train your model normally
# ============================================
model = tf.keras.Sequential([
    tf.keras.layers.Conv2D(32, 3, activation='relu', input_shape=(28, 28, 1)),
    tf.keras.layers.MaxPooling2D(),
    tf.keras.layers.Conv2D(64, 3, activation='relu'),
    tf.keras.layers.MaxPooling2D(),
    tf.keras.layers.Flatten(),
    tf.keras.layers.Dense(128, activation='relu'),
    tf.keras.layers.Dense(10, activation='softmax')
])
model.compile(optimizer='adam', loss='sparse_categorical_crossentropy', metrics=['accuracy'])
# model.fit(x_train, y_train, epochs=5)

# ============================================
# Step 2: Convert to TFLite
# ============================================

# Method 1: Basic conversion (float32, no optimization)
converter = tf.lite.TFLiteConverter.from_keras_model(model)
tflite_model = converter.convert()

# Save the .tflite file
with open('model.tflite', 'wb') as f:
    f.write(tflite_model)

print(f"Model size: {len(tflite_model) / 1024:.1f} KB")

# Method 2: From SavedModel
# converter = tf.lite.TFLiteConverter.from_saved_model('saved_model_dir')
# tflite_model = converter.convert()
```

### Quantization (Making Models Even Smaller)

```python
import tensorflow as tf
import numpy as np

# ============================================
# Dynamic Range Quantization (Easiest, ~4x smaller)
# ============================================
# Quantizes weights from float32 → int8 at conversion time
# Activations remain float32 at runtime

converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]

tflite_quantized = converter.convert()

with open('model_dynamic_quant.tflite', 'wb') as f:
    f.write(tflite_quantized)

print(f"Original size: {len(tflite_model) / 1024:.1f} KB")
print(f"Quantized size: {len(tflite_quantized) / 1024:.1f} KB")
# Typically 3-4x smaller!

# ============================================
# Full Integer Quantization (Smallest, needs calibration data)
# ============================================
# Both weights AND activations are int8
# Requires a representative dataset for calibration

def representative_dataset():
    """Yields sample inputs for calibration."""
    for i in range(100):
        # Use real data from your training set!
        sample = np.random.rand(1, 28, 28, 1).astype(np.float32)
        yield [sample]

converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.representative_dataset = representative_dataset

# Force full integer quantization
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type = tf.uint8   # Input as uint8
converter.inference_output_type = tf.uint8  # Output as uint8

tflite_int8 = converter.convert()

with open('model_int8.tflite', 'wb') as f:
    f.write(tflite_int8)

# ============================================
# Float16 Quantization (Good balance)
# ============================================
# Weights as float16, activations as float32
# ~2x smaller, runs fast on GPU delegates

converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.target_spec.supported_types = [tf.float16]

tflite_fp16 = converter.convert()
```

### Quantization Comparison

```
Quantization Types:

Type              | Weights  | Activations | Size    | Speed   | Accuracy
──────────────────|──────────|─────────────|─────────|─────────|──────────
No quantization   | float32  | float32     | 100%    | Baseline| Best
Dynamic range     | int8     | float32     | ~25%    | Faster  | Minimal loss
Float16           | float16  | float32     | ~50%    | Faster* | Minimal loss
Full integer      | int8     | int8        | ~25%    | Fastest | Some loss
                                                     
* Float16 is fastest on GPU delegates (mobile GPUs)
```

### Running TFLite Inference

```python
import tensorflow as tf
import numpy as np

# ============================================
# TFLite Inference in Python
# ============================================

# Load the TFLite model
interpreter = tf.lite.Interpreter(model_path='model.tflite')
interpreter.allocate_tensors()

# Get input and output details
input_details = interpreter.get_input_details()
output_details = interpreter.get_output_details()

print(f"Input shape: {input_details[0]['shape']}")    # [1, 28, 28, 1]
print(f"Input dtype: {input_details[0]['dtype']}")     # float32
print(f"Output shape: {output_details[0]['shape']}")   # [1, 10]

# Prepare input
input_data = np.random.rand(1, 28, 28, 1).astype(np.float32)

# Run inference
interpreter.set_tensor(input_details[0]['index'], input_data)
interpreter.invoke()

# Get output
output_data = interpreter.get_tensor(output_details[0]['index'])
predicted_class = np.argmax(output_data)
confidence = np.max(output_data)

print(f"Predicted class: {predicted_class}")
print(f"Confidence: {confidence:.4f}")

# ============================================
# Benchmark inference speed
# ============================================
import time

num_runs = 100
start = time.time()

for _ in range(num_runs):
    interpreter.set_tensor(input_details[0]['index'], input_data)
    interpreter.invoke()

elapsed = time.time() - start
print(f"Average inference time: {elapsed/num_runs*1000:.2f} ms")
```

### TFLite on Android (Java/Kotlin)

```kotlin
// Add dependency: implementation 'org.tensorflow:tensorflow-lite:2.14.0'

import org.tensorflow.lite.Interpreter

// Load model
val tflite = Interpreter(loadModelFile(activity))

// Prepare input
val input = Array(1) { FloatArray(784) }  // Flatten image
// ... fill with pixel values

// Prepare output
val output = Array(1) { FloatArray(10) }  // 10 classes

// Run inference
tflite.run(input, output)

// Get result
val predictedClass = output[0].indices.maxByOrNull { output[0][it] } ?: -1
```

---

## TensorFlow.js

### What is TF.js?

TensorFlow.js runs ML models directly **in the browser** or in **Node.js**. Zero server needed — computation happens on the user's device.

```
TF.js Execution Backends:

┌──────────────────────────────────────────────┐
│                  TF.js API                    │
├──────────┬───────────┬───────────┬───────────┤
│  WebGL   │  WebGPU   │   WASM    │   CPU     │
│  (GPU)   │  (GPU)    │  (SIMD)   │ (Fallback)│
│  Fast    │  Fastest  │  Medium   │  Slowest  │
│  Most    │  Modern   │  Any      │  Any      │
│  browsers│  browsers │  browser  │  browser  │
└──────────┴───────────┴───────────┴───────────┘
```

### Converting a Model for TF.js

```python
# pip install tensorflowjs

# ============================================
# Method 1: Command-line conversion
# ============================================
# Convert SavedModel to TF.js format
# tensorflowjs_converter \
#     --input_format=tf_saved_model \
#     --output_format=tfjs_graph_model \
#     saved_model_dir/ \
#     tfjs_model/

# Convert Keras .keras file
# tensorflowjs_converter \
#     --input_format=keras \
#     --output_format=tfjs_layers_model \
#     model.keras \
#     tfjs_model/

# With quantization (smaller model for web)
# tensorflowjs_converter \
#     --input_format=tf_saved_model \
#     --output_format=tfjs_graph_model \
#     --quantize_uint8 \
#     saved_model_dir/ \
#     tfjs_model_quantized/

# ============================================
# Method 2: Python API conversion
# ============================================
import tensorflowjs as tfjs

# From Keras model
model = tf.keras.models.load_model('my_model')
tfjs.converters.save_keras_model(model, 'tfjs_model/')

# Output files:
# tfjs_model/
#   model.json         ← Model architecture + weight manifest
#   group1-shard1of1.bin  ← Weight data (binary)
```

### Using TF.js in the Browser

```html
<!-- Include TF.js from CDN -->
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.0.0"></script>

<script>
// ============================================
// Load and run a converted model
// ============================================
async function loadAndPredict() {
    // Load model from URL or local path
    const model = await tf.loadLayersModel('tfjs_model/model.json');
    
    // Prepare input (e.g., from a canvas element)
    const input = tf.tensor2d([[0.1, 0.2, 0.3, /* ... 784 values */]]);
    
    // Run prediction
    const prediction = model.predict(input);
    const probabilities = await prediction.data();
    const predictedClass = prediction.argMax(-1).dataSync()[0];
    
    console.log(`Predicted class: ${predictedClass}`);
    console.log(`Probabilities: ${probabilities}`);
    
    // IMPORTANT: Dispose tensors to prevent memory leaks!
    input.dispose();
    prediction.dispose();
}

// ============================================
// Image classification from webcam
// ============================================
async function classifyWebcam() {
    const model = await tf.loadLayersModel('model/model.json');
    const video = document.getElementById('webcam');
    
    // Get webcam stream
    const stream = await navigator.mediaDevices.getUserMedia({ video: true });
    video.srcObject = stream;
    
    // Classify every frame
    async function predict() {
        // Convert video frame to tensor
        const img = tf.browser.fromPixels(video)
            .resizeBilinear([224, 224])  // Resize to model input size
            .expandDims(0)                // Add batch dimension
            .div(255.0);                  // Normalize to [0, 1]
        
        const prediction = model.predict(img);
        const classIndex = prediction.argMax(-1).dataSync()[0];
        
        document.getElementById('result').textContent = `Class: ${classIndex}`;
        
        // Clean up tensors
        img.dispose();
        prediction.dispose();
        
        // Request next frame
        requestAnimationFrame(predict);
    }
    
    predict();
}

// ============================================
// Train a model IN the browser
// ============================================
async function trainInBrowser() {
    // Define model
    const model = tf.sequential();
    model.add(tf.layers.dense({ units: 64, activation: 'relu', inputShape: [10] }));
    model.add(tf.layers.dense({ units: 1, activation: 'sigmoid' }));
    
    model.compile({
        optimizer: 'adam',
        loss: 'binaryCrossentropy',
        metrics: ['accuracy']
    });
    
    // Generate sample data
    const xs = tf.randomNormal([1000, 10]);
    const ys = tf.randomUniform([1000, 1]).round();
    
    // Train
    await model.fit(xs, ys, {
        epochs: 10,
        batchSize: 32,
        callbacks: {
            onEpochEnd: (epoch, logs) => {
                console.log(`Epoch ${epoch}: loss = ${logs.loss.toFixed(4)}`);
            }
        }
    });
    
    // Save model to browser's IndexedDB
    await model.save('indexeddb://my-browser-model');
    
    // Load it later
    const loadedModel = await tf.loadLayersModel('indexeddb://my-browser-model');
}
</script>
```

---

## Model Optimization

### Overview of Optimization Techniques

```
Optimization Landscape:

Technique          │ Size Reduction │ Speed Improvement │ Accuracy Impact
───────────────────│────────────────│───────────────────│─────────────────
Quantization       │ 2-4x           │ 2-4x              │ Minimal (< 1%)
Pruning            │ 2-10x          │ 1-3x (with HW)    │ Small (< 2%)
Clustering         │ 2-5x           │ 1-2x              │ Minimal
Knowledge Distill. │ 5-50x          │ 5-50x             │ Some (2-5%)
Architecture Search│ N/A            │ 2-10x             │ Can improve!
```

### Pruning (Removing Unnecessary Weights)

```python
# pip install tensorflow-model-optimization

import tensorflow as tf
import tensorflow_model_optimization as tfmot

# ============================================
# Weight Pruning: Set small weights to ZERO
# ============================================
"""
Intuition: Most neural network weights are close to zero and don't 
contribute much. Pruning sets them exactly to zero.

Before pruning:                After 50% pruning:
[0.23, -0.01, 0.87,           [0.23, 0.00, 0.87,
 0.02, -0.95, 0.00,            0.00, -0.95, 0.00,
 -0.44, 0.01, 0.56]            -0.44, 0.00, 0.56]

Zeros can be compressed efficiently → smaller model file!
With hardware support, zeros can be skipped → faster inference!
"""

# Build your model
model = tf.keras.Sequential([
    tf.keras.layers.Dense(128, activation='relu', input_shape=(784,)),
    tf.keras.layers.Dense(64, activation='relu'),
    tf.keras.layers.Dense(10, activation='softmax')
])

# Define pruning schedule
pruning_params = {
    'pruning_schedule': tfmot.sparsity.keras.PolynomialDecay(
        initial_sparsity=0.10,   # Start with 10% zeros
        final_sparsity=0.80,     # End with 80% zeros
        begin_step=1000,         # Start pruning at step 1000
        end_step=5000            # Stop pruning at step 5000
    )
}

# Wrap model with pruning
pruned_model = tfmot.sparsity.keras.prune_low_magnitude(model, **pruning_params)

# Must use this callback during training!
callbacks = [tfmot.sparsity.keras.UpdatePruningStep()]

pruned_model.compile(
    optimizer='adam',
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

# Train with pruning
# pruned_model.fit(x_train, y_train, epochs=10, callbacks=callbacks)

# Strip pruning wrappers for deployment
final_model = tfmot.sparsity.keras.strip_pruning(pruned_model)

# Convert to TFLite with pruning benefits
converter = tf.lite.TFLiteConverter.from_keras_model(final_model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
tflite_pruned = converter.convert()

# The sparse model compresses much better!
# Unpruned TFLite: ~500 KB
# 80% pruned TFLite: ~100 KB (after gzip compression)
```

### Weight Clustering

```python
import tensorflow_model_optimization as tfmot

# ============================================
# Weight Clustering: Group weights into N clusters
# ============================================
"""
Instead of millions of unique float32 values, 
reduce to N cluster centroids.

Before clustering (unique values):
[0.23, 0.25, 0.87, 0.91, -0.44, -0.42]

After clustering (K=3):
Centroids: [-0.43, 0.24, 0.89]
Indices:   [1, 1, 2, 2, 0, 0]

Store indices (small integers) + centroids → much smaller!
"""

# Apply clustering
cluster_weights = tfmot.clustering.keras.cluster_weights

clustering_params = {
    'number_of_clusters': 16,  # Only 16 unique weight values!
    'cluster_centroids_init': tfmot.clustering.keras.CentroidInitialization.LINEAR
}

clustered_model = cluster_weights(model, **clustering_params)
clustered_model.compile(
    optimizer='adam',
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

# Fine-tune
# clustered_model.fit(x_train, y_train, epochs=3)

# Strip clustering for deployment
final_model = tfmot.clustering.keras.strip_clustering(clustered_model)
```

### Combining Optimizations (Maximum Compression)

```python
"""
Pipeline: Train → Prune → Cluster → Quantize

Each technique stacks on top of the previous one:
• Pruning: removes unnecessary weights (sets to zero)
• Clustering: groups remaining weights into clusters
• Quantization: reduces precision of cluster centroids

Result: 10-50x smaller model with minimal accuracy loss!
"""

# Step 1: Train base model
model = build_and_train_model()

# Step 2: Apply pruning
pruned_model = tfmot.sparsity.keras.prune_low_magnitude(model)
# Fine-tune with pruning...

# Step 3: Apply clustering to pruned model
stripped_pruned = tfmot.sparsity.keras.strip_pruning(pruned_model)
clustered_model = tfmot.clustering.keras.cluster_weights(stripped_pruned)
# Fine-tune with clustering...

# Step 4: Quantize
stripped_clustered = tfmot.clustering.keras.strip_clustering(clustered_model)
converter = tf.lite.TFLiteConverter.from_keras_model(stripped_clustered)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
final_tflite = converter.convert()

# Compare sizes
print(f"Original: {original_size:.1f} KB")
print(f"After all optimizations: {len(final_tflite)/1024:.1f} KB")
# Can achieve 10-50x compression!
```

---

## End-to-End Deployment Patterns

### Pattern 1: REST API with Flask

```python
from flask import Flask, request, jsonify
import tensorflow as tf
import numpy as np

app = Flask(__name__)

# Load model once at startup (NOT on every request!)
model = tf.keras.models.load_model('my_model')

@app.route('/predict', methods=['POST'])
def predict():
    """
    Accepts JSON input, returns predictions.
    
    Request:  {"instances": [[0.1, 0.2, ...784 values...]]}
    Response: {"predictions": [[0.01, 0.02, ..., 0.90, ...]]}
    """
    try:
        data = request.get_json()
        instances = np.array(data['instances'], dtype=np.float32)
        
        # Input validation
        if instances.shape[-1] != 784:
            return jsonify({'error': f'Expected 784 features, got {instances.shape[-1]}'}), 400
        
        # Make prediction
        predictions = model.predict(instances)
        
        return jsonify({
            'predictions': predictions.tolist(),
            'classes': np.argmax(predictions, axis=1).tolist()
        })
    
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/health', methods=['GET'])
def health():
    return jsonify({'status': 'healthy'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### Pattern 2: FastAPI (Modern, Async)

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import tensorflow as tf
import numpy as np
from typing import List

app = FastAPI(title="ML Model API", version="1.0")

# Load model
model = tf.keras.models.load_model('my_model')

class PredictionRequest(BaseModel):
    instances: List[List[float]]

class PredictionResponse(BaseModel):
    predictions: List[List[float]]
    classes: List[int]

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    """Make predictions with the ML model."""
    try:
        instances = np.array(request.instances, dtype=np.float32)
        predictions = model.predict(instances)
        
        return PredictionResponse(
            predictions=predictions.tolist(),
            classes=np.argmax(predictions, axis=1).tolist()
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    return {"status": "healthy", "model_loaded": model is not None}

# Run with: uvicorn main:app --host 0.0.0.0 --port 8000
```

### Pattern 3: Docker Deployment

```dockerfile
# Dockerfile for ML model serving

FROM python:3.10-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy model and code
COPY my_model/ ./my_model/
COPY main.py .

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s \
    CMD curl -f http://localhost:8000/health || exit 1

# Run the server
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```text
# requirements.txt
tensorflow==2.15.0
fastapi==0.104.0
uvicorn==0.24.0
numpy==1.26.0
```

```bash
# Build and run
docker build -t ml-model-api .
docker run -p 8000:8000 ml-model-api
```

### Pattern 4: Model Versioning and A/B Testing

```python
import tensorflow as tf
import numpy as np
import random

class ModelManager:
    """
    Manages multiple model versions for A/B testing and gradual rollout.
    """
    def __init__(self):
        self.models = {}
        self.traffic_split = {}  # {version: percentage}
    
    def load_model(self, version: str, path: str):
        """Load a model version."""
        self.models[version] = tf.keras.models.load_model(path)
        print(f"Loaded model version {version} from {path}")
    
    def set_traffic_split(self, split: dict):
        """
        Set traffic split between versions.
        Example: {'v1': 0.9, 'v2': 0.1}  # 90% v1, 10% v2
        """
        total = sum(split.values())
        if abs(total - 1.0) > 0.01:
            raise ValueError(f"Traffic split must sum to 1.0, got {total}")
        self.traffic_split = split
    
    def predict(self, input_data) -> dict:
        """
        Route prediction to a model version based on traffic split.
        Returns prediction along with version info (for logging).
        """
        # Select version based on traffic weights
        r = random.random()
        cumulative = 0.0
        selected_version = None
        
        for version, weight in self.traffic_split.items():
            cumulative += weight
            if r <= cumulative:
                selected_version = version
                break
        
        if selected_version is None or selected_version not in self.models:
            selected_version = list(self.models.keys())[-1]
        
        model = self.models[selected_version]
        prediction = model.predict(input_data)
        
        return {
            'predictions': prediction.tolist(),
            'model_version': selected_version
        }

# Usage
manager = ModelManager()
manager.load_model('v1', 'models/v1/')
manager.load_model('v2', 'models/v2/')
manager.set_traffic_split({'v1': 0.8, 'v2': 0.2})  # 80/20 split

# result = manager.predict(input_data)
# Log result['model_version'] to compare performance!
```

---

## Common Mistakes

### Mistake 1: Shipping the Wrong Model Format

```python
# ❌ WRONG: Saving as .h5 for production serving
model.save('model.h5')  # Legacy format, limited functionality

# ✅ CORRECT: Use SavedModel for serving
model.save('model_dir/')  # SavedModel format with signatures

# ✅ CORRECT: Use .keras for training checkpoints
model.save('model.keras')  # Keras native format
```

### Mistake 2: Not Including Preprocessing in the Model

```python
# ❌ WRONG: Preprocessing is separate from the model
# Client must know to normalize, resize, etc.
def preprocess(image):
    return image / 255.0  # Client must do this!

prediction = model.predict(preprocess(image))

# ✅ CORRECT: Include preprocessing IN the model
inputs = tf.keras.Input(shape=(None, None, 3))  # Accept any size
x = tf.keras.layers.Resizing(224, 224)(inputs)  # Resize to expected
x = tf.keras.layers.Rescaling(1./255)(x)        # Normalize
x = base_model(x)
outputs = tf.keras.layers.Dense(10, activation='softmax')(x)

serving_model = tf.keras.Model(inputs, outputs)
serving_model.save('serving_model/')
# Now clients just send raw images — no preprocessing needed!
```

### Mistake 3: Not Versioning Models

```python
# ❌ WRONG: Overwriting the same model
model.save('my_model/')  # Overwritten every time!

# ✅ CORRECT: Version-based directory structure
import datetime

version = datetime.datetime.now().strftime('%Y%m%d_%H%M%S')
model.save(f'models/my_model/{version}/')

# Or use integer versions for TF Serving
model.save(f'models/my_model/1/')  # Version 1
# Later...
model.save(f'models/my_model/2/')  # Version 2
```

### Mistake 4: Loading Model on Every Request

```python
# ❌ WRONG: Loading model per request (SLOW!)
@app.route('/predict', methods=['POST'])
def predict():
    model = tf.keras.models.load_model('my_model/')  # 5-30 seconds!
    return model.predict(data)

# ✅ CORRECT: Load once, reuse
model = tf.keras.models.load_model('my_model/')  # Load at startup

@app.route('/predict', methods=['POST'])
def predict():
    return model.predict(data)  # Instant!
```

### Mistake 5: Not Testing TFLite Output Against Original

```python
# ❌ WRONG: Convert and deploy without validation
converter = tf.lite.TFLiteConverter.from_keras_model(model)
tflite_model = converter.convert()
# Deploy immediately... 😱

# ✅ CORRECT: Validate TFLite matches original
original_pred = model.predict(test_input)

interpreter = tf.lite.Interpreter(model_content=tflite_model)
interpreter.allocate_tensors()
input_details = interpreter.get_input_details()
output_details = interpreter.get_output_details()

interpreter.set_tensor(input_details[0]['index'], test_input.astype(np.float32))
interpreter.invoke()
tflite_pred = interpreter.get_tensor(output_details[0]['index'])

# Compare
max_diff = np.max(np.abs(original_pred - tflite_pred))
print(f"Max prediction difference: {max_diff:.6f}")
assert max_diff < 0.01, "TFLite conversion caused significant accuracy loss!"
```

### Mistake 6: Ignoring Model Size for Mobile

```python
# ❌ WRONG: Deploying ResNet152 on a phone (232 MB!)
model = tf.keras.applications.ResNet152(weights='imagenet')

# ✅ CORRECT: Use a mobile-friendly architecture
model = tf.keras.applications.MobileNetV2(weights='imagenet')  # 14 MB
# + Quantization → ~3.5 MB
```

---

## Interview Questions

### Q1: What is SavedModel and how is it different from .h5?
**A**: SavedModel is TensorFlow's standard serialization format containing:
- Computation graph (saved_model.pb)
- Variable values (variables/)
- Serving signatures (input/output specs)
- Assets (vocab files, etc.)

Differences from .h5:
- SavedModel supports serving signatures (required for TF Serving)
- SavedModel preserves `@tf.function` decorated methods
- SavedModel supports custom objects without registration
- SavedModel is the required format for TF Serving, TFLite, and TF.js conversion
- .h5 is a legacy format with limited functionality

### Q2: What is quantization and what are the types?
**A**: Quantization reduces model precision to decrease size and increase speed:

- **Dynamic Range**: Weights quantized to int8 at conversion time; activations quantized dynamically at runtime. Easiest, ~4x smaller.
- **Full Integer (int8)**: Both weights and activations are int8. Requires calibration data. Fastest on CPUs, ~4x smaller.
- **Float16**: Weights stored as float16. ~2x smaller. Best on GPU delegates.

Trade-off: Lower precision → smaller model and faster inference, but potential accuracy loss (typically < 1-2%).

### Q3: Explain TF Serving architecture.
**A**: TF Serving has these key components:
- **Servables**: Model versions that can be loaded/served
- **Sources**: Monitor filesystem for new model versions
- **Loaders**: Load/unload model versions
- **Manager**: Manages lifecycle of servables
- **Serving Core**: Handles incoming requests, routes to correct model version

Key features: automatic model hot-swapping, version management, request batching, REST and gRPC APIs, hardware acceleration.

### Q4: How do you deploy ML models on mobile devices?
**A**:
1. Train model in TensorFlow
2. Optimize the model (pruning, quantization)
3. Convert to TFLite format (`.tflite`)
4. Integrate with mobile app using TFLite runtime (Android/iOS SDK)
5. Choose appropriate delegate (GPU, NNAPI, CoreML) for hardware acceleration
6. Use model bundling or download from server

Key considerations: model size (< 10MB ideal), latency (< 100ms), power consumption, memory usage.

### Q5: What is the difference between REST and gRPC for model serving?
**A**:

| Aspect | REST | gRPC |
|--------|------|------|
| Protocol | HTTP 1.1 | HTTP/2 |
| Format | JSON (text) | Protobuf (binary) |
| Speed | Slower | 2-5x faster |
| Complexity | Simple | More setup |
| Browser | Native | Needs grpc-web |
| Streaming | No | Yes |
| Best for | Testing, simple apps | Production, high throughput |

### Q6: What is model optimization and why is it important?
**A**: Model optimization reduces model size and improves inference speed for deployment on resource-constrained devices:
- **Pruning**: Removes low-magnitude weights (10-80% sparsity)
- **Quantization**: Reduces numerical precision (float32 → int8)
- **Clustering**: Groups weights into K clusters
- **Knowledge Distillation**: Trains a small model to mimic a large one
- **Architecture Search**: Finds efficient architectures automatically

These techniques can be combined for cumulative compression (10-50x smaller).

### Q7: How do you handle model versioning in production?
**A**:
- **Directory-based versioning**: Each version in a numbered directory (TF Serving auto-detects)
- **A/B testing**: Route traffic between versions to compare performance
- **Canary deployments**: Gradually shift traffic from old to new version
- **Shadow mode**: New model runs alongside old one without serving (compare outputs)
- **Rollback**: Maintain previous versions for quick rollback if issues arise
- **Model registry**: Use MLflow, Vertex AI, or similar for tracking lineage and metadata

### Q8: How would you reduce latency of a deployed model?
**A**:
1. **Model optimization**: Quantization, pruning, distillation
2. **Batching**: Group requests for GPU efficiency
3. **Caching**: Cache frequent predictions
4. **Hardware**: Use GPU/TPU for inference
5. **Async processing**: Non-blocking request handling
6. **Model architecture**: Use lighter models (MobileNet vs ResNet)
7. **Preprocessing in model**: Avoid client-server preprocessing round-trips
8. **gRPC**: Use binary protocol instead of REST/JSON
9. **Edge deployment**: Run model on-device, no network latency

---

## Quick Reference

### Deployment Decision Matrix

| Target | Format | Tool | Size Priority |
|--------|--------|------|---------------|
| Server (high throughput) | SavedModel | TF Serving | Low |
| Server (simple API) | SavedModel | Flask/FastAPI | Low |
| Android/iOS | .tflite | TFLite runtime | High |
| Browser | TFJS layers/graph | TF.js | High |
| Edge devices | .tflite (int8) | TFLite + delegates | Very High |
| Cloud managed | SavedModel | Vertex AI / SageMaker | Low |

### SavedModel Commands Cheat Sheet

```python
# Save
model.save('model_dir/')                    # Keras model
tf.saved_model.save(module, 'model_dir/')   # tf.Module

# Load  
model = tf.keras.models.load_model('model_dir/')  # Keras (full)
loaded = tf.saved_model.load('model_dir/')          # Generic (inference only)

# Inspect (CLI)
# saved_model_cli show --dir model_dir --all
# saved_model_cli show --dir model_dir --tag_set serve --signature_def serving_default
```

### TFLite Conversion Cheat Sheet

```python
# Basic conversion
converter = tf.lite.TFLiteConverter.from_keras_model(model)
tflite = converter.convert()

# Dynamic range quantization
converter.optimizations = [tf.lite.Optimize.DEFAULT]

# Full integer quantization
converter.representative_dataset = representative_fn
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]

# Float16 quantization
converter.target_spec.supported_types = [tf.float16]
```

### Optimization Techniques Summary

| Technique | Size Reduction | Speed Gain | Accuracy Loss | Difficulty |
|-----------|---------------|------------|---------------|------------|
| Dynamic Quantization | 4x | 2-3x | < 1% | Easy |
| Full Int8 Quantization | 4x | 3-4x | 1-2% | Medium |
| Float16 Quantization | 2x | 1.5-2x | < 0.5% | Easy |
| Pruning (80%) | 3-5x* | 1-3x | 1-2% | Medium |
| Weight Clustering | 3-5x* | 1-2x | < 1% | Medium |
| Knowledge Distillation | 5-50x | 5-50x | 2-5% | Hard |

*After compression (gzip/zip)

### TF Serving Docker Quick Start

```bash
# Serve a model
docker run -p 8501:8501 \
  -v /path/to/models:/models \
  -e MODEL_NAME=my_model \
  tensorflow/serving

# REST predict
curl -d '{"instances": [[1.0, 2.0, 3.0]]}' \
  -X POST http://localhost:8501/v1/models/my_model:predict

# Check status
curl http://localhost:8501/v1/models/my_model

# Get metadata
curl http://localhost:8501/v1/models/my_model/metadata
```

---

*Previous: [08-Custom-Training-Loops](./08-Custom-Training-Loops.md) | Next: [10-Distributed-Training-TF](./10-Distributed-Training-TF.md)*
