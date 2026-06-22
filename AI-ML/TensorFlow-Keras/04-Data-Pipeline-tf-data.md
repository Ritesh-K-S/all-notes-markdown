# Chapter 04: Data Pipeline with tf.data

## Table of Contents
- [What is tf.data?](#what-is-tfdata)
- [Why tf.data Matters](#why-tfdata-matters)
- [How tf.data Works — The Pipeline Model](#how-tfdata-works)
- [Creating Datasets](#creating-datasets)
- [Dataset Transformations](#dataset-transformations)
- [Batching Strategies](#batching-strategies)
- [Performance Optimization](#performance-optimization)
- [TFRecord Format](#tfrecord-format)
- [Data Augmentation in the Pipeline](#data-augmentation-in-the-pipeline)
- [Working with Large Datasets](#working-with-large-datasets)
- [Real-World Pipeline Examples](#real-world-pipeline-examples)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is tf.data?

### Simple Explanation

Imagine you're a chef cooking for 1,000 people. You wouldn't:
- Buy ALL ingredients at once (runs out of fridge space = RAM)
- Prepare one dish at a time while the oven sits empty (wasted time = idle GPU)

Instead, you'd:
- Have a helper fetch ingredients **while** you cook (prefetching)
- Prepare food in **batches** of 50 plates (batching)
- Pre-chop vegetables in bulk (mapping/transformation)

**`tf.data`** is that smart kitchen management system for your ML model. It loads, transforms, and feeds data to your model efficiently — keeping the GPU busy and memory usage low.

### Technical Definition

`tf.data.Dataset` is TensorFlow's API for building **input pipelines** — sequences of operations that load raw data, transform it, and deliver it to your model in optimized batches. It handles:

- **Lazy evaluation** — nothing runs until you iterate
- **Parallel processing** — multiple CPU cores transform data simultaneously
- **Prefetching** — loads next batch while GPU trains on current batch
- **Streaming** — processes data that doesn't fit in memory

---

## Why tf.data Matters

### The Bottleneck Problem

```
Without tf.data:
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ Load    │────▶│Transform│────▶│ Train   │────▶│ Load    │──▶ ...
│ Batch 1 │     │ Batch 1 │     │ Batch 1 │     │ Batch 2 │
└─────────┘     └─────────┘     └─────────┘     └─────────┘
   CPU              CPU            GPU              CPU
                                  ↑ IDLE ↑

With tf.data (prefetch + parallel map):
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Load    │─▶│Transform│─▶│ READY   │  ← Batch 1
└─────────┘  └─────────┘  └─────────┘
   ┌─────────┐  ┌─────────┐              ← Batch 2 (parallel)
   │ Load    │─▶│Transform│
   └─────────┘  └─────────┘
                              ┌─────────┐
                              │ Train   │  ← GPU always busy
                              │ Batch 1 │
                              └─────────┘
```

### When to Use tf.data

| Scenario | Use tf.data? | Alternative |
|----------|-------------|-------------|
| Training on GPU with large dataset | ✅ Yes | — |
| Dataset doesn't fit in RAM | ✅ Yes (streaming) | Generators (slower) |
| Need data augmentation | ✅ Yes (in pipeline) | ImageDataGenerator (legacy) |
| Small dataset, quick prototype | Optional | `model.fit(x, y)` with NumPy |
| Reading from TFRecord files | ✅ Yes | — |
| Multi-GPU / TPU training | ✅ Required | — |
| Production ML pipeline | ✅ Yes | TFX (uses tf.data internally) |

### Performance Impact

```
Naive data loading:     ~40% GPU utilization
tf.data with prefetch:  ~85% GPU utilization  
tf.data optimized:      ~95%+ GPU utilization
```

---

## How tf.data Works

### The Pipeline Model

`tf.data` works like a **lazy assembly line**:

```
Source → Transform → Transform → ... → Consumer

[from_tensor_slices] → [map] → [shuffle] → [batch] → [prefetch] → model.fit()
```

**Key insight**: Nothing actually happens until you start consuming (iterating). Each operation returns a **new Dataset object** — it's **functional programming** for data.

### Dataset Types

```
tf.data.Dataset
├── from_tensor_slices()     ← In-memory data (NumPy arrays, tensors)
├── from_tensors()           ← Entire dataset as single element
├── from_generator()         ← Python generator function
├── TextLineDataset()        ← Lines from text files
├── TFRecordDataset()        ← TFRecord binary files
├── list_files()             ← File paths matching a pattern
└── experimental.*           ← CSV, SQL, etc.
```

### Execution Model

```python
# This does NOT load any data — it creates a computation graph
dataset = tf.data.Dataset.from_tensor_slices([1, 2, 3, 4, 5])
dataset = dataset.map(lambda x: x * 2)
dataset = dataset.batch(2)

# Data flows ONLY when you iterate
for batch in dataset:
    print(batch)  # tf.Tensor([2 4]), tf.Tensor([6 8]), tf.Tensor([10])
```

---

## Creating Datasets

### From NumPy Arrays (Most Common)

```python
import tensorflow as tf
import numpy as np

# Create sample data
X = np.random.randn(1000, 28, 28).astype(np.float32)  # 1000 images, 28x28
y = np.random.randint(0, 10, size=(1000,))              # 10 classes

# Create dataset — each element is one (image, label) pair
dataset = tf.data.Dataset.from_tensor_slices((X, y))

# Check what we got
print(dataset.element_spec)
# (TensorSpec(shape=(28, 28), dtype=tf.float32),
#  TensorSpec(shape=(), dtype=tf.int32))

# Peek at first element
for image, label in dataset.take(1):
    print(f"Image shape: {image.shape}, Label: {label.numpy()}")
# Image shape: (28, 28), Label: 3
```

> **Important**: `from_tensor_slices()` slices along the **first dimension**. If you pass a tensor of shape `(1000, 28, 28)`, you get 1000 elements each of shape `(28, 28)`.

### From Tensors (Single Element)

```python
# from_tensors treats the ENTIRE input as a single element
dataset = tf.data.Dataset.from_tensors([1, 2, 3, 4, 5])

for elem in dataset:
    print(elem)  # tf.Tensor([1 2 3 4 5]) — one element!

# vs from_tensor_slices — 5 separate elements
dataset = tf.data.Dataset.from_tensor_slices([1, 2, 3, 4, 5])

for elem in dataset:
    print(elem.numpy())  # 1, 2, 3, 4, 5 — five elements!
```

### From Python Generator

```python
# Useful when data doesn't fit in memory or comes from complex logic
def data_generator():
    """Simulate reading from a database or API"""
    for i in range(10000):
        # Generate one sample at a time — memory efficient
        image = np.random.randn(224, 224, 3).astype(np.float32)
        label = np.random.randint(0, 10)
        yield image, label

# Must specify output types and shapes
dataset = tf.data.Dataset.from_generator(
    data_generator,
    output_signature=(
        tf.TensorSpec(shape=(224, 224, 3), dtype=tf.float32),
        tf.TensorSpec(shape=(), dtype=tf.int32)
    )
)

# Now use like any other dataset
dataset = dataset.batch(32).prefetch(tf.data.AUTOTUNE)
```

> **Warning**: Generator-based datasets cannot be parallelized across workers and are slower than `from_tensor_slices` or `TFRecordDataset`. Use them only when necessary.

### From Files

```python
# List all JPEG files
file_dataset = tf.data.Dataset.list_files("data/images/*.jpg", shuffle=True)

# Read and decode each file
def load_image(file_path):
    # Read raw bytes from disk
    raw = tf.io.read_file(file_path)
    # Decode JPEG to tensor
    image = tf.io.decode_jpeg(raw, channels=3)
    # Resize to consistent shape
    image = tf.image.resize(image, [224, 224])
    # Normalize to [0, 1]
    image = image / 255.0
    # Extract label from directory name
    parts = tf.strings.split(file_path, os.sep)
    label = parts[-2]  # e.g., "cats" or "dogs"
    return image, label

dataset = file_dataset.map(load_image, num_parallel_calls=tf.data.AUTOTUNE)
```

### From Text Files

```python
# Read CSV line by line
dataset = tf.data.TextLineDataset("data/train.csv")

# Skip header
dataset = dataset.skip(1)

# Parse each line
def parse_csv_line(line):
    # Define column types
    defaults = [tf.float32, tf.float32, tf.float32, tf.int32]
    fields = tf.io.decode_csv(line, record_defaults=defaults)
    features = tf.stack(fields[:-1])  # All columns except last
    label = fields[-1]                 # Last column is label
    return features, label

dataset = dataset.map(parse_csv_line)
```

---

## Dataset Transformations

### map() — Apply a Function to Each Element

```python
dataset = tf.data.Dataset.from_tensor_slices(
    (np.random.randn(100, 28, 28).astype(np.float32),
     np.random.randint(0, 10, 100))
)

# Normalize images and one-hot encode labels
def preprocess(image, label):
    image = image / 255.0                              # Normalize
    image = tf.expand_dims(image, -1)                  # Add channel dim: (28,28) → (28,28,1)
    label = tf.one_hot(label, depth=10)                # One-hot encode
    return image, label

dataset = dataset.map(preprocess, num_parallel_calls=tf.data.AUTOTUNE)
# num_parallel_calls=AUTOTUNE lets TF decide optimal parallelism
```

> **Pro Tip**: Functions inside `map()` must use **TensorFlow ops only** (not NumPy). If you need Python/NumPy operations, wrap them with `tf.py_function()`, but this disables parallelism.

```python
# When you MUST use Python code (slower, use sparingly)
def python_processing(image, label):
    # tf.py_function wraps arbitrary Python
    image = tf.py_function(
        func=lambda x: some_numpy_operation(x.numpy()),
        inp=[image],
        Tout=tf.float32
    )
    image.set_shape([28, 28, 1])  # Must set shape manually after py_function
    return image, label
```

### filter() — Keep Only Matching Elements

```python
dataset = tf.data.Dataset.range(100)

# Keep only even numbers
even_dataset = dataset.filter(lambda x: x % 2 == 0)
# Result: 0, 2, 4, 6, ..., 98

# Filter by label — e.g., keep only digits 0-4
def keep_low_digits(image, label):
    return label < 5

filtered = dataset.filter(keep_low_digits)
```

### shuffle() — Randomize Order

```python
dataset = tf.data.Dataset.range(10)

# buffer_size determines randomness quality
# Larger buffer = better shuffling but more memory
shuffled = dataset.shuffle(buffer_size=5, seed=42)

# How shuffle works:
# 1. Fills buffer with first 5 elements: [0, 1, 2, 3, 4]
# 2. Randomly picks one (say 3), outputs it
# 3. Replaces with next element (5): [0, 1, 2, 5, 4]
# 4. Randomly picks another, and so on...
```

```
buffer_size effect on shuffling quality:

buffer_size=1:    No shuffling at all (elements come in order)
buffer_size=N:    Perfectly random (N = dataset size)
buffer_size=1000: Good balance for large datasets

Rule of thumb: buffer_size >= dataset_size for perfect shuffle
                buffer_size >= 10 * batch_size for "good enough"
```

> **Warning**: `shuffle(buffer_size=1)` does NOT shuffle! This is a common bug. Always set buffer_size to at least your batch size.

### take() and skip() — Slice the Dataset

```python
dataset = tf.data.Dataset.range(100)

# Take first 10 elements (for debugging/preview)
preview = dataset.take(10)    # Elements 0-9

# Skip first 80 elements (for validation split)
val_set = dataset.skip(80)    # Elements 80-99
train_set = dataset.take(80)  # Elements 0-79
```

### repeat() — Loop the Dataset

```python
dataset = tf.data.Dataset.range(3)  # [0, 1, 2]

# Repeat 2 times
repeated = dataset.repeat(2)  # [0, 1, 2, 0, 1, 2]

# Repeat forever (for training)
infinite = dataset.repeat()    # [0, 1, 2, 0, 1, 2, 0, 1, 2, ...]
# Must specify steps_per_epoch in model.fit() when using repeat()
```

### concatenate() — Merge Datasets

```python
dataset1 = tf.data.Dataset.range(5)      # [0, 1, 2, 3, 4]
dataset2 = tf.data.Dataset.range(5, 10)  # [5, 6, 7, 8, 9]

combined = dataset1.concatenate(dataset2)  # [0, 1, 2, ..., 9]
```

### zip() — Combine Multiple Datasets

```python
images = tf.data.Dataset.from_tensor_slices(np.random.randn(100, 28, 28))
labels = tf.data.Dataset.from_tensor_slices(np.random.randint(0, 10, 100))

# Zip them together
dataset = tf.data.Dataset.zip((images, labels))

for image, label in dataset.take(1):
    print(image.shape, label.numpy())
```

### flat_map() — Map and Flatten

```python
# Useful for reading multiple files
files = tf.data.Dataset.list_files("data/shards/*.tfrecord")

# flat_map reads each file and flattens all records into one stream
dataset = files.flat_map(
    lambda filename: tf.data.TFRecordDataset(filename)
)
```

### window() — Sliding Window (for Time Series)

```python
dataset = tf.data.Dataset.range(10)  # [0, 1, 2, ..., 9]

# Create windows of size 5, shifting by 1
windowed = dataset.window(size=5, shift=1, drop_remainder=True)

# Each window is a sub-dataset, so flatten it
windowed = windowed.flat_map(lambda w: w.batch(5))

for window in windowed:
    print(window.numpy())
# [0 1 2 3 4]
# [1 2 3 4 5]
# [2 3 4 5 6]
# ...
```

---

## Batching Strategies

### Basic batch()

```python
dataset = tf.data.Dataset.range(10)

# Group into batches of 3
batched = dataset.batch(3)
for batch in batched:
    print(batch.numpy())
# [0 1 2]
# [3 4 5]
# [6 7 8]
# [9]          ← Last batch is smaller!

# Drop the incomplete last batch
batched = dataset.batch(3, drop_remainder=True)
# [0 1 2], [3 4 5], [6 7 8]  ← [9] is dropped
```

> **Pro Tip**: Use `drop_remainder=True` when training on TPU (requires fixed batch sizes) or when using batch normalization (small last batch can cause instability).

### padded_batch() — For Variable-Length Sequences

```python
# Common in NLP — sentences have different lengths
dataset = tf.data.Dataset.from_generator(
    lambda: iter([
        ([1, 2, 3], 1),
        ([4, 5], 0),
        ([6, 7, 8, 9], 1),
        ([10], 0)
    ]),
    output_signature=(
        tf.TensorSpec(shape=(None,), dtype=tf.int32),
        tf.TensorSpec(shape=(), dtype=tf.int32)
    )
)

# Pad sequences to same length within each batch
padded = dataset.padded_batch(
    batch_size=2,
    padded_shapes=([None], []),     # None = pad to max length in batch
    padding_values=(0, -1)          # Pad sequences with 0
)

for sequences, labels in padded:
    print("Sequences:", sequences.numpy())
    print("Labels:", labels.numpy())
# Sequences: [[1 2 3]
#              [4 5 0]]     ← padded with 0
# Labels: [1 0]
```

### unbatch() — Reverse Batching

```python
batched = tf.data.Dataset.from_tensor_slices([[1, 2], [3, 4], [5, 6]])
# Elements: [1,2], [3,4], [5,6]

unbatched = batched.unbatch()
# Elements: 1, 2, 3, 4, 5, 6
```

---

## Performance Optimization

### The Golden Rule: prefetch + parallel map

```python
# THE most important optimization pattern
dataset = (
    tf.data.Dataset.from_tensor_slices((X_train, y_train))
    .shuffle(buffer_size=10000)
    .map(preprocess_fn, num_parallel_calls=tf.data.AUTOTUNE)  # ← Parallel
    .batch(32)
    .prefetch(tf.data.AUTOTUNE)  # ← Overlap data prep with training
)
```

### What AUTOTUNE Does

```python
tf.data.AUTOTUNE  # = -1

# TensorFlow dynamically tunes:
# - num_parallel_calls: how many map() operations run in parallel
# - prefetch buffer_size: how many batches to prepare ahead
# It monitors runtime and adjusts automatically
```

### cache() — Store in Memory or Disk

```python
# Cache in memory (fast, but limited by RAM)
dataset = dataset.cache()

# Cache to disk (slower than RAM, but unlimited)
dataset = dataset.cache("/tmp/my_dataset_cache")

# IMPORTANT: cache BEFORE random transformations, AFTER deterministic ones
dataset = (
    tf.data.Dataset.from_tensor_slices((X, y))
    .map(deterministic_preprocess)    # ← Decode, resize (do once)
    .cache()                           # ← Cache here
    .map(random_augmentation)          # ← Random transforms (different each epoch)
    .shuffle(1000)
    .batch(32)
    .prefetch(tf.data.AUTOTUNE)
)
```

```
Cache placement matters:

WRONG:  source → shuffle → augment → cache → batch → prefetch
        (caches augmented data — same augmentations every epoch!)

RIGHT:  source → decode/resize → cache → shuffle → augment → batch → prefetch
        (caches decoded data, re-augments each epoch)
```

### interleave() — Parallel File Reading

```python
# Read from multiple TFRecord files in parallel
files = tf.data.Dataset.list_files("data/train-*.tfrecord")

dataset = files.interleave(
    lambda x: tf.data.TFRecordDataset(x),
    cycle_length=4,                          # Read 4 files simultaneously
    num_parallel_calls=tf.data.AUTOTUNE,     # Parallelize
    deterministic=False                       # Allow non-deterministic order (faster)
)
```

```
interleave with cycle_length=3:

File A:  a1, a2, a3, a4, ...
File B:  b1, b2, b3, b4, ...
File C:  c1, c2, c3, c4, ...

Output:  a1, b1, c1, a2, b2, c2, a3, b3, c3, ...
         (round-robin from each file)
```

### Complete Optimized Pipeline

```python
def build_optimized_pipeline(file_pattern, batch_size=32, is_training=True):
    """Production-grade tf.data pipeline"""
    
    # Step 1: List and optionally shuffle files
    files = tf.data.Dataset.list_files(file_pattern, shuffle=is_training)
    
    # Step 2: Read files in parallel with interleave
    dataset = files.interleave(
        lambda f: tf.data.TFRecordDataset(f, compression_type="GZIP"),
        cycle_length=8,
        num_parallel_calls=tf.data.AUTOTUNE,
        deterministic=not is_training  # Non-deterministic for training (faster)
    )
    
    # Step 3: Parse and decode (deterministic) → cache
    dataset = dataset.map(parse_tfrecord, num_parallel_calls=tf.data.AUTOTUNE)
    
    if not is_training:
        dataset = dataset.cache()  # Cache validation data (usually smaller)
    
    # Step 4: Shuffle (training only)
    if is_training:
        dataset = dataset.shuffle(buffer_size=10000)
    
    # Step 5: Augment (training only, random transforms)
    if is_training:
        dataset = dataset.map(augment, num_parallel_calls=tf.data.AUTOTUNE)
    
    # Step 6: Batch
    dataset = dataset.batch(batch_size, drop_remainder=is_training)
    
    # Step 7: Prefetch
    dataset = dataset.prefetch(tf.data.AUTOTUNE)
    
    return dataset
```

### Performance Comparison

| Technique | Speedup | When to Use |
|-----------|---------|-------------|
| `prefetch(AUTOTUNE)` | 2-3x | **Always** |
| `map(num_parallel_calls=AUTOTUNE)` | 2-5x | Heavy preprocessing |
| `cache()` | 2-10x | Dataset fits in RAM, or repeated epochs |
| `interleave()` | 2-4x | Multiple input files |
| `deterministic=False` | 1.1-1.5x | Training (order doesn't matter) |
| `TFRecord` format | 2-5x | Large datasets, distributed training |

---

## TFRecord Format

### What is TFRecord?

TFRecord is TensorFlow's **binary storage format**. Think of it as a highly optimized filing cabinet specifically designed for ML data.

```
Why TFRecord?
┌────────────────────────┬────────────────────────┐
│ Individual Files       │ TFRecord               │
├────────────────────────┼────────────────────────┤
│ 1M small file reads    │ Sequential large reads │
│ Slow random access     │ Fast streaming         │
│ High filesystem overhead│ Low overhead          │
│ Hard to shard           │ Easy to split         │
│ No compression          │ Built-in GZIP/ZLIB   │
└────────────────────────┴────────────────────────┘
```

### Writing TFRecords

```python
import tensorflow as tf
import numpy as np

# Helper functions for creating tf.train.Feature
def _bytes_feature(value):
    """Returns a bytes_list Feature."""
    if isinstance(value, type(tf.constant(0))):
        value = value.numpy()
    return tf.train.Feature(bytes_list=tf.train.BytesList(value=[value]))

def _float_feature(value):
    """Returns a float_list Feature."""
    return tf.train.Feature(float_list=tf.train.FloatList(value=[value]))

def _int64_feature(value):
    """Returns an int64_list Feature."""
    return tf.train.Feature(int64_list=tf.train.Int64List(value=[value]))

def _float_array_feature(value):
    """Returns a float_list Feature for arrays."""
    return tf.train.Feature(float_list=tf.train.FloatList(value=value))


# Create a tf.train.Example for one sample
def create_example(image, label):
    """Convert one image-label pair to a tf.train.Example"""
    feature = {
        'image': _bytes_feature(tf.io.serialize_tensor(image)),
        'label': _int64_feature(label),
        'height': _int64_feature(image.shape[0]),
        'width': _int64_feature(image.shape[1]),
    }
    return tf.train.Example(
        features=tf.train.Features(feature=feature)
    )


# Write dataset to TFRecord file
def write_tfrecords(images, labels, filename):
    """Write images and labels to a TFRecord file"""
    with tf.io.TFRecordWriter(filename) as writer:
        for i in range(len(images)):
            example = create_example(images[i], labels[i])
            writer.write(example.SerializeToString())
    print(f"Wrote {len(images)} examples to {filename}")


# Example: Write MNIST-like data
X_train = np.random.randn(1000, 28, 28).astype(np.float32)
y_train = np.random.randint(0, 10, 1000)

write_tfrecords(X_train, y_train, "train.tfrecord")
```

### Writing Sharded TFRecords (Production)

```python
def write_sharded_tfrecords(images, labels, output_dir, num_shards=10):
    """Split data across multiple TFRecord files for parallel reading"""
    import os
    os.makedirs(output_dir, exist_ok=True)
    
    shard_size = len(images) // num_shards
    
    for shard_id in range(num_shards):
        start = shard_id * shard_size
        end = start + shard_size if shard_id < num_shards - 1 else len(images)
        
        filename = os.path.join(output_dir, f"train-{shard_id:05d}-of-{num_shards:05d}.tfrecord")
        
        with tf.io.TFRecordWriter(filename) as writer:
            for i in range(start, end):
                example = create_example(images[i], labels[i])
                writer.write(example.SerializeToString())
        
        print(f"Shard {shard_id}: {end - start} examples → {filename}")

# Write 10 shards
write_sharded_tfrecords(X_train, y_train, "data/tfrecords/", num_shards=10)
```

### Reading TFRecords

```python
# Define the feature description (must match what was written)
feature_description = {
    'image': tf.io.FixedLenFeature([], tf.string),    # Serialized tensor
    'label': tf.io.FixedLenFeature([], tf.int64),
    'height': tf.io.FixedLenFeature([], tf.int64),
    'width': tf.io.FixedLenFeature([], tf.int64),
}

def parse_tfrecord(serialized_example):
    """Parse a single TFRecord example"""
    # Parse the raw bytes
    example = tf.io.parse_single_example(serialized_example, feature_description)
    
    # Deserialize the image tensor
    image = tf.io.parse_tensor(example['image'], out_type=tf.float32)
    image = tf.reshape(image, [example['height'], example['width']])
    
    label = example['label']
    
    return image, label


# Read from TFRecord files
dataset = tf.data.TFRecordDataset(
    ["train.tfrecord"],
    compression_type=""  # or "GZIP" if compressed
)

dataset = (
    dataset
    .map(parse_tfrecord, num_parallel_calls=tf.data.AUTOTUNE)
    .shuffle(1000)
    .batch(32)
    .prefetch(tf.data.AUTOTUNE)
)

# Verify
for images, labels in dataset.take(1):
    print(f"Batch shape: {images.shape}, Labels: {labels.shape}")
```

### Compressed TFRecords

```python
# Write with GZIP compression (saves ~50-70% disk space)
options = tf.io.TFRecordOptions(compression_type="GZIP")
with tf.io.TFRecordWriter("train.tfrecord.gz", options=options) as writer:
    for i in range(len(X_train)):
        example = create_example(X_train[i], y_train[i])
        writer.write(example.SerializeToString())

# Read compressed
dataset = tf.data.TFRecordDataset("train.tfrecord.gz", compression_type="GZIP")
```

---

## Data Augmentation in the Pipeline

### Image Augmentation with tf.image

```python
def augment_image(image, label):
    """Apply random augmentations to an image"""
    # Random horizontal flip (50% chance)
    image = tf.image.random_flip_left_right(image)
    
    # Random brightness adjustment
    image = tf.image.random_brightness(image, max_delta=0.2)
    
    # Random contrast adjustment
    image = tf.image.random_contrast(image, lower=0.8, upper=1.2)
    
    # Random saturation (for color images)
    image = tf.image.random_saturation(image, lower=0.8, upper=1.2)
    
    # Random hue shift
    image = tf.image.random_hue(image, max_delta=0.1)
    
    # Random crop and resize
    image = tf.image.random_crop(image, size=[200, 200, 3])
    image = tf.image.resize(image, [224, 224])
    
    # Clip values to valid range
    image = tf.clip_by_value(image, 0.0, 1.0)
    
    return image, label


# Apply augmentation only during training
train_dataset = (
    raw_dataset
    .map(augment_image, num_parallel_calls=tf.data.AUTOTUNE)
    .batch(32)
    .prefetch(tf.data.AUTOTUNE)
)
```

### Using Keras Preprocessing Layers (Recommended for TF 2.6+)

```python
# Preprocessing layers can be part of the model OR the pipeline
augmentation_layers = tf.keras.Sequential([
    tf.keras.layers.RandomFlip("horizontal"),
    tf.keras.layers.RandomRotation(0.1),       # ±10% rotation
    tf.keras.layers.RandomZoom(0.1),            # ±10% zoom
    tf.keras.layers.RandomContrast(0.1),
    tf.keras.layers.RandomTranslation(0.1, 0.1),
])

def augment_with_layers(image, label):
    image = augmentation_layers(image, training=True)
    return image, label

# Apply in pipeline
train_dataset = train_dataset.map(
    augment_with_layers,
    num_parallel_calls=tf.data.AUTOTUNE
)
```

> **Pro Tip**: If you include augmentation layers **inside** your model, they automatically activate during training and deactivate during inference — no separate pipeline logic needed.

```python
# Augmentation as part of the model
model = tf.keras.Sequential([
    # Augmentation layers (only active during training)
    tf.keras.layers.RandomFlip("horizontal"),
    tf.keras.layers.RandomRotation(0.1),
    
    # Actual model layers
    tf.keras.layers.Conv2D(32, 3, activation='relu'),
    tf.keras.layers.MaxPooling2D(),
    tf.keras.layers.Flatten(),
    tf.keras.layers.Dense(10, activation='softmax'),
])
```

---

## Working with Large Datasets

### tf.keras.utils.image_dataset_from_directory (Easiest)

```python
# Automatically creates a tf.data.Dataset from a directory structure:
# data/
#   train/
#     cats/
#       cat001.jpg
#       cat002.jpg
#     dogs/
#       dog001.jpg
#       dog002.jpg

train_dataset = tf.keras.utils.image_dataset_from_directory(
    "data/train",
    image_size=(224, 224),      # Resize all images
    batch_size=32,
    label_mode='categorical',    # One-hot labels
    shuffle=True,
    seed=42,
    validation_split=0.2,        # Use 20% for validation
    subset="training"
)

val_dataset = tf.keras.utils.image_dataset_from_directory(
    "data/train",
    image_size=(224, 224),
    batch_size=32,
    label_mode='categorical',
    shuffle=True,
    seed=42,
    validation_split=0.2,
    subset="validation"
)

# Optimize
train_dataset = train_dataset.prefetch(tf.data.AUTOTUNE)
val_dataset = val_dataset.cache().prefetch(tf.data.AUTOTUNE)
```

### tf.keras.utils.text_dataset_from_directory

```python
# For text classification from directory structure:
# data/
#   train/
#     positive/
#       review1.txt
#     negative/
#       review2.txt

train_dataset = tf.keras.utils.text_dataset_from_directory(
    "data/train",
    batch_size=32,
    validation_split=0.2,
    subset="training",
    seed=42
)
```

### Handling Datasets Too Large for RAM

```python
# Strategy 1: TFRecord + streaming (no memory limit)
dataset = tf.data.TFRecordDataset(
    tf.io.gfile.glob("gs://bucket/data/train-*.tfrecord")
)

# Strategy 2: Generator with lazy loading
def lazy_loader():
    file_list = os.listdir("data/images/")
    for filename in file_list:
        image = tf.io.read_file(f"data/images/{filename}")
        image = tf.io.decode_jpeg(image)
        yield image, extract_label(filename)

dataset = tf.data.Dataset.from_generator(lazy_loader, ...)

# Strategy 3: Memory-mapped files with interleave
files = tf.data.Dataset.list_files("data/shards/*.tfrecord")
dataset = files.interleave(
    tf.data.TFRecordDataset,
    cycle_length=tf.data.AUTOTUNE,
    num_parallel_calls=tf.data.AUTOTUNE
)
```

---

## Real-World Pipeline Examples

### Complete Image Classification Pipeline

```python
import tensorflow as tf
import os

IMG_SIZE = 224
BATCH_SIZE = 32
AUTOTUNE = tf.data.AUTOTUNE

def build_image_pipeline(data_dir, is_training=True):
    """End-to-end image classification pipeline"""
    
    # Load from directory
    dataset = tf.keras.utils.image_dataset_from_directory(
        data_dir,
        image_size=(IMG_SIZE, IMG_SIZE),
        batch_size=None,  # We'll batch manually for more control
        label_mode='int',
        shuffle=is_training
    )
    
    # Normalize to [0, 1]
    normalization = tf.keras.layers.Rescaling(1.0 / 255)
    dataset = dataset.map(lambda x, y: (normalization(x), y), num_parallel_calls=AUTOTUNE)
    
    if is_training:
        # Augmentation
        augment = tf.keras.Sequential([
            tf.keras.layers.RandomFlip("horizontal"),
            tf.keras.layers.RandomRotation(0.2),
            tf.keras.layers.RandomZoom(0.2),
        ])
        dataset = dataset.map(
            lambda x, y: (augment(x, training=True), y),
            num_parallel_calls=AUTOTUNE
        )
        dataset = dataset.shuffle(1000)
    else:
        dataset = dataset.cache()
    
    dataset = dataset.batch(BATCH_SIZE)
    dataset = dataset.prefetch(AUTOTUNE)
    
    return dataset
```

### Complete NLP Pipeline

```python
import tensorflow as tf

MAX_TOKENS = 20000
MAX_LENGTH = 256
BATCH_SIZE = 64

def build_text_pipeline(texts, labels, is_training=True):
    """End-to-end text classification pipeline"""
    
    dataset = tf.data.Dataset.from_tensor_slices((texts, labels))
    
    if is_training:
        dataset = dataset.shuffle(10000)
    
    # Vectorize text
    vectorizer = tf.keras.layers.TextVectorization(
        max_tokens=MAX_TOKENS,
        output_mode='int',
        output_sequence_length=MAX_LENGTH
    )
    
    # Adapt vectorizer on training data
    if is_training:
        text_dataset = dataset.map(lambda x, y: x)
        vectorizer.adapt(text_dataset.batch(1000))
    
    def vectorize_text(text, label):
        text = tf.expand_dims(text, -1)
        text = vectorizer(text)
        text = tf.squeeze(text, axis=0)
        return text, label
    
    dataset = dataset.map(vectorize_text, num_parallel_calls=tf.data.AUTOTUNE)
    dataset = dataset.batch(BATCH_SIZE)
    dataset = dataset.prefetch(tf.data.AUTOTUNE)
    
    return dataset, vectorizer
```

### Time Series Pipeline

```python
def build_timeseries_pipeline(data, window_size=30, forecast_horizon=1, batch_size=32):
    """Pipeline for time series forecasting"""
    
    dataset = tf.data.Dataset.from_tensor_slices(data)
    
    # Create sliding windows
    dataset = dataset.window(
        size=window_size + forecast_horizon,
        shift=1,
        drop_remainder=True
    )
    
    # Flatten windows into batches
    dataset = dataset.flat_map(lambda w: w.batch(window_size + forecast_horizon))
    
    # Split into input (window) and target (next value)
    dataset = dataset.map(
        lambda w: (w[:window_size], w[window_size:])
    )
    
    dataset = dataset.shuffle(1000)
    dataset = dataset.batch(batch_size)
    dataset = dataset.prefetch(tf.data.AUTOTUNE)
    
    return dataset

# Usage
import numpy as np
time_series_data = np.sin(np.linspace(0, 100, 10000)).astype(np.float32)
dataset = build_timeseries_pipeline(time_series_data, window_size=50, forecast_horizon=1)

for inputs, targets in dataset.take(1):
    print(f"Input shape: {inputs.shape}")   # (32, 50)
    print(f"Target shape: {targets.shape}") # (32, 1)
```

---

## Common Mistakes

### 1. Shuffling with buffer_size=1

```python
# ❌ WRONG — No shuffling at all!
dataset = dataset.shuffle(buffer_size=1)

# ✅ CORRECT — Use a meaningful buffer size
dataset = dataset.shuffle(buffer_size=10000)
```

### 2. Wrong Order of Operations

```python
# ❌ WRONG — Shuffling after batching shuffles batches, not individual samples
dataset = dataset.batch(32).shuffle(100)

# ✅ CORRECT — Shuffle before batching
dataset = dataset.shuffle(10000).batch(32)
```

### 3. Caching After Random Augmentation

```python
# ❌ WRONG — Same augmentations every epoch!
dataset = dataset.map(random_augment).cache().batch(32)

# ✅ CORRECT — Cache raw data, then augment
dataset = dataset.cache().map(random_augment).batch(32)
```

### 4. Forgetting prefetch()

```python
# ❌ WRONG — GPU waits for data
dataset = dataset.shuffle(1000).batch(32)

# ✅ CORRECT — Always end with prefetch
dataset = dataset.shuffle(1000).batch(32).prefetch(tf.data.AUTOTUNE)
```

### 5. Using NumPy Inside map()

```python
# ❌ WRONG — NumPy operations won't work in tf.data graph
def preprocess(x, y):
    x = np.expand_dims(x, -1)  # This will fail!
    return x, y

# ✅ CORRECT — Use TensorFlow ops
def preprocess(x, y):
    x = tf.expand_dims(x, -1)
    return x, y
```

### 6. Not Setting Shapes After py_function

```python
# ❌ WRONG — Shape is unknown after py_function
def process(x):
    result = tf.py_function(my_func, [x], tf.float32)
    return result  # Shape is None — will cause errors downstream

# ✅ CORRECT — Set shape explicitly
def process(x):
    result = tf.py_function(my_func, [x], tf.float32)
    result.set_shape([224, 224, 3])  # Tell TF the expected shape
    return result
```

### 7. Using repeat() Without steps_per_epoch

```python
# ❌ WRONG — Training will never end!
dataset = dataset.repeat()
model.fit(dataset, epochs=10)

# ✅ CORRECT — Specify steps per epoch
dataset = dataset.repeat()
model.fit(dataset, epochs=10, steps_per_epoch=len(X_train) // batch_size)
```

---

## Interview Questions

### Q1: What is tf.data and why is it preferred over feeding NumPy arrays directly?

**Answer**: `tf.data.Dataset` is TensorFlow's input pipeline API. It's preferred because:
1. **Memory efficiency** — streams data instead of loading everything into RAM
2. **Performance** — prefetching overlaps data loading with model training
3. **Parallelism** — `num_parallel_calls` enables multi-threaded data preprocessing
4. **Flexibility** — can read from files, generators, TFRecords, cloud storage
5. **GPU utilization** — keeps GPU busy instead of waiting for data

### Q2: Explain the difference between `cache()`, `prefetch()`, and `shuffle()`.

**Answer**:
- **`cache()`**: Stores the dataset in memory (or disk) after the first epoch. Subsequent epochs read from cache instead of re-processing. Place it after expensive deterministic operations but before random augmentations.
- **`prefetch()`**: Prepares the next batch while the current batch is being trained on. Overlaps data preprocessing (CPU) with model execution (GPU).
- **`shuffle()`**: Randomizes element order using a fixed-size buffer. `buffer_size` controls randomness quality — larger buffer means better shuffling but more memory.

### Q3: What is the optimal order of tf.data operations?

**Answer**: The recommended order is:
```
list_files → interleave → map(decode) → cache → shuffle → map(augment) → batch → prefetch
```
Key rules:
- Shuffle **before** batch (shuffle individual samples, not batches)
- Cache **after** deterministic ops, **before** random ops
- Prefetch **always last**
- Augment **after** cache (so augmentation varies each epoch)

### Q4: What is TFRecord and when should you use it?

**Answer**: TFRecord is TensorFlow's binary serialization format. Use it when:
- Dataset has millions of small files (avoids filesystem overhead)
- Training on distributed systems / TPUs
- Need to read data sequentially (optimized for streaming)
- Want to shard data across multiple files for parallel reading
- Storing complex data types (images + bounding boxes + metadata)

### Q5: How does `interleave()` differ from `flat_map()`?

**Answer**:
- **`flat_map()`**: Processes sources **sequentially** — finishes one source before starting the next
- **`interleave()`**: Processes multiple sources **concurrently** — reads from `cycle_length` sources in round-robin or parallel fashion
- `interleave()` is preferred for reading multiple files because it achieves higher I/O throughput

### Q6: How would you handle a dataset that doesn't fit in memory?

**Answer**:
1. **TFRecord with streaming** — Write data to TFRecord files, read with `TFRecordDataset` (processes one example at a time)
2. **`from_generator()`** — Lazily generate samples from disk
3. **Sharded files + `interleave()`** — Split data across multiple files, read in parallel
4. **`cache()` to disk** — `dataset.cache("/tmp/cache")` stores processed data on disk
5. **Cloud storage** — Read directly from GCS/S3 with TFRecord

### Q7: What is `tf.data.AUTOTUNE` and when should you use it?

**Answer**: `tf.data.AUTOTUNE` (value = -1) tells TensorFlow to dynamically determine the optimal value at runtime. Use it for:
- `num_parallel_calls` in `map()` and `interleave()`
- `buffer_size` in `prefetch()`

It monitors pipeline performance and adjusts values automatically. **Always prefer AUTOTUNE** over hardcoded values unless you have a specific reason.

---

## Quick Reference

### Essential Pipeline Pattern

```python
dataset = (
    tf.data.Dataset.from_tensor_slices((X, y))
    .shuffle(buffer_size=10000)
    .map(preprocess_fn, num_parallel_calls=tf.data.AUTOTUNE)
    .batch(batch_size)
    .prefetch(tf.data.AUTOTUNE)
)
```

### Transformation Cheat Sheet

| Operation | Purpose | Key Parameter |
|-----------|---------|---------------|
| `from_tensor_slices()` | Create from arrays | Slices along axis 0 |
| `from_generator()` | Create from generator | `output_signature` required |
| `map(fn)` | Transform each element | `num_parallel_calls=AUTOTUNE` |
| `filter(fn)` | Keep matching elements | Returns bool |
| `shuffle(N)` | Randomize order | `buffer_size` ≥ dataset size ideal |
| `batch(N)` | Group into batches | `drop_remainder` for TPU |
| `padded_batch(N)` | Batch with padding | `padded_shapes` |
| `cache()` | Store in memory/disk | Optional filename for disk |
| `prefetch(N)` | Overlap prep & training | Always use AUTOTUNE |
| `repeat(N)` | Loop dataset N times | None = infinite |
| `take(N)` | First N elements | For debugging |
| `skip(N)` | Skip first N elements | For splits |
| `interleave()` | Parallel file reading | `cycle_length` |
| `unbatch()` | Reverse batching | — |
| `window()` | Sliding windows | `size`, `shift` |
| `flat_map()` | Map + flatten | — |

### Operation Order

```
Source → decode/resize → cache → shuffle → augment → batch → prefetch
```

### Key Constants

```python
tf.data.AUTOTUNE   # = -1, dynamic tuning
tf.data.INFINITE_CARDINALITY    # Dataset has infinite elements
tf.data.UNKNOWN_CARDINALITY     # Cardinality can't be determined
```

---

*Next: [05-CNN-with-TensorFlow.md](./05-CNN-with-TensorFlow.md) — Building CNNs with TensorFlow/Keras*
