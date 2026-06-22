# Chapter 07: Transfer Learning with PyTorch

## Table of Contents
- [What is Transfer Learning?](#what-is-transfer-learning)
- [Why Transfer Learning Matters](#why-transfer-learning-matters)
- [How Transfer Learning Works](#how-transfer-learning-works)
- [Pretrained Models in torchvision](#pretrained-models-in-torchvision)
- [Feature Extraction](#feature-extraction)
- [Fine-Tuning](#fine-tuning)
- [Advanced Fine-Tuning Strategies](#advanced-fine-tuning-strategies)
- [Domain-Specific Transfer Learning](#domain-specific-transfer-learning)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Transfer Learning?

**Simple Explanation:** Imagine you learned to ride a bicycle as a kid. When you later try to ride a motorcycle, you don't start from zero — you already know about balance, steering, and braking. Transfer learning is the same idea for neural networks: take knowledge learned from one task and apply it to a different (but related) task.

**Formal Definition:** Transfer learning is a machine learning technique where a model trained on a large dataset (source domain) is reused as the starting point for a model on a different dataset (target domain).

```
┌─────────────────────────────────────────────────────────┐
│              TRANSFER LEARNING CONCEPT                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Source Task (ImageNet)      Target Task (Your Data)     │
│  ┌──────────────────┐       ┌──────────────────┐       │
│  │ 1.2M images      │       │ 500 images       │       │
│  │ 1000 classes     │  ───► │ 5 classes        │       │
│  │ General features │       │ Specific features │       │
│  └──────────────────┘       └──────────────────┘       │
│                                                          │
│  What transfers: edges, textures, shapes, patterns       │
│  What's new: task-specific decision boundaries           │
└─────────────────────────────────────────────────────────┘
```

---

## Why Transfer Learning Matters

### Real-World Relevance

| Problem | Without Transfer Learning | With Transfer Learning |
|---------|--------------------------|----------------------|
| Data needed | 100K+ images | 100-1000 images |
| Training time | Days/weeks on GPU | Hours on GPU |
| Accuracy | Often poor | State-of-the-art |
| Cost | $1000s in compute | $10-50 in compute |
| Expertise needed | High (architecture design) | Moderate |

### When to Use Transfer Learning

1. **Limited labeled data** — You have < 10K samples
2. **Similar domain** — Your task relates to the pretrained model's domain
3. **Time/budget constraints** — Can't afford to train from scratch
4. **State-of-the-art performance** — Pretrained models encode years of research
5. **Production deployment** — Need reliable, tested architectures

### When NOT to Use Transfer Learning

- Your domain is radically different (e.g., medical spectrograms vs. natural images)
- You have massive amounts of labeled data (100M+)
- The pretrained model's architecture doesn't fit your constraints (mobile/edge)
- You need full control over every layer for interpretability

---

## How Transfer Learning Works

### The Feature Hierarchy

Neural networks learn hierarchical features. Earlier layers learn general features; later layers learn task-specific features:

```
┌─────────────────────────────────────────────────────────────┐
│                    CNN FEATURE HIERARCHY                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Layer 1-2:  Edges, colors, gradients        [VERY GENERAL] │
│              ░░▓▓ ──── ╱╲ ║║                                │
│                                                              │
│  Layer 3-5:  Textures, patterns, corners     [GENERAL]      │
│              ╔══╗ ┌┐┌┐ ⊞⊞⊞                                 │
│                                                              │
│  Layer 6-10: Parts (eyes, wheels, leaves)    [SEMI-SPECIFIC]│
│              👁️ ⚙️ 🍃                                        │
│                                                              │
│  Layer 11+:  Full objects, scenes            [TASK-SPECIFIC]│
│              🐕 🚗 🌳                                        │
│                                                              │
│  Final FC:   Class probabilities             [VERY SPECIFIC]│
│              [0.1, 0.7, 0.05, ...]                          │
└─────────────────────────────────────────────────────────────┘
```

### Two Main Approaches

```
┌──────────────────────────────────┬──────────────────────────────────┐
│       FEATURE EXTRACTION         │          FINE-TUNING             │
├──────────────────────────────────┼──────────────────────────────────┤
│                                  │                                  │
│  Freeze ALL pretrained layers    │  Unfreeze SOME/ALL layers        │
│  Only train new classifier head  │  Train with small learning rate  │
│                                  │                                  │
│  ┌────────┐  FROZEN             │  ┌────────┐  TRAINABLE (small lr)│
│  │ Conv1  │  ████████           │  │ Conv1  │  ░░░░░░░░           │
│  │ Conv2  │  ████████           │  │ Conv2  │  ░░░░░░░░           │
│  │ Conv3  │  ████████           │  │ Conv3  │  ░░░░░░░░           │
│  │ Conv4  │  ████████           │  │ Conv4  │  ▓▓▓▓▓▓▓▓           │
│  │ Conv5  │  ████████           │  │ Conv5  │  ▓▓▓▓▓▓▓▓           │
│  ├────────┤                     │  ├────────┤                      │
│  │ New FC │  ░░░░░░░░ TRAINABLE │  │ New FC │  ░░░░░░░░ TRAINABLE │
│  └────────┘                     │  └────────┘                      │
│                                  │                                  │
│  Best when:                      │  Best when:                      │
│  - Very small dataset            │  - Medium dataset (1K-100K)      │
│  - Similar to source domain      │  - Slightly different domain     │
│  - Limited compute               │  - Need higher accuracy          │
└──────────────────────────────────┴──────────────────────────────────┘
```

---

## Pretrained Models in torchvision

### Available Model Families

```python
import torchvision.models as models

# List all available models
print(dir(models))
```

| Model | Params | Top-1 Acc (ImageNet) | Speed | Best For |
|-------|--------|---------------------|-------|----------|
| ResNet-18 | 11.7M | 69.8% | Very Fast | Prototyping, mobile |
| ResNet-50 | 25.6M | 76.1% | Fast | General purpose |
| ResNet-152 | 60.2M | 78.3% | Slow | High accuracy needed |
| VGG-16 | 138M | 71.6% | Slow | Feature extraction |
| EfficientNet-B0 | 5.3M | 77.1% | Fast | Mobile/edge |
| EfficientNet-B7 | 66M | 84.3% | Very Slow | Maximum accuracy |
| ViT-B/16 | 86M | 81.1% | Medium | Modern tasks |
| ConvNeXt-Tiny | 28.6M | 82.1% | Medium | Best CNN alternative |

### Loading Pretrained Models (Modern API — PyTorch 2.0+)

```python
import torch
import torchvision.models as models
from torchvision.models import ResNet50_Weights

# Modern way (recommended) — uses new Weights enum
model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)

# Access the transforms that match the pretrained model
preprocess = ResNet50_Weights.IMAGENET1K_V2.transforms()

# Legacy way (deprecated but still works)
# model = models.resnet50(pretrained=True)  # Don't use this

# No pretrained weights (random initialization)
model = models.resnet50(weights=None)

print(f"Model parameters: {sum(p.numel() for p in model.parameters()):,}")
# Output: Model parameters: 25,557,032
```

### Inspecting Model Architecture

```python
import torch
import torchvision.models as models
from torchvision.models import ResNet50_Weights

model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)

# See the full architecture
print(model)

# See named children (top-level modules)
for name, module in model.named_children():
    print(f"{name}: {module.__class__.__name__}")
    
# Output:
# conv1: Conv2d
# bn1: BatchNorm2d
# relu: ReLU
# maxpool: MaxPool2d
# layer1: Sequential
# layer2: Sequential
# layer3: Sequential
# layer4: Sequential
# avgpool: AdaptiveAvgPool2d
# fc: Linear

# Check the final classifier
print(model.fc)
# Output: Linear(in_features=2048, out_features=1000, bias=True)
```

---

## Feature Extraction

### Concept

In feature extraction, we **freeze** all pretrained layers and only train a new classifier head. The pretrained network acts as a fixed feature extractor.

### Complete Feature Extraction Example

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader
from torchvision import datasets, transforms, models
from torchvision.models import ResNet50_Weights
import time

# ============================================================
# STEP 1: Define transforms matching the pretrained model
# ============================================================

# The pretrained model expects specific normalization
# ImageNet mean and std
IMAGENET_MEAN = [0.485, 0.456, 0.406]
IMAGENET_STD = [0.229, 0.224, 0.225]

train_transforms = transforms.Compose([
    transforms.RandomResizedCrop(224),      # Random crop to 224x224
    transforms.RandomHorizontalFlip(),       # Data augmentation
    transforms.ColorJitter(0.2, 0.2, 0.2),  # Color augmentation
    transforms.ToTensor(),                    # Convert to tensor [0, 1]
    transforms.Normalize(IMAGENET_MEAN, IMAGENET_STD)  # Normalize to ImageNet stats
])

val_transforms = transforms.Compose([
    transforms.Resize(256),                  # Resize shortest side to 256
    transforms.CenterCrop(224),              # Center crop to 224x224
    transforms.ToTensor(),
    transforms.Normalize(IMAGENET_MEAN, IMAGENET_STD)
])

# ============================================================
# STEP 2: Load your custom dataset
# ============================================================

# Assuming data is organized as:
# data/train/class1/, data/train/class2/, ...
# data/val/class1/, data/val/class2/, ...

train_dataset = datasets.ImageFolder('data/train', transform=train_transforms)
val_dataset = datasets.ImageFolder('data/val', transform=val_transforms)

train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True, num_workers=4)
val_loader = DataLoader(val_dataset, batch_size=32, shuffle=False, num_workers=4)

num_classes = len(train_dataset.classes)
print(f"Classes: {train_dataset.classes}")
print(f"Number of classes: {num_classes}")

# ============================================================
# STEP 3: Load pretrained model and freeze layers
# ============================================================

model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)

# FREEZE all parameters — this is the key step!
for param in model.parameters():
    param.requires_grad = False

# REPLACE the final fully connected layer
# The original fc outputs 1000 classes (ImageNet)
# We need it to output our number of classes
model.fc = nn.Sequential(
    nn.Linear(2048, 512),        # Reduce dimensions
    nn.ReLU(),
    nn.Dropout(0.3),             # Regularization
    nn.Linear(512, num_classes)  # Output layer
)

# New layers have requires_grad=True by default
# Verify: only new fc layer parameters are trainable
trainable_params = sum(p.numel() for p in model.parameters() if p.requires_grad)
total_params = sum(p.numel() for p in model.parameters())
print(f"Trainable: {trainable_params:,} / {total_params:,} "
      f"({100*trainable_params/total_params:.1f}%)")
# Output: Trainable: 1,049,600 / 24,607,552 (4.3%)

# ============================================================
# STEP 4: Setup training
# ============================================================

device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = model.to(device)

# Only optimize the parameters that require gradients
optimizer = optim.Adam(model.fc.parameters(), lr=0.001)
criterion = nn.CrossEntropyLoss()
scheduler = optim.lr_scheduler.StepLR(optimizer, step_size=5, gamma=0.1)

# ============================================================
# STEP 5: Training loop
# ============================================================

def train_one_epoch(model, loader, criterion, optimizer, device):
    model.train()
    running_loss = 0.0
    correct = 0
    total = 0
    
    for images, labels in loader:
        images, labels = images.to(device), labels.to(device)
        
        optimizer.zero_grad()
        outputs = model(images)
        loss = criterion(outputs, labels)
        loss.backward()
        optimizer.step()
        
        running_loss += loss.item() * images.size(0)
        _, predicted = outputs.max(1)
        total += labels.size(0)
        correct += predicted.eq(labels).sum().item()
    
    epoch_loss = running_loss / total
    epoch_acc = correct / total
    return epoch_loss, epoch_acc

def evaluate(model, loader, criterion, device):
    model.eval()
    running_loss = 0.0
    correct = 0
    total = 0
    
    with torch.no_grad():  # No gradient computation during evaluation
        for images, labels in loader:
            images, labels = images.to(device), labels.to(device)
            outputs = model(images)
            loss = criterion(outputs, labels)
            
            running_loss += loss.item() * images.size(0)
            _, predicted = outputs.max(1)
            total += labels.size(0)
            correct += predicted.eq(labels).sum().item()
    
    epoch_loss = running_loss / total
    epoch_acc = correct / total
    return epoch_loss, epoch_acc

# Train for a few epochs (feature extraction converges fast!)
num_epochs = 10
for epoch in range(num_epochs):
    train_loss, train_acc = train_one_epoch(model, train_loader, criterion, optimizer, device)
    val_loss, val_acc = evaluate(model, val_loader, criterion, device)
    scheduler.step()
    
    print(f"Epoch {epoch+1}/{num_epochs} | "
          f"Train Loss: {train_loss:.4f} Acc: {train_acc:.4f} | "
          f"Val Loss: {val_loss:.4f} Acc: {val_acc:.4f}")
```

### Pro Tip: Precompute Features for Speed

```python
import torch
import torch.nn as nn
from torchvision import models
from torchvision.models import ResNet50_Weights
from torch.utils.data import DataLoader, TensorDataset

# When doing feature extraction, the frozen backbone computes
# the SAME features every epoch. Precompute them once!

model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)
model.eval()

# Remove the final FC layer — keep everything else
feature_extractor = nn.Sequential(*list(model.children())[:-1])
feature_extractor = feature_extractor.to(device)

# Extract features once
def extract_features(loader, extractor, device):
    """Extract features from all images in the loader."""
    all_features = []
    all_labels = []
    
    with torch.no_grad():
        for images, labels in loader:
            images = images.to(device)
            features = extractor(images)
            features = features.squeeze()  # Remove spatial dims: (B, 2048, 1, 1) → (B, 2048)
            all_features.append(features.cpu())
            all_labels.append(labels)
    
    return torch.cat(all_features), torch.cat(all_labels)

# Precompute (run once)
train_features, train_labels = extract_features(train_loader, feature_extractor, device)
val_features, val_labels = extract_features(val_loader, feature_extractor, device)

# Now train only the classifier — this is MUCH faster
# No need to run images through the full CNN each epoch
feature_train_loader = DataLoader(
    TensorDataset(train_features, train_labels),
    batch_size=256, shuffle=True  # Can use much larger batch size!
)

# Simple classifier
classifier = nn.Sequential(
    nn.Linear(2048, 512),
    nn.ReLU(),
    nn.Dropout(0.3),
    nn.Linear(512, num_classes)
).to(device)

# Training is now 10-50x faster!
```

---

## Fine-Tuning

### Concept

In fine-tuning, we **unfreeze** some or all pretrained layers and train them with a very small learning rate. This allows the model to adapt its learned features to the new domain.

### Why Small Learning Rate?

The pretrained weights are already near a good solution. Large learning rates would destroy the useful features learned from millions of images.

$$\text{Fine-tuning LR} \approx \frac{\text{Training from scratch LR}}{10} \text{ to } \frac{1}{100}$$

### Complete Fine-Tuning Example

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import models
from torchvision.models import ResNet50_Weights

# ============================================================
# STEP 1: Load and modify the model
# ============================================================

model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)

# Replace the classifier (same as feature extraction)
num_classes = 10
model.fc = nn.Sequential(
    nn.Linear(2048, 512),
    nn.ReLU(),
    nn.Dropout(0.3),
    nn.Linear(512, num_classes)
)

# ============================================================
# STEP 2: Use DIFFERENT learning rates for different layers
# ============================================================

# This is the KEY technique for fine-tuning!
# Pretrained layers get a small LR, new layers get a larger LR

# Group parameters by layer depth
backbone_params = []
classifier_params = []

for name, param in model.named_parameters():
    if 'fc' in name:
        classifier_params.append(param)
    else:
        backbone_params.append(param)

# Discriminative learning rates (differential LR)
optimizer = optim.Adam([
    {'params': backbone_params, 'lr': 1e-5},    # Pretrained: very small LR
    {'params': classifier_params, 'lr': 1e-3},  # New layers: normal LR
], weight_decay=1e-4)

print(f"Backbone params: {sum(p.numel() for p in backbone_params):,}")
print(f"Classifier params: {sum(p.numel() for p in classifier_params):,}")
```

### Gradual Unfreezing (Layer-by-Layer Fine-Tuning)

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import models
from torchvision.models import ResNet50_Weights

model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)
num_classes = 10
model.fc = nn.Linear(2048, num_classes)

# ============================================================
# Strategy: Gradually unfreeze layers during training
# ============================================================

def freeze_all(model):
    """Freeze all layers except the classifier."""
    for param in model.parameters():
        param.requires_grad = False
    for param in model.fc.parameters():
        param.requires_grad = True

def unfreeze_layer(model, layer_name):
    """Unfreeze a specific layer."""
    for name, param in model.named_parameters():
        if layer_name in name:
            param.requires_grad = True

# Phase 1: Train only classifier (epochs 1-3)
freeze_all(model)
# Train...

# Phase 2: Unfreeze layer4 (epochs 4-6)
unfreeze_layer(model, 'layer4')
# Reduce LR and train...

# Phase 3: Unfreeze layer3 (epochs 7-9)
unfreeze_layer(model, 'layer3')
# Reduce LR further and train...

# Phase 4: Unfreeze everything (epochs 10-15)
for param in model.parameters():
    param.requires_grad = True
# Use very small LR and train...

# ============================================================
# Complete Gradual Unfreezing Implementation
# ============================================================

def gradual_unfreeze_training(model, train_loader, val_loader, device, num_classes):
    """Full implementation of gradual unfreezing."""
    
    criterion = nn.CrossEntropyLoss()
    
    # Define unfreezing schedule
    schedule = [
        # (layers_to_unfreeze, learning_rate, epochs)
        (['fc'], 1e-3, 3),
        (['layer4', 'fc'], 3e-4, 3),
        (['layer3', 'layer4', 'fc'], 1e-4, 3),
        (None, 3e-5, 5),  # None means unfreeze all
    ]
    
    for phase, (layers, lr, epochs) in enumerate(schedule):
        print(f"\n{'='*50}")
        print(f"Phase {phase+1}: Unfreezing {layers or 'ALL'}, LR={lr}")
        print(f"{'='*50}")
        
        # Freeze everything first
        for param in model.parameters():
            param.requires_grad = False
        
        # Unfreeze specified layers
        if layers is None:
            for param in model.parameters():
                param.requires_grad = True
        else:
            for name, param in model.named_parameters():
                if any(layer in name for layer in layers):
                    param.requires_grad = True
        
        # Count trainable params
        trainable = sum(p.numel() for p in model.parameters() if p.requires_grad)
        print(f"Trainable parameters: {trainable:,}")
        
        # New optimizer for this phase
        optimizer = optim.Adam(
            filter(lambda p: p.requires_grad, model.parameters()),
            lr=lr, weight_decay=1e-4
        )
        
        # Train for specified epochs
        for epoch in range(epochs):
            # ... training loop here ...
            pass
    
    return model
```

### Discriminative Learning Rates (Per-Layer LR)

```python
import torch.optim as optim
from torchvision import models
from torchvision.models import ResNet50_Weights

model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)

# Advanced: different LR for each layer group
# Earlier layers → smaller LR (more general features)
# Later layers → larger LR (more task-specific features)

param_groups = [
    {'params': model.conv1.parameters(), 'lr': 1e-6},
    {'params': model.bn1.parameters(), 'lr': 1e-6},
    {'params': model.layer1.parameters(), 'lr': 5e-6},
    {'params': model.layer2.parameters(), 'lr': 1e-5},
    {'params': model.layer3.parameters(), 'lr': 5e-5},
    {'params': model.layer4.parameters(), 'lr': 1e-4},
    {'params': model.fc.parameters(), 'lr': 1e-3},
]

optimizer = optim.AdamW(param_groups, weight_decay=0.01)
```

---

## Advanced Fine-Tuning Strategies

### 1. Learning Rate Finder

```python
import torch
import torch.nn as nn
import matplotlib.pyplot as plt

def find_lr(model, train_loader, criterion, optimizer, device,
            start_lr=1e-7, end_lr=1, num_steps=100):
    """
    Find optimal learning rate using the LR range test.
    Gradually increase LR and track loss.
    The optimal LR is where loss decreases fastest.
    """
    lrs = []
    losses = []
    
    # Save model state
    model_state = model.state_dict().copy()
    optimizer_state = optimizer.state_dict().copy()
    
    # Exponential LR increase
    lr_mult = (end_lr / start_lr) ** (1 / num_steps)
    lr = start_lr
    
    model.train()
    iterator = iter(train_loader)
    
    for step in range(num_steps):
        # Set learning rate
        for param_group in optimizer.param_groups:
            param_group['lr'] = lr
        
        # Get batch
        try:
            images, labels = next(iterator)
        except StopIteration:
            iterator = iter(train_loader)
            images, labels = next(iterator)
        
        images, labels = images.to(device), labels.to(device)
        
        # Forward pass
        optimizer.zero_grad()
        outputs = model(images)
        loss = criterion(outputs, labels)
        
        # Stop if loss explodes
        if step > 0 and loss.item() > 4 * losses[0]:
            break
        
        lrs.append(lr)
        losses.append(loss.item())
        
        # Backward pass
        loss.backward()
        optimizer.step()
        
        lr *= lr_mult
    
    # Restore model state
    model.load_state_dict(model_state)
    optimizer.load_state_dict(optimizer_state)
    
    # Plot
    plt.figure(figsize=(10, 6))
    plt.semilogx(lrs, losses)
    plt.xlabel('Learning Rate')
    plt.ylabel('Loss')
    plt.title('Learning Rate Finder')
    plt.grid(True)
    plt.show()
    
    # Return the LR where loss decreases fastest
    # (steepest negative slope)
    min_loss_idx = losses.index(min(losses))
    suggested_lr = lrs[min_loss_idx] / 10  # Divide by 10 for safety
    print(f"Suggested LR: {suggested_lr:.2e}")
    return suggested_lr
```

### 2. Cosine Annealing with Warm Restarts

```python
import torch.optim as optim
from torch.optim.lr_scheduler import CosineAnnealingWarmRestarts, OneCycleLR

# Cosine Annealing — smoothly decreases LR, then restarts
scheduler = CosineAnnealingWarmRestarts(
    optimizer,
    T_0=10,      # First restart after 10 epochs
    T_mult=2,    # Double the period after each restart
    eta_min=1e-7 # Minimum learning rate
)

# OneCycleLR — best for fine-tuning (from fastai research)
scheduler = OneCycleLR(
    optimizer,
    max_lr=1e-3,           # Peak learning rate
    epochs=20,
    steps_per_epoch=len(train_loader),
    pct_start=0.3,         # 30% warmup
    anneal_strategy='cos', # Cosine annealing
    div_factor=25,         # start_lr = max_lr / 25
    final_div_factor=1000  # end_lr = max_lr / (25 * 1000)
)
```

### 3. Mixup and CutMix Augmentation for Fine-Tuning

```python
import torch
import numpy as np

def mixup_data(x, y, alpha=0.2):
    """
    Mixup: blend two random training examples.
    Creates virtual training examples between classes.
    
    Formula: x_new = λ*x_i + (1-λ)*x_j
             y_new = λ*y_i + (1-λ)*y_j
    """
    if alpha > 0:
        lam = np.random.beta(alpha, alpha)
    else:
        lam = 1.0
    
    batch_size = x.size(0)
    index = torch.randperm(batch_size).to(x.device)
    
    mixed_x = lam * x + (1 - lam) * x[index]
    y_a, y_b = y, y[index]
    
    return mixed_x, y_a, y_b, lam

def mixup_criterion(criterion, pred, y_a, y_b, lam):
    """Mixup loss."""
    return lam * criterion(pred, y_a) + (1 - lam) * criterion(pred, y_b)

# Usage in training loop
for images, labels in train_loader:
    images, labels = images.to(device), labels.to(device)
    
    # Apply mixup
    mixed_images, labels_a, labels_b, lam = mixup_data(images, labels, alpha=0.2)
    
    outputs = model(mixed_images)
    loss = mixup_criterion(criterion, outputs, labels_a, labels_b, lam)
    
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
```

### 4. Knowledge Distillation from Larger Model

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DistillationLoss(nn.Module):
    """
    Combine hard labels (ground truth) with soft labels (teacher predictions).
    The teacher model's soft predictions contain "dark knowledge" about
    class similarities that hard labels miss.
    """
    def __init__(self, temperature=4.0, alpha=0.7):
        super().__init__()
        self.temperature = temperature
        self.alpha = alpha  # Weight for distillation loss vs hard loss
    
    def forward(self, student_logits, teacher_logits, labels):
        # Hard loss: standard cross-entropy with true labels
        hard_loss = F.cross_entropy(student_logits, labels)
        
        # Soft loss: KL divergence between teacher and student soft predictions
        soft_student = F.log_softmax(student_logits / self.temperature, dim=1)
        soft_teacher = F.softmax(teacher_logits / self.temperature, dim=1)
        soft_loss = F.kl_div(soft_student, soft_teacher, reduction='batchmean')
        soft_loss *= self.temperature ** 2  # Scale by T^2 as per Hinton et al.
        
        # Combined loss
        return self.alpha * soft_loss + (1 - self.alpha) * hard_loss

# Usage
teacher = models.resnet152(weights='IMAGENET1K_V2').to(device)
teacher.eval()  # Teacher is always in eval mode

student = models.resnet18(weights='IMAGENET1K_V1').to(device)
# Modify student for your task...

distill_loss = DistillationLoss(temperature=4.0, alpha=0.7)

for images, labels in train_loader:
    images, labels = images.to(device), labels.to(device)
    
    with torch.no_grad():
        teacher_logits = teacher(images)
    
    student_logits = student(images)
    loss = distill_loss(student_logits, teacher_logits, labels)
    
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
```

---

## Domain-Specific Transfer Learning

### Medical Imaging

```python
import torch
import torch.nn as nn
from torchvision import models
from torchvision.models import DenseNet121_Weights

# Medical imaging often uses DenseNet (good for X-rays)
model = models.densenet121(weights=DenseNet121_Weights.IMAGENET1K_V1)

# DenseNet uses a 'classifier' attribute (not 'fc')
num_features = model.classifier.in_features
model.classifier = nn.Sequential(
    nn.Linear(num_features, 512),
    nn.ReLU(),
    nn.Dropout(0.5),  # Higher dropout for medical (small datasets)
    nn.Linear(512, 14)  # e.g., 14 pathologies in CheXpert
)

# Medical imaging tips:
# 1. Use higher resolution (384x384 or 512x512)
# 2. Use heavier augmentation (rotation, elastic deformation)
# 3. Consider multi-label classification (patients can have multiple conditions)
# 4. Use weighted loss for class imbalance
# 5. Grayscale → repeat channels: img.repeat(3, 1, 1) for 1-channel inputs
```

### Multi-Label Classification

```python
import torch
import torch.nn as nn
from torchvision import models
from torchvision.models import ResNet50_Weights

model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)

# For multi-label: use sigmoid instead of softmax
num_labels = 20
model.fc = nn.Linear(2048, num_labels)  # No softmax! BCEWithLogitsLoss handles it

# Loss function for multi-label
criterion = nn.BCEWithLogitsLoss()  # Applies sigmoid internally

# During inference
model.eval()
with torch.no_grad():
    outputs = model(images)
    predictions = torch.sigmoid(outputs) > 0.5  # Threshold at 0.5
```

### Using Different Architectures

```python
import torch
import torch.nn as nn
from torchvision import models
from torchvision.models import (
    EfficientNet_B0_Weights,
    VGG16_Weights,
    ViT_B_16_Weights
)

# ============================================================
# EfficientNet (best accuracy/efficiency tradeoff)
# ============================================================
efficientnet = models.efficientnet_b0(weights=EfficientNet_B0_Weights.IMAGENET1K_V1)
# EfficientNet uses 'classifier' with a sequential structure
efficientnet.classifier[1] = nn.Linear(1280, num_classes)

# ============================================================
# VGG (simple, good for feature extraction)
# ============================================================
vgg = models.vgg16(weights=VGG16_Weights.IMAGENET1K_V1)
# VGG uses 'classifier' which is a Sequential of linear layers
vgg.classifier[6] = nn.Linear(4096, num_classes)

# ============================================================
# Vision Transformer (ViT)
# ============================================================
vit = models.vit_b_16(weights=ViT_B_16_Weights.IMAGENET1K_V1)
# ViT uses 'heads' attribute
vit.heads.head = nn.Linear(768, num_classes)

# ============================================================
# How to find the classifier layer for ANY model
# ============================================================
def find_classifier(model):
    """Print the last few layers to identify the classifier."""
    children = list(model.named_children())
    for name, layer in children[-3:]:
        print(f"{name}: {layer}")

find_classifier(efficientnet)
find_classifier(vgg)
find_classifier(vit)
```

---

## Common Mistakes

### 1. Forgetting to Freeze Layers

```python
# ❌ WRONG: All parameters are trainable, defeating the purpose
model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)
model.fc = nn.Linear(2048, num_classes)
# Forgot to freeze! All 25M params will train with same LR

# ✅ CORRECT: Freeze backbone first
model = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)
for param in model.parameters():
    param.requires_grad = False
model.fc = nn.Linear(2048, num_classes)
# Only ~2K params are trainable now
```

### 2. Wrong Input Normalization

```python
# ❌ WRONG: Using [0, 1] range without ImageNet normalization
transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),  # Only converts to [0, 1]
])

# ✅ CORRECT: Must normalize with ImageNet stats
transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225])
])
```

### 3. Not Setting Model to Eval Mode

```python
# ❌ WRONG: BatchNorm and Dropout behave differently in train mode
model.load_state_dict(torch.load('best_model.pth'))
predictions = model(test_images)  # Wrong! Still in train mode

# ✅ CORRECT: Always set eval mode for inference
model.load_state_dict(torch.load('best_model.pth'))
model.eval()
with torch.no_grad():
    predictions = model(test_images)
```

### 4. Learning Rate Too High for Fine-Tuning

```python
# ❌ WRONG: Same LR for pretrained and new layers
optimizer = optim.Adam(model.parameters(), lr=0.001)  # Too high for pretrained layers!

# ✅ CORRECT: Discriminative learning rates
optimizer = optim.Adam([
    {'params': model.layer4.parameters(), 'lr': 1e-5},
    {'params': model.fc.parameters(), 'lr': 1e-3},
])
```

### 5. Forgetting to Update BatchNorm Statistics

```python
# ❌ WRONG: BatchNorm layers are frozen and use ImageNet statistics
for param in model.parameters():
    param.requires_grad = False
# BN running_mean and running_var won't update!

# ✅ CORRECT: Keep BN layers in train mode or fine-tune them
# Option A: Keep BN trainable
for module in model.modules():
    if isinstance(module, nn.BatchNorm2d):
        for param in module.parameters():
            param.requires_grad = True

# Option B: Freeze BN but keep them in eval mode explicitly
def freeze_bn(model):
    for module in model.modules():
        if isinstance(module, nn.BatchNorm2d):
            module.eval()
            for param in module.parameters():
                param.requires_grad = False
```

### 6. Input Size Mismatch

```python
# ❌ WRONG: Using wrong input size
# ResNet expects 224x224, but some models expect different sizes
transforms.Resize((300, 300))  # Wrong for ResNet!

# ✅ CORRECT: Check model's expected input size
# ResNet, VGG, DenseNet: 224x224
# Inception v3: 299x299
# EfficientNet-B0: 224x224, B7: 600x600
# ViT-B/16: 224x224

# Best: Use the model's built-in transforms
from torchvision.models import EfficientNet_B0_Weights
preprocess = EfficientNet_B0_Weights.IMAGENET1K_V1.transforms()
```

---

## Interview Questions

### Q1: What's the difference between feature extraction and fine-tuning?

**Answer:**
- **Feature extraction**: Freeze all pretrained layers, only train a new classifier head. The pretrained model is a fixed feature extractor. Best for small datasets similar to the source domain.
- **Fine-tuning**: Unfreeze some/all pretrained layers and train with a small learning rate. Allows the model to adapt features to the new domain. Best for medium datasets or different domains.

### Q2: Why do we use a smaller learning rate for pretrained layers?

**Answer:** Pretrained weights are already near a good local minimum from training on millions of images. A large learning rate would destroy these carefully learned features by making large, random weight updates. A small LR allows gentle adaptation without catastrophic forgetting.

### Q3: What is catastrophic forgetting and how do you prevent it?

**Answer:** Catastrophic forgetting occurs when a model trained on a new task completely forgets what it learned from the original task. Prevention strategies:
1. Use small learning rates for pretrained layers
2. Gradual unfreezing (train classifier first, then unfreeze layers bottom-up)
3. Regularization (L2, dropout, early stopping)
4. Elastic Weight Consolidation (EWC) — adds a penalty for changing important weights
5. Knowledge distillation — use the original model as a teacher

### Q4: When would you choose ResNet vs. EfficientNet vs. ViT for transfer learning?

**Answer:**
| Model | Choose When |
|-------|------------|
| ResNet-18/34 | Fast inference needed, limited GPU, prototyping |
| ResNet-50/101 | General purpose, well-understood, reliable |
| EfficientNet | Best accuracy/FLOPs tradeoff, mobile deployment |
| ViT | Large datasets (>10K images), need global context |
| ConvNeXt | Modern CNN alternative to ViT, fewer data needed |

### Q5: How do you handle class imbalance in transfer learning?

**Answer:**
1. **Weighted loss**: `nn.CrossEntropyLoss(weight=class_weights)`
2. **Oversampling**: `WeightedRandomSampler` in DataLoader
3. **Data augmentation**: More augmentation for minority classes
4. **Focal loss**: Down-weights easy examples, focuses on hard ones
5. **Two-stage training**: First train on balanced subset, then fine-tune on full data

### Q6: Can you transfer learn across different modalities (e.g., natural images → medical images)?

**Answer:** Yes, but with caveats:
- Early layers (edges, textures) transfer well across most image domains
- Later layers may need more fine-tuning
- For very different domains (satellite, microscopy), consider:
  - Using models pretrained on domain-specific data if available
  - Fine-tuning more aggressively (more layers, higher LR)
  - Training longer

### Q7: What is the "linear probing then fine-tuning" (LP-FT) approach?

**Answer:** A two-phase approach that often outperforms direct fine-tuning:
1. **Phase 1 (Linear Probing)**: Freeze all layers, train only the classification head until convergence. This finds a good initialization for the head.
2. **Phase 2 (Fine-Tuning)**: Unfreeze all layers and fine-tune with a small LR. The good head initialization prevents early layers from being distorted.

This avoids the "feature distortion" problem where fine-tuning with a randomly initialized head sends bad gradients to pretrained layers.

---

## Quick Reference

### Decision Flowchart

```
┌─────────────────────────────────────┐
│  How much labeled data do you have? │
└──────────────────┬──────────────────┘
                   │
         ┌─────────┼──────────┐
         │         │          │
      < 1K      1K-50K      > 50K
         │         │          │
         ▼         ▼          ▼
   Feature      Fine-      Train from
   Extraction   Tuning     scratch (or
                            fine-tune)
```

### Cheat Sheet

| Task | Code |
|------|------|
| Load pretrained model | `models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)` |
| Freeze all layers | `for p in model.parameters(): p.requires_grad = False` |
| Replace classifier | `model.fc = nn.Linear(2048, num_classes)` |
| Discriminative LR | `optim.Adam([{'params': ..., 'lr': 1e-5}, {'params': ..., 'lr': 1e-3}])` |
| Check trainable params | `sum(p.numel() for p in model.parameters() if p.requires_grad)` |
| Get transforms | `ResNet50_Weights.IMAGENET1K_V2.transforms()` |
| Freeze BatchNorm | `module.eval()` for each BN module |

### Model Classifier Locations

| Model | Classifier Attribute | Input Features |
|-------|---------------------|----------------|
| ResNet | `model.fc` | 512 (18/34) or 2048 (50/101/152) |
| VGG | `model.classifier[6]` | 4096 |
| DenseNet | `model.classifier` | 1024 |
| EfficientNet | `model.classifier[1]` | 1280 (B0) |
| ViT | `model.heads.head` | 768 (B) |
| MobileNetV3 | `model.classifier[3]` | 1280 |
| ConvNeXt | `model.classifier[2]` | 768 (tiny) |

### Key Hyperparameters

| Parameter | Feature Extraction | Fine-Tuning |
|-----------|-------------------|-------------|
| Backbone LR | 0 (frozen) | 1e-5 to 1e-4 |
| Classifier LR | 1e-3 to 1e-2 | 1e-3 |
| Epochs | 5-15 | 15-50 |
| Batch size | 32-128 | 16-64 |
| Weight decay | 0 to 1e-4 | 1e-4 to 1e-2 |
| Augmentation | Light | Heavy |
| Dropout | 0.2-0.5 | 0.1-0.3 |
