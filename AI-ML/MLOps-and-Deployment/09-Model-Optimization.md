# Model Optimization for Deployment

## Table of Contents
- [What is Model Optimization?](#what-is-model-optimization)
- [Why It Matters](#why-it-matters)
- [How It Works — The Big Picture](#how-it-works--the-big-picture)
- [Quantization](#quantization)
- [Pruning](#pruning)
- [Knowledge Distillation](#knowledge-distillation)
- [ONNX — Open Neural Network Exchange](#onnx--open-neural-network-exchange)
- [TensorRT](#tensorrt)
- [Other Optimization Techniques](#other-optimization-techniques)
- [Optimization Pipeline — Combining Techniques](#optimization-pipeline--combining-techniques)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Model Optimization?

### Simple Explanation

Imagine you wrote a 1000-page book (your trained model). It's amazing and detailed, but:
- It's too heavy to carry around (too large for mobile/edge devices)
- It takes 10 minutes to find any answer (too slow for real-time serving)
- You need a massive bookshelf to store it (too much memory)

**Model optimization** is like creating a pocket-sized version of that book that:
- Keeps 95-99% of the important information
- Fits in your pocket (smaller model size)
- Lets you find answers in seconds (faster inference)
- Costs less to store and transport (cheaper serving)

### Formal Definition

Model optimization encompasses techniques to reduce the computational cost, memory footprint, and latency of ML models while preserving prediction quality. It bridges the gap between large, accurate research models and the constraints of production deployment.

### The Optimization Triangle

```
                    ACCURACY
                       ▲
                      / \
                     /   \
                    /     \
                   / SWEET \
                  /  SPOT   \
                 /           \
                /─────────────\
               /               \
              /                 \
     SPEED ◀──────────────────────▶ SIZE
     (Latency)                  (Memory/Storage)

Goal: Find the sweet spot where you sacrifice minimal
accuracy for maximum gains in speed and size.
```

---

## Why It Matters

### Real-World Impact

| Scenario | Unoptimized | Optimized | Impact |
|----------|-------------|-----------|--------|
| **BERT inference** | 110M params, 440MB | 28M params, 30MB (quantized) | 14x smaller, 4x faster |
| **ResNet-50** | 98MB, 11ms/image | 25MB, 3ms/image (TensorRT) | 4x smaller, 3.7x faster |
| **GPT-2** | 1.5GB, 200ms/token | 380MB, 50ms/token (INT8) | 4x smaller, 4x faster |
| **Mobile deployment** | Won't fit on device | Runs at 30fps | Enables new use cases |
| **Serving cost** | $10,000/month (GPU) | $2,500/month | 75% cost reduction |

### When You Need Optimization

- **Edge/Mobile deployment** — Models must fit in limited memory (< 100MB)
- **Real-time serving** — Latency SLA < 50ms per request
- **Cost reduction** — Serving thousands of requests/second on fewer GPUs
- **Batch processing** — Process millions of items faster
- **Embedded systems** — IoT devices, cameras, drones with no GPU

### The Golden Rule

$$\text{Optimization Value} = \frac{\text{Speedup} \times \text{Size Reduction}}{\text{Accuracy Loss}}$$

If this ratio is high → optimization is worth it. If accuracy loss is too high → reconsider.

---

## How It Works — The Big Picture

### Optimization Techniques Taxonomy

```
Model Optimization
├── Weight Reduction
│   ├── Quantization (reduce precision: FP32 → INT8)
│   ├── Pruning (remove unnecessary weights)
│   └── Low-Rank Factorization (decompose weight matrices)
│
├── Architecture Optimization
│   ├── Knowledge Distillation (train smaller "student" model)
│   ├── Neural Architecture Search (NAS) (find efficient architectures)
│   └── Efficient Architectures (MobileNet, EfficientNet, SqueezeNet)
│
├── Runtime Optimization
│   ├── ONNX Runtime (cross-platform inference)
│   ├── TensorRT (NVIDIA GPU optimization)
│   ├── OpenVINO (Intel CPU/GPU optimization)
│   ├── CoreML (Apple devices)
│   └── TFLite (Mobile/Edge)
│
└── Computational Tricks
    ├── Operator Fusion (combine multiple ops)
    ├── Batching (process multiple inputs together)
    ├── Caching (key-value cache for transformers)
    └── Mixed Precision Inference (FP16 where safe)
```

### When to Use Which Technique

| Technique | Accuracy Loss | Speed Gain | Size Reduction | Effort |
|-----------|--------------|------------|----------------|--------|
| **FP16 Quantization** | ~0% | 1.5-2x | 2x | Very Low |
| **INT8 Quantization** | 0.5-2% | 2-4x | 4x | Low |
| **Pruning (50%)** | 0.5-3% | 1.5-2x | 2x | Medium |
| **Distillation** | 1-5% | 2-10x | 2-10x | High |
| **ONNX Runtime** | 0% | 1.2-2x | 0% | Low |
| **TensorRT** | 0-1% | 2-5x | Varies | Medium |
| **Combining all** | 2-5% | 5-20x | 5-15x | High |

---

## Quantization

### What is Quantization?

Quantization reduces the numerical precision of model weights and activations. Instead of storing every number as a 32-bit float (FP32), you use fewer bits (FP16, INT8, or even INT4).

### Analogy

Think of measuring temperature:
- **FP32:** "It's 72.384729°F" — Absurdly precise, nobody needs this
- **FP16:** "It's 72.38°F" — Still very accurate, half the storage
- **INT8:** "It's 72°F" — Good enough for daily life, quarter the storage
- **INT4:** "It's about 70°F" — Rough estimate, but sometimes sufficient

### How Quantization Works

```
FP32 (32 bits per weight):
┌──────────────────────────────────────────┐
│ Sign(1) │ Exponent(8) │ Mantissa(23)      │
└──────────────────────────────────────────┘
Range: ±3.4 × 10³⁸, Precision: ~7 decimal digits

FP16 (16 bits per weight):
┌────────────────────────┐
│ Sign(1) │ Exp(5) │ Man(10) │
└────────────────────────┘
Range: ±65504, Precision: ~3 decimal digits

INT8 (8 bits per weight):
┌──────────┐
│ 8-bit int │
└──────────┘
Range: -128 to 127, maps to original value range

Quantization Mapping (Linear):
   FP32 value                        INT8 value
   -1.5 ─────────── scale + zero ──────── -128
    0.0 ────────────────────────────────── 0
    1.5 ─────────────────────────────────  127
```

### Mathematical Foundation

**Linear quantization** maps a floating-point range $[x_{min}, x_{max}]$ to integer range $[q_{min}, q_{max}]$:

$$\text{scale} = \frac{x_{max} - x_{min}}{q_{max} - q_{min}}$$

$$\text{zero\_point} = q_{min} - \text{round}\left(\frac{x_{min}}{\text{scale}}\right)$$

$$x_{quantized} = \text{round}\left(\frac{x}{\text{scale}}\right) + \text{zero\_point}$$

$$x_{dequantized} = (x_{quantized} - \text{zero\_point}) \times \text{scale}$$

### Types of Quantization

```
┌─────────────────────────────────────────────────────────────┐
│                    QUANTIZATION TYPES                         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. POST-TRAINING QUANTIZATION (PTQ)                         │
│     • Applied AFTER training is complete                      │
│     • No retraining needed                                    │
│     • Quick and easy                                          │
│     • Slight accuracy loss                                    │
│     ├── Dynamic Quantization (weights only, at runtime)      │
│     ├── Static Quantization (weights + activations,          │
│     │   needs calibration data)                               │
│     └── Weight-Only Quantization (for LLMs: GPTQ, AWQ)      │
│                                                               │
│  2. QUANTIZATION-AWARE TRAINING (QAT)                        │
│     • Simulates quantization DURING training                  │
│     • Model learns to be robust to lower precision            │
│     • Better accuracy than PTQ                                │
│     • Requires retraining (more effort)                       │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Code: PyTorch Quantization

```python
# quantization_pytorch.py
"""
Complete quantization examples using PyTorch.
Covers: Dynamic, Static, and Quantization-Aware Training.
"""

import torch
import torch.nn as nn
import torch.quantization as quant
import time
import os

# ============================================
# SETUP: Define a sample model
# ============================================

class SimpleCNN(nn.Module):
    """A simple CNN for demonstration."""
    def __init__(self, num_classes=10):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(3, 32, 3, padding=1),
            nn.ReLU(),
            nn.Conv2d(32, 64, 3, padding=1),
            nn.ReLU(),
            nn.AdaptiveAvgPool2d(1)
        )
        self.classifier = nn.Sequential(
            nn.Linear(64, 128),
            nn.ReLU(),
            nn.Linear(128, num_classes)
        )
    
    def forward(self, x):
        x = self.features(x)
        x = x.view(x.size(0), -1)
        x = self.classifier(x)
        return x

# Create and "train" a model (using random weights for demo)
model_fp32 = SimpleCNN()
model_fp32.eval()  # Must be in eval mode for quantization

# Helper: measure model size
def get_model_size_mb(model):
    """Get model size in megabytes."""
    torch.save(model.state_dict(), "temp_model.pt")
    size_mb = os.path.getsize("temp_model.pt") / (1024 * 1024)
    os.remove("temp_model.pt")
    return size_mb

# Helper: measure inference time
def measure_latency(model, input_tensor, num_runs=100):
    """Measure average inference latency in milliseconds."""
    # Warmup
    for _ in range(10):
        _ = model(input_tensor)
    
    start = time.perf_counter()
    for _ in range(num_runs):
        with torch.no_grad():
            _ = model(input_tensor)
    elapsed = (time.perf_counter() - start) / num_runs * 1000
    return elapsed

print(f"Original FP32 model size: {get_model_size_mb(model_fp32):.2f} MB")

dummy_input = torch.randn(1, 3, 32, 32)  # Single image


# ============================================
# METHOD 1: Dynamic Quantization
# ============================================
# Simplest approach — quantizes weights to INT8, activations
# remain in FP32 but are quantized dynamically at runtime.
# Best for: Models dominated by Linear/LSTM layers (NLP models)

print("\n--- Dynamic Quantization ---")

model_dynamic = torch.quantization.quantize_dynamic(
    model_fp32,                          # Original model
    {nn.Linear},                         # Which layers to quantize
    dtype=torch.qint8                    # Target dtype
)

print(f"Dynamic quantized size: {get_model_size_mb(model_dynamic):.2f} MB")

# Compare outputs
with torch.no_grad():
    output_fp32 = model_fp32(dummy_input)
    output_dynamic = model_dynamic(dummy_input)
    diff = (output_fp32 - output_dynamic).abs().mean()
    print(f"Average output difference: {diff:.6f}")


# ============================================
# METHOD 2: Static Quantization (Post-Training)
# ============================================
# Quantizes BOTH weights and activations to INT8.
# Requires a "calibration" step with representative data.
# Best for: CNN models, when you have calibration data.

print("\n--- Static Quantization ---")

class QuantizableCNN(nn.Module):
    """CNN with quantization stubs for static quantization."""
    def __init__(self, num_classes=10):
        super().__init__()
        # QuantStub converts float tensors to quantized
        self.quant = torch.quantization.QuantStub()
        
        self.features = nn.Sequential(
            nn.Conv2d(3, 32, 3, padding=1),
            nn.ReLU(),
            nn.Conv2d(32, 64, 3, padding=1),
            nn.ReLU(),
            nn.AdaptiveAvgPool2d(1)
        )
        self.classifier = nn.Sequential(
            nn.Linear(64, 128),
            nn.ReLU(),
            nn.Linear(128, num_classes)
        )
        
        # DeQuantStub converts quantized tensors back to float
        self.dequant = torch.quantization.DeQuantStub()
    
    def forward(self, x):
        x = self.quant(x)           # float → quantized
        x = self.features(x)
        x = x.view(x.size(0), -1)
        x = self.classifier(x)
        x = self.dequant(x)         # quantized → float
        return x

model_static = QuantizableCNN()
model_static.eval()

# Step 1: Specify quantization configuration
model_static.qconfig = torch.quantization.get_default_qconfig('x86')
# 'x86' for Intel CPUs, 'qnnpack' for ARM/mobile

# Step 2: Prepare model (inserts observers to track value ranges)
model_prepared = torch.quantization.prepare(model_static)

# Step 3: Calibrate with representative data
# Feed several batches through the model so observers can
# record the range of values for each tensor
print("Calibrating...")
calibration_data = [torch.randn(32, 3, 32, 32) for _ in range(10)]
with torch.no_grad():
    for batch in calibration_data:
        model_prepared(batch)

# Step 4: Convert to quantized model
model_quantized = torch.quantization.convert(model_prepared)

print(f"Static quantized size: {get_model_size_mb(model_quantized):.2f} MB")

# Measure latency improvement
lat_fp32 = measure_latency(model_fp32, dummy_input)
lat_quant = measure_latency(model_quantized, dummy_input)
print(f"FP32 latency: {lat_fp32:.2f} ms")
print(f"INT8 latency: {lat_quant:.2f} ms")
print(f"Speedup: {lat_fp32/lat_quant:.2f}x")


# ============================================
# METHOD 3: Quantization-Aware Training (QAT)
# ============================================
# Simulates quantization effects DURING training.
# Model learns weights that are robust to quantization noise.
# Best accuracy among all quantization methods.

print("\n--- Quantization-Aware Training ---")

model_qat = QuantizableCNN()
model_qat.train()  # Must be in training mode for QAT

# Step 1: Specify QAT config
model_qat.qconfig = torch.quantization.get_default_qat_qconfig('x86')

# Step 2: Prepare for QAT (inserts fake-quantization modules)
model_qat_prepared = torch.quantization.prepare_qat(model_qat)

# Step 3: Fine-tune (train for a few epochs with fake quantization)
# In a real scenario, you'd use your actual training loop here
optimizer = torch.optim.Adam(model_qat_prepared.parameters(), lr=1e-4)
criterion = nn.CrossEntropyLoss()

print("QAT fine-tuning...")
for epoch in range(3):
    model_qat_prepared.train()
    for _ in range(10):  # Simulated batches
        inputs = torch.randn(32, 3, 32, 32)
        targets = torch.randint(0, 10, (32,))
        
        outputs = model_qat_prepared(inputs)
        loss = criterion(outputs, targets)
        
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
    
    print(f"  Epoch {epoch+1}, Loss: {loss.item():.4f}")

# Step 4: Convert to actual quantized model
model_qat_prepared.eval()
model_qat_final = torch.quantization.convert(model_qat_prepared)

print(f"QAT model size: {get_model_size_mb(model_qat_final):.2f} MB")


# ============================================
# BONUS: Quantizing Hugging Face Transformers
# ============================================

print("\n--- Quantizing BERT with bitsandbytes ---")

# Method 1: Using bitsandbytes (for LLMs)
"""
from transformers import AutoModelForSequenceClassification, BitsAndBytesConfig

# 4-bit quantization config
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,                    # Use 4-bit quantization
    bnb_4bit_quant_type="nf4",            # Normal Float 4-bit
    bnb_4bit_compute_dtype=torch.float16, # Compute in FP16
    bnb_4bit_use_double_quant=True        # Double quantization (saves more)
)

model = AutoModelForSequenceClassification.from_pretrained(
    "bert-base-uncased",
    quantization_config=bnb_config,
    device_map="auto"
)

# Model is now ~4x smaller and uses much less GPU memory
print(f"Model memory: {model.get_memory_footprint() / 1e6:.1f} MB")
"""

# Method 2: Using GPTQ (for LLMs - best quality)
"""
from transformers import AutoModelForCausalLM, GPTQConfig

gptq_config = GPTQConfig(
    bits=4,                    # 4-bit quantization
    dataset="c4",              # Calibration dataset
    tokenizer=tokenizer,
    group_size=128             # Quantize in groups of 128
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b",
    quantization_config=gptq_config,
    device_map="auto"
)
"""

# Method 3: Using AWQ (Activation-aware Weight Quantization)
"""
from awq import AutoAWQForCausalLM

model = AutoAWQForCausalLM.from_pretrained("meta-llama/Llama-2-7b")
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-2-7b")

# Quantize
model.quantize(
    tokenizer,
    quant_config={"zero_point": True, "q_group_size": 128, "w_bit": 4}
)

# Save quantized model
model.save_quantized("llama-2-7b-awq")
"""
```

### Quantization Comparison Table

| Method | Precision | Accuracy Drop | Speed Gain | Use Case |
|--------|-----------|--------------|------------|----------|
| **FP32** (baseline) | 32-bit float | 0% | 1x | Training |
| **FP16** | 16-bit float | ~0% | 1.5-2x | GPU inference |
| **BF16** | 16-bit bfloat | ~0% | 1.5-2x | Training + inference |
| **INT8 Dynamic** | 8-bit int | 0.5-1% | 2-3x | NLP models (CPU) |
| **INT8 Static** | 8-bit int | 0.5-2% | 2-4x | CNN models (CPU) |
| **INT8 QAT** | 8-bit int | 0.1-0.5% | 2-4x | Best quality INT8 |
| **INT4 (GPTQ)** | 4-bit int | 1-3% | 2-3x | LLM deployment |
| **INT4 (AWQ)** | 4-bit int | 0.5-2% | 2-3x | LLM deployment |
| **NF4 (QLoRA)** | 4-bit NF | 1-2% | Fine-tuning | LLM fine-tuning |

---

## Pruning

### What is Pruning?

Pruning removes unnecessary weights (connections) from a neural network. Research shows that 50-90% of weights in most networks are redundant — they contribute almost nothing to the output.

### Analogy

Think of a road network in a city:
- **Before pruning:** Every street is paved, including dead-end roads nobody uses
- **After pruning:** You remove roads with <5 cars/day. The city still functions perfectly, but maintenance costs drop dramatically

### Types of Pruning

```
┌─────────────────────────────────────────────────────────────┐
│                      PRUNING TYPES                            │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  UNSTRUCTURED PRUNING          STRUCTURED PRUNING            │
│  (Individual weights)          (Entire channels/layers)      │
│                                                               │
│  Before:                       Before:                        │
│  ┌─────────────┐              ┌─────────────┐               │
│  │ 0.3  0.1 0.7│              │ Ch1 Ch2 Ch3 │               │
│  │ 0.0  0.5 0.0│              │  ●   ●   ●  │               │
│  │ 0.2  0.0 0.8│              │  ●   ●   ●  │               │
│  └─────────────┘              └─────────────┘               │
│                                                               │
│  After (50% sparse):          After (remove Ch2):            │
│  ┌─────────────┐              ┌─────────────┐               │
│  │ 0.0  0.0 0.7│              │ Ch1     Ch3 │               │
│  │ 0.0  0.5 0.0│              │  ●       ●  │               │
│  │ 0.0  0.0 0.8│              │  ●       ●  │               │
│  └─────────────┘              └─────────────┘               │
│                                                               │
│  ✓ Higher compression          ✓ Actual speedup on hardware │
│  ✗ Needs sparse hardware       ✓ No special hardware needed │
│    for speedup                 ✗ Less granular control       │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Code: PyTorch Pruning

```python
# pruning_pytorch.py
"""
Model pruning techniques using PyTorch.
Covers: Magnitude pruning, structured pruning, iterative pruning.
"""

import torch
import torch.nn as nn
import torch.nn.utils.prune as prune
import numpy as np
import copy

# ============================================
# SETUP: Model for pruning
# ============================================

class MLP(nn.Module):
    """Simple MLP for pruning demonstration."""
    def __init__(self):
        super().__init__()
        self.fc1 = nn.Linear(784, 512)
        self.relu1 = nn.ReLU()
        self.fc2 = nn.Linear(512, 256)
        self.relu2 = nn.ReLU()
        self.fc3 = nn.Linear(256, 10)
    
    def forward(self, x):
        x = self.relu1(self.fc1(x))
        x = self.relu2(self.fc2(x))
        return self.fc3(x)

model = MLP()

def count_parameters(model):
    """Count total and non-zero parameters."""
    total = 0
    nonzero = 0
    for param in model.parameters():
        total += param.numel()
        nonzero += (param != 0).sum().item()
    return total, nonzero

total, nonzero = count_parameters(model)
print(f"Original: {total:,} total params, {nonzero:,} non-zero ({nonzero/total:.1%})")


# ============================================
# METHOD 1: Unstructured Magnitude Pruning
# ============================================
# Remove individual weights with smallest absolute values.
# Intuition: Small weights contribute least to output.

print("\n--- Unstructured Magnitude Pruning ---")

model_pruned = copy.deepcopy(model)

# Prune 50% of weights in fc1 (by L1 magnitude)
prune.l1_unstructured(model_pruned.fc1, name='weight', amount=0.5)

# Prune 30% of weights in fc2
prune.l1_unstructured(model_pruned.fc2, name='weight', amount=0.3)

# Check sparsity
total, nonzero = count_parameters(model_pruned)
print(f"After pruning: {nonzero:,} non-zero out of {total:,} ({nonzero/total:.1%})")

# View the pruning mask
print(f"fc1 mask shape: {model_pruned.fc1.weight_mask.shape}")
print(f"fc1 sparsity: {(model_pruned.fc1.weight_mask == 0).sum().item() / model_pruned.fc1.weight_mask.numel():.1%}")


# ============================================
# METHOD 2: Global Unstructured Pruning
# ============================================
# Prune across ALL layers globally — the smallest weights
# anywhere in the network get pruned first.
# Better than per-layer because some layers are more sensitive.

print("\n--- Global Unstructured Pruning ---")

model_global = copy.deepcopy(model)

# Define which parameters to prune
parameters_to_prune = (
    (model_global.fc1, 'weight'),
    (model_global.fc2, 'weight'),
    (model_global.fc3, 'weight'),
)

# Globally prune 60% of weights with smallest magnitude
prune.global_unstructured(
    parameters_to_prune,
    pruning_method=prune.L1Unstructured,
    amount=0.6  # Remove 60% of all weights
)

# Check per-layer sparsity
for name, module in model_global.named_modules():
    if hasattr(module, 'weight_mask'):
        sparsity = (module.weight_mask == 0).sum().item() / module.weight_mask.numel()
        print(f"  {name} sparsity: {sparsity:.1%}")

total, nonzero = count_parameters(model_global)
print(f"Global pruning: {nonzero:,} non-zero ({nonzero/total:.1%})")


# ============================================
# METHOD 3: Structured Pruning (Channel Pruning)
# ============================================
# Remove entire channels/neurons instead of individual weights.
# Advantage: Actually speeds up inference on regular hardware.

print("\n--- Structured Pruning ---")

model_structured = copy.deepcopy(model)

# Prune 40% of output neurons in fc1 (entire rows of weight matrix)
prune.ln_structured(
    model_structured.fc1, 
    name='weight', 
    amount=0.4,      # Remove 40% of neurons
    n=2,             # L2 norm (prune channels with smallest L2 norm)
    dim=0            # dim=0 means output channels (neurons)
)

print(f"fc1 weight shape: {model_structured.fc1.weight.shape}")  # Still 512x784 but sparse
remaining = (model_structured.fc1.weight_mask.sum(dim=1) > 0).sum().item()
print(f"fc1 remaining neurons: {remaining} out of 512")


# ============================================
# METHOD 4: Iterative Pruning (Best Practice)
# ============================================
# Prune gradually over multiple iterations, fine-tuning between each.
# Much better accuracy than one-shot pruning.

print("\n--- Iterative Pruning Schedule ---")

"""
The process:
1. Train model to convergence
2. Prune X% of weights
3. Fine-tune for a few epochs
4. Repeat steps 2-3 until target sparsity

Example schedule for 90% total pruning:
  Round 1: Prune 20%, fine-tune → 80% weights remain
  Round 2: Prune 25%, fine-tune → 60% weights remain
  Round 3: Prune 33%, fine-tune → 40% weights remain
  Round 4: Prune 50%, fine-tune → 20% weights remain
  Round 5: Prune 50%, fine-tune → 10% weights remain (90% pruned!)
"""

def iterative_pruning(model, train_loader, val_loader, target_sparsity=0.9, num_rounds=5):
    """
    Iterative pruning with fine-tuning between rounds.
    
    Args:
        model: Trained model
        train_loader: Training data
        val_loader: Validation data
        target_sparsity: Final target (e.g., 0.9 = 90% weights removed)
        num_rounds: Number of pruning rounds
    """
    import math
    
    # Calculate per-round pruning amount
    # If target is 90% in 5 rounds: each round prunes ~37% of REMAINING weights
    per_round = 1 - (1 - target_sparsity) ** (1 / num_rounds)
    
    parameters_to_prune = [
        (module, 'weight') 
        for module in model.modules() 
        if isinstance(module, (nn.Linear, nn.Conv2d))
    ]
    
    for round_num in range(num_rounds):
        # Prune
        prune.global_unstructured(
            parameters_to_prune,
            pruning_method=prune.L1Unstructured,
            amount=per_round
        )
        
        # Count current sparsity
        total_zeros = sum(
            (module.weight == 0).sum().item() 
            for module, _ in parameters_to_prune
        )
        total_params = sum(
            module.weight.numel() 
            for module, _ in parameters_to_prune
        )
        current_sparsity = total_zeros / total_params
        
        print(f"Round {round_num + 1}: Sparsity = {current_sparsity:.1%}")
        
        # Fine-tune for a few epochs
        # fine_tune(model, train_loader, epochs=3)
        
        # Validate
        # accuracy = evaluate(model, val_loader)
        # print(f"  Accuracy after fine-tuning: {accuracy:.4f}")
    
    # Make pruning permanent (remove masks, apply pruning)
    for module, name in parameters_to_prune:
        prune.remove(module, name)
    
    return model

# Example usage (with dummy data):
# model = iterative_pruning(model, train_loader, val_loader)


# ============================================
# Making Pruning Permanent & Exporting
# ============================================

print("\n--- Making Pruning Permanent ---")

model_export = copy.deepcopy(model_global)

# Remove pruning reparameterization (make it permanent)
for module in model_export.modules():
    if hasattr(module, 'weight_orig'):
        prune.remove(module, 'weight')

# Now save the permanently pruned model
torch.save(model_export.state_dict(), 'pruned_model.pt')

# For actual speedup with unstructured pruning, 
# you need sparse format support:
sparse_weight = model_export.fc1.weight.to_sparse()
print(f"Sparse weight: {sparse_weight.shape}, nnz={sparse_weight._nnz()}")

# Clean up
import os
if os.path.exists('pruned_model.pt'):
    os.remove('pruned_model.pt')
```

---

## Knowledge Distillation

### What is Knowledge Distillation?

A large, accurate model (the **teacher**) transfers its knowledge to a smaller, faster model (the **student**). The student learns to mimic the teacher's outputs, not just the hard labels.

### Analogy

Think of an expert professor (teacher model) and a student:
- **Normal training:** Student reads the textbook and memorizes answers (hard labels: "this is a cat")
- **Distillation:** Student sits with the professor who says "This is 90% cat, 8% tiger, 2% lion" — the student learns WHY things are similar, not just what the answer is

### How Distillation Works

```
                    KNOWLEDGE DISTILLATION
                    
     Input Image ─────────┐
          │                │
          ▼                ▼
    ┌──────────┐    ┌──────────┐
    │ TEACHER  │    │ STUDENT  │
    │ (Large)  │    │ (Small)  │
    │ ResNet-50│    │ MobileNet│
    └────┬─────┘    └────┬─────┘
         │               │
         ▼               ▼
    Soft Logits      Soft Logits
    [0.7, 0.2, 0.1] [?, ?, ?]     ← Student tries to match these
                         │
                         │  Loss = α × KL(teacher_soft, student_soft)
                         │        + (1-α) × CE(student_hard, true_label)
                         │
                    ┌────┴────┐
                    │Combined │
                    │  Loss   │
                    └─────────┘
```

### Mathematical Formulation

The distillation loss combines two objectives:

$$\mathcal{L} = \alpha \cdot T^2 \cdot \text{KL}\left(\sigma\left(\frac{z_T}{T}\right), \sigma\left(\frac{z_S}{T}\right)\right) + (1-\alpha) \cdot \text{CE}(y, \sigma(z_S))$$

Where:
- $z_T$ = teacher logits, $z_S$ = student logits
- $T$ = temperature (higher = softer probabilities, typically 2-20)
- $\alpha$ = weight between distillation loss and hard label loss (typically 0.5-0.9)
- $\sigma$ = softmax function
- $T^2$ = scaling factor because gradients of soft targets are scaled by $1/T^2$

### Why Temperature Matters

```
Temperature = 1 (Normal softmax):
  Cat: 0.95  Dog: 0.03  Car: 0.02
  → Almost all information is "it's a cat". Student learns nothing about relationships.

Temperature = 5 (Soft softmax):
  Cat: 0.45  Dog: 0.35  Car: 0.20
  → Now the student learns: "cat and dog are related, car is different"
  → This is the "dark knowledge" — relationships between classes

Temperature = 20 (Very soft):
  Cat: 0.37  Dog: 0.34  Car: 0.29
  → Too uniform, losing too much signal
```

### Code: Knowledge Distillation

```python
# knowledge_distillation.py
"""
Knowledge Distillation: Train a small model using a large model's knowledge.
"""

import torch
import torch.nn as nn
import torch.nn.functional as F
from torchvision import datasets, transforms, models
from torch.utils.data import DataLoader

# ============================================
# STEP 1: Define Teacher and Student Models
# ============================================

class TeacherModel(nn.Module):
    """Large, accurate model (e.g., ResNet-like)."""
    def __init__(self, num_classes=10):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(1, 64, 3, padding=1), nn.ReLU(), nn.BatchNorm2d(64),
            nn.Conv2d(64, 128, 3, padding=1), nn.ReLU(), nn.BatchNorm2d(128),
            nn.MaxPool2d(2),
            nn.Conv2d(128, 256, 3, padding=1), nn.ReLU(), nn.BatchNorm2d(256),
            nn.AdaptiveAvgPool2d(1)
        )
        self.classifier = nn.Sequential(
            nn.Linear(256, 512), nn.ReLU(), nn.Dropout(0.3),
            nn.Linear(512, num_classes)
        )
    
    def forward(self, x):
        x = self.features(x)
        x = x.view(x.size(0), -1)
        return self.classifier(x)

class StudentModel(nn.Module):
    """Small, fast model (e.g., MobileNet-like)."""
    def __init__(self, num_classes=10):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(1, 16, 3, padding=1), nn.ReLU(),
            nn.MaxPool2d(2),
            nn.Conv2d(16, 32, 3, padding=1), nn.ReLU(),
            nn.AdaptiveAvgPool2d(1)
        )
        self.classifier = nn.Linear(32, num_classes)
    
    def forward(self, x):
        x = self.features(x)
        x = x.view(x.size(0), -1)
        return self.classifier(x)


# ============================================
# STEP 2: Distillation Loss Function
# ============================================

class DistillationLoss(nn.Module):
    """
    Combined distillation + classification loss.
    
    Args:
        temperature: Softening factor for teacher predictions (default: 4.0)
        alpha: Weight for distillation vs hard label loss (default: 0.7)
    """
    def __init__(self, temperature=4.0, alpha=0.7):
        super().__init__()
        self.temperature = temperature
        self.alpha = alpha
        self.kl_div = nn.KLDivLoss(reduction='batchmean')
        self.ce_loss = nn.CrossEntropyLoss()
    
    def forward(self, student_logits, teacher_logits, true_labels):
        """
        Compute distillation loss.
        
        Args:
            student_logits: Raw output from student model
            teacher_logits: Raw output from teacher model
            true_labels: Ground truth class labels
        """
        # Soft targets from teacher (high temperature)
        teacher_soft = F.softmax(teacher_logits / self.temperature, dim=1)
        student_soft = F.log_softmax(student_logits / self.temperature, dim=1)
        
        # KL divergence between soft distributions
        # Multiply by T² because gradients are scaled by 1/T²
        distillation_loss = self.kl_div(student_soft, teacher_soft) * (self.temperature ** 2)
        
        # Standard cross-entropy with true labels
        hard_loss = self.ce_loss(student_logits, true_labels)
        
        # Combined loss
        total_loss = (self.alpha * distillation_loss) + ((1 - self.alpha) * hard_loss)
        
        return total_loss


# ============================================
# STEP 3: Training Loop with Distillation
# ============================================

def train_with_distillation(
    teacher_model,
    student_model,
    train_loader,
    val_loader,
    epochs=10,
    temperature=4.0,
    alpha=0.7,
    lr=1e-3
):
    """
    Train student model using knowledge distillation.
    
    Args:
        teacher_model: Pre-trained teacher (frozen)
        student_model: Student model to train
        train_loader: Training data
        val_loader: Validation data
        epochs: Number of training epochs
        temperature: Distillation temperature
        alpha: Weight for distillation loss
        lr: Learning rate
    """
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    
    teacher_model = teacher_model.to(device)
    student_model = student_model.to(device)
    
    # IMPORTANT: Teacher is frozen — no gradient computation
    teacher_model.eval()
    for param in teacher_model.parameters():
        param.requires_grad = False
    
    optimizer = torch.optim.Adam(student_model.parameters(), lr=lr)
    scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=epochs)
    criterion = DistillationLoss(temperature=temperature, alpha=alpha)
    
    best_val_acc = 0
    
    for epoch in range(epochs):
        # --- Training ---
        student_model.train()
        train_loss = 0
        correct = 0
        total = 0
        
        for batch_idx, (inputs, targets) in enumerate(train_loader):
            inputs, targets = inputs.to(device), targets.to(device)
            
            # Get teacher predictions (no grad needed)
            with torch.no_grad():
                teacher_logits = teacher_model(inputs)
            
            # Get student predictions
            student_logits = student_model(inputs)
            
            # Compute distillation loss
            loss = criterion(student_logits, teacher_logits, targets)
            
            # Backprop (only student gets gradients)
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            train_loss += loss.item()
            _, predicted = student_logits.max(1)
            total += targets.size(0)
            correct += predicted.eq(targets).sum().item()
        
        train_acc = correct / total
        
        # --- Validation ---
        student_model.eval()
        val_correct = 0
        val_total = 0
        
        with torch.no_grad():
            for inputs, targets in val_loader:
                inputs, targets = inputs.to(device), targets.to(device)
                outputs = student_model(inputs)
                _, predicted = outputs.max(1)
                val_total += targets.size(0)
                val_correct += predicted.eq(targets).sum().item()
        
        val_acc = val_correct / val_total
        scheduler.step()
        
        print(f"Epoch {epoch+1}/{epochs} | "
              f"Train Loss: {train_loss/len(train_loader):.4f} | "
              f"Train Acc: {train_acc:.4f} | "
              f"Val Acc: {val_acc:.4f}")
        
        if val_acc > best_val_acc:
            best_val_acc = val_acc
            torch.save(student_model.state_dict(), 'best_student.pt')
    
    print(f"\nBest validation accuracy: {best_val_acc:.4f}")
    return student_model


# ============================================
# STEP 4: Compare Teacher vs Student
# ============================================

def compare_models(teacher, student, input_shape=(1, 1, 28, 28)):
    """Compare model sizes and speeds."""
    import time
    
    dummy = torch.randn(*input_shape)
    
    # Size comparison
    teacher_params = sum(p.numel() for p in teacher.parameters())
    student_params = sum(p.numel() for p in student.parameters())
    
    print(f"\n{'='*50}")
    print(f"MODEL COMPARISON")
    print(f"{'='*50}")
    print(f"Teacher parameters: {teacher_params:,}")
    print(f"Student parameters: {student_params:,}")
    print(f"Compression ratio:  {teacher_params/student_params:.1f}x")
    
    # Speed comparison
    teacher.eval()
    student.eval()
    
    # Warmup
    for _ in range(10):
        teacher(dummy)
        student(dummy)
    
    runs = 100
    
    start = time.perf_counter()
    for _ in range(runs):
        with torch.no_grad():
            teacher(dummy)
    teacher_time = (time.perf_counter() - start) / runs * 1000
    
    start = time.perf_counter()
    for _ in range(runs):
        with torch.no_grad():
            student(dummy)
    student_time = (time.perf_counter() - start) / runs * 1000
    
    print(f"Teacher latency:    {teacher_time:.2f} ms")
    print(f"Student latency:    {student_time:.2f} ms")
    print(f"Speedup:            {teacher_time/student_time:.1f}x")


# ============================================
# Run everything
# ============================================

if __name__ == "__main__":
    teacher = TeacherModel()
    student = StudentModel()
    
    # Compare sizes
    compare_models(teacher, student)
    
    # In practice, you would:
    # 1. Train teacher on full dataset to high accuracy
    # 2. Run distillation to train student
    # 3. Evaluate student accuracy vs teacher
    # 4. Deploy the student model (smaller, faster)
```

### Distillation Variants

| Variant | Description | Best For |
|---------|-------------|----------|
| **Response Distillation** | Match final output logits | General use |
| **Feature Distillation** | Match intermediate feature maps | CV models |
| **Attention Distillation** | Match attention maps | Transformers |
| **Self-Distillation** | Same architecture, deeper→shallower | ResNets |
| **Online Distillation** | Two models teach each other | No pre-trained teacher |
| **Multi-Teacher** | Multiple teachers for one student | Ensemble → single model |

---

## ONNX — Open Neural Network Exchange

### What is ONNX?

ONNX is an open format for representing ML models. It's like PDF for documents — no matter what tool created the document (Word, Google Docs, LaTeX), anyone can open a PDF.

### Why ONNX?

```
WITHOUT ONNX:                    WITH ONNX:
                                 
PyTorch model → PyTorch only     PyTorch → ONNX ──┬── ONNX Runtime (CPU)
TF model → TF Serving only                        ├── TensorRT (NVIDIA GPU)
Sklearn → custom API                               ├── OpenVINO (Intel)
                                                   ├── CoreML (Apple)
Vendor lock-in!                                    ├── TFLite (Mobile)
                                                   └── DirectML (Windows)
                                 
                                 One model, deploy everywhere!
```

### Code: ONNX Conversion & Inference

```python
# onnx_workflow.py
"""
ONNX: Convert, optimize, and run inference with ONNX Runtime.
"""

import torch
import torch.nn as nn
import numpy as np
import time

# ============================================
# STEP 1: Export PyTorch Model to ONNX
# ============================================

class ImageClassifier(nn.Module):
    def __init__(self, num_classes=10):
        super().__init__()
        self.conv1 = nn.Conv2d(3, 32, 3, padding=1)
        self.relu = nn.ReLU()
        self.pool = nn.AdaptiveAvgPool2d(1)
        self.fc = nn.Linear(32, num_classes)
    
    def forward(self, x):
        x = self.relu(self.conv1(x))
        x = self.pool(x)
        x = x.view(x.size(0), -1)
        return self.fc(x)

# Create model
model = ImageClassifier()
model.eval()

# Create dummy input (batch of 1 image, 3 channels, 224x224)
dummy_input = torch.randn(1, 3, 224, 224)

# Export to ONNX
torch.onnx.export(
    model,                          # Model to export
    dummy_input,                    # Example input
    "model.onnx",                   # Output file
    export_params=True,             # Include trained weights
    opset_version=17,               # ONNX opset version
    do_constant_folding=True,       # Optimize constant expressions
    input_names=['input'],          # Input tensor name
    output_names=['output'],        # Output tensor name
    dynamic_axes={                  # Allow variable batch size
        'input': {0: 'batch_size'},
        'output': {0: 'batch_size'}
    }
)

print("Model exported to ONNX format!")


# ============================================
# STEP 2: Validate ONNX Model
# ============================================

import onnx

# Load and check the ONNX model
onnx_model = onnx.load("model.onnx")
onnx.checker.check_model(onnx_model)  # Validates model structure
print("ONNX model is valid!")

# Print model info
print(f"IR version: {onnx_model.ir_version}")
print(f"Opset: {onnx_model.opset_import[0].version}")
print(f"Graph inputs: {[i.name for i in onnx_model.graph.input]}")
print(f"Graph outputs: {[o.name for o in onnx_model.graph.output]}")


# ============================================
# STEP 3: Run Inference with ONNX Runtime
# ============================================

import onnxruntime as ort

# Create inference session
# Providers specify execution backends (CPU, GPU, TensorRT, etc.)
session = ort.InferenceSession(
    "model.onnx",
    providers=[
        'CUDAExecutionProvider',     # Try GPU first
        'CPUExecutionProvider'       # Fall back to CPU
    ]
)

# Check which provider is being used
print(f"Using provider: {session.get_providers()}")

# Run inference
input_data = np.random.randn(1, 3, 224, 224).astype(np.float32)

# Get input/output names
input_name = session.get_inputs()[0].name
output_name = session.get_outputs()[0].name

# Predict
result = session.run(
    [output_name],                   # Output names to fetch
    {input_name: input_data}         # Input dict
)

print(f"Prediction shape: {result[0].shape}")
print(f"Predictions: {result[0][0][:5]}")  # First 5 class scores


# ============================================
# STEP 4: Optimize ONNX Model
# ============================================

# Method 1: Using ONNX optimizer (graph-level optimizations)
from onnxruntime.transformers import optimizer

"""
# For transformer models:
optimized_model = optimizer.optimize_model(
    "bert_model.onnx",
    model_type="bert",          # or "gpt2", "vit", etc.
    num_heads=12,
    hidden_size=768
)
optimized_model.save_model_to_file("bert_optimized.onnx")
"""

# Method 2: Using session options for runtime optimization
session_options = ort.SessionOptions()

# Graph optimization level
session_options.graph_optimization_level = ort.GraphOptimizationLevel.ORT_ENABLE_ALL
# Options:
# ORT_DISABLE_ALL     - No optimization
# ORT_ENABLE_BASIC    - Basic (constant folding, redundant node removal)
# ORT_ENABLE_EXTENDED - + complex node fusions
# ORT_ENABLE_ALL      - + layout optimization (recommended)

# Threading
session_options.intra_op_num_threads = 4  # Threads per operation
session_options.inter_op_num_threads = 2  # Threads between operations

# Memory optimization
session_options.enable_mem_pattern = True     # Optimize memory allocation
session_options.enable_cpu_mem_arena = True   # Use memory arena

# Save optimized model
session_options.optimized_model_filepath = "model_optimized.onnx"

optimized_session = ort.InferenceSession(
    "model.onnx",
    sess_options=session_options,
    providers=['CPUExecutionProvider']
)

# ============================================
# STEP 5: Benchmark PyTorch vs ONNX Runtime
# ============================================

def benchmark_pytorch(model, input_tensor, num_runs=500):
    """Benchmark PyTorch inference."""
    model.eval()
    # Warmup
    for _ in range(50):
        with torch.no_grad():
            _ = model(input_tensor)
    
    start = time.perf_counter()
    for _ in range(num_runs):
        with torch.no_grad():
            _ = model(input_tensor)
    elapsed = (time.perf_counter() - start) / num_runs * 1000
    return elapsed

def benchmark_onnx(session, input_data, num_runs=500):
    """Benchmark ONNX Runtime inference."""
    input_name = session.get_inputs()[0].name
    output_name = session.get_outputs()[0].name
    
    # Warmup
    for _ in range(50):
        session.run([output_name], {input_name: input_data})
    
    start = time.perf_counter()
    for _ in range(num_runs):
        session.run([output_name], {input_name: input_data})
    elapsed = (time.perf_counter() - start) / num_runs * 1000
    return elapsed

# Run benchmarks
pytorch_time = benchmark_pytorch(model, dummy_input)
onnx_time = benchmark_onnx(optimized_session, input_data)

print(f"\n{'='*40}")
print(f"BENCHMARK RESULTS")
print(f"{'='*40}")
print(f"PyTorch:      {pytorch_time:.2f} ms/inference")
print(f"ONNX Runtime: {onnx_time:.2f} ms/inference")
print(f"Speedup:      {pytorch_time/onnx_time:.2f}x")


# ============================================
# STEP 6: Convert Sklearn/XGBoost to ONNX
# ============================================

"""
# Sklearn to ONNX (using skl2onnx)
from skl2onnx import convert_sklearn
from skl2onnx.common.data_types import FloatTensorType
from sklearn.ensemble import RandomForestClassifier

# Train sklearn model
rf_model = RandomForestClassifier(n_estimators=100)
rf_model.fit(X_train, y_train)

# Convert to ONNX
initial_type = [('input', FloatTensorType([None, X_train.shape[1]]))]
onnx_model = convert_sklearn(rf_model, initial_types=initial_type)
onnx.save(onnx_model, "rf_model.onnx")

# XGBoost to ONNX (using onnxmltools)
import onnxmltools
from onnxmltools.convert.common.data_types import FloatTensorType

xgb_onnx = onnxmltools.convert_xgboost(
    xgb_model, 
    initial_types=[('input', FloatTensorType([None, num_features]))]
)
onnxmltools.utils.save_model(xgb_onnx, 'xgb_model.onnx')
"""

# Cleanup
import os
for f in ['model.onnx', 'model_optimized.onnx']:
    if os.path.exists(f):
        os.remove(f)
```

---

## TensorRT

### What is TensorRT?

TensorRT is NVIDIA's high-performance deep learning inference optimizer and runtime. It takes a trained model and produces a hyper-optimized engine specifically for your target NVIDIA GPU.

### How TensorRT Optimizes

```
Input Model (PyTorch/TF/ONNX)
         │
         ▼
┌─────────────────────────────┐
│   TensorRT Optimizations     │
├─────────────────────────────┤
│                               │
│  1. LAYER FUSION              │
│     Conv + BN + ReLU → 1 op  │
│                               │
│  2. PRECISION CALIBRATION     │
│     FP32 → FP16/INT8         │
│     (per-layer decision)      │
│                               │
│  3. KERNEL AUTO-TUNING        │
│     Test 100s of CUDA kernels │
│     Pick fastest for YOUR GPU │
│                               │
│  4. MEMORY OPTIMIZATION       │
│     Tensor memory reuse       │
│     Workspace allocation      │
│                               │
│  5. DYNAMIC TENSOR MEMORY     │
│     Allocate only what's      │
│     needed at each layer      │
│                               │
└──────────┬──────────────────┘
           │
           ▼
   TensorRT Engine (.engine/.plan)
   Optimized for your specific GPU!
```

### Code: TensorRT Optimization

```python
# tensorrt_optimization.py
"""
TensorRT: Convert ONNX model to optimized TensorRT engine.
Requires: NVIDIA GPU + TensorRT installed.
"""

import tensorrt as trt
import numpy as np
import pycuda.driver as cuda
import pycuda.autoinit  # Initializes CUDA

# ============================================
# STEP 1: Build TensorRT Engine from ONNX
# ============================================

def build_engine(onnx_path, engine_path, precision="fp16", max_batch_size=8):
    """
    Build TensorRT engine from ONNX model.
    
    Args:
        onnx_path: Path to ONNX model
        engine_path: Where to save the TensorRT engine
        precision: "fp32", "fp16", or "int8"
        max_batch_size: Maximum batch size the engine will support
    """
    logger = trt.Logger(trt.Logger.WARNING)
    builder = trt.Builder(logger)
    
    # Create network definition
    network = builder.create_network(
        1 << int(trt.NetworkDefinitionCreationFlag.EXPLICIT_BATCH)
    )
    
    # Parse ONNX model
    parser = trt.OnnxParser(network, logger)
    with open(onnx_path, 'rb') as f:
        if not parser.parse(f.read()):
            for error in range(parser.num_errors):
                print(f"ONNX Parse Error: {parser.get_error(error)}")
            raise RuntimeError("Failed to parse ONNX model")
    
    # Configure builder
    config = builder.create_builder_config()
    config.set_memory_pool_limit(trt.MemoryPoolType.WORKSPACE, 1 << 30)  # 1 GB
    
    # Set precision
    if precision == "fp16" and builder.platform_has_fast_fp16:
        config.set_flag(trt.BuilderFlag.FP16)
        print("Using FP16 precision")
    elif precision == "int8" and builder.platform_has_fast_int8:
        config.set_flag(trt.BuilderFlag.INT8)
        # INT8 requires calibration (see below)
        print("Using INT8 precision")
    else:
        print("Using FP32 precision")
    
    # Set dynamic batch size
    profile = builder.create_optimization_profile()
    input_shape = network.get_input(0).shape
    
    # (min_batch, opt_batch, max_batch) for dynamic batching
    profile.set_shape(
        network.get_input(0).name,
        min=(1, *input_shape[1:]),          # Minimum shape
        opt=(max_batch_size // 2, *input_shape[1:]),  # Optimal shape
        max=(max_batch_size, *input_shape[1:])        # Maximum shape
    )
    config.add_optimization_profile(profile)
    
    # Build engine (this takes minutes — it tests thousands of kernel configurations)
    print("Building TensorRT engine... (this may take several minutes)")
    engine = builder.build_serialized_network(network, config)
    
    if engine is None:
        raise RuntimeError("Failed to build TensorRT engine")
    
    # Save engine to file
    with open(engine_path, 'wb') as f:
        f.write(engine)
    
    print(f"Engine saved to {engine_path}")
    return engine


# ============================================
# STEP 2: Run Inference with TensorRT Engine
# ============================================

class TensorRTInference:
    """
    TensorRT inference wrapper.
    Handles memory allocation and inference execution.
    """
    
    def __init__(self, engine_path):
        """Load engine and allocate memory."""
        self.logger = trt.Logger(trt.Logger.WARNING)
        
        # Load engine
        with open(engine_path, 'rb') as f:
            runtime = trt.Runtime(self.logger)
            self.engine = runtime.deserialize_cuda_engine(f.read())
        
        self.context = self.engine.create_execution_context()
        
        # Allocate GPU memory for inputs/outputs
        self.inputs = []
        self.outputs = []
        self.bindings = []
        self.stream = cuda.Stream()
        
        for i in range(self.engine.num_io_tensors):
            name = self.engine.get_tensor_name(i)
            dtype = trt.nptype(self.engine.get_tensor_dtype(name))
            shape = self.engine.get_tensor_shape(name)
            size = abs(np.prod(shape)) * np.dtype(dtype).itemsize
            
            # Allocate device memory
            device_mem = cuda.mem_alloc(size)
            self.bindings.append(int(device_mem))
            
            if self.engine.get_tensor_mode(name) == trt.TensorIOMode.INPUT:
                self.inputs.append({
                    'name': name, 'dtype': dtype, 'shape': shape,
                    'device': device_mem, 'host': np.empty(shape, dtype=dtype)
                })
            else:
                self.outputs.append({
                    'name': name, 'dtype': dtype, 'shape': shape,
                    'device': device_mem, 'host': np.empty(shape, dtype=dtype)
                })
    
    def predict(self, input_data):
        """
        Run inference on input data.
        
        Args:
            input_data: numpy array matching input shape
            
        Returns:
            numpy array with model predictions
        """
        # Copy input to host buffer
        np.copyto(self.inputs[0]['host'], input_data.ravel())
        
        # Transfer input to GPU
        cuda.memcpy_htod_async(
            self.inputs[0]['device'],
            self.inputs[0]['host'],
            self.stream
        )
        
        # Set tensor addresses
        for inp in self.inputs:
            self.context.set_tensor_address(inp['name'], int(inp['device']))
        for out in self.outputs:
            self.context.set_tensor_address(out['name'], int(out['device']))
        
        # Run inference
        self.context.execute_async_v3(stream_handle=self.stream.handle)
        
        # Transfer output back to CPU
        cuda.memcpy_dtoh_async(
            self.outputs[0]['host'],
            self.outputs[0]['device'],
            self.stream
        )
        
        self.stream.synchronize()
        
        return self.outputs[0]['host'].copy()


# ============================================
# STEP 3: INT8 Calibration
# ============================================

class Int8Calibrator(trt.IInt8EntropyCalibrator2):
    """
    INT8 calibrator for TensorRT.
    Uses representative data to determine optimal quantization ranges.
    """
    
    def __init__(self, calibration_data, batch_size=32, cache_file="calibration.cache"):
        super().__init__()
        self.calibration_data = calibration_data
        self.batch_size = batch_size
        self.cache_file = cache_file
        self.current_index = 0
        
        # Allocate GPU memory for calibration batch
        self.device_input = cuda.mem_alloc(
            calibration_data[0:1].nbytes * batch_size
        )
    
    def get_batch_size(self):
        return self.batch_size
    
    def get_batch(self, names):
        """Supply calibration data batch by batch."""
        if self.current_index + self.batch_size > len(self.calibration_data):
            return None
        
        batch = self.calibration_data[self.current_index:self.current_index + self.batch_size]
        cuda.memcpy_htod(self.device_input, batch.ravel())
        self.current_index += self.batch_size
        
        return [int(self.device_input)]
    
    def read_calibration_cache(self):
        """Read cached calibration data if available."""
        import os
        if os.path.exists(self.cache_file):
            with open(self.cache_file, 'rb') as f:
                return f.read()
        return None
    
    def write_calibration_cache(self, cache):
        """Cache calibration data for reuse."""
        with open(self.cache_file, 'wb') as f:
            f.write(cache)


# ============================================
# STEP 4: Using TensorRT with torch-tensorrt
# ============================================

"""
# Easier approach: Use torch-tensorrt (PyTorch integration)

import torch
import torch_tensorrt

model = YourModel()
model.eval().cuda()

# Compile with TensorRT
trt_model = torch_tensorrt.compile(
    model,
    inputs=[
        torch_tensorrt.Input(
            min_shape=[1, 3, 224, 224],
            opt_shape=[8, 3, 224, 224],
            max_shape=[32, 3, 224, 224],
            dtype=torch.float16  # FP16 precision
        )
    ],
    enabled_precisions={torch.float16},  # Enable FP16
    workspace_size=1 << 30               # 1 GB workspace
)

# Use like a normal PyTorch model
input_data = torch.randn(1, 3, 224, 224).cuda().half()
output = trt_model(input_data)

# Save compiled model
torch.jit.save(trt_model, "model_trt.ts")
"""


# ============================================
# STEP 5: Benchmark Comparison
# ============================================

"""
Typical speedups with TensorRT (NVIDIA T4 GPU):

| Model        | PyTorch | TensorRT FP16 | TensorRT INT8 | Speedup |
|-------------|---------|---------------|---------------|---------|
| ResNet-50   | 8.2 ms  | 2.8 ms        | 1.5 ms        | 3-5.5x  |
| BERT-base   | 12.5 ms | 4.1 ms        | 2.2 ms        | 3-5.7x  |
| YOLOv5-s    | 6.4 ms  | 2.1 ms        | 1.3 ms        | 3-4.9x  |
| EfficientNet| 9.1 ms  | 3.2 ms        | 1.8 ms        | 2.8-5x  |
"""
```

---

## Other Optimization Techniques

### Operator Fusion

```python
# Operator fusion combines multiple operations into one GPU kernel
# This reduces memory reads/writes between operations

# WITHOUT fusion (3 separate GPU kernels):
#   Kernel 1: Conv2d → write to memory
#   Kernel 2: Read from memory → BatchNorm → write to memory
#   Kernel 3: Read from memory → ReLU → write to memory

# WITH fusion (1 fused GPU kernel):
#   Kernel 1: Conv2d + BatchNorm + ReLU → write to memory

# PyTorch automatically fuses some operations:
import torch

model = torch.jit.script(model)  # TorchScript enables fusion

# Or use torch.compile (PyTorch 2.0+):
model = torch.compile(model, mode="reduce-overhead")
# Modes:
#   "default"        - Good balance of compilation time and speedup
#   "reduce-overhead" - Best latency (longer compile time)
#   "max-autotune"   - Maximum performance (longest compile time)
```

### Mixed Precision Inference

```python
# Use FP16 for computation, FP32 for sensitive operations
# Typically gives 1.5-2x speedup on modern GPUs with ~0% accuracy loss

import torch

model = model.half().cuda()  # Convert model to FP16
input_data = input_data.half().cuda()  # Convert input to FP16

with torch.no_grad():
    # Inference in FP16
    output = model(input_data)
    # Convert output back to FP32 if needed
    output = output.float()

# For more control, use torch.cuda.amp
with torch.cuda.amp.autocast(dtype=torch.float16):
    output = model(input_data)
```

### Model Compilation (PyTorch 2.0+)

```python
# torch.compile — The easiest optimization for PyTorch models
import torch

model = YourModel().eval()

# One line to optimize!
compiled_model = torch.compile(model)

# First inference is slow (compilation), subsequent are fast
output = compiled_model(input_data)

# Advanced: specify backend
compiled_model = torch.compile(
    model,
    backend="inductor",          # Default backend (best for most cases)
    mode="max-autotune",         # Maximum optimization
    fullgraph=True               # Compile entire graph (no fallbacks)
)
```

---

## Optimization Pipeline — Combining Techniques

### Recommended Optimization Order

```
Step 1: torch.compile() or ONNX Runtime
        → Free speedup, no accuracy loss
        → Expected: 1.2-2x faster

Step 2: FP16 / Mixed Precision
        → Almost free, ~0% accuracy loss
        → Expected: 1.5-2x faster (cumulative: 2-4x)

Step 3: Quantization (INT8)
        → Small accuracy impact, big speed gain
        → Expected: 2-4x faster (cumulative: 4-8x)

Step 4: Pruning (if needed)
        → Removes redundant weights
        → Expected: 1.5-2x faster (cumulative: 6-16x)

Step 5: Distillation (if needed)
        → Replace with smaller architecture entirely
        → Expected: 2-10x faster (cumulative: 10-100x)

Step 6: TensorRT (GPU) or OpenVINO (CPU)
        → Platform-specific kernel optimization
        → Expected: 1.5-3x on top of everything
```

### Complete Pipeline Code

```python
# optimization_pipeline.py
"""
Complete model optimization pipeline: 
PyTorch → ONNX → Quantized ONNX → TensorRT
"""

import torch
import torch.nn as nn
import onnx
import onnxruntime as ort
from onnxruntime.quantization import quantize_dynamic, QuantType
import numpy as np
import time
import os

def full_optimization_pipeline(model, input_shape, output_dir="optimized_models"):
    """
    Apply all optimization techniques sequentially.
    
    Args:
        model: Trained PyTorch model (in eval mode)
        input_shape: Tuple for dummy input (e.g., (1, 3, 224, 224))
        output_dir: Directory to save optimized models
    """
    os.makedirs(output_dir, exist_ok=True)
    results = {}
    dummy_input = torch.randn(*input_shape)
    dummy_np = dummy_input.numpy()
    
    # --- Baseline: PyTorch ---
    model.eval()
    results['pytorch_fp32'] = benchmark(
        lambda: model(dummy_input), 
        "PyTorch FP32"
    )
    
    # --- Step 1: Export to ONNX ---
    onnx_path = os.path.join(output_dir, "model.onnx")
    torch.onnx.export(
        model, dummy_input, onnx_path,
        opset_version=17,
        input_names=['input'],
        output_names=['output'],
        dynamic_axes={'input': {0: 'batch'}, 'output': {0: 'batch'}}
    )
    
    # --- Step 2: ONNX Runtime ---
    ort_session = ort.InferenceSession(onnx_path, providers=['CPUExecutionProvider'])
    input_name = ort_session.get_inputs()[0].name
    results['onnx_runtime'] = benchmark(
        lambda: ort_session.run(None, {input_name: dummy_np}),
        "ONNX Runtime FP32"
    )
    
    # --- Step 3: ONNX Quantized (INT8) ---
    quant_path = os.path.join(output_dir, "model_int8.onnx")
    quantize_dynamic(
        onnx_path,
        quant_path,
        weight_type=QuantType.QInt8
    )
    
    quant_session = ort.InferenceSession(quant_path, providers=['CPUExecutionProvider'])
    results['onnx_int8'] = benchmark(
        lambda: quant_session.run(None, {input_name: dummy_np}),
        "ONNX Runtime INT8"
    )
    
    # --- Step 4: torch.compile (PyTorch 2.0+) ---
    try:
        compiled_model = torch.compile(model, mode="reduce-overhead")
        # Warmup compile
        compiled_model(dummy_input)
        results['torch_compile'] = benchmark(
            lambda: compiled_model(dummy_input),
            "torch.compile"
        )
    except Exception as e:
        print(f"torch.compile not available: {e}")
    
    # --- Print comparison ---
    print(f"\n{'='*60}")
    print(f"{'OPTIMIZATION RESULTS':^60}")
    print(f"{'='*60}")
    print(f"{'Method':<25} {'Latency (ms)':<15} {'Speedup':<10} {'Size (MB)':<10}")
    print(f"{'-'*60}")
    
    baseline = results['pytorch_fp32']['latency']
    sizes = {
        'pytorch_fp32': os.path.getsize(os.path.join(output_dir, "model.onnx")) / 1e6,
        'onnx_runtime': os.path.getsize(onnx_path) / 1e6,
        'onnx_int8': os.path.getsize(quant_path) / 1e6,
    }
    
    for name, data in results.items():
        speedup = baseline / data['latency']
        size = sizes.get(name, 0)
        print(f"{name:<25} {data['latency']:<15.2f} {speedup:<10.2f}x {size:<10.1f}")
    
    # Cleanup
    for f in [onnx_path, quant_path]:
        if os.path.exists(f):
            os.remove(f)
    if os.path.exists(output_dir):
        os.rmdir(output_dir)
    
    return results

def benchmark(fn, name, num_runs=200, warmup=50):
    """Benchmark a function."""
    # Warmup
    for _ in range(warmup):
        with torch.no_grad():
            fn()
    
    # Measure
    start = time.perf_counter()
    for _ in range(num_runs):
        with torch.no_grad():
            fn()
    avg_ms = (time.perf_counter() - start) / num_runs * 1000
    
    print(f"  {name}: {avg_ms:.2f} ms/inference")
    return {'latency': avg_ms}

# Usage:
# model = YourModel().eval()
# full_optimization_pipeline(model, input_shape=(1, 3, 224, 224))
```

---

## Common Mistakes

### 1. Optimizing Before Profiling
```python
# ❌ WRONG: Blindly apply every optimization technique
model = quantize(prune(distill(model)))  # Overkill?

# ✅ RIGHT: Profile first, then optimize the bottleneck
import torch.profiler

with torch.profiler.profile(
    activities=[torch.profiler.ProfilerActivity.CPU],
    record_shapes=True
) as prof:
    model(input_data)

print(prof.key_averages().table(sort_by="cpu_time_total", row_limit=10))
# Now you KNOW which layers are slow → optimize those
```

### 2. Not Validating After Each Step
```python
# ❌ WRONG: Optimize and deploy without checking accuracy
quantized_model = quantize(model)
deploy(quantized_model)  # Hope for the best!

# ✅ RIGHT: Validate at every step
original_acc = evaluate(model, test_loader)
quantized_acc = evaluate(quantized_model, test_loader)
print(f"Accuracy drop: {original_acc - quantized_acc:.4f}")

if original_acc - quantized_acc > 0.02:  # More than 2% drop
    print("WARNING: Significant accuracy loss! Reconsider quantization settings.")
```

### 3. Wrong Quantization Type for the Hardware
```python
# ❌ WRONG: Using x86 quantization config for ARM mobile
model.qconfig = torch.quantization.get_default_qconfig('x86')  # Wrong for mobile!

# ✅ RIGHT: Match quantization config to target hardware
if target == "mobile":
    model.qconfig = torch.quantization.get_default_qconfig('qnnpack')  # ARM
elif target == "server":
    model.qconfig = torch.quantization.get_default_qconfig('x86')  # Intel CPU
elif target == "gpu":
    # Use TensorRT or FP16 instead of CPU quantization
    pass
```

### 4. Ignoring Calibration Data Quality
```python
# ❌ WRONG: Calibrate INT8 with random data
calibration_data = np.random.randn(100, 3, 224, 224)  # Random garbage

# ✅ RIGHT: Use REPRESENTATIVE real data for calibration
calibration_data = load_subset_of_real_data(n=500)  # Actual images from your dataset
# Should cover the distribution of real inputs the model will see
```

### 5. TensorRT Engine Not Portable
```python
# ❌ WRONG: Build TensorRT engine on one GPU, deploy on another
# Engine built on A100 will NOT work on T4

# ✅ RIGHT: Build engine on the SAME GPU type as deployment target
# Or use ONNX Runtime with TensorRT execution provider (more portable)
session = ort.InferenceSession(
    "model.onnx",
    providers=['TensorrtExecutionProvider', 'CUDAExecutionProvider']
)
# This builds the engine at startup for the current GPU
```

---

## Interview Questions

### Conceptual

**Q1: Explain the trade-offs between quantization, pruning, and distillation.**
> **A:** Quantization reduces numerical precision (FP32→INT8) — easiest to apply, smallest accuracy loss (~0.5-2%), gives 2-4x speedup. Pruning removes weights — requires fine-tuning to recover accuracy, unstructured pruning needs special hardware for speedup, structured pruning works on any hardware. Distillation trains a smaller model from scratch using teacher knowledge — most effort but best compression ratios (10x+ possible), requires retraining, can change model architecture entirely. Use quantization first (low-hanging fruit), then pruning for moderate gains, distillation when you need radical size reduction.

**Q2: What is the difference between post-training quantization and quantization-aware training?**
> **A:** PTQ applies quantization after training — fast, no retraining needed, but accuracy drops more because the model wasn't trained to handle quantization noise. QAT simulates quantization during training using "fake quantization" nodes — model learns weights robust to rounding errors, resulting in 0.5-1% better accuracy than PTQ. Use PTQ for quick experiments and when accuracy loss is acceptable; use QAT when you need maximum accuracy at INT8 precision.

**Q3: Why is ONNX important for production ML?**
> **A:** ONNX provides framework interoperability — train in PyTorch, deploy in TensorRT. It eliminates vendor lock-in to specific frameworks. ONNX Runtime provides optimized inference across hardware (CPU, GPU, edge) without framework overhead. It enables hardware-specific optimizations via execution providers (TensorRT, OpenVINO, CoreML) through a single model format.

**Q4: How does TensorRT achieve such dramatic speedups?**
> **A:** TensorRT applies multiple optimizations: (1) Layer/operator fusion reduces memory transfers between operations, (2) Kernel auto-tuning tests hundreds of CUDA kernel implementations to find the fastest for your specific GPU, (3) Precision calibration selects optimal FP16/INT8 precision per-layer, (4) Memory optimization minimizes GPU memory usage through tensor memory reuse, (5) The engine is compiled and specialized for the exact GPU and batch size.

**Q5: How would you optimize a BERT model that needs to serve 1000 requests/second with <50ms latency?**
> **A:** Step 1: Export to ONNX and use ONNX Runtime (1.5-2x). Step 2: Apply dynamic INT8 quantization to linear layers (2-3x). Step 3: If on NVIDIA GPU, convert to TensorRT FP16 (additional 2-3x). Step 4: If still too slow, use DistilBERT (knowledge distillation — 2x smaller, 40% faster, 97% accuracy). Step 5: Enable batching to maximize GPU utilization. Step 6: Deploy multiple replicas behind a load balancer. Combined, you can achieve 10-20x speedup from baseline.

### Practical

**Q6: You optimized a model with INT8 quantization and accuracy dropped 5%. How do you fix it?**
> **A:** (1) Try quantization-aware training (QAT) instead of post-training quantization. (2) Use mixed-precision — keep sensitive layers (first/last convolutions, attention layers) in FP16 while quantizing others to INT8. (3) Improve calibration data — use more representative samples covering edge cases. (4) Try per-channel quantization instead of per-tensor for Conv layers. (5) If still too much loss, try FP16 instead of INT8 (less speedup but near-zero accuracy loss).

---

## Quick Reference

### Optimization Decision Flowchart

```
Start: Is the model too slow/large?
  │
  ├── Try torch.compile() or ONNX Runtime first (free speedup)
  │     └── Sufficient? → Done ✓
  │
  ├── Apply FP16 (GPU) or BF16 inference
  │     └── Sufficient? → Done ✓
  │
  ├── Apply INT8 Quantization (PTQ first, QAT if needed)
  │     └── Sufficient? → Done ✓
  │
  ├── Apply Pruning (50-80% sparsity with fine-tuning)
  │     └── Sufficient? → Done ✓
  │
  ├── Apply TensorRT/OpenVINO (platform-specific)
  │     └── Sufficient? → Done ✓
  │
  └── Knowledge Distillation (smaller architecture)
        └── Sufficient? → Done ✓
              └── No? → Rethink your latency/accuracy requirements
```

### Cheat Sheet Table

| Technique | Tool | One-Liner | Accuracy Impact |
|-----------|------|-----------|----------------|
| **ONNX Export** | `torch.onnx.export()` | Framework-independent format | 0% |
| **ONNX Runtime** | `ort.InferenceSession()` | Optimized cross-platform inference | 0% |
| **FP16** | `model.half()` | Half-precision on GPU | ~0% |
| **Dynamic Quant** | `quantize_dynamic()` | INT8 weights, easy to apply | 0.5-1% |
| **Static Quant** | `prepare() → calibrate → convert()` | INT8 weights+activations | 0.5-2% |
| **QAT** | `prepare_qat() → train → convert()` | Best INT8 quality | 0.1-0.5% |
| **Pruning** | `prune.global_unstructured()` | Remove small weights | 0.5-3% |
| **Distillation** | `DistillationLoss()` | Small student from big teacher | 1-5% |
| **TensorRT** | `trt.Builder()` | NVIDIA GPU-optimized engine | 0-1% |
| **torch.compile** | `torch.compile(model)` | PyTorch 2.0+ JIT compilation | 0% |

### Platform-Specific Optimization Tools

| Target Hardware | Best Tool | Format |
|----------------|-----------|--------|
| **NVIDIA GPU** | TensorRT | `.engine` / `.plan` |
| **Intel CPU** | OpenVINO | `.xml` + `.bin` |
| **Apple (iOS/Mac)** | CoreML | `.mlmodel` |
| **Android/Mobile** | TFLite | `.tflite` |
| **AMD GPU** | ROCm + MIGraphX | ONNX |
| **Any CPU** | ONNX Runtime | `.onnx` |
| **Any GPU** | ONNX Runtime + EP | `.onnx` |
| **Browser** | ONNX.js / TF.js | `.onnx` / `.json` |

---

> **Pro Tip:** The 80/20 rule applies here. `torch.compile()` or ONNX Runtime + FP16 gives you 80% of the possible speedup with 20% of the effort. Only go deeper (TensorRT, INT8, pruning) when those easy wins aren't enough.
