# Chapter 04: Data Loading and Transforms

## Table of Contents
- [1. Dataset and DataLoader — The Data Pipeline](#1-dataset-and-dataloader)
- [2. Built-in Datasets — Ready-Made Data](#2-built-in-datasets)
- [3. Custom Datasets — Your Own Data](#3-custom-datasets)
- [4. Transforms — Preprocessing and Augmentation](#4-transforms)
- [5. Advanced DataLoader Features](#5-advanced-dataloader-features)
- [6. Data Augmentation Strategies](#6-data-augmentation-strategies)
- [7. Working with Different Data Types](#7-working-with-different-data-types)
- [8. Performance Optimization](#8-performance-optimization)
- [9. Common Mistakes](#9-common-mistakes)
- [10. Interview Questions](#10-interview-questions)
- [11. Quick Reference](#11-quick-reference)

---

## 1. Dataset and DataLoader — The Data Pipeline {#1-dataset-and-dataloader}

### What It Is
PyTorch separates data handling into two core abstractions:
- **`Dataset`** — Defines *what* the data is and *how* to access individual samples (like a filing cabinet)
- **`DataLoader`** — Defines *how* to serve the data in batches during training (like a waiter bringing plates from the kitchen)

Think of it this way: `Dataset` is your entire photo album. `DataLoader` is the friend who picks 32 photos at a time, shuffles them, and hands them to you.

### Why It Matters
- Real-world data is too large to fit in memory all at once
- Training requires data in **mini-batches** (not all at once, not one-by-one)
- You need shuffling, parallel loading, and preprocessing — DataLoader handles all of this
- This abstraction lets you swap datasets without changing training code

### How It Works

```
┌──────────────────────────────────────────────────────────┐
│                     DATA PIPELINE                         │
│                                                          │
│  ┌──────────┐     ┌────────────┐     ┌──────────────┐   │
│  │  Files   │     │  Dataset   │     │  DataLoader  │   │
│  │ on Disk  │────▶│  __len__   │────▶│  batching    │──▶│ Training
│  │ (images, │     │  __getitem__│    │  shuffling   │   │  Loop
│  │  csv,    │     │  transform │     │  parallel    │   │
│  │  etc.)   │     │            │     │  loading     │   │
│  └──────────┘     └────────────┘     └──────────────┘   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### The Simplest Example

```python
import torch
from torch.utils.data import Dataset, DataLoader, TensorDataset

# ═══════════════════════════════════════════
# METHOD 1: TensorDataset (for in-memory data)
# ═══════════════════════════════════════════
X = torch.randn(1000, 10)  # 1000 samples, 10 features
y = torch.randint(0, 3, (1000,))  # 1000 labels (3 classes)

dataset = TensorDataset(X, y)  # Wraps tensors into a dataset
print(f"Dataset size: {len(dataset)}")  # 1000
print(f"Single sample: {dataset[0]}")   # (tensor([...]), tensor(2))

# Create DataLoader
loader = DataLoader(
    dataset,
    batch_size=32,     # 32 samples per batch
    shuffle=True,      # Randomize order each epoch
    num_workers=0,     # Parallel data loading processes
    drop_last=False    # Keep last incomplete batch
)

# Iterate over batches
for batch_X, batch_y in loader:
    print(f"Batch shape: X={batch_X.shape}, y={batch_y.shape}")
    break
# Output: Batch shape: X=torch.Size([32, 10]), y=torch.Size([32])
```

---

## 2. Built-in Datasets — Ready-Made Data {#2-built-in-datasets}

### What It Is
PyTorch provides ready-to-use datasets through companion libraries:
- **`torchvision.datasets`** — Image datasets (MNIST, CIFAR, ImageNet, etc.)
- **`torchtext.datasets`** — Text datasets (IMDB, AG_NEWS, etc.)
- **`torchaudio.datasets`** — Audio datasets (LibriSpeech, YESNO, etc.)

### torchvision Datasets

```python
import torchvision
import torchvision.transforms as transforms

# ═══════════════════════════════════════════
# MNIST — Handwritten Digits (28×28 grayscale)
# ═══════════════════════════════════════════
transform = transforms.Compose([
    transforms.ToTensor(),           # PIL Image → Tensor (also scales 0-255 → 0-1)
    transforms.Normalize((0.1307,), (0.3081,))  # Mean, Std of MNIST
])

train_dataset = torchvision.datasets.MNIST(
    root='./data',          # Where to store/find data
    train=True,             # Training split
    download=True,          # Download if not present
    transform=transform     # Apply transforms
)

test_dataset = torchvision.datasets.MNIST(
    root='./data',
    train=False,
    download=True,
    transform=transform
)

print(f"Training samples: {len(train_dataset)}")  # 60,000
print(f"Test samples: {len(test_dataset)}")        # 10,000

# Access a single sample
image, label = train_dataset[0]
print(f"Image shape: {image.shape}")  # torch.Size([1, 28, 28])  → (C, H, W)
print(f"Label: {label}")              # 5

# ═══════════════════════════════════════════
# CIFAR-10 — 10 classes of color images (32×32 RGB)
# ═══════════════════════════════════════════
transform_cifar = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((0.4914, 0.4822, 0.4465),   # CIFAR-10 channel means
                         (0.2470, 0.2435, 0.2616))    # CIFAR-10 channel stds
])

cifar_train = torchvision.datasets.CIFAR10(
    root='./data', train=True, download=True, transform=transform_cifar
)

classes = ('plane', 'car', 'bird', 'cat', 'deer',
           'dog', 'frog', 'horse', 'ship', 'truck')

image, label = cifar_train[0]
print(f"Image shape: {image.shape}")     # torch.Size([3, 32, 32])  → (C, H, W)
print(f"Class: {classes[label]}")        # frog

# ═══════════════════════════════════════════
# ImageFolder — Load from directory structure
# ═══════════════════════════════════════════
# Expected folder structure:
# data/
#   train/
#     cats/
#       cat001.jpg
#       cat002.jpg
#     dogs/
#       dog001.jpg
#       dog002.jpg
#   val/
#     cats/
#     dogs/

train_dataset = torchvision.datasets.ImageFolder(
    root='data/train',
    transform=transforms.Compose([
        transforms.Resize((224, 224)),
        transforms.ToTensor(),
        transforms.Normalize([0.485, 0.456, 0.406],   # ImageNet means
                             [0.229, 0.224, 0.225])    # ImageNet stds
    ])
)

print(f"Classes: {train_dataset.classes}")           # ['cats', 'dogs']
print(f"Class mapping: {train_dataset.class_to_idx}")  # {'cats': 0, 'dogs': 1}
```

### Common Built-in Datasets

| Dataset | Domain | Samples | Classes | Image Size |
|---------|--------|---------|---------|------------|
| MNIST | Digits | 70K | 10 | 28×28 gray |
| FashionMNIST | Clothing | 70K | 10 | 28×28 gray |
| CIFAR-10 | Objects | 60K | 10 | 32×32 RGB |
| CIFAR-100 | Objects | 60K | 100 | 32×32 RGB |
| ImageNet | Objects | 1.2M | 1000 | Various RGB |
| COCO | Detection | 330K | 80 | Various RGB |
| VOC | Detection | 11.5K | 20 | Various RGB |

---

## 3. Custom Datasets — Your Own Data {#3-custom-datasets}

### What It Is
When your data doesn't fit a built-in dataset, you create a custom `Dataset` class. You just need to implement three methods:
- `__init__()` — Set up file paths, load metadata
- `__len__()` — Return total number of samples
- `__getitem__(idx)` — Return one sample (data + label)

### Why It Matters
- Real projects almost always use custom data
- You need to handle CSV files, custom image folders, text files, databases, etc.
- Custom datasets let you implement lazy loading (load from disk on demand)

### Custom Dataset for Tabular Data (CSV)

```python
import pandas as pd
import torch
from torch.utils.data import Dataset, DataLoader

class TabularDataset(Dataset):
    """Custom dataset for CSV/tabular data."""
    
    def __init__(self, csv_file, target_column, feature_columns=None):
        """
        Args:
            csv_file (str): Path to CSV file
            target_column (str): Name of the target column
            feature_columns (list): Column names for features (None = all except target)
        """
        self.df = pd.read_csv(csv_file)
        self.target_column = target_column
        
        if feature_columns is None:
            feature_columns = [c for c in self.df.columns if c != target_column]
        
        # Convert to tensors once (fast access later)
        self.features = torch.tensor(
            self.df[feature_columns].values, dtype=torch.float32
        )
        self.targets = torch.tensor(
            self.df[target_column].values, dtype=torch.long
        )
    
    def __len__(self):
        return len(self.df)
    
    def __getitem__(self, idx):
        return self.features[idx], self.targets[idx]

# Usage
# dataset = TabularDataset('data.csv', target_column='label')
# loader = DataLoader(dataset, batch_size=64, shuffle=True)
```

### Custom Dataset for Images

```python
import os
from PIL import Image
import torch
from torch.utils.data import Dataset
import torchvision.transforms as transforms

class CustomImageDataset(Dataset):
    """Load images from a directory with a CSV annotation file."""
    
    def __init__(self, annotations_file, img_dir, transform=None):
        """
        Args:
            annotations_file (str): CSV with columns [filename, label]
            img_dir (str): Directory containing images
            transform: Transforms to apply to images
        """
        self.labels_df = pd.read_csv(annotations_file)
        self.img_dir = img_dir
        self.transform = transform
    
    def __len__(self):
        return len(self.labels_df)
    
    def __getitem__(self, idx):
        # Get image path and label
        img_name = self.labels_df.iloc[idx, 0]  # filename column
        img_path = os.path.join(self.img_dir, img_name)
        label = self.labels_df.iloc[idx, 1]      # label column
        
        # Load image (lazy loading — only loads when accessed!)
        image = Image.open(img_path).convert('RGB')
        
        # Apply transforms
        if self.transform:
            image = self.transform(image)
        
        return image, label

# Usage
transform = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225])
])

# dataset = CustomImageDataset('labels.csv', 'images/', transform=transform)
# loader = DataLoader(dataset, batch_size=32, shuffle=True, num_workers=4)
```

### Custom Dataset for Text

```python
import torch
from torch.utils.data import Dataset
from collections import Counter

class TextClassificationDataset(Dataset):
    """Custom dataset for text classification."""
    
    def __init__(self, texts, labels, vocab=None, max_length=256):
        """
        Args:
            texts (list): List of text strings
            labels (list): List of integer labels
            vocab (dict): Token-to-index mapping (built from training data)
            max_length (int): Maximum sequence length (pad/truncate to this)
        """
        self.labels = labels
        self.max_length = max_length
        
        # Build vocabulary from training data
        if vocab is None:
            word_counts = Counter()
            for text in texts:
                word_counts.update(text.lower().split())
            # Reserve 0=PAD, 1=UNK
            self.vocab = {word: idx + 2 for idx, (word, _) 
                         in enumerate(word_counts.most_common(10000))}
            self.vocab['<PAD>'] = 0
            self.vocab['<UNK>'] = 1
        else:
            self.vocab = vocab
        
        # Tokenize all texts
        self.encoded_texts = [self._encode(text) for text in texts]
    
    def _encode(self, text):
        """Convert text to padded/truncated integer sequence."""
        tokens = text.lower().split()
        indices = [self.vocab.get(t, 1) for t in tokens]  # 1 = UNK
        
        # Truncate
        indices = indices[:self.max_length]
        # Pad
        indices = indices + [0] * (self.max_length - len(indices))
        
        return indices
    
    def __len__(self):
        return len(self.labels)
    
    def __getitem__(self, idx):
        return (
            torch.tensor(self.encoded_texts[idx], dtype=torch.long),
            torch.tensor(self.labels[idx], dtype=torch.long)
        )

# Usage
# texts = ["this movie is great", "terrible film", ...]
# labels = [1, 0, ...]
# dataset = TextClassificationDataset(texts, labels)
```

### Map-Style vs Iterable-Style Datasets

```python
# ═══════════════════════════════════════════
# Map-Style Dataset (most common)
# Implements __getitem__ and __len__
# Supports random access: dataset[42]
# ═══════════════════════════════════════════
class MapDataset(Dataset):
    def __len__(self):
        return 1000
    def __getitem__(self, idx):
        return torch.randn(10), torch.randint(0, 3, ())

# ═══════════════════════════════════════════
# Iterable-Style Dataset
# Implements __iter__ (for streaming data)
# Use when data is too large or comes from a stream
# ═══════════════════════════════════════════
from torch.utils.data import IterableDataset

class StreamDataset(IterableDataset):
    """For streaming data from files, APIs, databases, etc."""
    
    def __init__(self, file_path):
        self.file_path = file_path
    
    def __iter__(self):
        with open(self.file_path, 'r') as f:
            for line in f:
                # Process each line on-the-fly
                parts = line.strip().split(',')
                features = torch.tensor([float(x) for x in parts[:-1]])
                label = torch.tensor(int(parts[-1]))
                yield features, label

# Usage
# dataset = StreamDataset('huge_file.csv')
# loader = DataLoader(dataset, batch_size=32)
# Note: shuffle=True does NOT work with IterableDataset!
```

---

## 4. Transforms — Preprocessing and Augmentation {#4-transforms}

### What It Is
Transforms are operations applied to data before feeding it to the model. Two types:
1. **Preprocessing** — Required steps (resize, normalize, convert to tensor)
2. **Augmentation** — Random modifications to increase data diversity (flip, rotate, crop)

Analogy: Preprocessing is like formatting a Word document (necessary). Augmentation is like paraphrasing sentences to create more training examples.

### Why It Matters
- Models expect specific input formats (tensors, normalized values, fixed size)
- Augmentation is the **cheapest way to get more training data**
- Good augmentation can improve accuracy by 5-15% without any model changes
- Prevents overfitting by making the model see different versions of the same image

### torchvision.transforms — The Classic API

```python
import torchvision.transforms as transforms
from PIL import Image

# ═══════════════════════════════════════════
# PREPROCESSING TRANSFORMS (Always Applied)
# ═══════════════════════════════════════════

# Resize — make all images the same size
transforms.Resize((224, 224))          # Resize to 224×224
transforms.Resize(256)                 # Resize smallest edge to 256

# CenterCrop — crop the center region
transforms.CenterCrop(224)             # Crop 224×224 from center

# ToTensor — PIL Image (H, W, C) uint8 → Tensor (C, H, W) float32 [0, 1]
transforms.ToTensor()                  # ALSO scales pixel values: 0-255 → 0.0-1.0

# Normalize — standardize using mean and std
transforms.Normalize(
    mean=[0.485, 0.456, 0.406],        # ImageNet means (per channel)
    std=[0.229, 0.224, 0.225]          # ImageNet stds (per channel)
)
# Formula: output = (input - mean) / std

# ═══════════════════════════════════════════
# AUGMENTATION TRANSFORMS (Training Only!)
# ═══════════════════════════════════════════

# RandomHorizontalFlip — 50% chance to flip left-right
transforms.RandomHorizontalFlip(p=0.5)

# RandomVerticalFlip — 50% chance to flip top-bottom
transforms.RandomVerticalFlip(p=0.5)

# RandomCrop — crop random region of specified size
transforms.RandomCrop(224)             # Crop random 224×224 region
transforms.RandomCrop(224, padding=4)  # Pad 4px first, then random crop

# RandomResizedCrop — crop random region and resize
transforms.RandomResizedCrop(
    224,                               # Output size
    scale=(0.08, 1.0),                 # Crop area: 8%-100% of original
    ratio=(0.75, 1.33)                 # Aspect ratio range
)

# RandomRotation
transforms.RandomRotation(degrees=15)  # Rotate ±15 degrees

# ColorJitter — random brightness, contrast, saturation, hue
transforms.ColorJitter(
    brightness=0.2,    # ±20% brightness
    contrast=0.2,      # ±20% contrast
    saturation=0.2,    # ±20% saturation
    hue=0.1            # ±10% hue shift
)

# RandomGrayscale — randomly convert to grayscale
transforms.RandomGrayscale(p=0.1)      # 10% chance

# GaussianBlur
transforms.GaussianBlur(kernel_size=3, sigma=(0.1, 2.0))

# RandomErasing — randomly erase a rectangle (after ToTensor!)
transforms.RandomErasing(p=0.5, scale=(0.02, 0.33))
```

### Composing Transforms

```python
# ═══════════════════════════════════════════
# TRAINING transforms (with augmentation)
# ═══════════════════════════════════════════
train_transform = transforms.Compose([
    transforms.RandomResizedCrop(224),       # Random crop + resize
    transforms.RandomHorizontalFlip(),       # 50% horizontal flip
    transforms.ColorJitter(0.2, 0.2, 0.2),  # Color augmentation
    transforms.ToTensor(),                   # Convert to tensor [0, 1]
    transforms.Normalize(                    # ImageNet normalization
        mean=[0.485, 0.456, 0.406],
        std=[0.229, 0.224, 0.225]
    ),
    transforms.RandomErasing(p=0.1),         # Random erasing (after ToTensor!)
])

# ═══════════════════════════════════════════
# VALIDATION/TEST transforms (NO augmentation)
# ═══════════════════════════════════════════
val_transform = transforms.Compose([
    transforms.Resize(256),                  # Resize smallest edge to 256
    transforms.CenterCrop(224),              # Center crop 224×224
    transforms.ToTensor(),
    transforms.Normalize(
        mean=[0.485, 0.456, 0.406],
        std=[0.229, 0.224, 0.225]
    ),
])

# Apply to datasets
train_dataset = torchvision.datasets.CIFAR10(
    root='./data', train=True, transform=train_transform
)
val_dataset = torchvision.datasets.CIFAR10(
    root='./data', train=False, transform=val_transform
)
```

> **Warning**: NEVER apply data augmentation to validation/test data! Augmentation is for training only. Validation transforms should be deterministic (always produce the same output).

### transforms.v2 — The Modern API (PyTorch ≥ 2.0)

```python
# v2 transforms work on tensors, PIL images, bounding boxes, masks, and videos
# They are faster and more flexible than v1
from torchvision.transforms import v2

train_transform = v2.Compose([
    v2.RandomResizedCrop(224, antialias=True),
    v2.RandomHorizontalFlip(p=0.5),
    v2.ColorJitter(brightness=0.2, contrast=0.2),
    v2.ToDtype(torch.float32, scale=True),  # Replaces ToTensor() + scales 0-1
    v2.Normalize(mean=[0.485, 0.456, 0.406],
                 std=[0.229, 0.224, 0.225]),
])

# v2 can also handle detection/segmentation data jointly
# (transforms bounding boxes and masks together with images)
```

### Custom Transforms

```python
# Custom transform as a callable class
class AddGaussianNoise:
    """Add random Gaussian noise to a tensor."""
    def __init__(self, mean=0.0, std=0.01):
        self.mean = mean
        self.std = std
    
    def __call__(self, tensor):
        noise = torch.randn_like(tensor) * self.std + self.mean
        return tensor + noise
    
    def __repr__(self):
        return f"{self.__class__.__name__}(mean={self.mean}, std={self.std})"

# Custom transform as a function with Lambda
custom_normalize = transforms.Lambda(lambda x: x * 2 - 1)  # Scale [0,1] → [-1,1]

# Use in Compose
transform = transforms.Compose([
    transforms.ToTensor(),
    AddGaussianNoise(std=0.05),
    custom_normalize,
])
```

---

## 5. Advanced DataLoader Features {#5-advanced-dataloader-features}

### DataLoader Parameters Deep Dive

```python
from torch.utils.data import DataLoader

loader = DataLoader(
    dataset,
    
    # ── BATCHING ──
    batch_size=64,          # Samples per batch
    drop_last=False,        # Drop last incomplete batch? (useful for BatchNorm)
    
    # ── ORDERING ──
    shuffle=True,           # Randomize sample order each epoch
    
    # ── PERFORMANCE ──
    num_workers=4,          # Parallel data loading processes
    pin_memory=True,        # Pin memory for faster GPU transfer
    prefetch_factor=2,      # Batches to prefetch per worker
    persistent_workers=True,  # Keep workers alive between epochs
    
    # ── ADVANCED ──
    collate_fn=None,        # Custom function to merge samples into batch
    sampler=None,           # Custom sampling strategy
    batch_sampler=None,     # Custom batch sampling strategy
)
```

### `num_workers` — Parallel Data Loading

```
num_workers=0 (default):
┌────────┐    ┌─────────┐    ┌───────┐
│  Load  │───▶│Transform│───▶│ Train │  ← Sequential: CPU loads, then GPU trains
│  Data  │    │  Data   │    │ Batch │     (GPU waits while CPU works!)
└────────┘    └─────────┘    └───────┘

num_workers=4:
┌─Worker 1─┐ ┌─Worker 2─┐ ┌─Worker 3─┐ ┌─Worker 4─┐
│Load+Trans│ │Load+Trans│ │Load+Trans│ │Load+Trans│  ← Parallel loading
└────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘
     └────────────┼────────────┼────────────┘
                  ▼
            ┌───────────┐
            │ GPU Train │  ← GPU gets batches without waiting!
            └───────────┘
```

```python
# Rule of thumb for num_workers:
# - Start with num_workers = 4
# - Max: number of CPU cores (os.cpu_count())
# - On Windows: num_workers=0 may be needed (multiprocessing issues)
# - Too many workers → overhead from process creation/communication

import os
num_workers = min(4, os.cpu_count())
```

### `pin_memory` — Faster CPU-to-GPU Transfer

```python
# pin_memory=True: Uses page-locked (pinned) memory
# This allows asynchronous CPU→GPU transfer (non_blocking=True)
# Always use when training on GPU!

loader = DataLoader(dataset, batch_size=64, pin_memory=True)

# In training loop, use non_blocking=True for overlap
for inputs, targets in loader:
    inputs = inputs.to(device, non_blocking=True)   # Async transfer
    targets = targets.to(device, non_blocking=True)  # Async transfer
    # Training step...
```

### Custom `collate_fn` — Variable-Length Data

```python
# Default collate stacks tensors of SAME size into a batch
# For variable-length sequences, you need custom collation

def pad_collate_fn(batch):
    """Pad variable-length sequences to the longest in the batch."""
    # batch = [(seq1, label1), (seq2, label2), ...]
    sequences, labels = zip(*batch)
    
    # Find max length in this batch
    lengths = [len(seq) for seq in sequences]
    max_len = max(lengths)
    
    # Pad sequences
    padded = torch.zeros(len(sequences), max_len, dtype=torch.long)
    for i, seq in enumerate(sequences):
        padded[i, :len(seq)] = seq
    
    labels = torch.stack(labels)
    lengths = torch.tensor(lengths)
    
    return padded, labels, lengths

# Usage
loader = DataLoader(dataset, batch_size=32, collate_fn=pad_collate_fn)

# Pro Tip: Use torch.nn.utils.rnn.pad_sequence instead of manual padding
from torch.nn.utils.rnn import pad_sequence

def collate_fn(batch):
    sequences, labels = zip(*batch)
    padded = pad_sequence(sequences, batch_first=True, padding_value=0)
    labels = torch.stack(labels)
    return padded, labels
```

### Samplers — Custom Sampling Strategies

```python
from torch.utils.data import (
    RandomSampler, SequentialSampler, SubsetRandomSampler,
    WeightedRandomSampler
)

# ═══════════════════════════════════════════
# WeightedRandomSampler — handle class imbalance
# ═══════════════════════════════════════════
# If dataset has 90% class-0, 10% class-1, oversample class-1
labels = [0]*900 + [1]*100  # Imbalanced dataset

# Compute sample weights (inverse class frequency)
class_counts = [900, 100]
class_weights = 1.0 / torch.tensor(class_counts, dtype=torch.float)
sample_weights = [class_weights[label] for label in labels]

sampler = WeightedRandomSampler(
    weights=sample_weights,
    num_samples=len(labels),  # Draw this many samples per epoch
    replacement=True           # Allow resampling (needed for oversampling)
)

loader = DataLoader(dataset, batch_size=32, sampler=sampler)
# Note: When using a sampler, do NOT set shuffle=True (mutually exclusive)

# ═══════════════════════════════════════════
# SubsetRandomSampler — use subset of data
# ═══════════════════════════════════════════
# Useful for train/val split
indices = list(range(len(dataset)))
train_indices = indices[:800]
val_indices = indices[800:]

train_loader = DataLoader(dataset, batch_size=32,
                          sampler=SubsetRandomSampler(train_indices))
val_loader = DataLoader(dataset, batch_size=32,
                        sampler=SubsetRandomSampler(val_indices))
```

### Train/Validation Split

```python
from torch.utils.data import random_split

# Method 1: random_split (recommended)
full_dataset = torchvision.datasets.CIFAR10(root='./data', train=True,
                                             transform=train_transform)

train_size = int(0.8 * len(full_dataset))
val_size = len(full_dataset) - train_size

train_dataset, val_dataset = random_split(
    full_dataset, [train_size, val_size],
    generator=torch.Generator().manual_seed(42)  # Reproducible split
)

# ⚠️ Problem: Both splits use the SAME transform (train_transform with augmentation)
# Solution: Override the transform for the validation split

# Method 2: Apply different transforms to train/val splits
class TransformSubset(torch.utils.data.Dataset):
    """Wraps a Subset with a custom transform."""
    def __init__(self, subset, transform):
        self.subset = subset
        self.transform = transform
    
    def __len__(self):
        return len(self.subset)
    
    def __getitem__(self, idx):
        image, label = self.subset[idx]
        if self.transform:
            # If image is already a tensor from a parent transform,
            # you may need ToPILImage() first
            image = self.transform(image)
        return image, label

# Better approach: Load dataset WITHOUT transform, apply in wrapper
base_dataset = torchvision.datasets.CIFAR10(root='./data', train=True)
train_sub, val_sub = random_split(base_dataset, [train_size, val_size])

train_dataset = TransformSubset(train_sub, train_transform)
val_dataset = TransformSubset(val_sub, val_transform)
```

---

## 6. Data Augmentation Strategies {#6-data-augmentation-strategies}

### What It Is
Data augmentation creates modified versions of existing data to increase training diversity. Instead of collecting more data, you create plausible variations.

### Standard Augmentation Recipes

```python
# ═══════════════════════════════════════════
# CIFAR-10 / Small Image Classification
# ═══════════════════════════════════════════
cifar_train_transform = transforms.Compose([
    transforms.RandomCrop(32, padding=4),      # Standard for CIFAR
    transforms.RandomHorizontalFlip(),
    transforms.ToTensor(),
    transforms.Normalize((0.4914, 0.4822, 0.4465),
                         (0.2470, 0.2435, 0.2616)),
])

# ═══════════════════════════════════════════
# ImageNet / Large Image Classification
# ═══════════════════════════════════════════
imagenet_train_transform = transforms.Compose([
    transforms.RandomResizedCrop(224),         # Standard for ImageNet
    transforms.RandomHorizontalFlip(),
    transforms.ColorJitter(0.4, 0.4, 0.4),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406],
                         [0.229, 0.224, 0.225]),
    transforms.RandomErasing(p=0.25),
])

# ═══════════════════════════════════════════
# Medical Imaging (be careful with flips!)
# ═══════════════════════════════════════════
medical_transform = transforms.Compose([
    transforms.RandomAffine(
        degrees=10,             # Small rotation (anatomy shouldn't be upside down)
        translate=(0.05, 0.05), # Small shift
        scale=(0.95, 1.05),     # Slight zoom
    ),
    transforms.RandomHorizontalFlip(),  # Only if anatomically valid!
    # NO vertical flip for chest X-rays!
    transforms.ColorJitter(brightness=0.1, contrast=0.1),
    transforms.ToTensor(),
    transforms.Normalize([0.5], [0.5]),
])
```

### Advanced Augmentation: AutoAugment and RandAugment

```python
# AutoAugment — learned augmentation policy (from Google)
transform = transforms.Compose([
    transforms.AutoAugment(transforms.AutoAugmentPolicy.IMAGENET),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
])

# RandAugment — simplified random augmentation (recommended!)
transform = transforms.Compose([
    transforms.RandAugment(num_ops=2, magnitude=9),  # Apply 2 random transforms
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
])

# TrivialAugmentWide — even simpler, one random transform at random magnitude
transform = transforms.Compose([
    transforms.TrivialAugmentWide(),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
])
```

### MixUp and CutMix (Batch-Level Augmentation)

```python
# MixUp: Blend two images and their labels
def mixup_data(x, y, alpha=0.2):
    """
    MixUp: x_new = λ*x_i + (1-λ)*x_j
           y_new = λ*y_i + (1-λ)*y_j
    """
    lam = torch.distributions.Beta(alpha, alpha).sample().item()
    batch_size = x.size(0)
    index = torch.randperm(batch_size, device=x.device)
    
    mixed_x = lam * x + (1 - lam) * x[index]
    y_a, y_b = y, y[index]
    
    return mixed_x, y_a, y_b, lam

def mixup_criterion(criterion, pred, y_a, y_b, lam):
    """MixUp loss: weighted combination of both label losses."""
    return lam * criterion(pred, y_a) + (1 - lam) * criterion(pred, y_b)

# Usage in training loop
for inputs, targets in train_loader:
    inputs, targets = inputs.to(device), targets.to(device)
    
    # Apply MixUp
    mixed_inputs, targets_a, targets_b, lam = mixup_data(inputs, targets)
    
    outputs = model(mixed_inputs)
    loss = mixup_criterion(criterion, outputs, targets_a, targets_b, lam)
    
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
```

---

## 7. Working with Different Data Types {#7-working-with-different-data-types}

### Multi-Input Dataset (Image + Metadata)

```python
class MultiModalDataset(Dataset):
    """Dataset that returns image + tabular metadata."""
    
    def __init__(self, img_dir, metadata_df, transform=None):
        self.img_dir = img_dir
        self.metadata = metadata_df
        self.transform = transform
    
    def __len__(self):
        return len(self.metadata)
    
    def __getitem__(self, idx):
        row = self.metadata.iloc[idx]
        
        # Load image
        img_path = os.path.join(self.img_dir, row['filename'])
        image = Image.open(img_path).convert('RGB')
        if self.transform:
            image = self.transform(image)
        
        # Get tabular features
        features = torch.tensor(
            [row['age'], row['weight'], row['height']],
            dtype=torch.float32
        )
        
        # Get label
        label = torch.tensor(row['label'], dtype=torch.long)
        
        return image, features, label

# Custom collate for multiple inputs
def multi_collate(batch):
    images, features, labels = zip(*batch)
    images = torch.stack(images)
    features = torch.stack(features)
    labels = torch.stack(labels)
    return images, features, labels

loader = DataLoader(dataset, batch_size=32, collate_fn=multi_collate)
```

### Large File Handling (HDF5 / LMDB)

```python
import h5py

class HDF5Dataset(Dataset):
    """Memory-efficient dataset for large data stored in HDF5."""
    
    def __init__(self, h5_path, transform=None):
        self.h5_path = h5_path
        self.transform = transform
        
        # Read metadata only (don't load data into memory)
        with h5py.File(h5_path, 'r') as f:
            self.length = len(f['images'])
    
    def __len__(self):
        return self.length
    
    def __getitem__(self, idx):
        # Open file each time (HDF5 files are NOT fork-safe)
        # This is necessary for num_workers > 0
        with h5py.File(self.h5_path, 'r') as f:
            image = f['images'][idx]   # Lazy loading from disk
            label = f['labels'][idx]
        
        image = torch.tensor(image, dtype=torch.float32)
        if self.transform:
            image = self.transform(image)
        
        return image, torch.tensor(label, dtype=torch.long)
```

---

## 8. Performance Optimization {#8-performance-optimization}

### Data Loading Bottleneck Detection

```python
import time

def benchmark_dataloader(loader, num_batches=100):
    """Measure data loading speed."""
    start = time.time()
    for i, batch in enumerate(loader):
        if i >= num_batches:
            break
    elapsed = time.time() - start
    print(f"Loaded {num_batches} batches in {elapsed:.2f}s "
          f"({num_batches/elapsed:.1f} batches/sec)")

# Compare different num_workers
for nw in [0, 2, 4, 8]:
    loader = DataLoader(dataset, batch_size=64, num_workers=nw, pin_memory=True)
    print(f"num_workers={nw}: ", end="")
    benchmark_dataloader(loader)
```

### Performance Tips

| Tip | Impact | Details |
|-----|--------|---------|
| `pin_memory=True` | 2-3x faster GPU transfer | Always use with GPU training |
| `num_workers=4` | 2-5x faster loading | Start with 4, tune up to CPU count |
| `persistent_workers=True` | Faster epoch transitions | Avoids worker restart between epochs |
| `prefetch_factor=2` | Smoother training | Pre-loads next batches in background |
| `non_blocking=True` | Overlaps transfer with compute | Use with `pin_memory=True` |
| Pre-resize images | Less I/O, faster transforms | Save resized versions to disk |
| Cache in memory | Fastest access | Only if data fits in RAM |
| Use SSD over HDD | 10-100x faster I/O | Especially for many small files |

```python
# Optimal DataLoader for GPU training
train_loader = DataLoader(
    train_dataset,
    batch_size=64,
    shuffle=True,
    num_workers=4,
    pin_memory=True,
    persistent_workers=True,
    prefetch_factor=2,
    drop_last=True,  # Avoid BatchNorm issues with tiny last batch
)
```

### In-Memory Caching

```python
class CachedDataset(Dataset):
    """Cache all samples in memory after first load."""
    
    def __init__(self, base_dataset):
        self.base_dataset = base_dataset
        self.cache = {}
    
    def __len__(self):
        return len(self.base_dataset)
    
    def __getitem__(self, idx):
        if idx not in self.cache:
            self.cache[idx] = self.base_dataset[idx]
        return self.cache[idx]

# Usage (only for small-medium datasets that fit in RAM)
cached_dataset = CachedDataset(original_dataset)
```

---

## 9. Common Mistakes {#9-common-mistakes}

| Mistake | Problem | Fix |
|---------|---------|-----|
| Applying augmentation to val/test | Inconsistent evaluation results | Use separate transforms for train and val |
| `ToTensor()` on already-tensor data | Double conversion or wrong scaling | Only use on PIL Images or numpy arrays |
| `num_workers > 0` on Windows without `if __name__ == '__main__'` | Multiprocessing crash | Wrap main code in `if __name__ == '__main__':` |
| Forgetting `shuffle=True` for training | Model learns data order patterns | Always shuffle training data |
| `shuffle=True` with custom `sampler` | Raises error (mutually exclusive) | Use one or the other |
| Not normalizing data | Slower/worse convergence | Always normalize (use dataset-specific stats) |
| Loading entire dataset into `__init__` | RAM overflow for large datasets | Use lazy loading in `__getitem__` |
| Wrong normalization values | Degraded model performance | Use the correct mean/std for your data |
| Not using `drop_last=True` with BatchNorm | BatchNorm fails with batch_size=1 | Set `drop_last=True` in training loader |
| Forgetting `.convert('RGB')` for images | Grayscale/RGBA causes channel mismatch | Always convert to expected format |

---

## 10. Interview Questions {#10-interview-questions}

**Q1: What's the difference between `Dataset` and `DataLoader`?**
> `Dataset` defines how to access individual samples (implements `__getitem__` and `__len__`). `DataLoader` wraps a dataset and provides batching, shuffling, parallel loading, and memory pinning. Think of `Dataset` as the cookbook and `DataLoader` as the chef who prepares batches.

**Q2: Why does `ToTensor()` scale values to [0, 1]?**
> Neural networks work better with normalized inputs. `ToTensor()` converts uint8 pixel values (0-255) to float32 (0.0-1.0) by dividing by 255. This is a convenience — it avoids a separate normalization step for the basic scaling.

**Q3: What is `pin_memory` and when should you use it?**
> Pinned (page-locked) memory allows direct DMA transfer to GPU, bypassing the CPU pageable memory staging area. Use `pin_memory=True` whenever training on GPU. Combined with `.to(device, non_blocking=True)`, it allows CPU→GPU transfers to overlap with computation.

**Q4: How do you handle class imbalance in data loading?**
> Three approaches: (1) `WeightedRandomSampler` to oversample minority classes, (2) `weight` parameter in loss functions like `CrossEntropyLoss`, (3) Data augmentation specifically for minority classes. Use WeightedRandomSampler when you want balanced batches without modifying the loss.

**Q5: Why can't you use `shuffle=True` with `IterableDataset`?**
> `IterableDataset` produces data sequentially via `__iter__`, with no random access (`__getitem__`). Shuffling requires random access. For iterable datasets, implement shuffling within the iterator (e.g., buffer-based shuffling).

**Q6: What happens if you use data augmentation during validation?**
> Validation results become non-deterministic and unreliable. Each evaluation would give different results because augmentation is random. Validation must use deterministic preprocessing (resize, center crop, normalize) for consistent and comparable metrics.

**Q7: How do you choose `num_workers`?**
> Start with `num_workers=4`. If data loading is the bottleneck (GPU utilization < 90%), increase up to `os.cpu_count()`. Too many workers cause overhead from process creation, memory usage, and inter-process communication. Profile with different values.

---

## 11. Quick Reference {#11-quick-reference}

### DataLoader Cheat Sheet

```python
# Standard training loader
DataLoader(dataset, batch_size=64, shuffle=True, num_workers=4,
           pin_memory=True, drop_last=True, persistent_workers=True)

# Standard validation loader
DataLoader(dataset, batch_size=128, shuffle=False, num_workers=4,
           pin_memory=True, drop_last=False)
```

### Transform Pipeline Templates

```python
# ImageNet-style training
transforms.Compose([
    transforms.RandomResizedCrop(224),
    transforms.RandomHorizontalFlip(),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
])

# ImageNet-style validation
transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
])
```

### Custom Dataset Template

```python
class MyDataset(Dataset):
    def __init__(self, data_path, transform=None):
        self.data_path = data_path
        self.transform = transform
        self.samples = self._load_metadata()
    
    def _load_metadata(self):
        # Load file paths, labels, etc.
        return [...]
    
    def __len__(self):
        return len(self.samples)
    
    def __getitem__(self, idx):
        # Load single sample (lazy loading)
        data, label = self._load_sample(self.samples[idx])
        if self.transform:
            data = self.transform(data)
        return data, label
```

### Key Imports

```python
from torch.utils.data import (
    Dataset, DataLoader, TensorDataset,
    random_split, Subset, ConcatDataset,
    IterableDataset, WeightedRandomSampler
)
import torchvision.transforms as transforms
from torchvision.transforms import v2  # Modern API
import torchvision.datasets as datasets
```

---

*Previous: [03-Training-Loop-and-Optimization](03-Training-Loop-and-Optimization.md) | Next: [05-CNN-with-PyTorch](05-CNN-with-PyTorch.md)*
