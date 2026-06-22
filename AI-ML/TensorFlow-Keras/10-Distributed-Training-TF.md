# Chapter 10: Distributed Training in TensorFlow

## Table of Contents
- [10.1 Introduction to Distributed Training](#101-introduction-to-distributed-training)
- [10.2 Data Parallelism vs Model Parallelism](#102-data-parallelism-vs-model-parallelism)
- [10.3 tf.distribute.Strategy Overview](#103-tfdistributestrategy-overview)
- [10.4 MirroredStrategy (Single Machine, Multi-GPU)](#104-mirroredstrategy-single-machine-multi-gpu)
- [10.5 MultiWorkerMirroredStrategy (Multi-Machine)](#105-multiworkermirroredstrategy-multi-machine)
- [10.6 TPUStrategy (Training on TPUs)](#106-tpustrategy-training-on-tpus)
- [10.7 ParameterServerStrategy](#107-parameterserverstrategy)
- [10.8 Mixed Precision Training](#108-mixed-precision-training)
- [10.9 Performance Optimization for Distributed Training](#109-performance-optimization-for-distributed-training)
- [10.10 Common Mistakes](#1010-common-mistakes)
- [10.11 Interview Questions](#1011-interview-questions)
- [10.12 Quick Reference](#1012-quick-reference)

---

## 10.1 Introduction to Distributed Training

### What It Is
Distributed training is like splitting a huge homework assignment among several students. Instead of one person doing all the work, multiple GPUs (or machines) share the workload so the model trains much faster.

When you train a neural network, the math (matrix multiplications, gradient calculations) is computationally expensive. A single GPU might take days or weeks for large models. Distributed training lets you use **multiple accelerators** (GPUs/TPUs) to train in parallel, reducing that time dramatically.

### Why It Matters
- **Speed**: Train models 2x, 4x, 8x faster (or more) with multiple GPUs
- **Scale**: Modern models (GPT, BERT, ResNet-152) are too large or require too much data for a single GPU
- **Cost Efficiency**: Faster training = less cloud compute cost (even though you use more GPUs, total cost can be lower)
- **Production Necessity**: Every major AI company uses distributed training — it's not optional at scale

### Real-World Analogy
Imagine you need to read 1,000 pages for an exam:
- **Single GPU**: You read all 1,000 pages yourself — takes forever
- **Data Parallelism**: 4 friends each read all 1,000 pages but work on different practice problems (data) simultaneously, then share notes
- **Model Parallelism**: You split the book into 4 sections, each friend reads 250 pages, then you piece together the full understanding

### How Training Scales

```
Single GPU Training:
┌──────────┐
│   GPU 0  │ ──→ Process ALL batches sequentially
└──────────┘
Time: ████████████████████████████████ (100%)

4-GPU Distributed Training:
┌──────────┐
│   GPU 0  │ ──→ Process Batch 1, 5, 9, ...
├──────────┤
│   GPU 1  │ ──→ Process Batch 2, 6, 10, ...
├──────────┤
│   GPU 2  │ ──→ Process Batch 3, 7, 11, ...
├──────────┤
│   GPU 3  │ ──→ Process Batch 4, 8, 12, ...
└──────────┘
Time: ████████ (~25% + overhead)
```

---

## 10.2 Data Parallelism vs Model Parallelism

### What They Are

**Data Parallelism**: Every GPU has a **complete copy** of the model. Each GPU processes a **different batch** of data. Gradients are averaged across GPUs, and weights are synchronized.

**Model Parallelism**: The model is **split across GPUs**. Each GPU holds a **portion** of the model. Data flows through GPUs sequentially (pipeline parallelism) or specific layers live on specific GPUs.

### Visual Comparison

```
DATA PARALLELISM:
┌─────────────────────────────────────────────────┐
│                  Training Data                   │
│  [Batch1] [Batch2] [Batch3] [Batch4] ...        │
└──────┬──────┬──────┬──────┬─────────────────────┘
       │      │      │      │
       ▼      ▼      ▼      ▼
   ┌──────┐┌──────┐┌──────┐┌──────┐
   │GPU 0 ││GPU 1 ││GPU 2 ││GPU 3 │  ← Each has FULL model copy
   │Model ││Model ││Model ││Model │
   │Copy  ││Copy  ││Copy  ││Copy  │
   └──┬───┘└──┬───┘└──┬───┘└──┬───┘
      │       │       │       │
      └───────┴───┬───┴───────┘
                  │
          AllReduce (average gradients)
                  │
          Synchronized Weights

MODEL PARALLELISM:
┌─────────────────┐
│  Training Data  │
│  [Full Batch]   │
└────────┬────────┘
         │
         ▼
   ┌──────────┐
   │  GPU 0   │  ← Layers 1-10
   │ Layer1-10│
   └────┬─────┘
        │ (activations)
        ▼
   ┌──────────┐
   │  GPU 1   │  ← Layers 11-20
   │Layer11-20│
   └────┬─────┘
        │ (activations)
        ▼
   ┌──────────┐
   │  GPU 2   │  ← Layers 21-30
   │Layer21-30│
   └──────────┘
```

### When to Use What

| Aspect | Data Parallelism | Model Parallelism |
|--------|-----------------|-------------------|
| **Use when** | Model fits in single GPU memory | Model is too large for single GPU |
| **Scales** | Linearly with #GPUs (near-ideal) | Harder to scale, pipeline bubbles |
| **Complexity** | Easy (TF handles it) | Complex (manual splitting) |
| **Communication** | Gradient sync after each step | Activation passing between GPUs |
| **Example** | ResNet-50 on 8 GPUs | GPT-3 (175B parameters) |
| **TF Support** | Excellent (built-in strategies) | Limited (manual or use Mesh TF) |
| **Most common** | ✅ Yes, default choice | For very large models only |

> **Pro Tip**: Start with data parallelism. Only use model parallelism when your model literally doesn't fit in a single GPU's memory.

---

## 10.3 tf.distribute.Strategy Overview

### What It Is
`tf.distribute.Strategy` is TensorFlow's API for distributing training across multiple GPUs, machines, or TPUs. It's a **clean abstraction** — you write your training code once, and the strategy handles all the complexity of distribution.

### Why It Matters
Without this API, you'd have to manually:
- Copy model weights to each GPU
- Split data across GPUs
- Collect and average gradients
- Synchronize weight updates

That's hundreds of lines of error-prone code. `tf.distribute.Strategy` does it in **2-3 lines**.

### Strategy Family

```
tf.distribute.Strategy (Base Class)
├── MirroredStrategy          → Single machine, multi-GPU (synchronous)
├── MultiWorkerMirroredStrategy → Multi-machine, multi-GPU (synchronous)
├── TPUStrategy               → Google TPU training
├── ParameterServerStrategy   → Multi-machine with param servers (async)
├── CentralStorageStrategy    → Single machine, CPU stores variables
├── OneDeviceStrategy         → Single device (for testing)
└── experimental.*            → Experimental strategies
```

### Strategy Comparison

| Strategy | Machines | GPUs | Sync Type | Best For |
|----------|----------|------|-----------|----------|
| `MirroredStrategy` | 1 | 2-8 | Synchronous | Most common, single workstation |
| `MultiWorkerMirroredStrategy` | 2+ | Any | Synchronous | Large-scale cluster training |
| `TPUStrategy` | 1+ | TPU cores | Synchronous | Google Cloud TPU training |
| `ParameterServerStrategy` | 2+ | Any | Asynchronous | Very large clusters, fault tolerance |
| `CentralStorageStrategy` | 1 | 2+ | Synchronous | When GPU memory is limited |

### Basic Usage Pattern

```python
import tensorflow as tf

# Step 1: Create a strategy
strategy = tf.distribute.MirroredStrategy()

# Step 2: Open strategy scope
with strategy.scope():
    # Step 3: Create model inside scope
    model = tf.keras.Sequential([
        tf.keras.layers.Dense(128, activation='relu'),
        tf.keras.layers.Dense(10, activation='softmax')
    ])
    
    # Step 4: Compile inside scope
    model.compile(
        optimizer='adam',
        loss='sparse_categorical_crossentropy',
        metrics=['accuracy']
    )

# Step 5: Train normally — strategy handles distribution
model.fit(train_dataset, epochs=10, validation_data=val_dataset)
```

> **Key Insight**: The only change to your normal training code is wrapping model creation in `strategy.scope()`. Everything else (model.fit, evaluation, prediction) works exactly the same.

---

## 10.4 MirroredStrategy (Single Machine, Multi-GPU)

### What It Is
`MirroredStrategy` is the most commonly used distribution strategy. It creates a **mirror (copy) of the model on each GPU** in a single machine. Each GPU processes different data, computes gradients, and then all GPUs **synchronize** (AllReduce) to keep weights identical.

### Why It Matters
- Most accessible — works with any multi-GPU workstation
- Near-linear scaling (2 GPUs ≈ 1.8x faster, 4 GPUs ≈ 3.5x faster)
- Zero code change to model architecture
- Best starting point for distributed training

### How It Works (Step by Step)

```
Step 1: Model Replication
┌─────────────────────────────┐
│       Original Model        │
│  (Layers, Weights, Optimizer)│
└──────────┬──────────────────┘
           │ Copy to each GPU
     ┌─────┼─────┐
     ▼     ▼     ▼
  ┌─────┐┌─────┐┌─────┐
  │GPU 0││GPU 1││GPU 2│
  │Copy ││Copy ││Copy │
  └─────┘└─────┘└─────┘

Step 2: Data Distribution (each step)
  Global Batch (e.g., 256 samples)
  ┌─────┬─────┬─────┐
  │ 86  │ 85  │ 85  │  (split among 3 GPUs)
  └──┬──┘└──┬──┘└──┬──┘
     ▼      ▼      ▼
  GPU 0   GPU 1   GPU 2

Step 3: Forward + Backward Pass (parallel)
  GPU 0: loss₀, grads₀
  GPU 1: loss₁, grads₁
  GPU 2: loss₂, grads₂

Step 4: AllReduce (synchronize gradients)
  avg_grad = (grads₀ + grads₁ + grads₂) / 3
  → Each GPU gets identical avg_grad

Step 5: Weight Update (identical on all GPUs)
  weights = weights - lr * avg_grad
  → All GPUs have identical weights ✓
```

### AllReduce Algorithms

```
RING ALLREDUCE (Default in TF):
Each GPU sends a chunk to its neighbor in a ring.
After N-1 steps, all GPUs have the sum.

GPU 0 ──→ GPU 1 ──→ GPU 2 ──→ GPU 3
  ▲                                │
  └────────────────────────────────┘

NCCL (NVIDIA Collective Communications Library):
Optimized GPU-to-GPU communication.
Uses NVLink/PCIe for fast data transfer.
Default backend for NVIDIA GPUs in TensorFlow.
```

### Complete Code Example

```python
import tensorflow as tf
import numpy as np

# ─── Check available GPUs ───
print("Available GPUs:", tf.config.list_physical_devices('GPU'))
print("Num GPUs:", len(tf.config.list_physical_devices('GPU')))

# ─── Create MirroredStrategy ───
# By default, uses ALL available GPUs
strategy = tf.distribute.MirroredStrategy()

# Or specify specific GPUs:
# strategy = tf.distribute.MirroredStrategy(devices=["/gpu:0", "/gpu:1"])

# Or specify cross-device communication:
# strategy = tf.distribute.MirroredStrategy(
#     cross_device_ops=tf.distribute.HierarchicalCopyAllReduce()
# )

print(f"Number of replicas: {strategy.num_replicas_in_sync}")

# ─── Prepare Data ───
# IMPORTANT: Global batch size = per_replica_batch_size × num_replicas
BATCH_SIZE_PER_REPLICA = 64
GLOBAL_BATCH_SIZE = BATCH_SIZE_PER_REPLICA * strategy.num_replicas_in_sync

# Load dataset
(x_train, y_train), (x_test, y_test) = tf.keras.datasets.mnist.load_data()
x_train = x_train.astype('float32') / 255.0  # Normalize to [0, 1]
x_test = x_test.astype('float32') / 255.0

# Create tf.data.Dataset with GLOBAL batch size
train_dataset = tf.data.Dataset.from_tensor_slices((x_train, y_train))
train_dataset = train_dataset.shuffle(10000).batch(GLOBAL_BATCH_SIZE).prefetch(tf.data.AUTOTUNE)

test_dataset = tf.data.Dataset.from_tensor_slices((x_test, y_test))
test_dataset = test_dataset.batch(GLOBAL_BATCH_SIZE).prefetch(tf.data.AUTOTUNE)

# ─── Build Model Inside Strategy Scope ───
with strategy.scope():
    model = tf.keras.Sequential([
        tf.keras.layers.Flatten(input_shape=(28, 28)),        # 784 inputs
        tf.keras.layers.Dense(512, activation='relu'),         # Hidden layer
        tf.keras.layers.Dropout(0.2),                          # Regularization
        tf.keras.layers.Dense(256, activation='relu'),         # Hidden layer
        tf.keras.layers.Dropout(0.2),                          # Regularization
        tf.keras.layers.Dense(10, activation='softmax')        # 10 classes
    ])
    
    model.compile(
        optimizer=tf.keras.optimizers.Adam(learning_rate=0.001),
        loss='sparse_categorical_crossentropy',
        metrics=['accuracy']
    )

model.summary()

# ─── Train ───
history = model.fit(
    train_dataset,
    epochs=10,
    validation_data=test_dataset,
    callbacks=[
        tf.keras.callbacks.EarlyStopping(patience=3, restore_best_weights=True),
        tf.keras.callbacks.ReduceLROnPlateau(factor=0.5, patience=2)
    ]
)

# ─── Evaluate ───
test_loss, test_acc = model.evaluate(test_dataset)
print(f"Test Accuracy: {test_acc:.4f}")
```

### Cross-Device Communication Options

```python
# Option 1: NCCL (Default for NVIDIA GPUs — fastest)
strategy = tf.distribute.MirroredStrategy(
    cross_device_ops=tf.distribute.NcclAllReduce()
)

# Option 2: Hierarchical Copy (good for multi-node within a machine)
strategy = tf.distribute.MirroredStrategy(
    cross_device_ops=tf.distribute.HierarchicalCopyAllReduce()
)

# Option 3: Reduction to one device (simple but slower)
strategy = tf.distribute.MirroredStrategy(
    cross_device_ops=tf.distribute.ReductionToOneDevice()
)
```

### Batch Size Considerations

```
The CRITICAL concept: Global Batch Size vs Per-Replica Batch Size

If you have 4 GPUs and want each GPU to process 64 samples:
  Global Batch Size = 64 × 4 = 256

If you set batch_size=256 in your dataset:
  TF automatically splits: 256 / 4 = 64 per GPU ✓

WRONG approach:
  Setting batch_size=64 with 4 GPUs → each GPU gets 16 samples
  → Effectively training with batch_size=16 per GPU (too small!)
```

### Learning Rate Scaling Rule

When you increase the global batch size, you should generally **scale the learning rate** proportionally:

$$lr_{scaled} = lr_{base} \times \frac{\text{global\_batch\_size}}{\text{base\_batch\_size}}$$

```python
# Linear scaling rule
BASE_LR = 0.001
BASE_BATCH_SIZE = 64
GLOBAL_BATCH_SIZE = 64 * strategy.num_replicas_in_sync

scaled_lr = BASE_LR * (GLOBAL_BATCH_SIZE / BASE_BATCH_SIZE)

# With warmup (recommended for large batch sizes)
initial_lr = scaled_lr / 10  # Start with 1/10th of target LR
warmup_steps = 1000

lr_schedule = tf.keras.optimizers.schedules.PolynomialDecay(
    initial_learning_rate=scaled_lr,
    decay_steps=total_steps,
    end_learning_rate=scaled_lr * 0.01
)

# Warmup wrapper
class WarmupSchedule(tf.keras.optimizers.schedules.LearningRateSchedule):
    def __init__(self, target_lr, warmup_steps, decay_schedule):
        super().__init__()
        self.target_lr = target_lr
        self.warmup_steps = warmup_steps
        self.decay_schedule = decay_schedule
    
    def __call__(self, step):
        # Linear warmup
        warmup_lr = self.target_lr * (step / self.warmup_steps)
        # After warmup, use decay schedule
        return tf.cond(
            step < self.warmup_steps,
            lambda: warmup_lr,
            lambda: self.decay_schedule(step - self.warmup_steps)
        )

with strategy.scope():
    optimizer = tf.keras.optimizers.Adam(
        learning_rate=WarmupSchedule(scaled_lr, warmup_steps, lr_schedule)
    )
```

> **Pro Tip**: The linear scaling rule works well up to a point. For very large batch sizes (>8K), consider using LARS or LAMB optimizers, which handle large-batch training more robustly.

---

## 10.5 MultiWorkerMirroredStrategy (Multi-Machine)

### What It Is
Extension of MirroredStrategy to **multiple machines** (workers). Each worker can have one or more GPUs. All workers synchronize gradients after every step — same principle as MirroredStrategy but across a network.

### Why It Matters
- When you need more GPUs than a single machine can hold (typically 4-8)
- Production training at scale (e.g., training on 32+ GPUs across 4 machines)
- Used by most AI companies for large-scale training

### How It Works

```
Machine 1 (Worker 0)           Machine 2 (Worker 1)
┌──────────────────┐           ┌──────────────────┐
│ GPU0  GPU1  GPU2 │           │ GPU0  GPU1  GPU2 │
│  ▼     ▼     ▼   │           │  ▼     ▼     ▼   │
│ ┌──┐ ┌──┐ ┌──┐  │           │ ┌──┐ ┌──┐ ┌──┐  │
│ │M │ │M │ │M │  │           │ │M │ │M │ │M │  │
│ └──┘ └──┘ └──┘  │           │ └──┘ └──┘ └──┘  │
│    Local Reduce   │           │    Local Reduce   │
│        ↕          │◄─Network─►│        ↕          │
│   Local Grads     │  AllReduce│   Local Grads     │
└──────────────────┘           └──────────────────┘
         ↕                              ↕
     Global AllReduce (across machines)
         ↕                              ↕
  Synchronized Weights on ALL GPUs across ALL machines
```

### Configuration via TF_CONFIG

```python
import json
import os

# TF_CONFIG must be set as an environment variable on EACH worker
# Worker 0 config:
tf_config_worker0 = {
    "cluster": {
        "worker": [
            "machine1.example.com:12345",   # Worker 0
            "machine2.example.com:12345",   # Worker 1
            "machine3.example.com:12345"    # Worker 2
        ]
    },
    "task": {
        "type": "worker",
        "index": 0          # THIS worker's index
    }
}

os.environ['TF_CONFIG'] = json.dumps(tf_config_worker0)
```

### Complete Multi-Worker Training Script

```python
"""
multi_worker_train.py
Run on each machine with appropriate TF_CONFIG set.
"""
import tensorflow as tf
import json
import os

# ─── Strategy Setup ───
communication_options = tf.distribute.experimental.CommunicationOptions(
    implementation=tf.distribute.experimental.CommunicationImplementation.NCCL
)

strategy = tf.distribute.MultiWorkerMirroredStrategy(
    communication_options=communication_options
)

print(f"Number of replicas: {strategy.num_replicas_in_sync}")

# ─── Global Batch Size ───
PER_REPLICA_BATCH = 64
GLOBAL_BATCH = PER_REPLICA_BATCH * strategy.num_replicas_in_sync

# ─── Data Loading ───
# IMPORTANT: Each worker must load the SAME dataset
# The strategy handles sharding automatically
def make_dataset():
    (x_train, y_train), (x_test, y_test) = tf.keras.datasets.mnist.load_data()
    x_train = x_train.astype('float32') / 255.0
    x_test = x_test.astype('float32') / 255.0
    
    train_ds = tf.data.Dataset.from_tensor_slices((x_train, y_train))
    train_ds = train_ds.shuffle(60000).batch(GLOBAL_BATCH)
    
    # AUTO sharding: each worker gets different portion of data
    options = tf.data.Options()
    options.experimental_distribute.auto_shard_policy = (
        tf.data.experimental.AutoShardPolicy.DATA
    )
    train_ds = train_ds.with_options(options)
    
    test_ds = tf.data.Dataset.from_tensor_slices((x_test, y_test))
    test_ds = test_ds.batch(GLOBAL_BATCH).with_options(options)
    
    return train_ds, test_ds

train_dataset, test_dataset = make_dataset()

# ─── Model (inside strategy scope) ───
with strategy.scope():
    model = tf.keras.Sequential([
        tf.keras.layers.Flatten(input_shape=(28, 28)),
        tf.keras.layers.Dense(256, activation='relu'),
        tf.keras.layers.Dense(10, activation='softmax')
    ])
    
    model.compile(
        optimizer='adam',
        loss='sparse_categorical_crossentropy',
        metrics=['accuracy']
    )

# ─── Callbacks ───
# Use BackupAndRestore for fault tolerance
callbacks = [
    tf.keras.callbacks.BackupAndRestore(
        backup_dir='/tmp/training_backup'
    ),
]

# Only save model from chief worker (worker 0) to avoid conflicts
task_type = strategy.cluster_resolver.task_type if hasattr(strategy, 'cluster_resolver') else 'worker'
task_id = strategy.cluster_resolver.task_id if hasattr(strategy, 'cluster_resolver') else 0

if task_id == 0:
    callbacks.append(
        tf.keras.callbacks.ModelCheckpoint(
            filepath='saved_model/best_model.keras',
            save_best_only=True
        )
    )

# ─── Train ───
model.fit(
    train_dataset,
    epochs=10,
    validation_data=test_dataset,
    callbacks=callbacks
)
```

### Data Sharding Policies

```python
# AUTO (default): TF decides best sharding
options.experimental_distribute.auto_shard_policy = (
    tf.data.experimental.AutoShardPolicy.AUTO
)

# DATA: Shard by data — each worker gets different data portions
# Use when: reading from same source (e.g., in-memory data)
options.experimental_distribute.auto_shard_policy = (
    tf.data.experimental.AutoShardPolicy.DATA
)

# FILE: Shard by file — each worker reads different files
# Use when: data is in multiple TFRecord files
options.experimental_distribute.auto_shard_policy = (
    tf.data.experimental.AutoShardPolicy.FILE
)

# OFF: No sharding — each worker gets all data
# Use when: you handle sharding yourself
options.experimental_distribute.auto_shard_policy = (
    tf.data.experimental.AutoShardPolicy.OFF
)
```

### Fault Tolerance

```python
# BackupAndRestore callback saves training state periodically
# If a worker crashes and restarts, training resumes from last backup

callbacks = [
    tf.keras.callbacks.BackupAndRestore(
        backup_dir='/shared/nfs/backup',  # Must be accessible by all workers
        save_freq='epoch',                # Save every epoch
        delete_checkpoint=True            # Clean up after training
    )
]
```

> **Important**: In multi-worker training, **all workers must execute the same code**. The only difference is the `TF_CONFIG` environment variable which identifies each worker's role and index.

---

## 10.6 TPUStrategy (Training on TPUs)

### What It Is
TPUs (Tensor Processing Units) are Google's custom-built AI accelerators. `TPUStrategy` lets you train models on TPUs, which are available through Google Cloud and Google Colab.

### Why It Matters
- TPUs can be **2-5x faster** than GPUs for many workloads
- Free TPU access in Google Colab (TPU v2 with 8 cores)
- Designed specifically for matrix operations in neural networks
- Used internally by Google for training their largest models

### TPU Architecture

```
TPU v2/v3 Pod Slice:
┌──────────────────────────────────┐
│           TPU Device             │
│  ┌──────┐ ┌──────┐ ┌──────┐     │
│  │Core 0│ │Core 1│ │Core 2│ ... │  ← 8 cores per TPU
│  │128x  │ │128x  │ │128x  │     │
│  │128   │ │128   │ │128   │     │  ← Each core has 128×128
│  │MXU   │ │MXU   │ │MXU   │     │    Matrix Multiply Unit
│  └──────┘ └──────┘ └──────┘     │
│         High Bandwidth Memory    │
│         (HBM): 16-32 GB         │
└──────────────────────────────────┘

MXU = Matrix Multiply Unit
- Performs 128×128 matrix multiply per cycle
- Optimized for bfloat16 (brain floating point)
- Much faster than GPU for matmuls
```

### Google Colab TPU Example

```python
import tensorflow as tf

# ─── Connect to TPU ───
try:
    # For Google Colab
    tpu = tf.distribute.cluster_resolver.TPUClusterResolver()
    print(f"Running on TPU: {tpu.master()}")
except ValueError:
    tpu = None
    print("No TPU found, falling back to GPU/CPU")

if tpu:
    tf.config.experimental_connect_to_cluster(tpu)
    tf.tpu.experimental.initialize_tpu_system(tpu)
    strategy = tf.distribute.TPUStrategy(tpu)
else:
    strategy = tf.distribute.MirroredStrategy()

print(f"Number of replicas: {strategy.num_replicas_in_sync}")

# ─── Data Pipeline (TPU-optimized) ───
# TPU REQUIREMENTS:
# 1. Batch size must be divisible by number of replicas (8 for TPU v2)
# 2. Use tf.data for data loading (no NumPy feeding)
# 3. Fixed-shape tensors only (no ragged tensors on TPU)

BATCH_SIZE = 128 * strategy.num_replicas_in_sync  # 128 × 8 = 1024

def get_dataset():
    (x_train, y_train), (x_test, y_test) = tf.keras.datasets.cifar10.load_data()
    
    x_train = tf.cast(x_train, tf.float32) / 255.0
    x_test = tf.cast(x_test, tf.float32) / 255.0
    y_train = tf.cast(y_train, tf.int32)
    y_test = tf.cast(y_test, tf.int32)
    
    train_ds = tf.data.Dataset.from_tensor_slices((x_train, y_train))
    train_ds = (train_ds
                .shuffle(50000)
                .batch(BATCH_SIZE, drop_remainder=True)  # MUST drop remainder for TPU
                .prefetch(tf.data.AUTOTUNE))
    
    test_ds = tf.data.Dataset.from_tensor_slices((x_test, y_test))
    test_ds = (test_ds
               .batch(BATCH_SIZE, drop_remainder=True)
               .prefetch(tf.data.AUTOTUNE))
    
    return train_ds, test_ds

train_ds, test_ds = get_dataset()

# ─── Model ───
with strategy.scope():
    model = tf.keras.Sequential([
        tf.keras.layers.Conv2D(32, 3, activation='relu', input_shape=(32, 32, 3)),
        tf.keras.layers.MaxPooling2D(),
        tf.keras.layers.Conv2D(64, 3, activation='relu'),
        tf.keras.layers.MaxPooling2D(),
        tf.keras.layers.Conv2D(128, 3, activation='relu'),
        tf.keras.layers.GlobalAveragePooling2D(),
        tf.keras.layers.Dense(128, activation='relu'),
        tf.keras.layers.Dense(10, activation='softmax')
    ])
    
    model.compile(
        optimizer='adam',
        loss='sparse_categorical_crossentropy',
        metrics=['accuracy']
    )

# ─── Train ───
history = model.fit(
    train_ds,
    epochs=20,
    validation_data=test_ds
)
```

### TPU vs GPU Comparison

| Feature | GPU (NVIDIA A100) | TPU v3 | TPU v4 |
|---------|-------------------|--------|--------|
| **Peak TFLOPS (fp16)** | 312 | 420 | 275 (bf16) |
| **Memory** | 40/80 GB HBM2e | 32 GB HBM | 32 GB HBM |
| **Best for** | General ML, GANs | Large batch training | Large models |
| **Programming** | CUDA, cuDNN | XLA only | XLA only |
| **Cost (Cloud/hr)** | ~$3-4 | ~$8 (full pod slice) | ~$12 |
| **Availability** | Everywhere | Google Cloud only | Google Cloud only |

> **Key TPU Constraint**: TPU requires **static shapes** and `drop_remainder=True` in batching. Dynamic shapes cause recompilation and slow training dramatically.

---

## 10.7 ParameterServerStrategy

### What It Is
Unlike MirroredStrategy (where each worker has full model copy), ParameterServerStrategy uses **dedicated parameter server** machines that store variables, while **worker** machines do the computation. Workers pull weights, compute gradients, and push updates to parameter servers.

### Why It Matters
- Supports **asynchronous training** — workers don't wait for each other
- Better **fault tolerance** — if a worker dies, others continue
- Scales to **very large clusters** (100+ workers)
- Preferred for production systems at companies like Google

### Architecture

```
PARAMETER SERVER STRATEGY:

Parameter Servers (store model weights):
┌──────────┐  ┌──────────┐
│  PS 0    │  │  PS 1    │
│ Weights  │  │ Weights  │
│ Part A   │  │ Part B   │
└─────┬────┘  └────┬─────┘
      │            │
      │   Network  │
      │            │
┌─────┴────┐  ┌────┴─────┐  ┌──────────┐
│ Worker 0 │  │ Worker 1 │  │ Worker 2 │
│ Compute  │  │ Compute  │  │ Compute  │
│ Gradients│  │ Gradients│  │ Gradients│
└──────────┘  └──────────┘  └──────────┘

Flow:
1. Worker pulls latest weights from PS
2. Worker computes forward/backward pass
3. Worker pushes gradients to PS
4. PS updates weights
5. Repeat (asynchronously — workers don't wait)
```

### Code Example

```python
import tensorflow as tf
import json
import os

# ─── TF_CONFIG for Parameter Server Training ───
# Each process needs its own TF_CONFIG

# For a parameter server:
tf_config_ps = {
    "cluster": {
        "worker": ["worker0:2222", "worker1:2222"],
        "ps": ["ps0:2222"],
        "chief": ["chief0:2222"]
    },
    "task": {"type": "ps", "index": 0}
}

# For a worker:
tf_config_worker = {
    "cluster": {
        "worker": ["worker0:2222", "worker1:2222"],
        "ps": ["ps0:2222"],
        "chief": ["chief0:2222"]
    },
    "task": {"type": "worker", "index": 0}
}

os.environ['TF_CONFIG'] = json.dumps(tf_config_worker)

# ─── Setup ───
cluster_resolver = tf.distribute.cluster_resolver.TFConfigClusterResolver()
strategy = tf.distribute.ParameterServerStrategy(cluster_resolver)

# ─── Coordinator (runs on chief) ───
# The coordinator distributes work and manages training

coordinator = tf.distribute.coordinator.ClusterCoordinator(strategy)

# ─── Dataset ───
def dataset_fn(input_context):
    """Each worker calls this to get its shard of data."""
    global_batch_size = 256
    batch_size = input_context.get_per_replica_batch_size(global_batch_size)
    
    (x_train, y_train), _ = tf.keras.datasets.mnist.load_data()
    x_train = x_train.astype('float32') / 255.0
    
    dataset = tf.data.Dataset.from_tensor_slices((x_train, y_train))
    dataset = dataset.shuffle(60000)
    dataset = dataset.shard(
        num_shards=input_context.num_input_pipelines,
        index=input_context.input_pipeline_id
    )
    dataset = dataset.batch(batch_size)
    dataset = dataset.prefetch(tf.data.AUTOTUNE)
    return dataset

distributed_dataset = coordinator.create_per_worker_dataset(dataset_fn)

# ─── Model ───
with strategy.scope():
    model = tf.keras.Sequential([
        tf.keras.layers.Flatten(input_shape=(28, 28)),
        tf.keras.layers.Dense(256, activation='relu'),
        tf.keras.layers.Dense(10, activation='softmax')
    ])
    
    optimizer = tf.keras.optimizers.Adam()
    loss_fn = tf.keras.losses.SparseCategoricalCrossentropy()
    accuracy = tf.keras.metrics.SparseCategoricalAccuracy()

# ─── Custom Training Step ───
@tf.function
def train_step(iterator):
    def step_fn(inputs):
        x, y = inputs
        with tf.GradientTape() as tape:
            predictions = model(x, training=True)
            loss = loss_fn(y, predictions)
        gradients = tape.gradient(loss, model.trainable_variables)
        optimizer.apply_gradients(zip(gradients, model.trainable_variables))
        accuracy.update_state(y, predictions)
        return loss
    
    per_replica_losses = strategy.run(step_fn, args=(next(iterator),))
    return strategy.reduce(
        tf.distribute.ReduceOp.MEAN, per_replica_losses, axis=None
    )

# ─── Training Loop ───
for epoch in range(10):
    accuracy.reset_states()
    distributed_iter = iter(distributed_dataset)
    
    for step in range(steps_per_epoch):
        coordinator.schedule(train_step, args=(distributed_iter,))
    
    coordinator.join()  # Wait for all scheduled steps to complete
    print(f"Epoch {epoch+1}, Accuracy: {accuracy.result():.4f}")
```

> **Pro Tip**: ParameterServerStrategy is overkill for most users. Use MirroredStrategy for single-machine training and MultiWorkerMirroredStrategy for 2-8 machines. Only reach for ParameterServerStrategy when you need asynchronous training or have 10+ workers.

---

## 10.8 Mixed Precision Training

### What It Is
Mixed precision training uses **two number formats** simultaneously:
- **float16** (half precision) for most computations → **faster math, less memory**
- **float32** (full precision) for critical operations → **numerical stability**

Think of it like using rough estimates for most calculations but switching to a calculator for the final answer.

### Why It Matters
- **2-3x faster training** on modern GPUs (NVIDIA Volta, Turing, Ampere)
- **~50% less GPU memory** → can train with larger batch sizes or bigger models
- **Near-identical accuracy** to full float32 training
- **Free performance** — minimal code changes required

### How Float16 Works

```
float32: 1 sign + 8 exponent + 23 mantissa = 32 bits
  Range: ±3.4 × 10³⁸, Precision: ~7 decimal digits
  Memory per value: 4 bytes

float16: 1 sign + 5 exponent + 10 mantissa = 16 bits
  Range: ±65,504, Precision: ~3 decimal digits
  Memory per value: 2 bytes

bfloat16: 1 sign + 8 exponent + 7 mantissa = 16 bits
  Range: ±3.4 × 10³⁸, Precision: ~2 decimal digits
  Memory per value: 2 bytes (TPU-optimized)

Why mixed?
┌─────────────────────────────────────────┐
│  Forward pass: float16 (fast)           │
│  Loss computation: float32 (accurate)   │
│  Gradient computation: float16 (fast)   │
│  Weight update: float32 (accurate)      │
│  Stored weights: float32 (master copy)  │
└─────────────────────────────────────────┘
```

### The Loss Scaling Problem

```
Problem: float16 has limited range → small gradients become ZERO

Gradient values:                    float16 representable range:
   ┌──────────────────┐            ┌──────────────────┐
   │ 0.00001          │            │ 0.00006 (min)    │
   │ 0.00003 ← ZERO  │──becomes──►│                  │
   │ 0.0002           │ in fp16    │ 0.0002  (OK)     │
   │ 0.15             │            │ 0.15    (OK)     │
   └──────────────────┘            └──────────────────┘

Solution: LOSS SCALING
  1. Multiply loss by a large factor (e.g., 1024)
  2. All gradients are scaled up → no underflow
  3. After gradient computation, divide by the same factor
  4. Update weights with correctly-scaled gradients

Dynamic Loss Scaling:
  - Start with large scale (e.g., 2¹⁵)
  - If gradients overflow (Inf/NaN), reduce scale by 2
  - If no overflow for N steps, increase scale by 2
  - TF handles this automatically!
```

### Simple Mixed Precision (Recommended)

```python
import tensorflow as tf

# ─── Enable Mixed Precision Globally ───
# This single line enables mixed precision for ALL layers
tf.keras.mixed_precision.set_global_policy('mixed_float16')

# Check current policy
print(tf.keras.mixed_precision.global_policy())  # <Policy "mixed_float16">

# ─── Build Model (layers automatically use float16 for compute) ───
model = tf.keras.Sequential([
    tf.keras.layers.Flatten(input_shape=(28, 28)),
    tf.keras.layers.Dense(512, activation='relu'),      # Computes in fp16
    tf.keras.layers.Dense(256, activation='relu'),      # Computes in fp16
    tf.keras.layers.Dense(10, dtype='float32')          # Output layer: MUST be fp32
    # For softmax/sigmoid output: always use float32 to avoid numerical issues
])

# ─── Compile with Loss Scaling (automatic) ───
optimizer = tf.keras.optimizers.Adam(learning_rate=0.001)
# Loss scaling is handled automatically by Keras

model.compile(
    optimizer=optimizer,
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

# ─── Train Normally ───
(x_train, y_train), (x_test, y_test) = tf.keras.datasets.mnist.load_data()
x_train = x_train.astype('float32') / 255.0
x_test = x_test.astype('float32') / 255.0

model.fit(x_train, y_train, epochs=5, batch_size=128, validation_split=0.1)
```

### Mixed Precision + Distributed Training

```python
import tensorflow as tf

# ─── Enable Mixed Precision ───
tf.keras.mixed_precision.set_global_policy('mixed_float16')

# ─── Setup Distribution ───
strategy = tf.distribute.MirroredStrategy()
print(f"Replicas: {strategy.num_replicas_in_sync}")

BATCH_SIZE = 256 * strategy.num_replicas_in_sync

# ─── Data ───
(x_train, y_train), (x_test, y_test) = tf.keras.datasets.cifar10.load_data()
x_train = x_train.astype('float32') / 255.0
x_test = x_test.astype('float32') / 255.0

train_ds = (tf.data.Dataset.from_tensor_slices((x_train, y_train))
            .shuffle(50000)
            .batch(BATCH_SIZE)
            .prefetch(tf.data.AUTOTUNE))

test_ds = (tf.data.Dataset.from_tensor_slices((x_test, y_test))
           .batch(BATCH_SIZE)
           .prefetch(tf.data.AUTOTUNE))

# ─── Model ───
with strategy.scope():
    base_model = tf.keras.applications.ResNet50(
        weights=None,
        input_shape=(32, 32, 3),
        classes=10
    )
    
    # Ensure output is float32 for mixed precision
    outputs = tf.keras.layers.Activation('linear', dtype='float32')(base_model.output)
    model = tf.keras.Model(inputs=base_model.input, outputs=outputs)
    
    model.compile(
        optimizer=tf.keras.optimizers.Adam(0.001),
        loss='sparse_categorical_crossentropy',
        metrics=['accuracy']
    )

model.fit(train_ds, epochs=20, validation_data=test_ds)
```

### Custom Training Loop with Mixed Precision

```python
import tensorflow as tf

tf.keras.mixed_precision.set_global_policy('mixed_float16')

# Build model
model = tf.keras.Sequential([
    tf.keras.layers.Dense(512, activation='relu', input_shape=(784,)),
    tf.keras.layers.Dense(10)   # No activation — raw logits
])

optimizer = tf.keras.optimizers.Adam(0.001)
loss_fn = tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True)

@tf.function
def train_step(x, y):
    with tf.GradientTape() as tape:
        predictions = model(x, training=True)       # fp16 computation
        # Cast predictions to fp32 for loss computation
        predictions = tf.cast(predictions, tf.float32)
        loss = loss_fn(y, predictions)
        # Note: In Keras, loss scaling is automatic
        # For manual control:
        # scaled_loss = optimizer.get_scaled_loss(loss)
    
    gradients = tape.gradient(loss, model.trainable_variables)
    # Unscale gradients (if using manual scaling):
    # gradients = optimizer.get_unscaled_gradients(gradients)
    optimizer.apply_gradients(zip(gradients, model.trainable_variables))
    return loss

# Training loop
for epoch in range(5):
    for x_batch, y_batch in train_dataset:
        loss = train_step(x_batch, y_batch)
    print(f"Epoch {epoch+1}, Loss: {loss:.4f}")
```

### When to Use Mixed Precision

| GPU Architecture | Mixed Precision Support | Speedup |
|-----------------|------------------------|---------|
| NVIDIA Volta (V100) | ✅ Tensor Cores (fp16) | 2-3x |
| NVIDIA Turing (T4, RTX 2080) | ✅ Tensor Cores (fp16) | 2-3x |
| NVIDIA Ampere (A100, RTX 3090) | ✅ Tensor Cores (fp16/bf16/tf32) | 2-3x |
| NVIDIA Hopper (H100) | ✅ Tensor Cores (fp8/fp16/bf16) | 3-4x |
| Google TPU | ✅ bfloat16 native | Already fast |
| Older NVIDIA (Pascal, Maxwell) | ❌ No benefit | 1x (slower due to conversion) |

> **Warning**: Don't use mixed precision on GPUs without Tensor Cores (pre-Volta). The fp16↔fp32 conversion overhead will actually make training **slower**.

---

## 10.9 Performance Optimization for Distributed Training

### Data Pipeline Optimization

```python
import tensorflow as tf

# ─── Optimized Data Pipeline for Distributed Training ───

# 1. Interleave: Read from multiple files in parallel
files = tf.data.Dataset.list_files("data/train-*.tfrecord")
dataset = files.interleave(
    tf.data.TFRecordDataset,
    cycle_length=tf.data.AUTOTUNE,     # Number of files to read in parallel
    num_parallel_calls=tf.data.AUTOTUNE
)

# 2. Parse and preprocess in parallel
def parse_fn(serialized):
    features = tf.io.parse_single_example(serialized, feature_description)
    image = tf.io.decode_jpeg(features['image'], channels=3)
    image = tf.cast(image, tf.float32) / 255.0
    image = tf.image.resize(image, [224, 224])
    label = features['label']
    return image, label

dataset = dataset.map(parse_fn, num_parallel_calls=tf.data.AUTOTUNE)

# 3. Cache (if dataset fits in memory)
dataset = dataset.cache()

# 4. Shuffle
dataset = dataset.shuffle(buffer_size=10000)

# 5. Batch with proper global batch size
strategy = tf.distribute.MirroredStrategy()
GLOBAL_BATCH = 64 * strategy.num_replicas_in_sync
dataset = dataset.batch(GLOBAL_BATCH, drop_remainder=True)

# 6. Prefetch (overlap data loading with training)
dataset = dataset.prefetch(tf.data.AUTOTUNE)

# VISUALIZATION OF PIPELINE:
# ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
# │ Read │→│ Map  │→│Cache │→│Shuf. │→│Batch │→│Pref. │→ Training
# └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └──────┘
#    ↑         ↑                                    ↑
# Parallel  Parallel                             Overlap with
# file read  parsing                             GPU compute
```

### Communication Optimization

```python
# ─── Gradient Compression ───
# Reduces communication overhead by compressing gradients

# Use tf.distribute communication options
strategy = tf.distribute.MirroredStrategy(
    cross_device_ops=tf.distribute.NcclAllReduce(num_packs=1)
)

# ─── Gradient Accumulation ───
# When GPU memory is limited, accumulate gradients over multiple mini-batches

class GradientAccumulator:
    """Simulates larger batch by accumulating gradients."""
    
    def __init__(self, model, optimizer, loss_fn, accumulation_steps=4):
        self.model = model
        self.optimizer = optimizer
        self.loss_fn = loss_fn
        self.accumulation_steps = accumulation_steps
        self.gradient_accumulator = [
            tf.Variable(tf.zeros_like(v), trainable=False)
            for v in model.trainable_variables
        ]
    
    def reset_gradients(self):
        for acc in self.gradient_accumulator:
            acc.assign(tf.zeros_like(acc))
    
    @tf.function
    def train_step(self, x, y, step):
        with tf.GradientTape() as tape:
            predictions = self.model(x, training=True)
            loss = self.loss_fn(y, predictions)
            # Scale loss by accumulation steps
            scaled_loss = loss / self.accumulation_steps
        
        gradients = tape.gradient(scaled_loss, self.model.trainable_variables)
        
        # Accumulate
        for acc, grad in zip(self.gradient_accumulator, gradients):
            acc.assign_add(grad)
        
        # Apply when we've accumulated enough
        if (step + 1) % self.accumulation_steps == 0:
            self.optimizer.apply_gradients(
                zip(self.gradient_accumulator, self.model.trainable_variables)
            )
            self.reset_gradients()
        
        return loss
```

### Memory Optimization

```python
# ─── GPU Memory Growth ───
# Prevent TF from grabbing all GPU memory at once
gpus = tf.config.experimental.list_physical_devices('GPU')
for gpu in gpus:
    tf.config.experimental.set_memory_growth(gpu, True)

# Or set a memory limit
tf.config.set_logical_device_configuration(
    gpus[0],
    [tf.config.LogicalDeviceConfiguration(memory_limit=4096)]  # 4GB limit
)

# ─── Gradient Checkpointing (Trade Compute for Memory) ───
# Instead of storing all activations, recompute them during backward pass
# Saves memory at cost of ~20-30% slower training

class CheckpointedModel(tf.keras.Model):
    def __init__(self):
        super().__init__()
        self.dense1 = tf.keras.layers.Dense(1024, activation='relu')
        self.dense2 = tf.keras.layers.Dense(1024, activation='relu')
        self.dense3 = tf.keras.layers.Dense(10)
    
    @tf.function
    def call(self, x, training=False):
        # tf.recompute_grad wraps a function to recompute activations
        # during backward pass instead of storing them
        @tf.recompute_grad
        def heavy_computation(inputs):
            h = self.dense1(inputs)
            h = self.dense2(h)
            return h
        
        h = heavy_computation(x)
        return self.dense3(h)
```

### XLA Compilation

```python
# ─── XLA (Accelerated Linear Algebra) ───
# Compiles TF operations into optimized machine code
# Can give 10-30% speedup

# Method 1: Enable globally
tf.config.optimizer.set_jit(True)

# Method 2: Per-function with jit_compile
@tf.function(jit_compile=True)
def train_step(x, y):
    with tf.GradientTape() as tape:
        predictions = model(x, training=True)
        loss = loss_fn(y, predictions)
    gradients = tape.gradient(loss, model.trainable_variables)
    optimizer.apply_gradients(zip(gradients, model.trainable_variables))
    return loss

# Method 3: In model.compile
model.compile(
    optimizer='adam',
    loss='sparse_categorical_crossentropy',
    jit_compile=True   # Enable XLA for the whole model
)

# When XLA helps:
# ✅ Many small operations → fused into fewer large ones
# ✅ Lots of element-wise ops (activation functions, normalization)
# ✅ TPU training (XLA is required for TPUs)
#
# When XLA hurts:
# ❌ Dynamic shapes (causes recompilation)
# ❌ Python control flow in tf.function
# ❌ Very simple models (compilation overhead > speedup)
```

### Profiling Distributed Training

```python
import tensorflow as tf

# ─── TensorBoard Profiler ───
# Identify bottlenecks in your training pipeline

log_dir = "logs/profile"
tensorboard_callback = tf.keras.callbacks.TensorBoard(
    log_dir=log_dir,
    profile_batch='10,20'  # Profile batches 10 through 20
)

model.fit(
    train_dataset,
    epochs=5,
    callbacks=[tensorboard_callback]
)

# Then run: tensorboard --logdir=logs/profile
# Navigate to the "Profile" tab to see:
# - GPU utilization timeline
# - Data pipeline bottlenecks
# - Operation-level timing
# - Memory usage

# ─── Programmatic Profiling ───
tf.profiler.experimental.start(log_dir)

for step, (x, y) in enumerate(train_dataset):
    if step == 20:
        break
    train_step(x, y)

tf.profiler.experimental.stop()
```

---

## 10.10 Common Mistakes

### Mistake 1: Wrong Batch Size Scaling
```python
# ❌ WRONG: Using same batch size regardless of GPU count
model.fit(train_data.batch(64), ...)  # Each GPU gets 64/N samples

# ✅ CORRECT: Scale batch size with GPU count
GLOBAL_BATCH = 64 * strategy.num_replicas_in_sync
model.fit(train_data.batch(GLOBAL_BATCH), ...)  # Each GPU gets 64 samples
```

### Mistake 2: Creating Model Outside Strategy Scope
```python
# ❌ WRONG: Model created outside scope
model = tf.keras.Sequential([...])  # Created on single device
with strategy.scope():
    model.compile(...)  # Too late — model already on one GPU

# ✅ CORRECT: Model AND compile inside scope
with strategy.scope():
    model = tf.keras.Sequential([...])
    model.compile(...)
```

### Mistake 3: Not Using drop_remainder for TPUs
```python
# ❌ WRONG: Variable batch sizes break TPU
dataset = dataset.batch(BATCH_SIZE)  # Last batch might be smaller

# ✅ CORRECT: Always drop remainder for TPU
dataset = dataset.batch(BATCH_SIZE, drop_remainder=True)
```

### Mistake 4: Not Scaling Learning Rate
```python
# ❌ WRONG: Same LR with larger effective batch
with strategy.scope():
    model.compile(optimizer=tf.keras.optimizers.Adam(0.001), ...)

# ✅ CORRECT: Scale LR proportionally
scaled_lr = 0.001 * strategy.num_replicas_in_sync
with strategy.scope():
    model.compile(optimizer=tf.keras.optimizers.Adam(scaled_lr), ...)
```

### Mistake 5: Saving Model from All Workers
```python
# ❌ WRONG: All workers try to save (file corruption)
model.save('my_model.keras')

# ✅ CORRECT: Only chief (worker 0) saves
if strategy.cluster_resolver.task_id == 0:
    model.save('my_model.keras')
```

### Mistake 6: Mixed Precision Output Layer
```python
# ❌ WRONG: Softmax in float16 causes numerical issues
model.add(tf.keras.layers.Dense(10, activation='softmax'))

# ✅ CORRECT: Output layer in float32
model.add(tf.keras.layers.Dense(10, activation='softmax', dtype='float32'))
# OR
model.add(tf.keras.layers.Dense(10))
model.add(tf.keras.layers.Activation('softmax', dtype='float32'))
```

---

## 10.11 Interview Questions

### Q1: What is the difference between data parallelism and model parallelism?
**Answer**: Data parallelism replicates the entire model on each GPU and splits data across GPUs. Each GPU processes different data, computes gradients, then gradients are averaged (AllReduce). Model parallelism splits the model itself across GPUs — different layers live on different GPUs. Data parallelism is simpler, scales better, and is the default choice. Model parallelism is needed when a model doesn't fit in a single GPU's memory.

### Q2: What is AllReduce and how does it work?
**Answer**: AllReduce is a collective communication operation where each participant (GPU) contributes a value, a reduction (typically sum or average) is performed, and every participant receives the result. In distributed training, it's used to average gradients across GPUs. The Ring AllReduce algorithm is most common — each GPU sends/receives chunks in a ring pattern. After 2(N-1) steps (N = number of GPUs), all GPUs have the complete reduced result. NCCL is NVIDIA's optimized implementation.

### Q3: Why do we need loss scaling in mixed precision training?
**Answer**: Float16 has a limited representable range (minimum positive ≈ 6×10⁻⁵). Many gradient values during backpropagation are smaller than this and become zero (underflow), causing training to stall. Loss scaling multiplies the loss by a large factor before backpropagation, shifting all gradients into the representable range. After gradient computation, we divide by the same factor. Dynamic loss scaling automatically adjusts this factor during training.

### Q4: How does `tf.distribute.MirroredStrategy` handle batch normalization?
**Answer**: MirroredStrategy uses `tf.keras.layers.BatchNormalization` which by default computes mean and variance **per replica** (not across all GPUs). This means each GPU uses its own mini-batch statistics. For `SyncBatchNormalization`, statistics are computed across all replicas using AllReduce, which is more accurate but slower. Use `tf.keras.layers.experimental.SyncBatchNormalization` when global batch statistics matter (e.g., small per-GPU batch sizes).

### Q5: What is the linear scaling rule for learning rates?
**Answer**: When you multiply the batch size by $k$, you should also multiply the learning rate by $k$: $lr_{new} = lr_{base} \times k$. This maintains the same expected gradient magnitude. However, this rule has limits — for very large batch sizes, training can become unstable. The solution is to use a **warmup period** where the learning rate gradually increases from a small value to the target LR over the first few thousand steps.

### Q6: How do you handle data loading in multi-worker training?
**Answer**: Each worker must see different data to avoid duplicate computation. TensorFlow provides automatic data sharding via `tf.data.experimental.AutoShardPolicy`:
- `FILE`: Shard by input file (best for TFRecords)
- `DATA`: Shard by data elements (best for in-memory data)
- `AUTO`: TF chooses based on dataset structure
- `OFF`: No automatic sharding (manual handling)

### Q7: What happens if a worker fails during MultiWorkerMirroredStrategy training?
**Answer**: By default, training hangs because synchronous training requires all workers. To handle this: use `BackupAndRestore` callback to save checkpoints, so training can resume after the failed worker restarts. For truly fault-tolerant training, `ParameterServerStrategy` with asynchronous updates is better — remaining workers continue training even if one fails.

### Q8: When would you use mixed precision with bfloat16 vs float16?
**Answer**: float16 has more mantissa bits (10 vs 7) so better precision, but limited range requiring loss scaling. bfloat16 has the same range as float32 (8 exponent bits) so no loss scaling needed, but lower precision. Use bfloat16 on TPUs (native support) and NVIDIA Ampere+ GPUs. Use float16 on NVIDIA Volta/Turing GPUs. bfloat16 is simpler (no loss scaling) and increasingly preferred.

---

## 10.12 Quick Reference

### Strategy Selection Cheatsheet

```
How many machines?
├── 1 machine
│   ├── How many GPUs?
│   │   ├── 1 GPU → No strategy needed (or OneDeviceStrategy)
│   │   └── 2+ GPUs → MirroredStrategy ✅
│   └── TPU? → TPUStrategy
└── 2+ machines
    ├── Synchronous training? → MultiWorkerMirroredStrategy
    └── Asynchronous / large cluster? → ParameterServerStrategy
```

### Key APIs

| API | Purpose |
|-----|---------|
| `tf.distribute.MirroredStrategy()` | Multi-GPU on single machine |
| `tf.distribute.MultiWorkerMirroredStrategy()` | Multi-machine synchronous |
| `tf.distribute.TPUStrategy(resolver)` | TPU training |
| `tf.distribute.ParameterServerStrategy(resolver)` | Async multi-machine |
| `strategy.scope()` | Context for creating distributed model |
| `strategy.num_replicas_in_sync` | Number of GPUs/TPU cores |
| `tf.keras.mixed_precision.set_global_policy('mixed_float16')` | Enable mixed precision |
| `tf.keras.callbacks.BackupAndRestore()` | Fault-tolerant checkpointing |
| `tf.config.experimental.set_memory_growth()` | Prevent GPU memory hogging |
| `tf.function(jit_compile=True)` | XLA compilation |

### Batch Size Formula

$$\text{Global Batch Size} = \text{Per-Replica Batch} \times \text{num\_replicas\_in\_sync}$$

### Learning Rate Scaling

$$lr_{scaled} = lr_{base} \times \frac{\text{Global Batch Size}}{\text{Base Batch Size}}$$

### Performance Tips Summary

| Optimization | Speedup | Effort | When to Use |
|-------------|---------|--------|-------------|
| Mixed Precision | 2-3x | Low | Always (on supported GPUs) |
| Multi-GPU (Mirrored) | ~Nx | Low | Multiple GPUs available |
| XLA Compilation | 10-30% | Low | Static-shape models |
| tf.data Optimization | 20-50% | Medium | Data pipeline is bottleneck |
| Gradient Accumulation | N/A | Medium | GPU memory limited |
| Gradient Checkpointing | N/A (saves memory) | Medium | Very deep models |
| Multi-Worker | ~Nx | High | Need more than 8 GPUs |

### Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `TF_CONFIG` | Multi-worker cluster config | `{"cluster": {...}, "task": {...}}` |
| `CUDA_VISIBLE_DEVICES` | Limit visible GPUs | `"0,1"` |
| `TF_FORCE_GPU_ALLOW_GROWTH` | Enable memory growth | `"true"` |
| `TF_GPU_THREAD_MODE` | GPU thread mode | `"gpu_private"` |
| `NCCL_DEBUG` | NCCL debugging | `"INFO"` |
| `TF_XLA_FLAGS` | XLA options | `"--tf_xla_auto_jit=2"` |
