# Chapter 09: Transfer Learning and Fine-Tuning

## Table of Contents
- [1. Introduction to Transfer Learning](#1-introduction-to-transfer-learning)
- [2. How Transfer Learning Works](#2-how-transfer-learning-works)
- [3. Pretrained Models Zoo](#3-pretrained-models-zoo)
- [4. Fine-Tuning Strategies](#4-fine-tuning-strategies)
- [5. Feature Extraction vs Fine-Tuning](#5-feature-extraction-vs-fine-tuning)
- [6. Domain Adaptation](#6-domain-adaptation)
- [7. Few-Shot and Zero-Shot Learning](#7-few-shot-and-zero-shot-learning)
- [8. Practical Implementation Guide](#8-practical-implementation-guide)
- [9. Advanced Techniques](#9-advanced-techniques)
- [10. Common Mistakes](#10-common-mistakes)
- [11. Interview Questions](#11-interview-questions)
- [12. Quick Reference](#12-quick-reference)

---

## 1. Introduction to Transfer Learning

### What It Is
**Transfer learning** means taking a model trained on one task (like classifying millions of images) and reusing its learned knowledge for a different but related task (like detecting diseases in X-rays). Instead of training from scratch, you start with a model that already "understands" visual patterns, language structure, etc.

**Analogy**: A chef who's mastered French cuisine can learn Italian cooking much faster than someone who's never cooked. They already understand knife skills, flavor combinations, heat control — the foundational knowledge transfers.

### Why It Matters
- **Less data required**: Works with 100-1000 images instead of millions
- **Faster training**: Converges in minutes/hours instead of days/weeks
- **Better performance**: Pretrained features are often superior to training from scratch
- **Accessible**: Use state-of-the-art models without massive compute budgets
- **Industry standard**: 90%+ of real-world deep learning uses transfer learning

### The Fundamental Insight

Deep networks learn **hierarchical features**:

```
Layer 1-2:    Edges, colors, textures       (UNIVERSAL — transfers everywhere)
Layer 3-5:    Shapes, patterns, parts        (GENERAL — transfers to most tasks)
Layer 6-8:    Object parts, compositions     (TASK-SPECIFIC — may need fine-tuning)
Final Layer:  Task-specific classes           (MUST be replaced for new task)

ImageNet Model:
┌─────────────────────────────────────────────────────────────┐
│  Edges → Textures → Parts → Objects → [1000 ImageNet classes]│
│  ◄──── Freeze These ────►  ◄── Fine-tune ──►  ◄─ Replace ─► │
└─────────────────────────────────────────────────────────────┘
                                    ↓
Your Task (e.g., Cat vs Dog):
┌─────────────────────────────────────────────────────────────┐
│  Edges → Textures → Parts → Objects → [2 classes: cat/dog]  │
└─────────────────────────────────────────────────────────────┘
```

Early layers learn generic features useful for almost any visual task. Later layers specialize.

---

## 2. How Transfer Learning Works

### Transfer Learning Taxonomy

```
                    Transfer Learning
                         │
           ┌─────────────┼─────────────┐
           │             │             │
    Feature Extraction  Fine-Tuning  Domain Adaptation
    (freeze backbone)  (unfreeze    (shift distributions)
                        partially)
           │             │             │
      Small dataset   Medium data   Different domain
      Similar domain  Similar task  Same task
```

### When Does Transfer Help?

| Source → Target | Strategy | Example |
|----------------|----------|---------|
| Large, similar domain | Feature extraction | ImageNet → Pet breeds |
| Large, different domain | Fine-tune deeply | ImageNet → Medical X-rays |
| Small, similar domain | Fine-tune top layers | Dogs → Wolves |
| Small, different domain | Fine-tune carefully with regularization | Photos → Sketches |

### The Transfer Learning Decision Matrix

```
                    Target dataset size
                    Small          Large
Source/Target    ┌────────────┬────────────┐
Similarity       │            │            │
Similar         │ Feature    │ Fine-tune  │
                │ Extraction │ all layers │
                ├────────────┼────────────┤
Different       │ Fine-tune  │ Fine-tune  │
                │ carefully  │ from       │
                │ (regularize)│ scratch?   │
                └────────────┴────────────┘
```

---

## 3. Pretrained Models Zoo

### Computer Vision Models

| Model | Year | Top-1 Acc (ImageNet) | Params | Best For |
|-------|------|---------------------|--------|----------|
| ResNet-50 | 2015 | 76.1% | 25M | General baseline |
| ResNet-152 | 2015 | 78.3% | 60M | Higher accuracy needs |
| EfficientNet-B0 | 2019 | 77.1% | 5.3M | Mobile/efficient |
| EfficientNet-B7 | 2019 | 84.3% | 66M | Maximum accuracy |
| ViT-Base | 2020 | 81.8% | 86M | When data is abundant |
| ConvNeXt-T | 2022 | 82.1% | 28M | Modern ConvNet |
| DINOv2 | 2023 | 86.3% | 1.1B | Self-supervised features |

### NLP Models

| Model | Year | Params | Best For |
|-------|------|--------|----------|
| BERT-base | 2018 | 110M | Classification, NER |
| RoBERTa | 2019 | 125M | Better BERT |
| GPT-2 | 2019 | 1.5B | Text generation |
| T5-base | 2019 | 220M | Any text-to-text task |
| LLaMA 2 | 2023 | 7-70B | General purpose |
| Mistral 7B | 2023 | 7B | Efficient inference |

### Where to Find Pretrained Models

```python
# PyTorch Hub / torchvision
import torchvision.models as models
model = models.resnet50(weights='IMAGENET1K_V2')

# Hugging Face (NLP + Vision + Audio)
from transformers import AutoModel
model = AutoModel.from_pretrained("bert-base-uncased")

# timm (PyTorch Image Models — 700+ architectures)
import timm
model = timm.create_model('efficientnet_b0', pretrained=True)
```

---

## 4. Fine-Tuning Strategies

### Strategy 1: Replace Final Layer Only (Feature Extraction)

Freeze entire backbone, only train a new classification head.

```python
import torch
import torch.nn as nn
import torchvision.models as models

# Load pretrained ResNet-50
model = models.resnet50(weights='IMAGENET1K_V2')

# Freeze ALL parameters
for param in model.parameters():
    param.requires_grad = False

# Replace final FC layer (originally 1000 classes → your task)
num_classes = 5  # e.g., 5 flower species
model.fc = nn.Sequential(
    nn.Linear(2048, 512),
    nn.ReLU(),
    nn.Dropout(0.3),
    nn.Linear(512, num_classes),
)
# Only the new layers have requires_grad=True by default
```

### Strategy 2: Gradual Unfreezing

Start frozen, then progressively unfreeze from top to bottom.

```python
import torch.optim as optim

class GradualUnfreezer:
    """Unfreeze layers progressively during training"""
    
    def __init__(self, model, unfreeze_schedule):
        """
        unfreeze_schedule: dict mapping epoch → layers to unfreeze
        Example: {0: ['fc'], 5: ['layer4'], 10: ['layer3'], 15: ['layer2']}
        """
        self.model = model
        self.schedule = unfreeze_schedule
        
        # Start with everything frozen
        for param in model.parameters():
            param.requires_grad = False
    
    def step(self, epoch):
        """Call at the start of each epoch"""
        if epoch in self.schedule:
            layers_to_unfreeze = self.schedule[epoch]
            for name, param in self.model.named_parameters():
                if any(layer in name for layer in layers_to_unfreeze):
                    param.requires_grad = True
                    print(f"  Unfreezing: {name}")

# Usage
model = models.resnet50(weights='IMAGENET1K_V2')
model.fc = nn.Linear(2048, 5)

unfreezer = GradualUnfreezer(model, {
    0: ['fc'],           # Epoch 0: only train classifier
    5: ['layer4'],       # Epoch 5: unfreeze last ResNet block
    10: ['layer3'],      # Epoch 10: unfreeze third block
    15: ['layer2'],      # Epoch 15: unfreeze second block
})
```

### Strategy 3: Discriminative Learning Rates

Different learning rates for different layers — lower for early layers, higher for later layers.

```python
def get_discriminative_lr_params(model, base_lr=1e-5, multiplier=2.5):
    """
    Assign progressively higher learning rates to later layers.
    Early layers (general features) → small LR (barely modify)
    Late layers (task-specific) → large LR (adapt more)
    """
    params = []
    
    # Group 1: Early layers (lowest LR)
    params.append({
        'params': list(model.layer1.parameters()) + list(model.layer2.parameters()),
        'lr': base_lr
    })
    
    # Group 2: Middle layers
    params.append({
        'params': model.layer3.parameters(),
        'lr': base_lr * multiplier
    })
    
    # Group 3: Late layers
    params.append({
        'params': model.layer4.parameters(),
        'lr': base_lr * multiplier ** 2
    })
    
    # Group 4: Classification head (highest LR)
    params.append({
        'params': model.fc.parameters(),
        'lr': base_lr * multiplier ** 3
    })
    
    return params

model = models.resnet50(weights='IMAGENET1K_V2')
model.fc = nn.Linear(2048, 5)
param_groups = get_discriminative_lr_params(model, base_lr=1e-5)
optimizer = optim.Adam(param_groups)

# Results in:
# layer1, layer2: lr = 1e-5
# layer3:         lr = 2.5e-5
# layer4:         lr = 6.25e-5
# fc:             lr = 1.56e-4
```

### Strategy 4: Learning Rate Warmup + Cosine Decay

```python
from torch.optim.lr_scheduler import CosineAnnealingWarmRestarts, LinearLR, SequentialLR

optimizer = optim.AdamW(model.parameters(), lr=1e-4, weight_decay=0.01)

# Warmup for 5 epochs, then cosine decay
warmup_scheduler = LinearLR(optimizer, start_factor=0.1, total_iters=5)
cosine_scheduler = CosineAnnealingWarmRestarts(optimizer, T_0=10, eta_min=1e-6)

scheduler = SequentialLR(optimizer, 
                         schedulers=[warmup_scheduler, cosine_scheduler],
                         milestones=[5])
```

---

## 5. Feature Extraction vs Fine-Tuning

### Complete Comparison

| Aspect | Feature Extraction | Fine-Tuning |
|--------|-------------------|-------------|
| What's trained | Only new head | Some/all layers |
| Training speed | Very fast (minutes) | Slower (hours) |
| Data needed | 100-1000 samples | 1000-10000+ samples |
| Risk of overfitting | Low | Higher (use regularization) |
| Final performance | Good baseline | Usually better |
| When to use | Small data, similar domain | Enough data, or different domain |

### Code Example: Complete Pipeline

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms, models
from torch.utils.data import DataLoader, random_split
import time

# ============================================================
# STEP 1: Data Preparation with Proper Augmentation
# ============================================================
# Key: Use the SAME normalization as the pretrained model was trained with
IMAGENET_MEAN = [0.485, 0.456, 0.406]
IMAGENET_STD = [0.229, 0.224, 0.225]

train_transforms = transforms.Compose([
    transforms.RandomResizedCrop(224),      # Random crop + resize to 224x224
    transforms.RandomHorizontalFlip(),       # 50% chance flip
    transforms.RandomRotation(15),           # ±15 degree rotation
    transforms.ColorJitter(brightness=0.2, contrast=0.2, saturation=0.2),
    transforms.ToTensor(),
    transforms.Normalize(IMAGENET_MEAN, IMAGENET_STD),  # MUST match pretrained!
])

val_transforms = transforms.Compose([
    transforms.Resize(256),                  # Resize shorter side to 256
    transforms.CenterCrop(224),              # Center crop to 224x224
    transforms.ToTensor(),
    transforms.Normalize(IMAGENET_MEAN, IMAGENET_STD),
])

# Assuming data organized as: data/train/class1/, data/train/class2/, etc.
train_dataset = datasets.ImageFolder('data/train', transform=train_transforms)
val_dataset = datasets.ImageFolder('data/val', transform=val_transforms)

train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True, 
                         num_workers=4, pin_memory=True)
val_loader = DataLoader(val_dataset, batch_size=32, shuffle=False,
                       num_workers=4, pin_memory=True)

num_classes = len(train_dataset.classes)
print(f"Classes: {train_dataset.classes}")
print(f"Training samples: {len(train_dataset)}")

# ============================================================
# STEP 2: Load Pretrained Model + Modify Head
# ============================================================
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

model = models.resnet50(weights='IMAGENET1K_V2')

# Option A: Simple head replacement
# model.fc = nn.Linear(2048, num_classes)

# Option B: More complex head (usually better)
model.fc = nn.Sequential(
    nn.Linear(2048, 512),
    nn.ReLU(),
    nn.Dropout(0.4),       # Regularization crucial for small datasets
    nn.Linear(512, 128),
    nn.ReLU(),
    nn.Dropout(0.2),
    nn.Linear(128, num_classes),
)

model = model.to(device)

# ============================================================
# STEP 3: Freeze Strategy
# ============================================================
# Phase 1: Freeze backbone, train only head
for name, param in model.named_parameters():
    if 'fc' not in name:
        param.requires_grad = False

# Count trainable parameters
trainable = sum(p.numel() for p in model.parameters() if p.requires_grad)
total = sum(p.numel() for p in model.parameters())
print(f"Trainable: {trainable:,} / {total:,} ({100*trainable/total:.1f}%)")

# ============================================================
# STEP 4: Training Function
# ============================================================
def train_model(model, train_loader, val_loader, criterion, optimizer, 
                scheduler=None, num_epochs=10):
    """Complete training loop with validation"""
    best_val_acc = 0
    history = {'train_loss': [], 'val_loss': [], 'train_acc': [], 'val_acc': []}
    
    for epoch in range(num_epochs):
        # ─── Training Phase ───
        model.train()
        running_loss = 0.0
        correct = 0
        total = 0
        
        for inputs, labels in train_loader:
            inputs, labels = inputs.to(device), labels.to(device)
            
            optimizer.zero_grad()
            outputs = model(inputs)
            loss = criterion(outputs, labels)
            loss.backward()
            
            # Gradient clipping prevents explosion when unfreezing
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            optimizer.step()
            
            running_loss += loss.item() * inputs.size(0)
            _, predicted = torch.max(outputs, 1)
            total += labels.size(0)
            correct += (predicted == labels).sum().item()
        
        train_loss = running_loss / total
        train_acc = correct / total
        
        # ─── Validation Phase ───
        model.eval()
        val_loss = 0.0
        val_correct = 0
        val_total = 0
        
        with torch.no_grad():
            for inputs, labels in val_loader:
                inputs, labels = inputs.to(device), labels.to(device)
                outputs = model(inputs)
                loss = criterion(outputs, labels)
                
                val_loss += loss.item() * inputs.size(0)
                _, predicted = torch.max(outputs, 1)
                val_total += labels.size(0)
                val_correct += (predicted == labels).sum().item()
        
        val_loss /= val_total
        val_acc = val_correct / val_total
        
        if scheduler:
            scheduler.step()
        
        # Save best model
        if val_acc > best_val_acc:
            best_val_acc = val_acc
            torch.save(model.state_dict(), 'best_model.pth')
        
        history['train_loss'].append(train_loss)
        history['val_loss'].append(val_loss)
        history['train_acc'].append(train_acc)
        history['val_acc'].append(val_acc)
        
        print(f"Epoch [{epoch+1}/{num_epochs}] | "
              f"Train Loss: {train_loss:.4f} Acc: {train_acc:.4f} | "
              f"Val Loss: {val_loss:.4f} Acc: {val_acc:.4f}")
    
    print(f"\nBest Validation Accuracy: {best_val_acc:.4f}")
    return history

# ============================================================
# STEP 5: Phase 1 — Train Head Only
# ============================================================
criterion = nn.CrossEntropyLoss(label_smoothing=0.1)  # Label smoothing helps
optimizer = optim.Adam(filter(lambda p: p.requires_grad, model.parameters()), 
                       lr=1e-3)

print("=" * 50)
print("Phase 1: Training head only (backbone frozen)")
print("=" * 50)
history1 = train_model(model, train_loader, val_loader, criterion, optimizer, 
                       num_epochs=10)

# ============================================================
# STEP 6: Phase 2 — Unfreeze and Fine-tune
# ============================================================
# Unfreeze last two ResNet blocks
for name, param in model.named_parameters():
    if 'layer3' in name or 'layer4' in name or 'fc' in name:
        param.requires_grad = True

# Use discriminative LR: lower for pretrained, higher for head
optimizer = optim.AdamW([
    {'params': model.layer3.parameters(), 'lr': 1e-5},
    {'params': model.layer4.parameters(), 'lr': 5e-5},
    {'params': model.fc.parameters(), 'lr': 1e-4},
], weight_decay=0.01)

scheduler = optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=20, eta_min=1e-6)

print("\n" + "=" * 50)
print("Phase 2: Fine-tuning (layer3, layer4, fc unfrozen)")
print("=" * 50)
history2 = train_model(model, train_loader, val_loader, criterion, optimizer, 
                       scheduler, num_epochs=20)
```

---

## 6. Domain Adaptation

### What It Is
**Domain adaptation** handles the case where your source domain (training data) has a different distribution than your target domain (deployment data), but the task is the same.

**Example**: Model trained on professional product photos → deployed on user-uploaded phone photos (different lighting, angles, backgrounds).

### Types of Domain Shift

```
┌─────────────────────────────────────────────┐
│           Types of Domain Shift              │
├─────────────────────────────────────────────┤
│ Covariate Shift    │ P(X) changes           │
│                    │ P(Y|X) stays same       │
│ (Most common)      │ e.g., daylight → night │
├────────────────────┼────────────────────────┤
│ Label Shift        │ P(Y) changes           │
│                    │ P(X|Y) stays same       │
│                    │ e.g., disease prevalence│
├────────────────────┼────────────────────────┤
│ Concept Drift      │ P(Y|X) changes         │
│                    │ Same input, diff labels │
│                    │ e.g., spam evolves      │
└────────────────────┴────────────────────────┘
```

### Unsupervised Domain Adaptation: DANN

**Domain-Adversarial Neural Network**: Learn features that are discriminative for the task BUT invariant to the domain.

```python
import torch
import torch.nn as nn
from torch.autograd import Function

class GradientReversalLayer(Function):
    """
    Reverses gradients during backprop.
    Forward: identity (no change)
    Backward: multiply gradient by -λ
    
    This CONFUSES the domain classifier, forcing the feature extractor
    to produce domain-invariant features.
    """
    @staticmethod
    def forward(ctx, x, lambda_val):
        ctx.lambda_val = lambda_val
        return x.clone()
    
    @staticmethod
    def backward(ctx, grad_output):
        return -ctx.lambda_val * grad_output, None


class DANN(nn.Module):
    """
    Domain-Adversarial Neural Network
    
    Architecture:
    Input → Feature Extractor → [Task Classifier (for labels)
                                  Domain Classifier (real or fake domain)]
    
    The gradient reversal layer makes features domain-invariant.
    """
    def __init__(self, num_classes=10):
        super(DANN, self).__init__()
        
        # Shared feature extractor
        self.feature_extractor = nn.Sequential(
            nn.Conv2d(3, 64, 5),
            nn.BatchNorm2d(64),
            nn.ReLU(),
            nn.MaxPool2d(2),
            nn.Conv2d(64, 128, 5),
            nn.BatchNorm2d(128),
            nn.ReLU(),
            nn.MaxPool2d(2),
            nn.Flatten(),
            nn.Linear(128 * 4 * 4, 256),
            nn.ReLU(),
        )
        
        # Task classifier (predict class labels)
        self.task_classifier = nn.Sequential(
            nn.Linear(256, 128),
            nn.ReLU(),
            nn.Linear(128, num_classes),
        )
        
        # Domain classifier (predict source vs target domain)
        self.domain_classifier = nn.Sequential(
            nn.Linear(256, 128),
            nn.ReLU(),
            nn.Linear(128, 2),  # 2 domains: source / target
        )
    
    def forward(self, x, lambda_val=1.0):
        features = self.feature_extractor(x)
        
        # Task prediction (normal gradient flow)
        class_output = self.task_classifier(features)
        
        # Domain prediction (REVERSED gradient!)
        reversed_features = GradientReversalLayer.apply(features, lambda_val)
        domain_output = self.domain_classifier(reversed_features)
        
        return class_output, domain_output


def train_dann(model, source_loader, target_loader, optimizer, 
               num_epochs=50, device='cuda'):
    """
    Train DANN: use source labels + domain confusion on both.
    """
    task_criterion = nn.CrossEntropyLoss()
    domain_criterion = nn.CrossEntropyLoss()
    
    for epoch in range(num_epochs):
        model.train()
        
        # Lambda increases from 0 to 1 during training (curriculum)
        p = epoch / num_epochs
        lambda_val = 2.0 / (1.0 + torch.exp(torch.tensor(-10.0 * p))) - 1.0
        
        for (src_data, src_labels), (tgt_data, _) in zip(source_loader, target_loader):
            src_data = src_data.to(device)
            src_labels = src_labels.to(device)
            tgt_data = tgt_data.to(device)
            
            batch_size = src_data.size(0)
            
            # Domain labels
            source_domain = torch.zeros(batch_size, dtype=torch.long).to(device)
            target_domain = torch.ones(tgt_data.size(0), dtype=torch.long).to(device)
            
            optimizer.zero_grad()
            
            # Source: task loss + domain loss
            class_out, domain_out = model(src_data, lambda_val.item())
            task_loss = task_criterion(class_out, src_labels)
            src_domain_loss = domain_criterion(domain_out, source_domain)
            
            # Target: domain loss only (no labels!)
            _, tgt_domain_out = model(tgt_data, lambda_val.item())
            tgt_domain_loss = domain_criterion(tgt_domain_out, target_domain)
            
            # Total loss
            loss = task_loss + src_domain_loss + tgt_domain_loss
            loss.backward()
            optimizer.step()
```

---

## 7. Few-Shot and Zero-Shot Learning

### Few-Shot Learning

**What it is**: Learn to classify with only 1-5 examples per class. Crucial when data collection is expensive (medical, rare defects).

**Terminology**: 
- **N-way K-shot**: N classes, K examples each
- **Support set**: The few labeled examples
- **Query set**: Images to classify

### Siamese Networks (One-Shot Learning)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SiameseNetwork(nn.Module):
    """
    Learn a similarity function: are these two images the same class?
    Uses shared weights for both branches.
    """
    def __init__(self, backbone='resnet18'):
        super(SiameseNetwork, self).__init__()
        
        # Shared feature extractor (pretrained)
        import torchvision.models as models
        resnet = models.resnet18(weights='IMAGENET1K_V1')
        self.backbone = nn.Sequential(*list(resnet.children())[:-1])  # Remove FC
        
        # Embedding head
        self.embedding = nn.Sequential(
            nn.Flatten(),
            nn.Linear(512, 256),
            nn.ReLU(),
            nn.Linear(256, 128),  # 128-dim embedding
        )
    
    def forward_one(self, x):
        """Extract embedding for one image"""
        features = self.backbone(x)
        return self.embedding(features)
    
    def forward(self, x1, x2):
        """Get embeddings for both images"""
        embed1 = self.forward_one(x1)
        embed2 = self.forward_one(x2)
        return embed1, embed2


def contrastive_loss(embed1, embed2, label, margin=1.0):
    """
    Contrastive loss:
    - Same class (label=1): minimize distance
    - Different class (label=0): maximize distance (up to margin)
    
    L = y * d² + (1-y) * max(margin - d, 0)²
    """
    distance = F.pairwise_distance(embed1, embed2)
    loss = (label * distance.pow(2) + 
            (1 - label) * F.relu(margin - distance).pow(2))
    return loss.mean()


# Alternative: Triplet Loss (often works better)
def triplet_loss(anchor, positive, negative, margin=0.3):
    """
    Pull anchor-positive close, push anchor-negative apart.
    L = max(d(a,p) - d(a,n) + margin, 0)
    """
    pos_dist = F.pairwise_distance(anchor, positive)
    neg_dist = F.pairwise_distance(anchor, negative)
    loss = F.relu(pos_dist - neg_dist + margin)
    return loss.mean()
```

### Prototypical Networks

```python
class PrototypicalNetwork(nn.Module):
    """
    Few-shot classification by computing class prototypes.
    
    Idea: Average the embeddings of the K support examples per class
    to get a "prototype". Classify queries by nearest prototype.
    """
    def __init__(self, backbone):
        super(PrototypicalNetwork, self).__init__()
        self.backbone = backbone  # Any pretrained feature extractor
    
    def compute_prototypes(self, support_embeddings, support_labels, n_way):
        """
        Compute class prototypes (mean embedding per class).
        
        support_embeddings: (n_way * k_shot, embed_dim)
        support_labels: (n_way * k_shot,)
        """
        prototypes = []
        for c in range(n_way):
            class_mask = (support_labels == c)
            class_embeddings = support_embeddings[class_mask]
            prototype = class_embeddings.mean(dim=0)  # Average embedding
            prototypes.append(prototype)
        return torch.stack(prototypes)  # (n_way, embed_dim)
    
    def forward(self, support_images, support_labels, query_images, n_way):
        """
        1. Embed support and query images
        2. Compute prototypes from support
        3. Classify queries by distance to prototypes
        """
        # Get embeddings
        support_embed = self.backbone(support_images)  # (n_way*k_shot, dim)
        query_embed = self.backbone(query_images)      # (n_query, dim)
        
        # Compute prototypes
        prototypes = self.compute_prototypes(support_embed, support_labels, n_way)
        
        # Compute distances from each query to each prototype
        # Using negative squared Euclidean distance as logits
        dists = torch.cdist(query_embed, prototypes)  # (n_query, n_way)
        logits = -dists  # Negative distance = similarity
        
        return logits  # Apply softmax + cross entropy for loss
```

### Zero-Shot Learning with CLIP

```python
# Zero-shot classification using OpenAI's CLIP
# Classify images into categories NEVER seen during training!

import torch
# pip install open_clip_torch
import open_clip

# Load CLIP model
model, _, preprocess = open_clip.create_model_and_transforms(
    'ViT-B-32', pretrained='laion2b_s34b_b79k'
)
tokenizer = open_clip.get_tokenizer('ViT-B-32')

def zero_shot_classify(image, class_names, model, preprocess, tokenizer):
    """
    Classify an image into classes using text descriptions.
    No training needed — just describe the classes in words!
    """
    device = next(model.parameters()).device
    
    # Encode image
    image_input = preprocess(image).unsqueeze(0).to(device)
    
    # Encode text descriptions of each class
    text_descriptions = [f"a photo of a {name}" for name in class_names]
    text_tokens = tokenizer(text_descriptions).to(device)
    
    with torch.no_grad():
        image_features = model.encode_image(image_input)
        text_features = model.encode_text(text_tokens)
        
        # Normalize embeddings
        image_features = image_features / image_features.norm(dim=-1, keepdim=True)
        text_features = text_features / text_features.norm(dim=-1, keepdim=True)
        
        # Cosine similarity between image and each class description
        similarity = (image_features @ text_features.T).softmax(dim=-1)
    
    # Get prediction
    probs = similarity[0].cpu().numpy()
    for name, prob in sorted(zip(class_names, probs), key=lambda x: -x[1]):
        print(f"  {name}: {prob:.3f}")
    
    return class_names[probs.argmax()]

# Usage:
# predicted = zero_shot_classify(
#     image, 
#     ["cat", "dog", "bird", "car", "airplane"],
#     model, preprocess, tokenizer
# )
```

---

## 8. Practical Implementation Guide

### Using timm (Recommended for Vision)

```python
import timm
import torch
import torch.nn as nn

# List available pretrained models
print(timm.list_models('efficientnet*', pretrained=True)[:10])

# Create model with custom number of classes
model = timm.create_model(
    'efficientnet_b3',
    pretrained=True,
    num_classes=10,        # Automatically replaces head
    drop_rate=0.3,         # Dropout in head
    drop_path_rate=0.2,    # Stochastic depth (advanced regularization)
)

# Get model-specific transforms (IMPORTANT: each model has its own!)
data_config = timm.data.resolve_model_data_config(model)
train_transforms = timm.data.create_transform(**data_config, is_training=True)
val_transforms = timm.data.create_transform(**data_config, is_training=False)

# For feature extraction only:
model = timm.create_model('efficientnet_b3', pretrained=True, num_classes=0)
# num_classes=0 removes the head, outputs raw features
features = model(torch.randn(1, 3, 300, 300))  # (1, 1536) feature vector
```

### Using Hugging Face for NLP Transfer Learning

```python
from transformers import (
    AutoTokenizer, AutoModelForSequenceClassification,
    TrainingArguments, Trainer
)
from datasets import load_dataset
import numpy as np

# Load pretrained BERT for classification
model_name = "bert-base-uncased"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSequenceClassification.from_pretrained(
    model_name, 
    num_labels=2,  # Binary classification
)

# Load dataset
dataset = load_dataset("imdb")

# Tokenize
def tokenize_function(examples):
    return tokenizer(examples["text"], padding="max_length", 
                    truncation=True, max_length=512)

tokenized_datasets = dataset.map(tokenize_function, batched=True)

# Fine-tune with Trainer API
training_args = TrainingArguments(
    output_dir="./results",
    num_train_epochs=3,
    per_device_train_batch_size=16,
    per_device_eval_batch_size=64,
    learning_rate=2e-5,              # Lower LR for fine-tuning!
    weight_decay=0.01,
    warmup_steps=500,                # Warmup prevents destroying pretrained weights
    evaluation_strategy="epoch",
    save_strategy="epoch",
    load_best_model_at_end=True,
    fp16=True,                       # Mixed precision for speed
)

def compute_metrics(eval_pred):
    logits, labels = eval_pred
    predictions = np.argmax(logits, axis=-1)
    accuracy = (predictions == labels).mean()
    return {"accuracy": accuracy}

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_datasets["train"],
    eval_dataset=tokenized_datasets["test"],
    compute_metrics=compute_metrics,
)

trainer.train()
```

### Transfer Learning for Object Detection

```python
import torchvision
from torchvision.models.detection import fasterrcnn_resnet50_fpn_v2, FasterRCNN_ResNet50_FPN_V2_Weights
from torchvision.models.detection.faster_rcnn import FastRCNNPredictor

def get_detection_model(num_classes):
    """
    Load pretrained Faster R-CNN and replace the head 
    for your custom number of classes.
    """
    # Load pretrained on COCO (91 classes)
    model = fasterrcnn_resnet50_fpn_v2(
        weights=FasterRCNN_ResNet50_FPN_V2_Weights.DEFAULT
    )
    
    # Replace the classification head
    in_features = model.roi_heads.box_predictor.cls_score.in_features
    model.roi_heads.box_predictor = FastRCNNPredictor(in_features, num_classes)
    
    return model

# Example: Fine-tune for 3 classes (background + 2 objects)
model = get_detection_model(num_classes=3)
```

---

## 9. Advanced Techniques

### LoRA (Low-Rank Adaptation)

**What it is**: Instead of fine-tuning all weights, inject small trainable matrices into each layer. Trains only ~0.1% of parameters with comparable performance.

```python
import torch
import torch.nn as nn

class LoRALinear(nn.Module):
    """
    LoRA: Low-Rank Adaptation
    
    Instead of modifying W directly, learn W + BA where:
    - B: (d, r) with r << d
    - A: (r, k)
    
    Only B and A are trainable (much fewer parameters!)
    Original weight W stays frozen.
    """
    def __init__(self, original_linear, rank=8, alpha=16):
        super(LoRALinear, self).__init__()
        
        self.original = original_linear
        self.original.weight.requires_grad = False  # Freeze original
        if self.original.bias is not None:
            self.original.bias.requires_grad = False
        
        d_out, d_in = original_linear.weight.shape
        
        # Low-rank matrices
        self.lora_A = nn.Parameter(torch.randn(rank, d_in) * 0.01)
        self.lora_B = nn.Parameter(torch.zeros(d_out, rank))
        
        # Scaling factor
        self.scaling = alpha / rank
    
    def forward(self, x):
        # Original output + low-rank adaptation
        original_output = self.original(x)
        lora_output = (x @ self.lora_A.T @ self.lora_B.T) * self.scaling
        return original_output + lora_output


def apply_lora(model, rank=8, target_modules=['query', 'value']):
    """Apply LoRA to specific layers in a model"""
    for name, module in model.named_modules():
        if any(t in name for t in target_modules):
            if isinstance(module, nn.Linear):
                parent_name = '.'.join(name.split('.')[:-1])
                child_name = name.split('.')[-1]
                parent = dict(model.named_modules())[parent_name]
                setattr(parent, child_name, LoRALinear(module, rank=rank))
    
    # Count parameters
    trainable = sum(p.numel() for p in model.parameters() if p.requires_grad)
    total = sum(p.numel() for p in model.parameters())
    print(f"LoRA: Trainable {trainable:,} / {total:,} = {100*trainable/total:.2f}%")
    return model
```

### Knowledge Distillation

**What it is**: Transfer knowledge from a large model (teacher) to a small model (student) by training the student to match the teacher's soft outputs.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DistillationLoss(nn.Module):
    """
    Knowledge Distillation Loss
    
    Combines:
    1. Hard loss: student prediction vs true labels (standard CE)
    2. Soft loss: student logits vs teacher's soft predictions
    
    Temperature T softens the probability distribution,
    revealing "dark knowledge" (relationships between classes).
    """
    def __init__(self, temperature=4.0, alpha=0.7):
        super(DistillationLoss, self).__init__()
        self.T = temperature   # Higher T = softer distributions
        self.alpha = alpha     # Weight for soft loss (0.7 = mostly teacher)
    
    def forward(self, student_logits, teacher_logits, true_labels):
        # Hard loss: standard cross-entropy with true labels
        hard_loss = F.cross_entropy(student_logits, true_labels)
        
        # Soft loss: KL divergence between softened distributions
        soft_student = F.log_softmax(student_logits / self.T, dim=1)
        soft_teacher = F.softmax(teacher_logits / self.T, dim=1)
        soft_loss = F.kl_div(soft_student, soft_teacher, reduction='batchmean')
        soft_loss *= (self.T ** 2)  # Scale by T² (gradient magnitude correction)
        
        # Combined loss
        return self.alpha * soft_loss + (1 - self.alpha) * hard_loss


def distill(teacher, student, train_loader, optimizer, 
            temperature=4.0, alpha=0.7, num_epochs=20, device='cuda'):
    """Train student to mimic teacher"""
    teacher.eval()  # Teacher is frozen
    criterion = DistillationLoss(temperature, alpha)
    
    for epoch in range(num_epochs):
        student.train()
        total_loss = 0
        
        for inputs, labels in train_loader:
            inputs, labels = inputs.to(device), labels.to(device)
            
            # Get teacher predictions (no gradient needed)
            with torch.no_grad():
                teacher_logits = teacher(inputs)
            
            # Get student predictions
            student_logits = student(inputs)
            
            # Distillation loss
            loss = criterion(student_logits, teacher_logits, labels)
            
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            total_loss += loss.item()
        
        print(f"Epoch [{epoch+1}/{num_epochs}] Loss: {total_loss/len(train_loader):.4f}")
```

### Mixed Precision Training (Speed + Memory)

```python
from torch.cuda.amp import autocast, GradScaler

# Mixed precision: use float16 where safe, float32 where needed
# ~2x speedup, ~50% less memory
scaler = GradScaler()

for inputs, labels in train_loader:
    inputs, labels = inputs.to(device), labels.to(device)
    
    optimizer.zero_grad()
    
    # Forward pass in mixed precision
    with autocast():
        outputs = model(inputs)
        loss = criterion(outputs, labels)
    
    # Backward pass with gradient scaling
    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

---

## 10. Common Mistakes

### Mistake 1: Wrong Input Normalization
**Problem**: Using your own normalization instead of the pretrained model's.
**Solution**: ALWAYS use the exact mean/std the model was trained with.
```python
# For ImageNet pretrained models:
transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
# For CLIP:
transforms.Normalize(mean=[0.48145466, 0.4578275, 0.40821073], 
                     std=[0.26862954, 0.26130258, 0.27577711])
```

### Mistake 2: Too High Learning Rate When Fine-Tuning
**Problem**: Large LR destroys pretrained features in early layers.
**Solution**: Use 10-100x smaller LR than training from scratch. Start with 1e-5 to 1e-4.

### Mistake 3: Not Freezing BatchNorm During Fine-Tuning
**Problem**: BatchNorm running statistics get corrupted by small dataset.
**Solution**: 
```python
# Freeze BatchNorm layers even when unfreezing other layers
for module in model.modules():
    if isinstance(module, (nn.BatchNorm2d, nn.BatchNorm1d)):
        module.eval()  # Use running stats, don't update
        for param in module.parameters():
            param.requires_grad = False
```

### Mistake 4: Using the Wrong Input Size
**Problem**: Resizing images to wrong dimensions (e.g., 224 for a model that expects 299).
**Solution**: Check model documentation. Common sizes: ResNet=224, Inception=299, EfficientNet=various (B0=224, B7=600).

### Mistake 5: Not Using Data Augmentation
**Problem**: Overfitting with small datasets.
**Solution**: Aggressive augmentation for small datasets:
```python
transforms.Compose([
    transforms.RandomResizedCrop(224, scale=(0.6, 1.0)),
    transforms.RandomHorizontalFlip(),
    transforms.RandomVerticalFlip(),  # If task allows
    transforms.RandomRotation(30),
    transforms.ColorJitter(0.4, 0.4, 0.4, 0.1),
    transforms.RandomGrayscale(p=0.1),
    transforms.GaussianBlur(kernel_size=3),
    transforms.RandomErasing(p=0.2),
])
```

### Mistake 6: Fine-Tuning All Layers with Tiny Dataset
**Problem**: With <500 samples, fine-tuning causes severe overfitting.
**Solution**: Use feature extraction only (frozen backbone + linear probe). If you must fine-tune, use strong regularization (dropout=0.5, weight_decay=0.1).

### Mistake 7: Ignoring Class Imbalance
**Problem**: Model predicts only majority class.
**Solution**:
```python
# Weighted loss
from collections import Counter
class_counts = Counter(train_dataset.targets)
weights = [1.0 / class_counts[i] for i in range(num_classes)]
weights = torch.FloatTensor(weights).to(device)
criterion = nn.CrossEntropyLoss(weight=weights)
```

---

## 11. Interview Questions

### Conceptual

**Q1: Why does transfer learning work?**
> **A**: Neural networks learn hierarchical features: early layers learn universal patterns (edges, textures) useful for any visual task. These features transfer across tasks because the visual world shares common low-level structure. Only the high-level, task-specific features need relearning.

**Q2: When would transfer learning NOT help?**
> **A**: When source and target domains are extremely different (e.g., ImageNet → radio signals), when the target task requires fundamentally different features, or when you have abundant target data (might be better to train from scratch). Also fails with very different input modalities.

**Q3: Explain the difference between fine-tuning and feature extraction.**
> **A**: Feature extraction = freeze pretrained backbone, only train new classifier head. Fine-tuning = unfreeze some/all pretrained layers and train them with a low learning rate. Feature extraction is safer with small data; fine-tuning gives better results with enough data.

**Q4: Why use a lower learning rate for fine-tuning?**
> **A**: Pretrained weights are already in a good region of the loss landscape. A high LR would jump far from this region, destroying useful learned features (catastrophic forgetting). Low LR makes small adjustments to adapt without destroying.

**Q5: What is catastrophic forgetting?**
> **A**: When fine-tuning on a new task, the model "forgets" what it learned from the original task. The new gradients overwrite the pretrained weights. Solutions: low LR, freeze early layers, elastic weight consolidation (EWC), progressive unfreezing.

**Q6: How does LoRA achieve parameter-efficient fine-tuning?**
> **A**: Instead of modifying all weights, LoRA adds low-rank matrices (rank r << d) to each layer: W_new = W_frozen + BA. Only B and A are trained (~0.1% of parameters). Based on the insight that weight updates during fine-tuning have low intrinsic rank.

**Q7: When would you use knowledge distillation?**
> **A**: When you need a small, fast model for deployment but have a large accurate model. The student learns not just the correct class but the teacher's full distribution (e.g., "this 7 looks like a 1 more than a 3") — dark knowledge that improves generalization.

**Q8: How do you handle domain shift between training and deployment?**
> **A**: Options: (1) Domain adaptation (DANN, adversarial training), (2) Data augmentation to simulate target domain, (3) Collect small labeled set from target domain for fine-tuning, (4) Test-time augmentation, (5) Self-supervised adaptation on unlabeled target data.

### Practical

**Q9: You have 200 labeled medical images and a pretrained ImageNet model. What's your strategy?**
> **A**: 
> 1. Use feature extraction first (frozen backbone + new head)
> 2. Heavy data augmentation (rotation, flip, color jitter)
> 3. Use larger input resolution if possible
> 4. Try multiple architectures (EfficientNet, ResNet, DenseNet)
> 5. If results insufficient, careful fine-tuning of last 1-2 blocks with very low LR (1e-5)
> 6. Use class-weighted loss if imbalanced
> 7. Cross-validation for reliable evaluation

**Q10: What preprocessing must match the pretrained model?**
> **A**: Input resolution, normalization (mean/std), color channel order (RGB vs BGR), value range ([0,1] vs [0,255] vs [-1,1]). Getting any of these wrong silently degrades performance.

---

## 12. Quick Reference

### Transfer Learning Decision Flowchart

```
Start
  │
  ▼
How much target data? ──────── Lots (>10k) ──▶ Fine-tune all layers
  │                                            (or train from scratch)
  │
  ├── Medium (1k-10k) ──▶ Fine-tune top layers + discriminative LR
  │
  └── Small (<1k) ──▶ Is domain similar to source?
                         │
                    Yes  │  No
                    ▼    ▼
            Feature   Fine-tune carefully
            Extraction  with heavy regularization
                        (dropout, weight decay, augmentation)
```

### Key Hyperparameters

| Parameter | Feature Extraction | Fine-Tuning |
|-----------|-------------------|-------------|
| Learning rate | 1e-3 to 1e-2 | 1e-5 to 1e-4 |
| Epochs | 10-30 | 20-100 |
| Batch size | 32-128 | 16-64 |
| Weight decay | 0.0001 | 0.01-0.1 |
| Dropout | 0.2-0.5 | 0.3-0.5 |
| Warmup epochs | 0-2 | 3-5 |

### Model Selection Guide

| Scenario | Recommended Model | Why |
|----------|------------------|-----|
| General classification | EfficientNet-B0/B3 | Best accuracy/compute trade-off |
| Mobile deployment | MobileNetV3 | Tiny, fast, still accurate |
| Maximum accuracy | EfficientNet-B7 or ViT-L | Highest ImageNet scores |
| Object detection | ResNet-50 + FPN | Proven backbone for detection |
| NLP classification | BERT-base / RoBERTa | Standard and effective |
| Zero-shot | CLIP ViT-B/32 | No training needed |
| Few-shot | Prototypical Networks + DINO | Best with minimal data |
| Parameter-efficient | LoRA on any large model | 0.1% params, 95% performance |

### Framework Comparison

| Framework | Strengths | Use When |
|-----------|----------|----------|
| torchvision | Native PyTorch, simple | Quick prototyping |
| timm | 700+ models, best configs | Production vision |
| Hugging Face | NLP+Vision+Audio, Trainer API | NLP or multimodal |
| Lightning | Reduces boilerplate | Complex training loops |
| fastai | High-level API, smart defaults | Rapid experimentation |

### Pro Tips

```
1. Always start with feature extraction — it's your baseline
2. Use model-specific transforms (timm.data.resolve_model_data_config)
3. Freeze BatchNorm when fine-tuning on small datasets
4. Use mixup/cutmix augmentation for better generalization
5. Learning rate finder helps choose the right LR
6. EMA (Exponential Moving Average) of weights improves stability
7. Test-time augmentation (TTA) boosts accuracy 1-3% for free
8. Save the full model + optimizer state for reproducibility
9. Monitor both train and val loss — divergence = overfitting
10. When in doubt, use a smaller model with more regularization
```

---

> **Key Takeaway**: Transfer learning is the default approach in modern deep learning. Start with a pretrained model, freeze early layers, and fine-tune later layers with a low learning rate. For small datasets (<1000 samples), feature extraction alone often outperforms fine-tuning. Use LoRA for parameter-efficient adaptation of large models. Always match your preprocessing to the pretrained model's requirements.
