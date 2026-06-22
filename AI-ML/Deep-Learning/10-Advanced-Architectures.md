# Chapter 10: Advanced Architectures

## U-Net, EfficientNet, Vision Transformers (ViT), and Diffusion Models

---

## Table of Contents
- [10.1 U-Net — Segmentation King](#101-u-net--segmentation-king)
- [10.2 EfficientNet — Scaling Done Right](#102-efficientnet--scaling-done-right)
- [10.3 Vision Transformers (ViT)](#103-vision-transformers-vit)
- [10.4 Diffusion Models](#104-diffusion-models)
- [10.5 Comparison and Selection Guide](#105-comparison-and-selection-guide)
- [10.6 Interview Questions](#106-interview-questions)
- [10.7 Quick Reference](#107-quick-reference)

---

## 10.1 U-Net — Segmentation King

### What It Is
U-Net is a neural network architecture shaped like the letter "U" that takes an image and outputs a **pixel-by-pixel map** telling you what each pixel belongs to. Think of it like a smart coloring tool — given a medical scan, it can color every pixel that's a tumor in red and every pixel that's healthy tissue in blue.

### Why It Matters
- **Medical imaging**: Segmenting tumors, organs, cells from scans
- **Autonomous driving**: Segmenting roads, pedestrians, vehicles
- **Satellite imagery**: Land cover classification
- **Industrial**: Defect detection on manufacturing lines
- Works incredibly well **even with very few training images** (originally designed for biomedical data where annotations are expensive)

### How It Works

#### Architecture Intuition
Imagine squeezing a sponge (encoder) — you compress the information but lose spatial details. Then you expand it back (decoder) — but you've lost fine details! U-Net's genius: **skip connections** copy the detailed information from the squeezing phase directly to the expanding phase.

```
INPUT IMAGE (572×572)
    │
    ▼
┌─────────────────────────────────────────────────┐
│  ENCODER (Contracting Path)                      │
│                                                   │
│  [Conv 3×3 → ReLU → Conv 3×3 → ReLU]  ──────────┼──── Skip Connection 1
│           │                                       │
│      MaxPool 2×2                                  │
│           │                                       │
│  [Conv 3×3 → ReLU → Conv 3×3 → ReLU]  ──────────┼──── Skip Connection 2
│           │                                       │
│      MaxPool 2×2                                  │
│           │                                       │
│  [Conv 3×3 → ReLU → Conv 3×3 → ReLU]  ──────────┼──── Skip Connection 3
│           │                                       │
│      MaxPool 2×2                                  │
│           │                                       │
│  [Conv 3×3 → ReLU → Conv 3×3 → ReLU]  ──────────┼──── Skip Connection 4
│           │                                       │
│      MaxPool 2×2                                  │
│           ▼                                       │
│  ┌─────────────────────┐                          │
│  │     BOTTLENECK      │                          │
│  │ Conv 3×3 → Conv 3×3 │                          │
│  └─────────────────────┘                          │
│           │                                       │
│  DECODER (Expanding Path)                         │
│           │                                       │
│      UpConv 2×2  ◄───────────────────────────────┼──── Skip Connection 4
│           │                                       │
│  [Conv 3×3 → ReLU → Conv 3×3 → ReLU]             │
│           │                                       │
│      UpConv 2×2  ◄───────────────────────────────┼──── Skip Connection 3
│           │                                       │
│  [Conv 3×3 → ReLU → Conv 3×3 → ReLU]             │
│           │                                       │
│      UpConv 2×2  ◄───────────────────────────────┼──── Skip Connection 2
│           │                                       │
│  [Conv 3×3 → ReLU → Conv 3×3 → ReLU]             │
│           │                                       │
│      UpConv 2×2  ◄───────────────────────────────┼──── Skip Connection 1
│           │                                       │
│  [Conv 3×3 → ReLU → Conv 3×3 → ReLU]             │
│           │                                       │
│      Conv 1×1 (Final)                             │
│           ▼                                       │
│    SEGMENTATION MAP                               │
└─────────────────────────────────────────────────┘
```

#### Key Components:

1. **Encoder (Contracting Path)**: Each block has two 3×3 convolutions + ReLU, followed by 2×2 max pooling. Channels double at each level (64→128→256→512→1024).

2. **Bottleneck**: Deepest part with the most compressed representation.

3. **Decoder (Expanding Path)**: Each block uses transposed convolution (up-conv 2×2) to upsample, concatenates with skip connection, then two 3×3 convolutions + ReLU.

4. **Skip Connections**: Copy-and-crop feature maps from encoder to decoder at the same level. This preserves fine-grained spatial information.

5. **Final Layer**: 1×1 convolution mapping feature vectors to desired number of classes.

#### The Math

For each convolutional block:

$$z = W * x + b$$
$$a = \text{ReLU}(z) = \max(0, z)$$

Skip connection concatenation:

$$x_{decoder} = \text{Concat}(\text{UpConv}(x_{lower}), \text{Crop}(x_{encoder}))$$

Loss function (typically):

$$\mathcal{L} = -\frac{1}{N}\sum_{i=1}^{N}\left[y_i \log(\hat{y}_i) + (1-y_i)\log(1-\hat{y}_i)\right]$$

For class imbalance, **Dice Loss** is preferred:

$$\mathcal{L}_{Dice} = 1 - \frac{2|X \cap Y|}{|X| + |Y|} = 1 - \frac{2\sum_i x_i y_i}{\sum_i x_i + \sum_i y_i}$$

### Code Example

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DoubleConv(nn.Module):
    """Two consecutive 3x3 conv layers with BatchNorm and ReLU."""
    def __init__(self, in_channels, out_channels):
        super().__init__()
        self.double_conv = nn.Sequential(
            nn.Conv2d(in_channels, out_channels, kernel_size=3, padding=1),  # padding=1 keeps spatial dims
            nn.BatchNorm2d(out_channels),  # normalize activations for stable training
            nn.ReLU(inplace=True),         # non-linearity
            nn.Conv2d(out_channels, out_channels, kernel_size=3, padding=1),
            nn.BatchNorm2d(out_channels),
            nn.ReLU(inplace=True),
        )
    
    def forward(self, x):
        return self.double_conv(x)


class UNet(nn.Module):
    """Full U-Net architecture for semantic segmentation."""
    def __init__(self, in_channels=1, out_channels=2, features=[64, 128, 256, 512]):
        super().__init__()
        self.encoder = nn.ModuleList()  # store encoder blocks
        self.decoder = nn.ModuleList()  # store decoder blocks
        self.pool = nn.MaxPool2d(kernel_size=2, stride=2)  # halves spatial dimensions
        
        # Build encoder (downsampling path)
        for feature in features:
            self.encoder.append(DoubleConv(in_channels, feature))
            in_channels = feature  # output channels become next input
        
        # Bottleneck — deepest layer
        self.bottleneck = DoubleConv(features[-1], features[-1] * 2)  # 512 → 1024
        
        # Build decoder (upsampling path)
        for feature in reversed(features):
            # Transposed conv to upsample spatial dimensions by 2x
            self.decoder.append(nn.ConvTranspose2d(feature * 2, feature, kernel_size=2, stride=2))
            # After concatenation with skip connection, input channels = feature * 2
            self.decoder.append(DoubleConv(feature * 2, feature))
        
        # Final 1×1 convolution maps to desired number of classes
        self.final_conv = nn.Conv2d(features[0], out_channels, kernel_size=1)
    
    def forward(self, x):
        skip_connections = []  # store encoder outputs for skip connections
        
        # Encoder path
        for encode_block in self.encoder:
            x = encode_block(x)
            skip_connections.append(x)  # save before pooling
            x = self.pool(x)
        
        # Bottleneck
        x = self.bottleneck(x)
        
        # Reverse skip connections (deepest first)
        skip_connections = skip_connections[::-1]
        
        # Decoder path
        for idx in range(0, len(self.decoder), 2):
            x = self.decoder[idx](x)       # upsample (transposed conv)
            skip = skip_connections[idx // 2]
            
            # Handle size mismatch (if input isn't perfectly divisible by 16)
            if x.shape != skip.shape:
                x = F.interpolate(x, size=skip.shape[2:], mode='bilinear', align_corners=True)
            
            x = torch.cat((skip, x), dim=1)  # concatenate along channel dimension
            x = self.decoder[idx + 1](x)     # double conv after concatenation
        
        return self.final_conv(x)


# --- Usage Example ---
if __name__ == "__main__":
    # Create model: 3 input channels (RGB), 1 output channel (binary segmentation)
    model = UNet(in_channels=3, out_channels=1, features=[64, 128, 256, 512])
    
    # Random input: batch_size=2, 3 channels, 256×256 image
    x = torch.randn(2, 3, 256, 256)
    output = model(x)
    
    print(f"Input shape:  {x.shape}")      # [2, 3, 256, 256]
    print(f"Output shape: {output.shape}")  # [2, 1, 256, 256]
    print(f"Parameters:   {sum(p.numel() for p in model.parameters()):,}")
    # Output:
    # Input shape:  torch.Size([2, 3, 256, 256])
    # Output shape: torch.Size([2, 1, 256, 256])
    # Parameters:   31,043,521
```

#### Advanced: U-Net with Dice Loss for Training

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, Dataset

class DiceLoss(nn.Module):
    """Dice Loss — better than BCE for imbalanced segmentation tasks."""
    def __init__(self, smooth=1.0):
        super().__init__()
        self.smooth = smooth  # prevents division by zero
    
    def forward(self, predictions, targets):
        predictions = torch.sigmoid(predictions)  # convert logits to probabilities
        
        # Flatten spatial dimensions
        predictions = predictions.view(-1)
        targets = targets.view(-1)
        
        intersection = (predictions * targets).sum()
        dice = (2.0 * intersection + self.smooth) / (
            predictions.sum() + targets.sum() + self.smooth
        )
        return 1 - dice  # we minimize loss, so 1 - dice_coefficient


class CombinedLoss(nn.Module):
    """Combine BCE + Dice for best of both worlds."""
    def __init__(self, bce_weight=0.5, dice_weight=0.5):
        super().__init__()
        self.bce = nn.BCEWithLogitsLoss()
        self.dice = DiceLoss()
        self.bce_weight = bce_weight
        self.dice_weight = dice_weight
    
    def forward(self, predictions, targets):
        return (self.bce_weight * self.bce(predictions, targets) + 
                self.dice_weight * self.dice(predictions, targets))


# Training loop
def train_unet(model, train_loader, epochs=50, lr=1e-4):
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    model = model.to(device)
    
    criterion = CombinedLoss()
    optimizer = optim.Adam(model.parameters(), lr=lr)
    scheduler = optim.lr_scheduler.ReduceLROnPlateau(optimizer, patience=5, factor=0.5)
    
    for epoch in range(epochs):
        model.train()
        epoch_loss = 0
        
        for batch_idx, (images, masks) in enumerate(train_loader):
            images = images.to(device)
            masks = masks.to(device)
            
            # Forward pass
            predictions = model(images)
            loss = criterion(predictions, masks)
            
            # Backward pass
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
            epoch_loss += loss.item()
        
        avg_loss = epoch_loss / len(train_loader)
        scheduler.step(avg_loss)
        
        if (epoch + 1) % 10 == 0:
            print(f"Epoch [{epoch+1}/{epochs}] Loss: {avg_loss:.4f}")
```

### Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using only BCE loss | Doesn't handle class imbalance (most pixels are background) | Use Dice Loss or combined BCE+Dice |
| Input size not divisible by 16 | Feature maps don't align in skip connections | Pad inputs or use `F.interpolate` for alignment |
| Not using data augmentation | Overfitting with small medical datasets | Apply elastic deformations, flips, rotations |
| Ignoring boundary pixels | Model predicts blobs instead of precise edges | Use weighted cross-entropy with higher weight on boundaries |
| Too deep for small images | Information gets lost in too many pooling operations | Reduce depth or use smaller pooling |

### Pro Tips

> **Tip 1**: For 3D medical data (CT/MRI volumes), use **3D U-Net** — replace all 2D operations with 3D equivalents.

> **Tip 2**: **Attention U-Net** adds attention gates to skip connections, letting the model focus on relevant regions and ignore irrelevant background.

> **Tip 3**: Use **test-time augmentation (TTA)** — predict on flipped/rotated versions and average results for better segmentation masks.

---

## 10.2 EfficientNet — Scaling Done Right

### What It Is
EfficientNet is a family of models (B0 to B7) that found the **optimal way to scale up** a neural network. Instead of blindly making networks deeper or wider, it scales depth, width, and resolution together using a **compound scaling** method. Think of it like growing a plant proportionally — you don't just make the trunk taller, you also thicken it and spread the branches.

### Why It Matters
- **State-of-the-art accuracy** with far fewer parameters than alternatives
- EfficientNet-B7 achieves better accuracy than GPipe (which has 8.4× more parameters)
- Perfect for **mobile/edge deployment** (B0-B2) and **research** (B5-B7)
- Foundation for EfficientDet (object detection) and EfficientNetV2

### How It Works

#### The Scaling Problem

Traditional approaches scale one dimension at a time:

| Strategy | Example | Problem |
|----------|---------|---------|
| Deeper | ResNet-18 → ResNet-152 | Vanishing gradients, diminishing returns |
| Wider | More filters per layer | Captures more fine-grained features, but plateaus quickly |
| Higher Resolution | 224→331→480 | More detail, but expensive, diminishing returns alone |

#### Compound Scaling (The Key Innovation)

EfficientNet scales all three dimensions simultaneously using a **compound coefficient** $\phi$:

$$\text{depth}: d = \alpha^{\phi}$$
$$\text{width}: w = \beta^{\phi}$$
$$\text{resolution}: r = \gamma^{\phi}$$

Subject to the constraint:

$$\alpha \cdot \beta^2 \cdot \gamma^2 \approx 2$$

This ensures that when $\phi$ increases by 1, FLOPS roughly double.

**Found values** (via grid search on B0):
- $\alpha = 1.2$ (depth)
- $\beta = 1.1$ (width)  
- $\gamma = 1.15$ (resolution)

```
               Compound Scaling
               
Width Only:    Depth Only:    Resolution Only:   COMPOUND (EfficientNet):
┌──────────┐   ┌────┐         ┌────────────┐     ┌──────────────┐
│          │   │    │         │            │     │              │
│          │   │    │         │            │     │              │
│          │   │    │         │            │     │              │
│          │   │    │         │            │     │              │
│          │   │    │         │            │     │              │
│          │   │    │         │            │     │              │
└──────────┘   │    │         └────────────┘     │              │
               │    │                            │              │
               │    │                            │              │
               │    │                            └──────────────┘
               │    │                            
               └────┘         Bigger input,       All three grow
Wide but       Deep but      but same model       proportionally!
shallow        narrow        capacity
```

#### MBConv Block (Building Block)

EfficientNet is built from **Mobile Inverted Bottleneck Convolution (MBConv)** blocks:

```
Input
  │
  ▼
[1×1 Conv] ──── Expand channels (×6 expansion ratio)
  │
  ▼
[Depthwise 3×3 or 5×5 Conv] ──── Spatial filtering (efficient!)
  │
  ▼
[Squeeze-and-Excitation] ──── Channel attention (learns which channels matter)
  │
  ▼
[1×1 Conv] ──── Project back to smaller channels
  │
  ▼
[+ Residual Connection] ──── Add input (if dimensions match)
  │
  ▼
Output
```

**Squeeze-and-Excitation (SE) block:**

$$s = \sigma(W_2 \cdot \text{ReLU}(W_1 \cdot \text{GAP}(x)))$$

Where GAP = Global Average Pooling. The SE block learns to **weight each channel** by importance.

#### EfficientNet Family

| Model | Resolution | Parameters | Top-1 Acc (ImageNet) | FLOPS |
|-------|-----------|------------|---------------------|-------|
| B0 | 224 | 5.3M | 77.1% | 0.39B |
| B1 | 240 | 7.8M | 79.1% | 0.70B |
| B2 | 260 | 9.2M | 80.1% | 1.0B |
| B3 | 300 | 12M | 81.6% | 1.8B |
| B4 | 380 | 19M | 82.9% | 4.2B |
| B5 | 456 | 30M | 83.6% | 9.9B |
| B6 | 528 | 43M | 84.0% | 19B |
| B7 | 600 | 66M | 84.3% | 37B |

### Code Example

```python
import torch
import torch.nn as nn
import torchvision.models as models
from torchvision import transforms

# --- Using Pretrained EfficientNet ---

# Load pretrained EfficientNet-B0
model = models.efficientnet_b0(weights='IMAGENET1K_V1')

# Check architecture
print(f"Total parameters: {sum(p.numel() for p in model.parameters()):,}")
# Output: Total parameters: 5,288,548

# Modify for custom number of classes (e.g., 10 classes)
num_classes = 10
model.classifier = nn.Sequential(
    nn.Dropout(p=0.2, inplace=True),  # regularization
    nn.Linear(model.classifier[1].in_features, num_classes)  # 1280 → 10
)

# Proper preprocessing for EfficientNet
preprocess = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),  # B0 uses 224×224
    transforms.ToTensor(),
    transforms.Normalize(
        mean=[0.485, 0.456, 0.406],  # ImageNet statistics
        std=[0.229, 0.224, 0.225]
    ),
])

# Forward pass
dummy_input = torch.randn(1, 3, 224, 224)
output = model(dummy_input)
print(f"Output shape: {output.shape}")  # [1, 10]
```

#### Advanced: Fine-Tuning with Progressive Resizing

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import models, transforms
from torch.utils.data import DataLoader

def create_efficientnet(model_name='efficientnet_b3', num_classes=100, pretrained=True):
    """Create EfficientNet with custom head for transfer learning."""
    
    # Load pretrained model
    weights = 'IMAGENET1K_V1' if pretrained else None
    model = getattr(models, model_name)(weights=weights)
    
    # Freeze backbone initially (for feature extraction phase)
    for param in model.features.parameters():
        param.requires_grad = False
    
    # Replace classifier head
    in_features = model.classifier[1].in_features
    model.classifier = nn.Sequential(
        nn.Dropout(p=0.3),
        nn.Linear(in_features, 512),
        nn.ReLU(),
        nn.Dropout(p=0.2),
        nn.Linear(512, num_classes)
    )
    
    return model


def progressive_training(model, train_dataset, val_dataset, device='cuda'):
    """
    Progressive resizing: start with small images, gradually increase.
    This acts as regularization and speeds up early training.
    """
    resolutions = [224, 260, 300]  # progressively increase
    epochs_per_resolution = [10, 10, 20]
    learning_rates = [1e-3, 5e-4, 1e-4]
    
    model = model.to(device)
    
    for res, epochs, lr in zip(resolutions, epochs_per_resolution, learning_rates):
        print(f"\n{'='*50}")
        print(f"Training at resolution: {res}×{res}, LR: {lr}, Epochs: {epochs}")
        print(f"{'='*50}")
        
        # Update transforms for current resolution
        train_transform = transforms.Compose([
            transforms.RandomResizedCrop(res),
            transforms.RandomHorizontalFlip(),
            transforms.ColorJitter(0.2, 0.2, 0.2),
            transforms.ToTensor(),
            transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
        ])
        
        train_dataset.transform = train_transform
        train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True, num_workers=4)
        
        # Unfreeze backbone for fine-tuning at higher resolutions
        if res > 224:
            for param in model.features[-3:].parameters():  # unfreeze last 3 blocks
                param.requires_grad = True
        
        optimizer = optim.AdamW(
            filter(lambda p: p.requires_grad, model.parameters()),
            lr=lr, weight_decay=1e-4
        )
        criterion = nn.CrossEntropyLoss(label_smoothing=0.1)  # label smoothing helps
        
        for epoch in range(epochs):
            model.train()
            running_loss = 0.0
            correct = 0
            total = 0
            
            for images, labels in train_loader:
                images, labels = images.to(device), labels.to(device)
                
                optimizer.zero_grad()
                outputs = model(images)
                loss = criterion(outputs, labels)
                loss.backward()
                
                # Gradient clipping — prevents exploding gradients during fine-tuning
                torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
                optimizer.step()
                
                running_loss += loss.item()
                _, predicted = outputs.max(1)
                total += labels.size(0)
                correct += predicted.eq(labels).sum().item()
            
            acc = 100.0 * correct / total
            avg_loss = running_loss / len(train_loader)
            
            if (epoch + 1) % 5 == 0:
                print(f"  Epoch {epoch+1}/{epochs} — Loss: {avg_loss:.4f}, Acc: {acc:.2f}%")
    
    return model


# Usage
model = create_efficientnet('efficientnet_b3', num_classes=100)
print(f"Trainable params: {sum(p.numel() for p in model.parameters() if p.requires_grad):,}")
# Initially only classifier is trainable
```

### Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using wrong input resolution | Each B-variant expects specific resolution | B0=224, B1=240, B2=260, B3=300, B4=380, B5=456, B6=528, B7=600 |
| Not using proper normalization | ImageNet pretrained weights expect specific stats | Always use mean=[0.485,0.456,0.406], std=[0.229,0.224,0.225] |
| Unfreezing all layers immediately | Destroys pretrained features | Freeze backbone first, train head, then gradually unfreeze |
| Using too large batch sizes with B5+ | OOM errors on GPU | Reduce batch size or use gradient accumulation |
| Ignoring stochastic depth | Missing important regularization | EfficientNet uses stochastic depth (drop_connect_rate) |

---

## 10.3 Vision Transformers (ViT)

### What It Is
Vision Transformer (ViT) applies the **Transformer architecture** (originally designed for text) directly to images. Instead of processing words, it splits an image into patches (like puzzle pieces) and processes them as a sequence. It proved that you **don't need convolutions** for computer vision — pure attention can match or beat CNNs.

### Why It Matters
- Demonstrated that **attention alone** can achieve SOTA on image classification
- Foundation for modern vision models (DINO, DINOv2, SAM, CLIP)
- Scales better than CNNs with more data and compute
- Unified architecture for vision + language (multimodal models)
- Powers modern image generation (Stable Diffusion uses transformer-like components)

### How It Works

#### From Text to Images — The Key Idea

Text Transformer: sentence → split into **words/tokens** → process sequence

Vision Transformer: image → split into **patches** → treat patches as tokens → process sequence

```
┌─────────────────────────────────────────────────────┐
│                  ORIGINAL IMAGE (224×224)            │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┐    │
│  │ P1 │ P2 │ P3 │ P4 │ P5 │ P6 │ P7 │ P8 │... │    │
│  ├────┼────┼────┼────┼────┼────┼────┼────┼────┤    │
│  │P15 │P16 │P17 │P18 │P19 │P20 │P21 │P22 │... │    │
│  ├────┼────┼────┼────┼────┼────┼────┼────┼────┤    │
│  │... │... │... │... │... │... │... │... │... │    │
│  └────┴────┴────┴────┴────┴────┴────┴────┴────┘    │
│                                                       │
│  Each patch = 16×16 pixels                           │
│  224/16 = 14 patches per side                        │
│  Total: 14×14 = 196 patches                         │
└─────────────────────────────────────────────────────┘

         │  Flatten each patch → Linear Projection
         ▼

┌─────────────────────────────────────────────────────┐
│  Patch Embeddings + Position Embeddings              │
│                                                       │
│  [CLS] [P1] [P2] [P3] ... [P196]                    │
│    │     │    │    │         │                       │
│    ▼     ▼    ▼    ▼         ▼                       │
│  ┌─────────────────────────────────────────────┐     │
│  │         Transformer Encoder × L             │     │
│  │  ┌───────────────────────────────────────┐  │     │
│  │  │  Multi-Head Self-Attention            │  │     │
│  │  │  LayerNorm + Residual                 │  │     │
│  │  │  MLP (FFN)                            │  │     │
│  │  │  LayerNorm + Residual                 │  │     │
│  │  └───────────────────────────────────────┘  │     │
│  └─────────────────────────────────────────────┘     │
│    │                                                  │
│    ▼                                                  │
│  [CLS] token → MLP Head → Class Prediction           │
└─────────────────────────────────────────────────────┘
```

#### Step-by-Step Process

**Step 1: Patch Embedding**

Split image $x \in \mathbb{R}^{H \times W \times C}$ into $N$ patches of size $P \times P$:

$$N = \frac{H \times W}{P^2}$$

Each patch is flattened and linearly projected:

$$z_0^i = x_p^i \cdot E + e_{pos}^i$$

Where $E \in \mathbb{R}^{P^2 \cdot C \times D}$ is the projection matrix, $D$ is the embedding dimension.

**Step 2: Add [CLS] Token + Position Embeddings**

$$z_0 = [x_{cls}; x_p^1 E; x_p^2 E; \ldots; x_p^N E] + E_{pos}$$

- $x_{cls}$: learnable classification token (like BERT's [CLS])
- $E_{pos} \in \mathbb{R}^{(N+1) \times D}$: learnable position embeddings

**Step 3: Transformer Encoder (×L layers)**

Each layer consists of:

$$z'_l = \text{MSA}(\text{LN}(z_{l-1})) + z_{l-1}$$
$$z_l = \text{MLP}(\text{LN}(z'_l)) + z'_l$$

Where MSA = Multi-head Self-Attention, LN = Layer Normalization.

**Step 4: Classification**

$$y = \text{MLP}_{head}(\text{LN}(z_L^0))$$

Only the [CLS] token's output is used for classification.

#### ViT Variants

| Model | Layers | Hidden Size | MLP Size | Heads | Parameters |
|-------|--------|-------------|----------|-------|-----------|
| ViT-Base (ViT-B/16) | 12 | 768 | 3072 | 12 | 86M |
| ViT-Large (ViT-L/16) | 24 | 1024 | 4096 | 16 | 307M |
| ViT-Huge (ViT-H/14) | 32 | 1280 | 5120 | 16 | 632M |

The `/16` or `/14` refers to patch size.

#### CNN vs ViT — Key Differences

| Aspect | CNN | ViT |
|--------|-----|-----|
| Inductive bias | Locality + translation invariance | Minimal — learns everything from data |
| Small data | Better (built-in priors help) | Worse (needs lots of data to learn spatial structure) |
| Large data | Plateaus eventually | Keeps improving (better scaling) |
| Receptive field | Grows gradually (local → global) | Global from layer 1 (every patch attends to every other) |
| Compute | Efficient for high-res images | Quadratic with number of patches |

### Code Example

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class PatchEmbedding(nn.Module):
    """Split image into patches and project to embedding dimension."""
    def __init__(self, img_size=224, patch_size=16, in_channels=3, embed_dim=768):
        super().__init__()
        self.img_size = img_size
        self.patch_size = patch_size
        self.n_patches = (img_size // patch_size) ** 2  # 196 for 224/16
        
        # Conv2d with kernel_size=patch_size and stride=patch_size
        # This is equivalent to splitting into patches + linear projection
        self.proj = nn.Conv2d(
            in_channels, embed_dim,
            kernel_size=patch_size, stride=patch_size
        )
    
    def forward(self, x):
        # x: (batch, 3, 224, 224)
        x = self.proj(x)         # (batch, embed_dim, 14, 14)
        x = x.flatten(2)         # (batch, embed_dim, 196)
        x = x.transpose(1, 2)   # (batch, 196, embed_dim)
        return x


class MultiHeadAttention(nn.Module):
    """Multi-Head Self-Attention mechanism."""
    def __init__(self, embed_dim=768, num_heads=12, dropout=0.0):
        super().__init__()
        self.num_heads = num_heads
        self.head_dim = embed_dim // num_heads  # 768/12 = 64
        self.scale = self.head_dim ** -0.5       # 1/sqrt(d_k) for scaled dot-product
        
        self.qkv = nn.Linear(embed_dim, embed_dim * 3)  # project to Q, K, V simultaneously
        self.proj = nn.Linear(embed_dim, embed_dim)       # output projection
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, x):
        batch_size, seq_len, embed_dim = x.shape
        
        # Generate Q, K, V
        qkv = self.qkv(x).reshape(batch_size, seq_len, 3, self.num_heads, self.head_dim)
        qkv = qkv.permute(2, 0, 3, 1, 4)  # (3, batch, heads, seq_len, head_dim)
        q, k, v = qkv[0], qkv[1], qkv[2]
        
        # Scaled dot-product attention
        attn = (q @ k.transpose(-2, -1)) * self.scale  # (batch, heads, seq, seq)
        attn = attn.softmax(dim=-1)
        attn = self.dropout(attn)
        
        # Apply attention to values
        x = (attn @ v).transpose(1, 2).reshape(batch_size, seq_len, embed_dim)
        x = self.proj(x)
        
        return x


class TransformerBlock(nn.Module):
    """Single Transformer encoder block."""
    def __init__(self, embed_dim=768, num_heads=12, mlp_ratio=4.0, dropout=0.0):
        super().__init__()
        self.norm1 = nn.LayerNorm(embed_dim)
        self.attn = MultiHeadAttention(embed_dim, num_heads, dropout)
        self.norm2 = nn.LayerNorm(embed_dim)
        self.mlp = nn.Sequential(
            nn.Linear(embed_dim, int(embed_dim * mlp_ratio)),  # 768 → 3072
            nn.GELU(),                                          # GELU > ReLU for transformers
            nn.Dropout(dropout),
            nn.Linear(int(embed_dim * mlp_ratio), embed_dim),  # 3072 → 768
            nn.Dropout(dropout),
        )
    
    def forward(self, x):
        # Pre-norm architecture (norm before attention/MLP)
        x = x + self.attn(self.norm1(x))   # residual + attention
        x = x + self.mlp(self.norm2(x))    # residual + MLP
        return x


class VisionTransformer(nn.Module):
    """Complete Vision Transformer (ViT) implementation."""
    def __init__(
        self,
        img_size=224,
        patch_size=16,
        in_channels=3,
        num_classes=1000,
        embed_dim=768,
        depth=12,
        num_heads=12,
        mlp_ratio=4.0,
        dropout=0.1,
    ):
        super().__init__()
        
        # Patch embedding
        self.patch_embed = PatchEmbedding(img_size, patch_size, in_channels, embed_dim)
        n_patches = self.patch_embed.n_patches
        
        # Learnable [CLS] token
        self.cls_token = nn.Parameter(torch.zeros(1, 1, embed_dim))
        
        # Learnable position embeddings (for N patches + 1 CLS token)
        self.pos_embed = nn.Parameter(torch.zeros(1, n_patches + 1, embed_dim))
        self.pos_drop = nn.Dropout(dropout)
        
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
        batch_size = x.shape[0]
        
        # Create patch embeddings
        x = self.patch_embed(x)  # (batch, 196, 768)
        
        # Prepend [CLS] token
        cls_tokens = self.cls_token.expand(batch_size, -1, -1)  # (batch, 1, 768)
        x = torch.cat([cls_tokens, x], dim=1)  # (batch, 197, 768)
        
        # Add position embeddings
        x = x + self.pos_embed
        x = self.pos_drop(x)
        
        # Pass through transformer blocks
        x = self.blocks(x)
        
        # Take [CLS] token output for classification
        x = self.norm(x[:, 0])  # (batch, 768) — only CLS token
        x = self.head(x)        # (batch, num_classes)
        
        return x


# --- Usage ---
if __name__ == "__main__":
    # Create ViT-Base/16
    model = VisionTransformer(
        img_size=224,
        patch_size=16,
        num_classes=10,
        embed_dim=768,
        depth=12,
        num_heads=12
    )
    
    x = torch.randn(4, 3, 224, 224)  # batch of 4 images
    output = model(x)
    
    print(f"Input:  {x.shape}")         # [4, 3, 224, 224]
    print(f"Output: {output.shape}")    # [4, 10]
    print(f"Params: {sum(p.numel() for p in model.parameters()):,}")
    # Output:
    # Input:  torch.Size([4, 3, 224, 224])
    # Output: torch.Size([4, 10])
    # Params: 85,806,346
```

#### Using Pretrained ViT from HuggingFace

```python
import torch
from transformers import ViTForImageClassification, ViTFeatureExtractor
from PIL import Image
import requests

# Load pretrained ViT
model_name = "google/vit-base-patch16-224"
model = ViTForImageClassification.from_pretrained(model_name)
feature_extractor = ViTFeatureExtractor.from_pretrained(model_name)

# Download sample image
url = "http://images.cocodataset.org/val2017/000000039769.jpg"
image = Image.open(requests.get(url, stream=True).raw)

# Preprocess and predict
inputs = feature_extractor(images=image, return_tensors="pt")
outputs = model(**inputs)
predicted_class = outputs.logits.argmax(-1).item()

print(f"Predicted class: {model.config.id2label[predicted_class]}")
# Output: Predicted class: Egyptian cat

# Fine-tuning ViT for custom dataset
from transformers import ViTForImageClassification, TrainingArguments, Trainer

# Load model with custom num_labels
model = ViTForImageClassification.from_pretrained(
    model_name,
    num_labels=10,                    # your number of classes
    ignore_mismatched_sizes=True      # allows changing classifier head
)

# Freeze all layers except classifier (for quick fine-tuning)
for param in model.vit.parameters():
    param.requires_grad = False

# Only classifier head is trainable
trainable = sum(p.numel() for p in model.parameters() if p.requires_grad)
total = sum(p.numel() for p in model.parameters())
print(f"Trainable: {trainable:,} / {total:,} ({100*trainable/total:.1f}%)")
```

### Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Training ViT from scratch on small data | Needs 14M+ images to learn spatial priors | Always use pretrained weights and fine-tune |
| Using wrong patch size for your resolution | Incorrect number of patches | Ensure img_size is divisible by patch_size |
| Ignoring position embeddings during fine-tuning at different resolutions | Position embeddings won't match | Interpolate position embeddings to new resolution |
| Not using enough augmentation | ViT overfits more easily than CNNs | Use strong augmentation (RandAugment, Mixup, CutMix) |
| Forgetting GELU activation | ReLU doesn't work as well in transformers | Always use GELU in transformer MLPs |

---

## 10.4 Diffusion Models

### What It Is
Diffusion models generate new data (images, audio, etc.) by learning to **reverse a noise-adding process**. Imagine taking a clear photo and slowly adding static/snow until it's pure noise. The model learns to do the reverse — take pure noise and gradually "denoise" it back into a beautiful image. This is the technology behind **Stable Diffusion, DALL-E 2, Midjourney, and Imagen**.

### Why It Matters
- **State-of-the-art image generation** — surpassed GANs in quality and diversity
- Power text-to-image systems (Stable Diffusion, DALL-E)
- More stable training than GANs (no mode collapse, no adversarial training)
- Applications: image generation, inpainting, super-resolution, video generation, drug discovery, 3D generation

### How It Works

#### Core Intuition

Think of it like a sculptor:
- **Forward process (adding noise)**: Take a marble statue and slowly chip random pieces off until it's just a shapeless block
- **Reverse process (denoising)**: Learn to sculpt a block of marble back into a statue, one careful chip at a time

```
FORWARD PROCESS (Fixed — just adding noise):

Clear Image → Slightly Noisy → More Noisy → ... → Pure Gaussian Noise
   x₀     →      x₁       →     x₂    → ... →       xₜ

   🖼️    →      🖼️+noise  →   🌫️     → ... →       📺(static)
   
   t=0           t=1            t=2              t=T (e.g., 1000)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

REVERSE PROCESS (Learned — neural network predicts noise to remove):

Pure Noise → Less Noisy → Less Noisy → ... → Generated Image
   xₜ     →    x_{t-1}  →   x_{t-2}  → ... →       x₀

   📺     →     🌫️      →    🖼️+noise → ... →      🖼️
   
   t=T           t=T-1         t=T-2             t=0
```

#### Mathematical Framework

**Forward Process (q)** — Add noise gradually:

$$q(x_t | x_{t-1}) = \mathcal{N}(x_t; \sqrt{1-\beta_t} \cdot x_{t-1}, \beta_t \mathbf{I})$$

Where $\beta_t$ is a noise schedule (small values, increasing from $\beta_1 = 10^{-4}$ to $\beta_T = 0.02$).

**Key trick** — Jump directly to any timestep $t$ without iterating:

$$q(x_t | x_0) = \mathcal{N}(x_t; \sqrt{\bar{\alpha}_t} \cdot x_0, (1 - \bar{\alpha}_t)\mathbf{I})$$

Where $\alpha_t = 1 - \beta_t$ and $\bar{\alpha}_t = \prod_{s=1}^{t} \alpha_s$

This means: $x_t = \sqrt{\bar{\alpha}_t} \cdot x_0 + \sqrt{1 - \bar{\alpha}_t} \cdot \epsilon$, where $\epsilon \sim \mathcal{N}(0, \mathbf{I})$

**Reverse Process (p)** — The neural network learns:

$$p_\theta(x_{t-1} | x_t) = \mathcal{N}(x_{t-1}; \mu_\theta(x_t, t), \sigma_t^2 \mathbf{I})$$

**Training Objective** — Simplified loss (predict the noise):

$$\mathcal{L}_{simple} = \mathbb{E}_{t, x_0, \epsilon}\left[\|\epsilon - \epsilon_\theta(x_t, t)\|^2\right]$$

The network $\epsilon_\theta$ takes a noisy image $x_t$ and timestep $t$, and predicts the noise $\epsilon$ that was added.

#### The Training Algorithm (DDPM)

```
Algorithm: Training
━━━━━━━━━━━━━━━━━━
1. Sample clean image x₀ from dataset
2. Sample random timestep t ~ Uniform(1, T)
3. Sample noise ε ~ N(0, I)
4. Create noisy image: x_t = √(ᾱ_t) · x₀ + √(1-ᾱ_t) · ε
5. Predict noise: ε̂ = model(x_t, t)
6. Loss = MSE(ε, ε̂)
7. Backpropagate and update model

Algorithm: Sampling (Generation)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Sample x_T ~ N(0, I)  (pure noise)
2. For t = T, T-1, ..., 1:
   a. Predict noise: ε̂ = model(x_t, t)
   b. Compute mean: μ = (1/√α_t)(x_t - (β_t/√(1-ᾱ_t))·ε̂)
   c. Sample x_{t-1} = μ + σ_t · z  (z ~ N(0,I) if t>1, else z=0)
3. Return x₀
```

#### The U-Net in Diffusion Models

The noise predictor $\epsilon_\theta$ is typically a **U-Net** with modifications:
- **Time conditioning**: Timestep $t$ is embedded via sinusoidal embeddings and injected into each block
- **Attention layers**: Self-attention at lower resolutions for global context
- **Cross-attention**: For text-conditioning (text-to-image)
- **ResNet blocks**: With GroupNorm instead of BatchNorm

```
                     Time Embedding (sinusoidal)
                            │
                            ▼
Noisy Image ──→ [Encoder with attention] ──→ [Bottleneck] ──→ [Decoder] ──→ Predicted Noise
   (x_t)            + skip connections                                          (ε̂)
                            ↑
                    Text Embedding (for text-to-image)
                    via Cross-Attention
```

### Code Example

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch.utils.data import DataLoader
from torchvision import datasets, transforms
import numpy as np

# --- Noise Schedule ---
def linear_beta_schedule(timesteps, beta_start=1e-4, beta_end=0.02):
    """Linear noise schedule from DDPM paper."""
    return torch.linspace(beta_start, beta_end, timesteps)

def cosine_beta_schedule(timesteps, s=0.008):
    """Cosine schedule — better for high-resolution images."""
    steps = timesteps + 1
    x = torch.linspace(0, timesteps, steps)
    alphas_cumprod = torch.cos(((x / timesteps) + s) / (1 + s) * torch.pi * 0.5) ** 2
    alphas_cumprod = alphas_cumprod / alphas_cumprod[0]
    betas = 1 - (alphas_cumprod[1:] / alphas_cumprod[:-1])
    return torch.clamp(betas, 0.0001, 0.9999)


class DiffusionModel:
    """DDPM (Denoising Diffusion Probabilistic Model) framework."""
    
    def __init__(self, timesteps=1000, beta_schedule='linear', device='cuda'):
        self.timesteps = timesteps
        self.device = device
        
        # Define noise schedule
        if beta_schedule == 'linear':
            self.betas = linear_beta_schedule(timesteps).to(device)
        else:
            self.betas = cosine_beta_schedule(timesteps).to(device)
        
        # Pre-compute useful quantities
        self.alphas = 1.0 - self.betas
        self.alphas_cumprod = torch.cumprod(self.alphas, dim=0)          # ᾱ_t
        self.alphas_cumprod_prev = F.pad(self.alphas_cumprod[:-1], (1, 0), value=1.0)
        
        # Calculations for diffusion q(x_t | x_0)
        self.sqrt_alphas_cumprod = torch.sqrt(self.alphas_cumprod)
        self.sqrt_one_minus_alphas_cumprod = torch.sqrt(1.0 - self.alphas_cumprod)
        
        # Calculations for posterior q(x_{t-1} | x_t, x_0)
        self.posterior_variance = (
            self.betas * (1.0 - self.alphas_cumprod_prev) / (1.0 - self.alphas_cumprod)
        )
    
    def q_sample(self, x_0, t, noise=None):
        """Forward process: add noise to x_0 to get x_t."""
        if noise is None:
            noise = torch.randn_like(x_0)
        
        # x_t = √(ᾱ_t) * x_0 + √(1-ᾱ_t) * ε
        sqrt_alphas_cumprod_t = self.sqrt_alphas_cumprod[t][:, None, None, None]
        sqrt_one_minus_t = self.sqrt_one_minus_alphas_cumprod[t][:, None, None, None]
        
        return sqrt_alphas_cumprod_t * x_0 + sqrt_one_minus_t * noise
    
    def compute_loss(self, model, x_0):
        """Training loss: predict noise from noisy image."""
        batch_size = x_0.shape[0]
        
        # Sample random timesteps
        t = torch.randint(0, self.timesteps, (batch_size,), device=self.device)
        
        # Sample noise
        noise = torch.randn_like(x_0)
        
        # Create noisy images
        x_t = self.q_sample(x_0, t, noise)
        
        # Predict noise
        noise_pred = model(x_t, t)
        
        # MSE loss between actual and predicted noise
        loss = F.mse_loss(noise_pred, noise)
        return loss
    
    @torch.no_grad()
    def p_sample(self, model, x_t, t):
        """Single reverse diffusion step."""
        betas_t = self.betas[t][:, None, None, None]
        sqrt_one_minus_alphas_cumprod_t = self.sqrt_one_minus_alphas_cumprod[t][:, None, None, None]
        sqrt_recip_alphas_t = (1.0 / torch.sqrt(self.alphas[t]))[:, None, None, None]
        
        # Predict noise
        noise_pred = model(x_t, t)
        
        # Compute mean of p(x_{t-1} | x_t)
        model_mean = sqrt_recip_alphas_t * (
            x_t - betas_t * noise_pred / sqrt_one_minus_alphas_cumprod_t
        )
        
        if t[0] > 0:
            noise = torch.randn_like(x_t)
            posterior_variance_t = self.posterior_variance[t][:, None, None, None]
            return model_mean + torch.sqrt(posterior_variance_t) * noise
        else:
            return model_mean
    
    @torch.no_grad()
    def sample(self, model, image_size, batch_size=16, channels=3):
        """Generate images by iterative denoising from pure noise."""
        model.eval()
        
        # Start from pure noise
        x = torch.randn(batch_size, channels, image_size, image_size, device=self.device)
        
        # Iteratively denoise
        for t_val in reversed(range(self.timesteps)):
            t = torch.full((batch_size,), t_val, device=self.device, dtype=torch.long)
            x = self.p_sample(model, x, t)
        
        # Clip to valid image range
        return torch.clamp(x, -1.0, 1.0)


# --- Simple U-Net for Noise Prediction ---
class SinusoidalPositionEmbedding(nn.Module):
    """Encode timestep as sinusoidal embedding (like in Transformer)."""
    def __init__(self, dim):
        super().__init__()
        self.dim = dim
    
    def forward(self, t):
        half_dim = self.dim // 2
        embeddings = np.log(10000) / (half_dim - 1)
        embeddings = torch.exp(torch.arange(half_dim, device=t.device) * -embeddings)
        embeddings = t[:, None] * embeddings[None, :]
        return torch.cat([embeddings.sin(), embeddings.cos()], dim=-1)


class SimpleUNetBlock(nn.Module):
    """Residual block with time conditioning."""
    def __init__(self, in_ch, out_ch, time_dim):
        super().__init__()
        self.conv1 = nn.Sequential(
            nn.GroupNorm(8, in_ch),
            nn.SiLU(),
            nn.Conv2d(in_ch, out_ch, 3, padding=1)
        )
        self.time_mlp = nn.Sequential(
            nn.SiLU(),
            nn.Linear(time_dim, out_ch)
        )
        self.conv2 = nn.Sequential(
            nn.GroupNorm(8, out_ch),
            nn.SiLU(),
            nn.Conv2d(out_ch, out_ch, 3, padding=1)
        )
        self.residual = nn.Conv2d(in_ch, out_ch, 1) if in_ch != out_ch else nn.Identity()
    
    def forward(self, x, t):
        h = self.conv1(x)
        # Add time information
        time_emb = self.time_mlp(t)[:, :, None, None]  # broadcast to spatial dims
        h = h + time_emb
        h = self.conv2(h)
        return h + self.residual(x)


class SimpleUNet(nn.Module):
    """Simplified U-Net for diffusion (works on 32×32 or 64×64 images)."""
    def __init__(self, in_channels=3, time_dim=256, channels=[64, 128, 256]):
        super().__init__()
        
        # Time embedding
        self.time_embed = nn.Sequential(
            SinusoidalPositionEmbedding(time_dim),
            nn.Linear(time_dim, time_dim),
            nn.SiLU(),
            nn.Linear(time_dim, time_dim),
        )
        
        # Encoder
        self.enc1 = SimpleUNetBlock(in_channels, channels[0], time_dim)
        self.enc2 = SimpleUNetBlock(channels[0], channels[1], time_dim)
        self.enc3 = SimpleUNetBlock(channels[1], channels[2], time_dim)
        
        self.pool = nn.MaxPool2d(2)
        
        # Bottleneck
        self.bot = SimpleUNetBlock(channels[2], channels[2], time_dim)
        
        # Decoder
        self.up3 = nn.ConvTranspose2d(channels[2], channels[2], 2, stride=2)
        self.dec3 = SimpleUNetBlock(channels[2] * 2, channels[1], time_dim)
        self.up2 = nn.ConvTranspose2d(channels[1], channels[1], 2, stride=2)
        self.dec2 = SimpleUNetBlock(channels[1] * 2, channels[0], time_dim)
        self.up1 = nn.ConvTranspose2d(channels[0], channels[0], 2, stride=2)
        self.dec1 = SimpleUNetBlock(channels[0] * 2, channels[0], time_dim)
        
        # Output
        self.out = nn.Conv2d(channels[0], in_channels, 1)
    
    def forward(self, x, t):
        # Time embedding
        t = self.time_embed(t.float())
        
        # Encoder
        e1 = self.enc1(x, t)
        e2 = self.enc2(self.pool(e1), t)
        e3 = self.enc3(self.pool(e2), t)
        
        # Bottleneck
        b = self.bot(self.pool(e3), t)
        
        # Decoder with skip connections
        d3 = self.dec3(torch.cat([self.up3(b), e3], dim=1), t)
        d2 = self.dec2(torch.cat([self.up2(d3), e2], dim=1), t)
        d1 = self.dec1(torch.cat([self.up1(d2), e1], dim=1), t)
        
        return self.out(d1)


# --- Training Loop ---
def train_diffusion():
    device = 'cuda' if torch.cuda.is_available() else 'cpu'
    
    # Hyperparameters
    image_size = 32
    batch_size = 64
    epochs = 100
    lr = 2e-4
    timesteps = 1000
    
    # Dataset (CIFAR-10 as example)
    transform = transforms.Compose([
        transforms.Resize(image_size),
        transforms.ToTensor(),
        transforms.Normalize([0.5], [0.5])  # scale to [-1, 1]
    ])
    dataset = datasets.CIFAR10(root='./data', train=True, download=True, transform=transform)
    dataloader = DataLoader(dataset, batch_size=batch_size, shuffle=True, num_workers=2)
    
    # Model and diffusion
    model = SimpleUNet(in_channels=3, time_dim=256, channels=[64, 128, 256]).to(device)
    diffusion = DiffusionModel(timesteps=timesteps, device=device)
    optimizer = torch.optim.Adam(model.parameters(), lr=lr)
    
    print(f"Model parameters: {sum(p.numel() for p in model.parameters()):,}")
    
    # Training
    for epoch in range(epochs):
        total_loss = 0
        for batch_idx, (images, _) in enumerate(dataloader):
            images = images.to(device)
            
            loss = diffusion.compute_loss(model, images)
            
            optimizer.zero_grad()
            loss.backward()
            torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
            optimizer.step()
            
            total_loss += loss.item()
        
        avg_loss = total_loss / len(dataloader)
        if (epoch + 1) % 10 == 0:
            print(f"Epoch {epoch+1}/{epochs}, Loss: {avg_loss:.4f}")
            
            # Generate samples
            samples = diffusion.sample(model, image_size, batch_size=16)
            print(f"  Generated sample range: [{samples.min():.2f}, {samples.max():.2f}]")


if __name__ == "__main__":
    train_diffusion()
```

#### DDIM — Fast Sampling (Fewer Steps)

```python
@torch.no_grad()
def ddim_sample(diffusion, model, image_size, batch_size=16, 
                channels=3, ddim_steps=50, eta=0.0):
    """
    DDIM sampling — generate images in fewer steps (50 instead of 1000).
    eta=0 → deterministic, eta=1 → equivalent to DDPM.
    """
    device = diffusion.device
    
    # Select subset of timesteps (evenly spaced)
    step_size = diffusion.timesteps // ddim_steps
    timesteps = list(range(0, diffusion.timesteps, step_size))
    timesteps = list(reversed(timesteps))  # from T to 0
    
    # Start from pure noise
    x = torch.randn(batch_size, channels, image_size, image_size, device=device)
    
    for i in range(len(timesteps) - 1):
        t_current = torch.full((batch_size,), timesteps[i], device=device, dtype=torch.long)
        t_next = torch.full((batch_size,), timesteps[i + 1], device=device, dtype=torch.long)
        
        # Get alpha values
        alpha_cumprod_t = diffusion.alphas_cumprod[t_current][:, None, None, None]
        alpha_cumprod_t_next = diffusion.alphas_cumprod[t_next][:, None, None, None]
        
        # Predict noise
        noise_pred = model(x, t_current)
        
        # Predict x_0
        x_0_pred = (x - torch.sqrt(1 - alpha_cumprod_t) * noise_pred) / torch.sqrt(alpha_cumprod_t)
        x_0_pred = torch.clamp(x_0_pred, -1, 1)  # clip for stability
        
        # Compute sigma for stochasticity
        sigma = eta * torch.sqrt(
            (1 - alpha_cumprod_t_next) / (1 - alpha_cumprod_t) * (1 - alpha_cumprod_t / alpha_cumprod_t_next)
        )
        
        # Predict direction
        direction = torch.sqrt(1 - alpha_cumprod_t_next - sigma**2) * noise_pred
        
        # Compute x_{t-1}
        x = torch.sqrt(alpha_cumprod_t_next) * x_0_pred + direction
        
        if eta > 0 and i < len(timesteps) - 2:
            x = x + sigma * torch.randn_like(x)
    
    return torch.clamp(x, -1.0, 1.0)
```

### Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using too few timesteps during training | Model can't learn fine denoising | Use T=1000 for training, then DDIM for fast sampling |
| Not normalizing images to [-1, 1] | Noise schedule assumes data in this range | Always normalize: `x = 2*x - 1` |
| Linear beta schedule for high-res | Too aggressive noise at start | Use cosine schedule for better results |
| Not clipping predictions | Numerical instability accumulates | Clip x_0 predictions to [-1, 1] during sampling |
| Ignoring EMA (Exponential Moving Average) | Final model has training noise | Always use EMA of model weights for sampling |

### Pro Tips

> **Classifier-Free Guidance (CFG)**: The technique behind high-quality text-to-image. During training, randomly drop the text condition (use empty string). During sampling: `noise_pred = noise_uncond + scale * (noise_cond - noise_uncond)`. Scale of 7.5 is typical.

> **Latent Diffusion (Stable Diffusion)**: Don't diffuse in pixel space! Encode images to a lower-dimensional latent space first (using a VAE), do diffusion there, then decode back. This makes training 4-8× cheaper.

---

## 10.5 Comparison and Selection Guide

### When to Use What

| Architecture | Best For | Data Requirement | Compute | Key Strength |
|-------------|----------|-----------------|---------|--------------|
| **U-Net** | Pixel-level segmentation | Small (100+ annotated) | Low-Medium | Works with few samples |
| **EfficientNet** | Classification (mobile to server) | Medium (1K+) | Low-High (B0-B7) | Best accuracy/efficiency ratio |
| **ViT** | Classification (large-scale) | Large (10K+ or pretrained) | High | Scales better with data |
| **Diffusion Models** | Image generation | Large (10K+) | Very High | Best generation quality |

### Architecture Decision Flowchart

```
What's your task?
│
├─── Classification?
│    ├─── Small dataset (<10K) → EfficientNet (transfer learning)
│    ├─── Medium dataset (10K-1M) → EfficientNet-B3/B4 or ViT (pretrained)
│    └─── Large dataset (1M+) → ViT-Large/Huge
│
├─── Segmentation?
│    ├─── Medical/few samples → U-Net
│    ├─── Real-time needed → U-Net with MobileNet encoder
│    └─── High accuracy → U-Net with EfficientNet/ViT encoder
│
├─── Generation?
│    ├─── Unconditional → DDPM/DDIM
│    ├─── Text-to-image → Latent Diffusion (Stable Diffusion)
│    └─── Fast generation needed → Consistency Models or DDIM (few steps)
│
└─── Detection?
     └─── See Object Detection chapter
```

---

## 10.6 Interview Questions

### Conceptual Questions

**Q1: Why does U-Net use skip connections? What would happen without them?**
> Skip connections carry fine-grained spatial information (edges, textures) from the encoder to the decoder. Without them, the decoder only has the compressed bottleneck features — resulting in blurry, imprecise segmentation boundaries. The decoder "knows" what objects are there but loses where exactly their boundaries are.

**Q2: Explain EfficientNet's compound scaling. Why is it better than scaling one dimension?**
> Compound scaling increases network depth ($\alpha^\phi$), width ($\beta^\phi$), and resolution ($\gamma^\phi$) simultaneously. Scaling only depth causes vanishing gradients; only width plateaus quickly; only resolution gains detail but model can't process it. Together, they balance: higher resolution needs more layers (depth) to process details, and wider layers to capture finer features.

**Q3: How does ViT handle the lack of inductive bias compared to CNNs?**
> CNNs have built-in locality (convolution kernels) and translation equivariance. ViT has no such priors — it must learn spatial relationships entirely from data. This means ViT needs significantly more training data (or pretraining on large datasets like ImageNet-21K or JFT-300M). However, once given enough data, ViT can discover better representations because it's not constrained by local receptive fields.

**Q4: In diffusion models, why do we predict noise $\epsilon$ instead of directly predicting $x_0$?**
> Predicting noise is mathematically equivalent but leads to more stable training. The noise has a standard normal distribution regardless of timestep, making the prediction target consistent across all timesteps. Predicting $x_0$ directly would require the model to output very different things depending on the noise level — near-perfect images at low timesteps and vague structure at high timesteps.

**Q5: What is classifier-free guidance and why does it produce better images?**
> During training, the text condition is randomly dropped (replaced with empty/null). During sampling, two predictions are made — one conditioned on text, one unconditional. The final prediction amplifies the difference: `output = uncond + scale * (cond - uncond)`. This "pushes" the generation toward images that strongly match the text prompt, at the cost of some diversity. Scale of 7.5 is typical for Stable Diffusion.

### Coding Questions

**Q6: Implement Dice Loss for segmentation.**
> See the U-Net section above for a complete implementation.

**Q7: Write the forward process of a diffusion model (adding noise at arbitrary timestep).**
> See `q_sample` in the Diffusion Models code section.

**Q8: What's the computational complexity of self-attention in ViT? How can it be reduced?**
> Self-attention is $O(N^2 \cdot D)$ where $N$ = number of patches. For a 224×224 image with 16×16 patches: $N=196$. For 512×512 with 16×16 patches: $N=1024$, making it 27× more expensive. Solutions: larger patches (ViT-H/14), windowed attention (Swin Transformer), linear attention approximations, or hierarchical designs.

---

## 10.7 Quick Reference

| Concept | Key Formula / Detail |
|---------|---------------------|
| U-Net structure | Encoder → Bottleneck → Decoder + Skip Connections |
| U-Net loss | Dice Loss: $1 - \frac{2 \sum x_i y_i}{\sum x_i + \sum y_i}$ |
| EfficientNet scaling | $d=\alpha^\phi$, $w=\beta^\phi$, $r=\gamma^\phi$, where $\alpha\beta^2\gamma^2 \approx 2$ |
| EfficientNet block | MBConv: Expand → Depthwise Conv → SE → Project |
| ViT patch embedding | Image → $N = (H/P)^2$ patches → Linear projection to $D$ dims |
| ViT total tokens | $N + 1$ (patches + [CLS] token) |
| ViT attention cost | $O(N^2 \cdot D)$ — quadratic with number of patches |
| Diffusion forward | $x_t = \sqrt{\bar\alpha_t} x_0 + \sqrt{1-\bar\alpha_t}\epsilon$ |
| Diffusion training loss | $\|\epsilon - \epsilon_\theta(x_t, t)\|^2$ |
| DDIM | Deterministic sampling in fewer steps (50 vs 1000) |
| Latent Diffusion | Diffuse in VAE latent space, not pixel space |
| Classifier-Free Guidance | `out = uncond + scale * (cond - uncond)` |

---

## Summary

```
Advanced Architectures Cheat Sheet:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
U-Net         → Segmentation specialist, skip connections are key
EfficientNet  → Best accuracy/size ratio, compound scaling
ViT           → Transformers for vision, needs big data or pretraining
Diffusion     → Best generation quality, learns to reverse noise
```

---

*Next: [Chapter 11 — Hyperparameter Tuning for Deep Learning](11-Hyperparameter-Tuning-DL.md)*
