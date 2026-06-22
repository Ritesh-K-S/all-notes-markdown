# Chapter 06: Vision Transformers — ViT, DeiT, Swin, BEiT & CLIP

---

## Table of Contents
1. [From CNNs to Transformers](#1-from-cnns-to-transformers)
2. [Vision Transformer (ViT)](#2-vision-transformer-vit)
3. [DeiT — Data-efficient Image Transformers](#3-deit--data-efficient-image-transformers)
4. [Swin Transformer](#4-swin-transformer)
5. [BEiT — BERT Pre-training for Vision](#5-beit--bert-pre-training-for-vision)
6. [CLIP — Connecting Vision and Language](#6-clip--connecting-vision-and-language)
7. [Other Notable Vision Transformers](#7-other-notable-vision-transformers)
8. [Practical Usage & Fine-tuning](#8-practical-usage--fine-tuning)
9. [Common Mistakes](#9-common-mistakes)
10. [Interview Questions](#10-interview-questions)
11. [Quick Reference](#11-quick-reference)

---

## 1. From CNNs to Transformers

### What It Is
Vision Transformers (ViTs) are a family of models that apply the Transformer architecture (originally designed for text/NLP) to images. Instead of using convolutions to process local neighborhoods, they split images into patches and use self-attention to let every patch "look at" every other patch.

### Why It Matters
- Transformers have largely replaced CNNs as the dominant architecture in computer vision
- They achieve state-of-the-art on nearly every vision benchmark
- They enable multi-modal models (CLIP, GPT-4V, LLaVA) that understand both images and text
- Transfer learning with ViTs is often superior to CNN-based transfer

### The Key Insight

```
CNN approach:                    Transformer approach:
Local → Global (hierarchical)   Global from the start (self-attention)

┌──────────────┐                ┌──────────────┐
│ ■■           │                │ ■ ■ ■ ■      │ 
│ ■■ ← 3×3    │                │              │  Every patch can
│    kernel    │                │ ■ ■ ■ ■      │  attend to every
│              │                │              │  other patch
│    sees only │                │ ■ ■ ■ ■      │  immediately!
│    local area│                │              │
└──────────────┘                └──────────────┘
```

**Why did CNNs dominate before 2020?**
- Images have strong **locality** (nearby pixels are related)
- CNNs encode this inductive bias directly (local kernels)
- Transformers need MORE DATA to learn these patterns from scratch
- ViT only worked well when trained on JFT-300M (300 million images)

**What changed?**
- Better training recipes (DeiT showed ViTs work with ImageNet alone)
- Efficient attention mechanisms (Swin, linear attention)
- Self-supervised pre-training (BEiT, MAE, DINO)
- Scale: bigger datasets + compute made global attention worthwhile

---

## 2. Vision Transformer (ViT)

### What It Is
ViT (Vision Transformer) was the breakthrough paper (Google, 2020) showing that a **pure** Transformer — with zero convolutions — can match or beat CNNs on image classification when given enough data.

### Why It Matters
- Proved Transformers work for vision (changed the entire field)
- Simple architecture: literally the NLP Transformer applied to image patches
- Foundation for all subsequent vision transformers
- Scales better than CNNs with more data and compute

### How It Works

**Step-by-Step:**

```
Input Image (224 × 224 × 3)
        │
        ▼
┌─────────────────────────────────────────────────────┐
│ 1. PATCH EMBEDDING                                   │
│    Split into 16×16 patches → 196 patches           │
│    Each patch: 16×16×3 = 768 values                 │
│    Linear projection: 768 → D (embedding dim)       │
│                                                      │
│    ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐    │
│    │p1│p2│p3│p4│p5│p6│p7│p8│p9│..│..│..│..│196│   │
│    └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘    │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│ 2. ADD [CLS] TOKEN + POSITION EMBEDDINGS            │
│                                                      │
│    [CLS] + [p1] + [p2] + ... + [p196]              │
│      ↓      ↓      ↓             ↓                  │
│    +pos0  +pos1  +pos2  ...    +pos196              │
│                                                      │
│    Total: 197 tokens, each of dimension D            │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│ 3. TRANSFORMER ENCODER (L layers)                    │
│                                                      │
│    For each layer:                                   │
│    ┌──────────────────────────────────────┐         │
│    │ Layer Norm                            │         │
│    │ Multi-Head Self-Attention (MHSA)     │         │
│    │ + Residual Connection                 │         │
│    │ Layer Norm                            │         │
│    │ MLP (Feed-Forward)                   │         │
│    │ + Residual Connection                 │         │
│    └──────────────────────────────────────┘         │
│                                                      │
└─────────────────────┬───────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────┐
│ 4. CLASSIFICATION HEAD                               │
│    Take [CLS] token → Layer Norm → Linear → Classes │
└─────────────────────────────────────────────────────┘
```

**The Math:**

Patch embedding:
$$\mathbf{z}_0 = [\mathbf{x}_{class}; \mathbf{x}_p^1\mathbf{E}; \mathbf{x}_p^2\mathbf{E}; \ldots; \mathbf{x}_p^N\mathbf{E}] + \mathbf{E}_{pos}$$

Where:
- $\mathbf{x}_p^i$ = flattened patch $i$ (size $P^2 \cdot C$)
- $\mathbf{E} \in \mathbb{R}^{(P^2 \cdot C) \times D}$ = patch projection matrix
- $\mathbf{E}_{pos} \in \mathbb{R}^{(N+1) \times D}$ = learnable position embeddings

Self-attention:
$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

### Code Example — ViT from Scratch

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class PatchEmbedding(nn.Module):
    """Convert image into patch embeddings."""
    
    def __init__(self, img_size=224, patch_size=16, in_channels=3, embed_dim=768):
        super().__init__()
        self.img_size = img_size
        self.patch_size = patch_size
        self.num_patches = (img_size // patch_size) ** 2  # 196 for 224/16
        
        # Linear projection of flattened patches
        # Using Conv2d with kernel=patch_size, stride=patch_size
        # This is equivalent to: flatten patch → linear layer
        self.projection = nn.Conv2d(
            in_channels, embed_dim,
            kernel_size=patch_size, stride=patch_size
        )
    
    def forward(self, x):
        """
        x: (B, C, H, W) → (B, num_patches, embed_dim)
        """
        # (B, 3, 224, 224) → (B, 768, 14, 14)
        x = self.projection(x)
        # (B, 768, 14, 14) → (B, 768, 196) → (B, 196, 768)
        x = x.flatten(2).transpose(1, 2)
        return x


class MultiHeadSelfAttention(nn.Module):
    """Multi-head self-attention mechanism."""
    
    def __init__(self, embed_dim=768, num_heads=12, dropout=0.0):
        super().__init__()
        self.num_heads = num_heads
        self.head_dim = embed_dim // num_heads
        self.scale = self.head_dim ** -0.5  # 1/sqrt(d_k)
        
        # Single linear layer for Q, K, V (more efficient than 3 separate)
        self.qkv = nn.Linear(embed_dim, embed_dim * 3)
        self.proj = nn.Linear(embed_dim, embed_dim)
        self.attn_dropout = nn.Dropout(dropout)
        self.proj_dropout = nn.Dropout(dropout)
    
    def forward(self, x):
        """
        x: (B, N, D) where N = num_patches + 1 (CLS)
        """
        B, N, D = x.shape
        
        # Compute Q, K, V in one shot
        qkv = self.qkv(x).reshape(B, N, 3, self.num_heads, self.head_dim)
        qkv = qkv.permute(2, 0, 3, 1, 4)  # (3, B, heads, N, head_dim)
        q, k, v = qkv[0], qkv[1], qkv[2]
        
        # Scaled dot-product attention
        attn = (q @ k.transpose(-2, -1)) * self.scale  # (B, heads, N, N)
        attn = attn.softmax(dim=-1)
        attn = self.attn_dropout(attn)
        
        # Apply attention to values
        x = (attn @ v).transpose(1, 2).reshape(B, N, D)  # (B, N, D)
        
        # Output projection
        x = self.proj(x)
        x = self.proj_dropout(x)
        
        return x


class TransformerBlock(nn.Module):
    """Single transformer encoder block."""
    
    def __init__(self, embed_dim=768, num_heads=12, mlp_ratio=4.0, dropout=0.0):
        super().__init__()
        
        self.norm1 = nn.LayerNorm(embed_dim)
        self.attn = MultiHeadSelfAttention(embed_dim, num_heads, dropout)
        
        self.norm2 = nn.LayerNorm(embed_dim)
        self.mlp = nn.Sequential(
            nn.Linear(embed_dim, int(embed_dim * mlp_ratio)),
            nn.GELU(),  # ViT uses GELU, not ReLU
            nn.Dropout(dropout),
            nn.Linear(int(embed_dim * mlp_ratio), embed_dim),
            nn.Dropout(dropout),
        )
    
    def forward(self, x):
        # Pre-norm architecture (norm before attention, not after)
        x = x + self.attn(self.norm1(x))   # Residual + attention
        x = x + self.mlp(self.norm2(x))    # Residual + FFN
        return x


class VisionTransformer(nn.Module):
    """
    Complete Vision Transformer (ViT) implementation.
    
    Variants:
    - ViT-Base:  D=768,  L=12, H=12, Params=86M
    - ViT-Large: D=1024, L=24, H=16, Params=307M
    - ViT-Huge:  D=1280, L=32, H=16, Params=632M
    """
    
    def __init__(self, img_size=224, patch_size=16, in_channels=3,
                 num_classes=1000, embed_dim=768, depth=12, 
                 num_heads=12, mlp_ratio=4.0, dropout=0.1):
        super().__init__()
        
        # Patch embedding
        self.patch_embed = PatchEmbedding(img_size, patch_size, in_channels, embed_dim)
        num_patches = self.patch_embed.num_patches
        
        # Learnable [CLS] token
        self.cls_token = nn.Parameter(torch.zeros(1, 1, embed_dim))
        
        # Learnable position embeddings
        self.pos_embed = nn.Parameter(torch.zeros(1, num_patches + 1, embed_dim))
        self.pos_dropout = nn.Dropout(dropout)
        
        # Transformer encoder blocks
        self.blocks = nn.Sequential(*[
            TransformerBlock(embed_dim, num_heads, mlp_ratio, dropout)
            for _ in range(depth)
        ])
        
        # Classification head
        self.norm = nn.LayerNorm(embed_dim)
        self.head = nn.Linear(embed_dim, num_classes)
        
        # Initialize weights
        nn.init.trunc_normal_(self.pos_embed, std=0.02)
        nn.init.trunc_normal_(self.cls_token, std=0.02)
    
    def forward(self, x):
        """
        x: (B, 3, 224, 224) → (B, num_classes)
        """
        B = x.shape[0]
        
        # 1. Patch embedding
        x = self.patch_embed(x)  # (B, 196, 768)
        
        # 2. Prepend [CLS] token
        cls_tokens = self.cls_token.expand(B, -1, -1)  # (B, 1, 768)
        x = torch.cat([cls_tokens, x], dim=1)  # (B, 197, 768)
        
        # 3. Add position embeddings
        x = x + self.pos_embed
        x = self.pos_dropout(x)
        
        # 4. Transformer encoder
        x = self.blocks(x)  # (B, 197, 768)
        
        # 5. Classification: use [CLS] token output
        x = self.norm(x[:, 0])  # (B, 768) — just the CLS token
        x = self.head(x)  # (B, num_classes)
        
        return x

# Create ViT-Base
model = VisionTransformer(
    img_size=224, patch_size=16, num_classes=1000,
    embed_dim=768, depth=12, num_heads=12
)
print(f"Parameters: {sum(p.numel() for p in model.parameters()) / 1e6:.1f}M")
# Output: Parameters: 86.6M

# Test forward pass
x = torch.randn(2, 3, 224, 224)
output = model(x)
print(f"Output shape: {output.shape}")  # (2, 1000)
```

### Position Embeddings — Deep Dive

```python
# Why position embeddings matter:
# Without them, the Transformer treats patches as an UNORDERED set
# (self-attention is permutation-equivariant)

# ViT uses LEARNABLE 1D position embeddings
# Interestingly, the model LEARNS a 2D grid structure!

import matplotlib.pyplot as plt
import numpy as np

def visualize_position_embeddings(pos_embed):
    """
    Visualize learned position embeddings.
    You'll see that nearby patches have similar embeddings!
    """
    # pos_embed: (1, 197, 768) — skip CLS token
    pos = pos_embed[0, 1:, :].detach().numpy()  # (196, 768)
    
    # Compute cosine similarity between all positions
    norms = np.linalg.norm(pos, axis=1, keepdims=True)
    pos_normalized = pos / norms
    similarity = pos_normalized @ pos_normalized.T  # (196, 196)
    
    # Reshape to 14×14 grid and visualize
    # Each row shows similarity of one patch to all others
    fig, axes = plt.subplots(4, 4, figsize=(12, 12))
    for idx, ax in enumerate(axes.flat):
        patch_idx = idx * 12  # Sample every 12th patch
        sim_map = similarity[patch_idx].reshape(14, 14)
        ax.imshow(sim_map, cmap='viridis')
        ax.set_title(f"Patch {patch_idx}")
        ax.axis('off')
    plt.suptitle("Position Embedding Similarities")
    plt.tight_layout()
    plt.savefig('pos_embed_viz.png')
```

---

## 3. DeiT — Data-efficient Image Transformers

### What It Is
DeiT (Facebook/Meta, 2021) showed that ViT can be trained effectively on **ImageNet alone** (1.2M images) — without the massive JFT-300M dataset. The secret: better training recipes + knowledge distillation.

### Why It Matters
- Made ViTs accessible to everyone (not just Google with 300M images)
- Introduced **distillation token** — a clever way to learn from CNNs
- Established training recipes that became standard for all ViTs
- Proved that ViT's original poor performance was due to training, not architecture

### How It Works

**Key Innovations:**

1. **Strong Data Augmentation:**
   - RandAugment, Mixup, CutMix, Random Erasing
   - These compensate for the lack of CNN's inductive bias

2. **Regularization:**
   - Stochastic Depth (drop entire layers randomly)
   - Label Smoothing
   - Repeated Augmentation

3. **Knowledge Distillation with Distillation Token:**

```
Standard ViT:                    DeiT with Distillation:
                                 
[CLS] [p1] [p2] ... [pN]       [CLS] [DIST] [p1] [p2] ... [pN]
  │                                │      │
  ▼                                ▼      ▼
Class                            Class   Teacher's
prediction                       pred    prediction
                                         (from CNN teacher)
```

The distillation token learns to mimic a CNN teacher's predictions. This gives DeiT both:
- The global attention of a Transformer (through CLS)
- The local bias of a CNN (through distillation from CNN teacher)

### Code Example

```python
import torch
import torch.nn as nn
from torchvision.models import vit_b_16, ViT_B_16_Weights

# ============================================
# Using Pre-trained DeiT/ViT (Recommended)
# ============================================

# Load DeiT-base from timm
import timm

# DeiT models available in timm
model = timm.create_model('deit_base_patch16_224', pretrained=True)
model.eval()

# Check model config
print(f"Model: {model.__class__.__name__}")
print(f"Num classes: {model.num_classes}")
print(f"Embed dim: {model.embed_dim}")

# Inference
from timm.data import resolve_data_config
from timm.data.transforms_factory import create_transform

config = resolve_data_config({}, model=model)
transform = create_transform(**config)

# Apply to image
from PIL import Image
# img = Image.open('test.jpg')
# input_tensor = transform(img).unsqueeze(0)
# output = model(input_tensor)
# pred = output.argmax(dim=1)
```

```python
# ============================================
# DeiT Training Recipe (Key Components)
# ============================================

import torch
import torch.nn as nn
from torchvision import transforms

# 1. Strong Data Augmentation (critical for ViTs!)
train_transform = transforms.Compose([
    transforms.RandomResizedCrop(224, scale=(0.08, 1.0)),
    transforms.RandomHorizontalFlip(),
    transforms.RandAugment(num_ops=2, magnitude=9),  # Key!
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],
                        std=[0.229, 0.224, 0.225]),
    transforms.RandomErasing(p=0.25),  # Random erasing
])

# 2. Mixup and CutMix (applied to batches)
from timm.data.mixup import Mixup

mixup_fn = Mixup(
    mixup_alpha=0.8,    # Mixup interpolation strength
    cutmix_alpha=1.0,   # CutMix area ratio
    prob=1.0,           # Probability of applying
    switch_prob=0.5,    # Probability of switching between mixup/cutmix
    mode='batch',
    num_classes=1000
)

# 3. Label Smoothing
criterion = nn.CrossEntropyLoss(label_smoothing=0.1)

# 4. Knowledge Distillation Loss
class DistillationLoss(nn.Module):
    """DeiT-style distillation loss."""
    
    def __init__(self, base_criterion, teacher_model, 
                 distillation_type='hard', alpha=0.5, tau=3.0):
        super().__init__()
        self.base_criterion = base_criterion  # Regular CE loss
        self.teacher_model = teacher_model    # CNN teacher
        self.type = distillation_type
        self.alpha = alpha    # Weight of distillation loss
        self.tau = tau        # Temperature for soft targets
    
    def forward(self, student_output, teacher_input, targets):
        """
        student_output: (cls_logits, dist_logits) from DeiT
        teacher_input: images for teacher model
        targets: ground truth labels
        """
        cls_output, dist_output = student_output
        
        # Standard classification loss on CLS token
        base_loss = self.base_criterion(cls_output, targets)
        
        # Get teacher predictions (no gradient!)
        with torch.no_grad():
            teacher_output = self.teacher_model(teacher_input)
        
        if self.type == 'hard':
            # Hard distillation: use teacher's argmax as target
            teacher_labels = teacher_output.argmax(dim=1)
            dist_loss = nn.functional.cross_entropy(dist_output, teacher_labels)
        else:
            # Soft distillation: KL divergence with temperature
            soft_teacher = nn.functional.softmax(teacher_output / self.tau, dim=1)
            soft_student = nn.functional.log_softmax(dist_output / self.tau, dim=1)
            dist_loss = nn.functional.kl_div(
                soft_student, soft_teacher, reduction='batchmean'
            ) * (self.tau ** 2)
        
        # Combined loss
        return (1 - self.alpha) * base_loss + self.alpha * dist_loss

# 5. Optimizer: AdamW with cosine schedule
def create_optimizer(model, lr=1e-3, weight_decay=0.05):
    """AdamW with proper weight decay (don't decay bias/norm)."""
    decay_params = []
    no_decay_params = []
    
    for name, param in model.named_parameters():
        if 'bias' in name or 'norm' in name or 'cls_token' in name or 'pos_embed' in name:
            no_decay_params.append(param)
        else:
            decay_params.append(param)
    
    return torch.optim.AdamW([
        {'params': decay_params, 'weight_decay': weight_decay},
        {'params': no_decay_params, 'weight_decay': 0.0}
    ], lr=lr, betas=(0.9, 0.999))
```

---

## 4. Swin Transformer

### What It Is
Swin Transformer (Microsoft, 2021) brings the **hierarchical structure** back to Transformers. It processes images at multiple scales (like CNNs) and uses **shifted windows** for efficient local attention.

### Why It Matters
- General-purpose backbone for ALL vision tasks (not just classification)
- Works for detection, segmentation, generation (unlike vanilla ViT)
- Linear complexity with image size (ViT is quadratic!)
- Became the de facto backbone replacing ResNets in many applications

### How It Works

**The Problem with ViT:**
- ViT outputs single-scale features (all at 1/16 resolution)
- Detection/segmentation need MULTI-SCALE features (1/4, 1/8, 1/16, 1/32)
- Full self-attention is $O(n^2)$ — prohibitive for high-res images

**Swin's Solution:**

```
Hierarchical Feature Maps (like CNN):

Stage 1: 56×56, dim=96      ← Fine-grained features
    │ Patch Merging (2× downsample)
    ▼
Stage 2: 28×28, dim=192     ← Medium features
    │ Patch Merging
    ▼
Stage 3: 14×14, dim=384     ← Coarse features
    │ Patch Merging
    ▼
Stage 4: 7×7, dim=768       ← Semantic features

(Compare: ViT only has 14×14, dim=768 — single scale!)
```

**Window-based Attention:**
```
Instead of:  Global attention (196 × 196 = expensive)
Use:         Window attention (49 × 49 = cheap!) in 7×7 windows

┌───────┬───────┬───────┬───────┐
│ Win 1 │ Win 2 │ Win 3 │ Win 4 │  Each window: 7×7 = 49 tokens
│ 7×7   │ 7×7   │ 7×7   │ 7×7   │  Self-attention only within window
├───────┼───────┼───────┼───────┤
│ Win 5 │ Win 6 │ Win 7 │ Win 8 │  Problem: No cross-window
│       │       │       │       │  communication!
└───────┴───────┴───────┴───────┘
```

**Shifted Window (the key innovation):**
```
Layer L: Regular windows        Layer L+1: SHIFTED windows
┌───────┬───────┐              ┌──┬────────┬──┐
│       │       │              │  │        │  │
│   A   │   B   │              │  │   E    │  │  Windows shift by
│       │       │              │──┼────────┼──│  (M/2, M/2) pixels
├───────┼───────┤              │  │        │  │
│       │       │              │  │   F    │  │  Now, tokens at
│   C   │   D   │              │  │        │  │  boundaries CAN
│       │       │              └──┴────────┴──┘  communicate!
└───────┴───────┘

Alternating regular ↔ shifted windows = global receptive field!
```

### Code Example

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class WindowAttention(nn.Module):
    """Window-based Multi-head Self Attention (W-MSA)."""
    
    def __init__(self, dim, window_size=7, num_heads=8):
        super().__init__()
        self.dim = dim
        self.window_size = window_size
        self.num_heads = num_heads
        head_dim = dim // num_heads
        self.scale = head_dim ** -0.5
        
        # Relative position bias (key for Swin's performance!)
        self.relative_position_bias_table = nn.Parameter(
            torch.zeros((2 * window_size - 1) * (2 * window_size - 1), num_heads)
        )
        
        # Compute relative position index
        coords_h = torch.arange(window_size)
        coords_w = torch.arange(window_size)
        coords = torch.stack(torch.meshgrid(coords_h, coords_w, indexing='ij'))
        coords_flatten = coords.view(2, -1)
        
        # Relative coords: (2, WH*WW, WH*WW)
        relative_coords = coords_flatten[:, :, None] - coords_flatten[:, None, :]
        relative_coords = relative_coords.permute(1, 2, 0).contiguous()
        relative_coords[:, :, 0] += window_size - 1  # Shift to start from 0
        relative_coords[:, :, 1] += window_size - 1
        relative_coords[:, :, 0] *= 2 * window_size - 1
        relative_position_index = relative_coords.sum(-1)
        self.register_buffer("relative_position_index", relative_position_index)
        
        self.qkv = nn.Linear(dim, dim * 3)
        self.proj = nn.Linear(dim, dim)
        
        nn.init.trunc_normal_(self.relative_position_bias_table, std=0.02)
    
    def forward(self, x, mask=None):
        """
        x: (num_windows*B, window_size*window_size, C)
        mask: for shifted window attention (mask some positions)
        """
        B_, N, C = x.shape  # N = window_size^2
        
        qkv = self.qkv(x).reshape(B_, N, 3, self.num_heads, C // self.num_heads)
        qkv = qkv.permute(2, 0, 3, 1, 4)
        q, k, v = qkv[0], qkv[1], qkv[2]
        
        attn = (q @ k.transpose(-2, -1)) * self.scale
        
        # Add relative position bias
        relative_position_bias = self.relative_position_bias_table[
            self.relative_position_index.view(-1)
        ].view(N, N, -1)
        relative_position_bias = relative_position_bias.permute(2, 0, 1)
        attn = attn + relative_position_bias.unsqueeze(0)
        
        # Apply attention mask for shifted windows
        if mask is not None:
            attn = attn + mask.unsqueeze(1).unsqueeze(0)
        
        attn = attn.softmax(dim=-1)
        
        x = (attn @ v).transpose(1, 2).reshape(B_, N, C)
        x = self.proj(x)
        return x


def window_partition(x, window_size):
    """Partition feature map into non-overlapping windows."""
    B, H, W, C = x.shape
    x = x.view(B, H // window_size, window_size, W // window_size, window_size, C)
    windows = x.permute(0, 1, 3, 2, 4, 5).contiguous()
    windows = windows.view(-1, window_size, window_size, C)
    return windows  # (num_windows*B, window_size, window_size, C)


def window_reverse(windows, window_size, H, W):
    """Reverse window partition."""
    B = int(windows.shape[0] / (H * W / window_size / window_size))
    x = windows.view(B, H // window_size, W // window_size, window_size, window_size, -1)
    x = x.permute(0, 1, 3, 2, 4, 5).contiguous().view(B, H, W, -1)
    return x


class SwinTransformerBlock(nn.Module):
    """Swin Transformer Block with (shifted) window attention."""
    
    def __init__(self, dim, num_heads, window_size=7, shift_size=0, mlp_ratio=4.0):
        super().__init__()
        self.dim = dim
        self.window_size = window_size
        self.shift_size = shift_size  # 0 for W-MSA, window_size//2 for SW-MSA
        
        self.norm1 = nn.LayerNorm(dim)
        self.attn = WindowAttention(dim, window_size, num_heads)
        self.norm2 = nn.LayerNorm(dim)
        self.mlp = nn.Sequential(
            nn.Linear(dim, int(dim * mlp_ratio)),
            nn.GELU(),
            nn.Linear(int(dim * mlp_ratio), dim),
        )
    
    def forward(self, x, H, W):
        """x: (B, H*W, C)"""
        B, L, C = x.shape
        
        x_reshaped = x.view(B, H, W, C)
        
        # Cyclic shift for shifted window attention
        if self.shift_size > 0:
            shifted_x = torch.roll(x_reshaped, 
                                   shifts=(-self.shift_size, -self.shift_size), 
                                   dims=(1, 2))
        else:
            shifted_x = x_reshaped
        
        # Partition into windows
        x_windows = window_partition(shifted_x, self.window_size)
        x_windows = x_windows.view(-1, self.window_size * self.window_size, C)
        
        # Window attention
        attn_windows = self.attn(x_windows)
        
        # Merge windows back
        attn_windows = attn_windows.view(-1, self.window_size, self.window_size, C)
        shifted_x = window_reverse(attn_windows, self.window_size, H, W)
        
        # Reverse shift
        if self.shift_size > 0:
            x_out = torch.roll(shifted_x, 
                              shifts=(self.shift_size, self.shift_size), 
                              dims=(1, 2))
        else:
            x_out = shifted_x
        
        x_out = x_out.view(B, H * W, C)
        
        # Residual + FFN
        x = x + x_out
        x = x + self.mlp(self.norm2(x))
        
        return x
```

```python
# ============================================
# Using Pre-trained Swin Transformer
# ============================================

import timm

# Available Swin variants
# swin_tiny_patch4_window7_224   - 28M params
# swin_small_patch4_window7_224  - 50M params
# swin_base_patch4_window7_224   - 88M params
# swin_large_patch4_window7_224  - 197M params

model = timm.create_model('swin_base_patch4_window7_224', pretrained=True)
model.eval()

# For detection/segmentation (use as backbone)
# Swin outputs multi-scale features: {stage1, stage2, stage3, stage4}
backbone = timm.create_model('swin_base_patch4_window7_224', 
                              pretrained=True,
                              features_only=True,  # Extract features
                              out_indices=(0, 1, 2, 3))

x = torch.randn(1, 3, 224, 224)
features = backbone(x)
for i, feat in enumerate(features):
    print(f"Stage {i}: {feat.shape}")
# Stage 0: torch.Size([1, 96, 56, 56])    - 1/4 resolution
# Stage 1: torch.Size([1, 192, 28, 28])   - 1/8 resolution
# Stage 2: torch.Size([1, 384, 14, 14])   - 1/16 resolution
# Stage 3: torch.Size([1, 768, 7, 7])     - 1/32 resolution
```

> **Key Insight:** Swin's multi-scale output makes it a drop-in replacement for ResNets in FPN-based detectors (Faster R-CNN, Mask R-CNN, DETR). Just swap the backbone!

---

## 5. BEiT — BERT Pre-training for Vision

### What It Is
BEiT (Microsoft, 2021) applies BERT's "masked prediction" pre-training to images. It masks random patches and trains the model to predict what was there — forcing it to understand image structure without any labels.

### Why It Matters
- Self-supervised pre-training: No labeled data needed!
- Learns rich representations from millions of unlabeled images
- After pre-training, fine-tuning with very few labels works great
- Led to MAE, DINO, and other self-supervised vision methods

### How It Works

```
BERT (text):                     BEiT (images):
"The [MASK] sat on the mat"     [patch1] [MASK] [patch3] [MASK] [patch5]
     → predict "cat"                     → predict visual tokens

But wait — what are "visual tokens" for images?
```

**BEiT's Two-Stage Process:**

```
Stage 1: Learn a Visual Tokenizer (dVAE)
┌─────────────────────────────────────────┐
│ Image → dVAE Encoder → Discrete Tokens  │
│ (like converting image to "visual words")│
│                                          │
│ 224×224 image → 14×14 = 196 tokens      │
│ Each token is one of 8192 possible values│
└─────────────────────────────────────────┘

Stage 2: Masked Image Modeling (MIM)
┌─────────────────────────────────────────────────┐
│                                                  │
│ Input:   [p1] [MASK] [p3] [MASK] [p5] [p6] ... │
│            │                                     │
│            ▼                                     │
│      Transformer Encoder                        │
│            │                                     │
│            ▼                                     │
│ Predict: [__] [tok2]  [__] [tok4]  [__] [__]   │
│                 ↑              ↑                  │
│           Ground truth from dVAE                 │
└─────────────────────────────────────────────────┘
```

**Comparison with MAE (Masked Autoencoder):**

| Aspect | BEiT | MAE |
|--------|------|-----|
| Target | Discrete visual tokens | Raw pixels |
| Masking ratio | 40% | 75% |
| Requires tokenizer | Yes (dVAE) | No (simpler!) |
| Decoder | Linear head | Lightweight decoder |
| Pre-training speed | Slower | Faster |

### Code Example

```python
# ============================================
# Masked Image Modeling (MAE-style, simplified)
# Easier to understand than BEiT's tokenizer
# ============================================

import torch
import torch.nn as nn

class MaskedAutoencoder(nn.Module):
    """
    Simplified MAE (He et al., 2022).
    Masks 75% of patches and reconstructs pixels.
    
    Key insight: The encoder only processes VISIBLE patches
    (saves 4× computation during pre-training!)
    """
    
    def __init__(self, img_size=224, patch_size=16, embed_dim=768,
                 encoder_depth=12, decoder_embed_dim=512, decoder_depth=8,
                 num_heads=12, mask_ratio=0.75):
        super().__init__()
        
        self.patch_size = patch_size
        self.mask_ratio = mask_ratio
        num_patches = (img_size // patch_size) ** 2  # 196
        
        # Encoder (processes only visible patches — very efficient!)
        self.patch_embed = PatchEmbedding(img_size, patch_size, 3, embed_dim)
        self.cls_token = nn.Parameter(torch.zeros(1, 1, embed_dim))
        self.pos_embed = nn.Parameter(torch.zeros(1, num_patches + 1, embed_dim))
        
        self.encoder_blocks = nn.Sequential(*[
            TransformerBlock(embed_dim, num_heads)
            for _ in range(encoder_depth)
        ])
        self.encoder_norm = nn.LayerNorm(embed_dim)
        
        # Decoder (reconstructs masked patches)
        self.decoder_embed = nn.Linear(embed_dim, decoder_embed_dim)
        self.mask_token = nn.Parameter(torch.zeros(1, 1, decoder_embed_dim))
        self.decoder_pos_embed = nn.Parameter(
            torch.zeros(1, num_patches + 1, decoder_embed_dim)
        )
        
        self.decoder_blocks = nn.Sequential(*[
            TransformerBlock(decoder_embed_dim, num_heads=8)
            for _ in range(decoder_depth)
        ])
        self.decoder_norm = nn.LayerNorm(decoder_embed_dim)
        
        # Prediction head: predict pixel values of masked patches
        self.decoder_pred = nn.Linear(
            decoder_embed_dim, patch_size ** 2 * 3  # 16×16×3 = 768
        )
    
    def random_masking(self, x, mask_ratio):
        """
        Random masking: keep (1-mask_ratio) patches, mask the rest.
        Returns visible patches and mask indices.
        """
        B, N, D = x.shape
        num_keep = int(N * (1 - mask_ratio))
        
        # Random shuffle → keep first num_keep
        noise = torch.rand(B, N, device=x.device)
        ids_shuffle = torch.argsort(noise, dim=1)
        ids_restore = torch.argsort(ids_shuffle, dim=1)
        
        # Keep visible patches
        ids_keep = ids_shuffle[:, :num_keep]
        x_visible = torch.gather(x, 1, ids_keep.unsqueeze(-1).expand(-1, -1, D))
        
        # Generate binary mask: 1=masked, 0=visible
        mask = torch.ones(B, N, device=x.device)
        mask[:, :num_keep] = 0
        mask = torch.gather(mask, 1, ids_restore)
        
        return x_visible, mask, ids_restore
    
    def forward(self, x):
        """
        Forward pass for pre-training.
        Returns reconstruction loss on masked patches.
        """
        # Patch embed
        patches = self.patch_embed(x)  # (B, 196, 768)
        patches = patches + self.pos_embed[:, 1:, :]  # Add pos embed
        
        # Mask: only keep 25% of patches for encoder!
        visible, mask, ids_restore = self.random_masking(patches, self.mask_ratio)
        
        # Encoder: process only visible patches (saves 75% compute!)
        cls = self.cls_token + self.pos_embed[:, :1, :]
        cls = cls.expand(visible.shape[0], -1, -1)
        visible = torch.cat([cls, visible], dim=1)
        
        encoded = self.encoder_blocks(visible)
        encoded = self.encoder_norm(encoded)
        
        # Decoder: reconstruct all patches
        decoded = self.decoder_embed(encoded)
        
        # Add mask tokens for masked positions
        B = decoded.shape[0]
        mask_tokens = self.mask_token.expand(B, ids_restore.shape[1] + 1 - decoded.shape[1], -1)
        full_tokens = torch.cat([decoded[:, 1:, :], mask_tokens], dim=1)
        full_tokens = torch.gather(
            full_tokens, 1, ids_restore.unsqueeze(-1).expand(-1, -1, decoded.shape[-1])
        )
        full_tokens = torch.cat([decoded[:, :1, :], full_tokens], dim=1)
        
        full_tokens = full_tokens + self.decoder_pos_embed
        decoded = self.decoder_blocks(full_tokens)
        decoded = self.decoder_norm(decoded)
        
        # Predict pixels
        pred = self.decoder_pred(decoded[:, 1:, :])  # Remove CLS
        
        # Compute loss only on masked patches
        target = self.patchify(x)  # (B, N, patch_size^2 * 3)
        loss = (pred - target) ** 2
        loss = loss.mean(dim=-1)  # Per-patch loss
        loss = (loss * mask).sum() / mask.sum()  # Only masked patches
        
        return loss
    
    def patchify(self, x):
        """Convert image to patches (ground truth target)."""
        B, C, H, W = x.shape
        p = self.patch_size
        h, w = H // p, W // p
        x = x.reshape(B, C, h, p, w, p)
        x = x.permute(0, 2, 4, 3, 5, 1).reshape(B, h * w, p * p * C)
        return x
```

---

## 6. CLIP — Connecting Vision and Language

### What It Is
CLIP (Contrastive Language-Image Pre-training, OpenAI, 2021) learns visual concepts from natural language descriptions. It jointly trains an image encoder and text encoder so that matching image-text pairs are close in embedding space.

### Why It Matters
- **Zero-shot classification:** Classify images into ANY categories without training!
- Foundation for DALL-E, Stable Diffusion, GPT-4V
- Enables text-based image search
- Robust to distribution shift (works on unusual images)
- The "backbone" of modern multi-modal AI

### How It Works

```
Training: Contrastive Learning on 400M image-text pairs

Image Encoder                    Text Encoder
(ViT or ResNet)                  (Transformer)
     │                                │
     ▼                                ▼
┌──────────┐                    ┌──────────┐
│ "A photo │                    │ "A photo │
│  of a    │ ←── MATCH ────→   │  of a    │
│  dog"    │                    │  dog"    │
└──────────┘                    └──────────┘
     │                                │
     ▼                                ▼
Image Embedding                 Text Embedding
   (512-d)                        (512-d)

Objective: Maximize similarity of matching pairs,
           minimize similarity of non-matching pairs

┌─────────────────────────────────────────────┐
│ Similarity Matrix (batch of N pairs):       │
│                                             │
│        Text₁  Text₂  Text₃  ...  TextN    │
│ Img₁  [ ✓     ✗      ✗           ✗   ]    │
│ Img₂  [ ✗     ✓      ✗           ✗   ]    │
│ Img₃  [ ✗     ✗      ✓           ✗   ]    │
│ ...                                         │
│ ImgN  [ ✗     ✗      ✗           ✓   ]    │
│                                             │
│ ✓ = high similarity (diagonal)             │
│ ✗ = low similarity (off-diagonal)          │
└─────────────────────────────────────────────┘
```

**The Loss:**

$$\mathcal{L} = -\frac{1}{N}\sum_{i=1}^{N}\log\frac{\exp(\text{sim}(I_i, T_i)/\tau)}{\sum_{j=1}^{N}\exp(\text{sim}(I_i, T_j)/\tau)}$$

Where $\text{sim}(I, T) = \frac{I \cdot T}{||I|| \cdot ||T||}$ (cosine similarity) and $\tau$ is a learned temperature.

### Code Examples

```python
# ============================================
# Zero-Shot Image Classification with CLIP
# No training on target classes needed!
# ============================================

# pip install open-clip-torch

import torch
import open_clip
from PIL import Image

# Load CLIP model
model, _, preprocess = open_clip.create_model_and_transforms(
    'ViT-B-32', pretrained='laion2b_s34b_b79k'
)
tokenizer = open_clip.get_tokenizer('ViT-B-32')
model.eval()

def zero_shot_classify(image_path, class_names):
    """
    Classify an image into ANY classes without training!
    
    How it works:
    1. Encode the image
    2. Encode text descriptions of each class
    3. Find which text is most similar to the image
    """
    # Encode image
    image = preprocess(Image.open(image_path)).unsqueeze(0)
    
    # Create text prompts (prompt engineering matters!)
    texts = [f"a photo of a {c}" for c in class_names]
    text_tokens = tokenizer(texts)
    
    with torch.no_grad():
        # Get embeddings
        image_features = model.encode_image(image)
        text_features = model.encode_text(text_tokens)
        
        # Normalize (cosine similarity)
        image_features /= image_features.norm(dim=-1, keepdim=True)
        text_features /= text_features.norm(dim=-1, keepdim=True)
        
        # Compute similarity
        similarity = (100.0 * image_features @ text_features.T).softmax(dim=-1)
    
    # Results
    results = [(class_names[i], similarity[0][i].item()) 
               for i in range(len(class_names))]
    results.sort(key=lambda x: x[1], reverse=True)
    
    return results

# Example: Classify into completely custom categories!
classes = ["cat", "dog", "bird", "car", "house", "tree", "person"]
# results = zero_shot_classify("test_image.jpg", classes)
# for cls, score in results:
#     print(f"  {cls:15s} {score:.3f}")
```

```python
# ============================================
# Image-Text Similarity Search with CLIP
# ============================================

import numpy as np
from pathlib import Path

class CLIPSearchEngine:
    """Search images using natural language queries."""
    
    def __init__(self, model_name='ViT-B-32'):
        self.model, _, self.preprocess = open_clip.create_model_and_transforms(
            model_name, pretrained='laion2b_s34b_b79k'
        )
        self.tokenizer = open_clip.get_tokenizer(model_name)
        self.model.eval()
        
        self.image_embeddings = None
        self.image_paths = []
    
    def index_images(self, image_dir):
        """Pre-compute embeddings for all images in a directory."""
        image_dir = Path(image_dir)
        self.image_paths = list(image_dir.glob('*.jpg')) + list(image_dir.glob('*.png'))
        
        embeddings = []
        for img_path in self.image_paths:
            image = self.preprocess(Image.open(img_path)).unsqueeze(0)
            with torch.no_grad():
                emb = self.model.encode_image(image)
                emb /= emb.norm(dim=-1, keepdim=True)
            embeddings.append(emb)
        
        self.image_embeddings = torch.cat(embeddings, dim=0)
        print(f"Indexed {len(self.image_paths)} images")
    
    def search(self, query, top_k=5):
        """Search images by text query."""
        # Encode query
        text = self.tokenizer([query])
        with torch.no_grad():
            text_emb = self.model.encode_text(text)
            text_emb /= text_emb.norm(dim=-1, keepdim=True)
        
        # Compute similarities
        similarities = (text_emb @ self.image_embeddings.T).squeeze(0)
        
        # Get top-k results
        top_indices = similarities.argsort(descending=True)[:top_k]
        
        results = []
        for idx in top_indices:
            results.append({
                'path': str(self.image_paths[idx]),
                'score': similarities[idx].item()
            })
        
        return results

# Usage
# engine = CLIPSearchEngine()
# engine.index_images('./my_photos/')
# results = engine.search("a sunset over the ocean")
# results = engine.search("people playing basketball")
# results = engine.search("a red car parked on the street")
```

```python
# ============================================
# CLIP for Image Captioning / Visual QA (Concept)
# ============================================

def clip_guided_caption(image_path, candidate_captions):
    """
    Given an image and candidate captions, 
    find the best matching caption using CLIP.
    """
    image = preprocess(Image.open(image_path)).unsqueeze(0)
    texts = tokenizer(candidate_captions)
    
    with torch.no_grad():
        image_features = model.encode_image(image)
        text_features = model.encode_text(texts)
        
        image_features /= image_features.norm(dim=-1, keepdim=True)
        text_features /= text_features.norm(dim=-1, keepdim=True)
        
        similarities = (image_features @ text_features.T).squeeze(0)
    
    best_idx = similarities.argmax().item()
    return candidate_captions[best_idx], similarities[best_idx].item()

# CLIP Prompt Engineering Tips:
# - "a photo of a {class}" works better than just "{class}"
# - "a good photo of a {class}" for high-quality images
# - "a blurry photo of a {class}" for low-quality
# - "a photo of a large {class}" for scale
# - Ensemble multiple prompts for better accuracy!

def ensemble_zero_shot(image_path, class_names, templates=None):
    """Zero-shot with prompt ensembling (more robust)."""
    if templates is None:
        templates = [
            "a photo of a {}",
            "a picture of a {}",
            "an image of a {}",
            "a good photo of a {}",
            "a photo of the {}",
            "a close-up photo of a {}",
        ]
    
    image = preprocess(Image.open(image_path)).unsqueeze(0)
    
    with torch.no_grad():
        image_features = model.encode_image(image)
        image_features /= image_features.norm(dim=-1, keepdim=True)
        
        # Average embeddings across templates for each class
        class_embeddings = []
        for cls in class_names:
            texts = [t.format(cls) for t in templates]
            text_tokens = tokenizer(texts)
            text_features = model.encode_text(text_tokens)
            text_features /= text_features.norm(dim=-1, keepdim=True)
            # Average across templates
            class_embeddings.append(text_features.mean(dim=0))
        
        class_embeddings = torch.stack(class_embeddings)
        class_embeddings /= class_embeddings.norm(dim=-1, keepdim=True)
        
        # Similarity
        similarity = (100.0 * image_features @ class_embeddings.T).softmax(dim=-1)
    
    return [(cls, sim.item()) for cls, sim in zip(class_names, similarity[0])]
```

---

## 7. Other Notable Vision Transformers

### Overview Table

| Model | Year | Key Innovation | Best For |
|-------|------|---------------|----------|
| CvT | 2021 | Convolutional token embedding + attention | Efficiency |
| PVT | 2021 | Pyramid structure (like Swin but different) | Dense prediction |
| SegFormer | 2021 | Lightweight decoder for segmentation | Semantic segmentation |
| DINO/DINOv2 | 2021/2023 | Self-supervised with self-distillation | Features without labels |
| SAM (Segment Anything) | 2023 | Foundation model for segmentation | Universal segmentation |
| EVA-02 | 2023 | Scaled ViT with CLIP pre-training | General purpose |

### DINOv2 — Self-Supervised Feature Learning

```python
# ============================================
# DINOv2: Powerful features WITHOUT any labels
# ============================================

import torch

# Load DINOv2 (Meta AI)
dinov2 = torch.hub.load('facebookresearch/dinov2', 'dinov2_vitb14')
dinov2.eval()

def extract_features_dino(image_tensor):
    """
    Extract rich visual features from DINOv2.
    These features work great for:
    - kNN classification (no training needed!)
    - Image retrieval
    - Segmentation (from patch tokens)
    """
    with torch.no_grad():
        # Get CLS token and patch tokens
        features = dinov2.forward_features(image_tensor)
        
        cls_token = features['x_norm_clstoken']   # (B, 768) - global
        patch_tokens = features['x_norm_patchtokens']  # (B, N, 768) - local
    
    return cls_token, patch_tokens

# DINOv2 patch features can be used directly for dense prediction!
# Reshape patch tokens to spatial grid for segmentation
def get_feature_map(image_tensor, model):
    """Get spatial feature map from DINOv2."""
    with torch.no_grad():
        features = model.forward_features(image_tensor)
        patch_tokens = features['x_norm_patchtokens']
    
    # Reshape to spatial: (B, H*W, D) → (B, D, H, W)
    B, N, D = patch_tokens.shape
    h = w = int(N ** 0.5)  # 16 for 224/14
    feature_map = patch_tokens.reshape(B, h, w, D).permute(0, 3, 1, 2)
    
    return feature_map  # (B, 768, 16, 16)
```

---

## 8. Practical Usage & Fine-tuning

### Fine-tuning a Vision Transformer

```python
# ============================================
# Fine-tuning ViT for Custom Classification
# ============================================

import torch
import torch.nn as nn
import timm
from torch.utils.data import DataLoader
from torchvision import datasets, transforms

# 1. Load pretrained model and modify head
model = timm.create_model('vit_base_patch16_224', pretrained=True, num_classes=10)

# 2. Freeze early layers (transfer learning strategy)
# Option A: Freeze everything except head
for param in model.parameters():
    param.requires_grad = False
model.head.weight.requires_grad = True
model.head.bias.requires_grad = True

# Option B: Freeze first N blocks (partial fine-tuning)
def partial_freeze(model, freeze_blocks=8):
    """Freeze first N transformer blocks."""
    for param in model.patch_embed.parameters():
        param.requires_grad = False
    
    for i, block in enumerate(model.blocks):
        if i < freeze_blocks:
            for param in block.parameters():
                param.requires_grad = False

# Option C: Full fine-tuning with lower LR for backbone
def create_param_groups(model, backbone_lr=1e-5, head_lr=1e-3):
    """Different learning rates for backbone vs head."""
    backbone_params = []
    head_params = []
    
    for name, param in model.named_parameters():
        if 'head' in name:
            head_params.append(param)
        else:
            backbone_params.append(param)
    
    return [
        {'params': backbone_params, 'lr': backbone_lr},
        {'params': head_params, 'lr': head_lr}
    ]

# 3. Training setup
param_groups = create_param_groups(model)
optimizer = torch.optim.AdamW(param_groups, weight_decay=0.01)

# Learning rate schedule (warmup + cosine decay)
from torch.optim.lr_scheduler import CosineAnnealingLR, LinearLR, SequentialLR

warmup = LinearLR(optimizer, start_factor=0.01, total_iters=500)
cosine = CosineAnnealingLR(optimizer, T_max=10000)
scheduler = SequentialLR(optimizer, [warmup, cosine], milestones=[500])

# 4. Data augmentation (CRITICAL for ViTs)
train_transform = transforms.Compose([
    transforms.RandomResizedCrop(224, scale=(0.5, 1.0)),
    transforms.RandomHorizontalFlip(),
    transforms.RandAugment(num_ops=2, magnitude=9),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
    transforms.RandomErasing(p=0.25),
])

val_transform = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
])

# 5. Training loop
def train_epoch(model, dataloader, criterion, optimizer, scheduler, device):
    model.train()
    total_loss = 0
    correct = 0
    total = 0
    
    for images, labels in dataloader:
        images, labels = images.to(device), labels.to(device)
        
        optimizer.zero_grad()
        outputs = model(images)
        loss = criterion(outputs, labels)
        loss.backward()
        
        # Gradient clipping (important for Transformers!)
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        
        optimizer.step()
        scheduler.step()
        
        total_loss += loss.item()
        _, predicted = outputs.max(1)
        total += labels.size(0)
        correct += predicted.eq(labels).sum().item()
    
    return total_loss / len(dataloader), 100. * correct / total

# Pro Tips for fine-tuning ViTs:
# 1. Use smaller learning rate than CNNs (1e-5 to 5e-5 for backbone)
# 2. Longer warmup (5-10% of training)
# 3. Strong augmentation is essential (RandAugment, Mixup, CutMix)
# 4. Label smoothing helps (0.1)
# 5. Layer-wise LR decay: lower LR for earlier layers
```

```python
# ============================================
# Layer-wise Learning Rate Decay (LLRD)
# Advanced technique: earlier layers get smaller LR
# ============================================

def create_llrd_param_groups(model, base_lr=1e-4, decay_rate=0.65):
    """
    Layer-wise Learning Rate Decay.
    
    Intuition: Early layers capture general features (don't change much),
    later layers are more task-specific (change more).
    
    Layer 12 (last): lr = base_lr
    Layer 11: lr = base_lr * decay_rate
    Layer 10: lr = base_lr * decay_rate^2
    ...
    Layer 0 (first): lr = base_lr * decay_rate^12
    """
    param_groups = []
    
    # Head gets full learning rate
    param_groups.append({
        'params': list(model.head.parameters()),
        'lr': base_lr
    })
    
    # Blocks get decaying learning rate
    num_blocks = len(model.blocks)
    for i, block in enumerate(reversed(model.blocks)):  # Start from last
        lr = base_lr * (decay_rate ** (i + 1))
        param_groups.append({
            'params': list(block.parameters()),
            'lr': lr
        })
    
    # Embedding layers get lowest learning rate
    embed_lr = base_lr * (decay_rate ** (num_blocks + 1))
    param_groups.append({
        'params': [model.pos_embed, model.cls_token] + 
                  list(model.patch_embed.parameters()),
        'lr': embed_lr
    })
    
    return param_groups
```

---

## 9. Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Training ViT from scratch on small datasets | ViTs need lots of data (no inductive bias like CNNs) | Always use pretrained models, fine-tune |
| Using CNN learning rates for ViTs | ViTs are more sensitive to high LR | Use 10-100× smaller LR (1e-5 to 5e-5) |
| No warmup in LR schedule | Transformers diverge without warmup | Use 5-10% warmup steps |
| Skipping data augmentation | ViTs overfit quickly without strong augmentation | Use RandAugment + Mixup + CutMix |
| Using ViT for object detection directly | ViT outputs single-scale features | Use Swin or add FPN adapter |
| Not normalizing CLIP embeddings | Cosine similarity requires unit vectors | Always L2-normalize before computing similarity |
| Wrong image preprocessing | Each model has specific normalization | Use model's built-in transforms |
| Ignoring position embedding interpolation | Changing resolution breaks pos embeddings | Use bicubic interpolation for pos embeddings |
| Not using gradient clipping | Transformers can have exploding gradients | Clip gradients to max_norm=1.0 |
| Using huge batch sizes without scaling LR | Effective LR changes with batch size | Linear LR scaling: lr_new = lr_base × (bs_new / bs_base) |

---

## 10. Interview Questions

### Conceptual

**Q1: How does ViT differ from CNN architectures fundamentally?**
> CNNs use local convolutions with built-in translation equivariance and locality bias. ViTs use global self-attention with no spatial inductive bias — every patch can attend to every other patch from layer 1. CNNs process images hierarchically (local → global), while ViTs have global receptive field immediately but need more data to learn spatial structure.

**Q2: Why did ViT initially require JFT-300M to work well?**
> Self-attention has no locality bias — it must learn from data that nearby pixels are related. CNNs encode this through local kernels. With small datasets, ViTs can't learn these patterns and overfit. DeiT later showed that strong augmentation + regularization + distillation compensates for limited data.

**Q3: Explain how Swin Transformer achieves linear complexity.**
> Swin computes attention only within local windows (e.g., 7×7 = 49 tokens). Cost per window: O(49²) = constant. With H×W image, number of windows = HW/49 → total cost is O(HW) = linear! Shifted windows in alternating layers provide cross-window communication.

**Q4: What is CLIP's key insight and limitation?**
> Key insight: Learn visual representations from natural language supervision. 400M image-text pairs provide diverse training signal. Limitation: CLIP is a matching model, not generative. It can tell you IF an image matches text, but can't describe novel concepts not in training data. Also struggles with counting, spatial relationships, and fine-grained attributes.

**Q5: Compare BEiT and MAE approaches to self-supervised vision.**
> BEiT predicts discrete visual tokens (from a pretrained dVAE). MAE predicts raw pixels. MAE is simpler (no tokenizer needed), faster (encodes only 25% visible patches), and often performs better. Both mask patches and predict the masked content, inspired by BERT's MLM.

**Q6: Why does CLIP use cosine similarity instead of dot product?**
> Cosine similarity normalizes by vector magnitude, making it scale-invariant. This is crucial because image and text embeddings may have different natural magnitudes. The learned temperature parameter τ controls the sharpness of the distribution.

### System Design

**Q7: Design a zero-shot image classification system for production.**
> (1) Use CLIP for encoding. (2) Pre-compute and cache text embeddings for all classes. (3) Ensemble multiple prompt templates per class. (4) For new images: encode once, compute similarity against cached text embeddings. (5) Add confidence thresholding. (6) For deployment: quantize CLIP model, batch inference, use ONNX/TensorRT.

**Q8: How would you adapt a ViT pretrained on 224×224 to 384×384 input?**
> The patch embedding works at any size (it's a convolution). The challenge is position embeddings — they were learned for 14×14 grid but now need 24×24. Solution: Bicubic interpolation of position embeddings from 14×14 → 24×24 grid, then fine-tune briefly at the new resolution.

---

## 11. Quick Reference

### Architecture Comparison

| Model | Params | ImageNet Top-1 | Attention Type | Multi-Scale |
|-------|--------|---------------|----------------|-------------|
| ViT-B/16 | 86M | 77.9% (IN-1K) | Global | No |
| ViT-L/16 | 307M | 85.2% (IN-21K→1K) | Global | No |
| DeiT-B | 86M | 81.8% | Global | No |
| Swin-T | 28M | 81.3% | Window (7×7) | Yes |
| Swin-B | 88M | 83.5% | Window (7×7) | Yes |
| Swin-L | 197M | 87.3% (IN-21K→1K) | Window (7×7) | Yes |
| BEiT-L | 307M | 87.5% | Global | No |
| EVA-02-L | 304M | 89.6% | Global | No |

### When to Use What

| Task | Best Choice | Why |
|------|-------------|-----|
| Image Classification | ViT/DeiT (pretrained) | Simple, effective |
| Object Detection | Swin Transformer | Multi-scale features |
| Segmentation | Swin + UPerNet, SegFormer | Hierarchical features |
| Zero-shot Tasks | CLIP | No training needed |
| Feature Extraction | DINOv2 | Best self-supervised features |
| Small Dataset | DeiT (fine-tuned) | Data-efficient training |
| Edge/Mobile | EfficientFormer, MobileViT | Optimized for speed |

### Key Hyperparameters for ViT Fine-tuning

| Parameter | Typical Value |
|-----------|--------------|
| Learning Rate (backbone) | 1e-5 to 5e-5 |
| Learning Rate (head) | 1e-3 to 1e-4 |
| Warmup Steps | 5-10% of total |
| Weight Decay | 0.01 - 0.05 |
| Batch Size | 32-128 |
| Epochs | 10-50 (fine-tune) |
| Label Smoothing | 0.1 |
| Mixup Alpha | 0.8 |
| CutMix Alpha | 1.0 |
| Drop Path Rate | 0.1-0.2 |
| Gradient Clip | 1.0 |
| LR Schedule | Cosine with warmup |
| Layer-wise LR Decay | 0.65-0.75 |

### CLIP Prompt Engineering

| Template | When to Use |
|----------|-------------|
| "a photo of a {class}" | General purpose |
| "a photo of a {class}, a type of pet" | When class is ambiguous |
| "a centered satellite photo of {class}" | Domain-specific |
| "a black and white photo of a {class}" | Style-specific |
| "itap of a {class}" | Internet photos |
| Ensemble 7+ templates | Production (always better) |

---

> **Key Takeaway:** Vision Transformers have become the dominant architecture in computer vision. Start with pretrained models (timm/torchvision), use strong augmentation, lower learning rates than CNNs, and choose Swin for tasks needing multi-scale features. CLIP enables zero-shot capabilities that were impossible before.
