# Chapter 08: Generative Adversarial Networks (GANs)

## Table of Contents
- [1. Introduction to GANs](#1-introduction-to-gans)
- [2. GAN Theory and Mathematics](#2-gan-theory-and-mathematics)
- [3. Vanilla GAN Implementation](#3-vanilla-gan-implementation)
- [4. DCGAN (Deep Convolutional GAN)](#4-dcgan-deep-convolutional-gan)
- [5. Conditional GAN (cGAN)](#5-conditional-gan-cgan)
- [6. Wasserstein GAN (WGAN)](#6-wasserstein-gan-wgan)
- [7. StyleGAN](#7-stylegan)
- [8. GAN Training Tricks and Best Practices](#8-gan-training-tricks-and-best-practices)
- [9. GAN Applications](#9-gan-applications)
- [10. Common Mistakes](#10-common-mistakes)
- [11. Interview Questions](#11-interview-questions)
- [12. Quick Reference](#12-quick-reference)

---

## 1. Introduction to GANs

### What It Is
A **Generative Adversarial Network** is two neural networks playing a game against each other:
- **Generator (G)**: Creates fake data trying to fool the discriminator (a counterfeiter)
- **Discriminator (D)**: Tries to distinguish real data from fake (a detective)

**Analogy**: Imagine a counterfeiter trying to make fake money, and a bank detective trying to catch fakes. Over time, the counterfeiter gets so good that even the detective can't tell the difference. That's a trained GAN — the generator produces data indistinguishable from real data.

### Why It Matters
- **Image Generation**: Create photorealistic faces, art, scenes (StyleGAN, DALL-E precursor)
- **Data Augmentation**: Generate synthetic training data
- **Super-Resolution**: Upscale low-res images (SRGAN)
- **Image-to-Image Translation**: Convert sketches to photos, day to night (pix2pix, CycleGAN)
- **Video Generation**: DeepFakes, video prediction
- **Drug Discovery**: Generate molecular structures
- **Anomaly Detection**: Discriminator scores as anomaly detector

### GAN Architecture Overview

```
                    ┌─────────────────────────────────────────┐
                    │              Training Loop               │
                    └─────────────────────────────────────────┘
                    
    Noise z ~ N(0,1)         Fake Images            Real Images
         │                       │                       │
         ▼                       ▼                       ▼
    ┌──────────┐          ┌──────────────┐        ┌──────────┐
    │Generator │──────────│Discriminator │────────│  Dataset  │
    │  G(z)    │          │   D(x)       │        │  (Real)   │
    └──────────┘          └──────────────┘        └──────────┘
         ▲                       │
         │                       ▼
         │                 Real or Fake?
         │                  (0 or 1)
         │                       │
         └───── Gradient ────────┘
              (make fakes more convincing)
```

---

## 2. GAN Theory and Mathematics

### The Minimax Game

The GAN training objective is a **minimax game**:

$$\min_G \max_D V(D, G) = \mathbb{E}_{x \sim p_{data}(x)}[\log D(x)] + \mathbb{E}_{z \sim p_z(z)}[\log(1 - D(G(z)))]$$

**Breaking it down:**
- $D(x)$: Probability that $x$ is real (should be close to 1 for real data)
- $D(G(z))$: Probability that generated sample is real (D wants this ≈ 0, G wants this ≈ 1)
- **Discriminator maximizes**: correctly classify real (high $\log D(x)$) and fake (high $\log(1-D(G(z)))$)
- **Generator minimizes**: make D(G(z)) close to 1, so $\log(1-D(G(z)))$ is minimized

### Optimal Discriminator

For a fixed generator G, the optimal discriminator is:

$$D^*_G(x) = \frac{p_{data}(x)}{p_{data}(x) + p_g(x)}$$

When G is perfect ($p_g = p_{data}$): $D^*(x) = 0.5$ everywhere (can't tell real from fake).

### Nash Equilibrium

At convergence:
- Generator produces data from the true distribution: $p_g = p_{data}$
- Discriminator outputs 0.5 for all inputs (50/50 guess)
- The global minimum of the minimax game equals $-\log 4$

### Jensen-Shannon Divergence

The GAN loss is related to the **JS divergence** between real and generated distributions:

$$V(D^*_G, G) = 2 \cdot JSD(p_{data} \| p_g) - \log 4$$

$$JSD(P \| Q) = \frac{1}{2}D_{KL}(P \| M) + \frac{1}{2}D_{KL}(Q \| M), \quad M = \frac{P+Q}{2}$$

### Non-Saturating Loss (Practical)

In practice, $\log(1 - D(G(z)))$ saturates when D is confident. Instead, we use:

**Generator loss** (maximize $\log D(G(z))$ instead of minimize $\log(1-D(G(z)))$):
$$\mathcal{L}_G = -\mathbb{E}_{z \sim p_z}[\log D(G(z))]$$

**Discriminator loss**:
$$\mathcal{L}_D = -\mathbb{E}_{x \sim p_{data}}[\log D(x)] - \mathbb{E}_{z \sim p_z}[\log(1 - D(G(z)))]$$

---

## 3. Vanilla GAN Implementation

### Code Example: GAN on MNIST

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms
from torch.utils.data import DataLoader
import matplotlib.pyplot as plt
import numpy as np

# ============================================================
# STEP 1: Generator Network
# ============================================================
class Generator(nn.Module):
    """
    Maps random noise z → fake image.
    Architecture: noise(100) → 256 → 512 → 1024 → 784 (28x28)
    """
    def __init__(self, latent_dim=100, img_dim=784):
        super(Generator, self).__init__()
        
        self.model = nn.Sequential(
            # Layer 1: Noise → 256 features
            nn.Linear(latent_dim, 256),
            nn.LeakyReLU(0.2),
            nn.BatchNorm1d(256),
            
            # Layer 2: 256 → 512 features
            nn.Linear(256, 512),
            nn.LeakyReLU(0.2),
            nn.BatchNorm1d(512),
            
            # Layer 3: 512 → 1024 features
            nn.Linear(512, 1024),
            nn.LeakyReLU(0.2),
            nn.BatchNorm1d(1024),
            
            # Output: 1024 → 784 (image pixels)
            nn.Linear(1024, img_dim),
            nn.Tanh(),  # Output in [-1, 1] to match normalized images
        )
    
    def forward(self, z):
        return self.model(z)

# ============================================================
# STEP 2: Discriminator Network
# ============================================================
class Discriminator(nn.Module):
    """
    Maps image → probability of being real.
    Architecture: 784 → 1024 → 512 → 256 → 1
    """
    def __init__(self, img_dim=784):
        super(Discriminator, self).__init__()
        
        self.model = nn.Sequential(
            # Layer 1: Image → 1024
            nn.Linear(img_dim, 1024),
            nn.LeakyReLU(0.2),
            nn.Dropout(0.3),  # Dropout helps prevent D from overpowering G
            
            # Layer 2: 1024 → 512
            nn.Linear(1024, 512),
            nn.LeakyReLU(0.2),
            nn.Dropout(0.3),
            
            # Layer 3: 512 → 256
            nn.Linear(512, 256),
            nn.LeakyReLU(0.2),
            nn.Dropout(0.3),
            
            # Output: Probability (real or fake)
            nn.Linear(256, 1),
            nn.Sigmoid(),  # Output probability in [0, 1]
        )
    
    def forward(self, x):
        return self.model(x)

# ============================================================
# STEP 3: Training Setup
# ============================================================
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
latent_dim = 100

# Initialize networks
G = Generator(latent_dim=latent_dim).to(device)
D = Discriminator().to(device)

# Optimizers — separate for G and D
# Lower learning rate for D to keep training balanced
optimizer_G = optim.Adam(G.parameters(), lr=2e-4, betas=(0.5, 0.999))
optimizer_D = optim.Adam(D.parameters(), lr=2e-4, betas=(0.5, 0.999))

# Binary Cross Entropy loss
criterion = nn.BCELoss()

# Data: MNIST normalized to [-1, 1] (to match Tanh output of G)
transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize([0.5], [0.5]),  # Maps [0,1] → [-1,1]
])
train_loader = DataLoader(
    datasets.MNIST('./data', train=True, transform=transform, download=True),
    batch_size=128, shuffle=True, drop_last=True
)

# ============================================================
# STEP 4: Training Loop
# ============================================================
num_epochs = 50
g_losses = []
d_losses = []

for epoch in range(num_epochs):
    for batch_idx, (real_images, _) in enumerate(train_loader):
        batch_size = real_images.size(0)
        real_images = real_images.view(batch_size, -1).to(device)
        
        # Labels
        real_labels = torch.ones(batch_size, 1).to(device)
        fake_labels = torch.zeros(batch_size, 1).to(device)
        
        # ─────────────────────────────────────────────
        # Train Discriminator: maximize log(D(x)) + log(1 - D(G(z)))
        # ─────────────────────────────────────────────
        optimizer_D.zero_grad()
        
        # Loss on real images
        outputs_real = D(real_images)
        d_loss_real = criterion(outputs_real, real_labels)
        
        # Loss on fake images
        z = torch.randn(batch_size, latent_dim).to(device)
        fake_images = G(z).detach()  # detach: don't compute G gradients here
        outputs_fake = D(fake_images)
        d_loss_fake = criterion(outputs_fake, fake_labels)
        
        # Total D loss
        d_loss = d_loss_real + d_loss_fake
        d_loss.backward()
        optimizer_D.step()
        
        # ─────────────────────────────────────────────
        # Train Generator: maximize log(D(G(z)))
        # ─────────────────────────────────────────────
        optimizer_G.zero_grad()
        
        z = torch.randn(batch_size, latent_dim).to(device)
        fake_images = G(z)
        outputs = D(fake_images)
        
        # G wants D to think fakes are real → use real_labels
        g_loss = criterion(outputs, real_labels)
        g_loss.backward()
        optimizer_G.step()
        
        g_losses.append(g_loss.item())
        d_losses.append(d_loss.item())
    
    # Print progress every 10 epochs
    if (epoch + 1) % 10 == 0:
        print(f"Epoch [{epoch+1}/{num_epochs}] | "
              f"D Loss: {d_loss.item():.4f} | G Loss: {g_loss.item():.4f} | "
              f"D(x): {outputs_real.mean().item():.3f} | "
              f"D(G(z)): {outputs.mean().item():.3f}")

# ============================================================
# STEP 5: Generate and Visualize Samples
# ============================================================
G.eval()
with torch.no_grad():
    z = torch.randn(64, latent_dim).to(device)
    generated = G(z).view(-1, 1, 28, 28).cpu()
    
    fig, axes = plt.subplots(8, 8, figsize=(10, 10))
    for i in range(64):
        axes[i//8, i%8].imshow(generated[i].squeeze(), cmap='gray')
        axes[i//8, i%8].axis('off')
    plt.suptitle(f'GAN Generated Digits (Epoch {num_epochs})')
    plt.tight_layout()
    plt.show()

# Plot training losses
plt.figure(figsize=(10, 5))
plt.plot(g_losses[::100], label='Generator', alpha=0.7)
plt.plot(d_losses[::100], label='Discriminator', alpha=0.7)
plt.xlabel('Iterations (x100)')
plt.ylabel('Loss')
plt.legend()
plt.title('GAN Training Loss')
plt.show()
```

---

## 4. DCGAN (Deep Convolutional GAN)

### What It Is
**DCGAN** replaces fully-connected layers with convolutional layers, following specific architectural guidelines that stabilize GAN training. It was the first GAN architecture that reliably produced good results.

### Key Architecture Rules (from the DCGAN paper)
1. Replace pooling with strided convolutions (D) / transposed convolutions (G)
2. Use BatchNorm in both G and D (except G output and D input)
3. Remove fully connected hidden layers
4. Use ReLU in G (except output: Tanh)
5. Use LeakyReLU in D

### Code Example: DCGAN

```python
import torch
import torch.nn as nn

class DCGANGenerator(nn.Module):
    """
    Generator using transposed convolutions.
    Input: noise vector z (latent_dim x 1 x 1)
    Output: image (channels x 64 x 64)
    
    Architecture: z → 4x4 → 8x8 → 16x16 → 32x32 → 64x64
    """
    def __init__(self, latent_dim=100, channels=3, features_g=64):
        super(DCGANGenerator, self).__init__()
        
        self.net = nn.Sequential(
            # Input: (latent_dim, 1, 1) → (features_g*8, 4, 4)
            self._block(latent_dim, features_g * 8, kernel_size=4, 
                       stride=1, padding=0),
            
            # (features_g*8, 4, 4) → (features_g*4, 8, 8)
            self._block(features_g * 8, features_g * 4, kernel_size=4, 
                       stride=2, padding=1),
            
            # (features_g*4, 8, 8) → (features_g*2, 16, 16)
            self._block(features_g * 4, features_g * 2, kernel_size=4, 
                       stride=2, padding=1),
            
            # (features_g*2, 16, 16) → (features_g, 32, 32)
            self._block(features_g * 2, features_g, kernel_size=4, 
                       stride=2, padding=1),
            
            # (features_g, 32, 32) → (channels, 64, 64)
            nn.ConvTranspose2d(features_g, channels, kernel_size=4, 
                              stride=2, padding=1),
            nn.Tanh(),  # Output in [-1, 1]
        )
    
    def _block(self, in_channels, out_channels, kernel_size, stride, padding):
        """ConvTranspose → BatchNorm → ReLU"""
        return nn.Sequential(
            nn.ConvTranspose2d(in_channels, out_channels, kernel_size, 
                              stride, padding, bias=False),
            nn.BatchNorm2d(out_channels),
            nn.ReLU(True),  # inplace=True for memory efficiency
        )
    
    def forward(self, z):
        # z shape: (batch, latent_dim) → reshape to (batch, latent_dim, 1, 1)
        return self.net(z.view(z.size(0), -1, 1, 1))


class DCGANDiscriminator(nn.Module):
    """
    Discriminator using strided convolutions.
    Input: image (channels x 64 x 64)
    Output: probability (scalar)
    
    Architecture: 64x64 → 32x32 → 16x16 → 8x8 → 4x4 → 1x1
    """
    def __init__(self, channels=3, features_d=64):
        super(DCGANDiscriminator, self).__init__()
        
        self.net = nn.Sequential(
            # (channels, 64, 64) → (features_d, 32, 32)
            # No BatchNorm on first layer!
            nn.Conv2d(channels, features_d, kernel_size=4, 
                     stride=2, padding=1),
            nn.LeakyReLU(0.2),
            
            # (features_d, 32, 32) → (features_d*2, 16, 16)
            self._block(features_d, features_d * 2, kernel_size=4, 
                       stride=2, padding=1),
            
            # (features_d*2, 16, 16) → (features_d*4, 8, 8)
            self._block(features_d * 2, features_d * 4, kernel_size=4, 
                       stride=2, padding=1),
            
            # (features_d*4, 8, 8) → (features_d*8, 4, 4)
            self._block(features_d * 4, features_d * 8, kernel_size=4, 
                       stride=2, padding=1),
            
            # (features_d*8, 4, 4) → (1, 1, 1)
            nn.Conv2d(features_d * 8, 1, kernel_size=4, stride=1, padding=0),
            nn.Sigmoid(),
        )
    
    def _block(self, in_channels, out_channels, kernel_size, stride, padding):
        """Conv → BatchNorm → LeakyReLU"""
        return nn.Sequential(
            nn.Conv2d(in_channels, out_channels, kernel_size, 
                     stride, padding, bias=False),
            nn.BatchNorm2d(out_channels),
            nn.LeakyReLU(0.2),
        )
    
    def forward(self, x):
        return self.net(x).view(-1, 1)


# ============================================================
# Weight Initialization (Critical for DCGAN!)
# ============================================================
def weights_init(m):
    """
    Custom weight initialization from DCGAN paper.
    All weights initialized from N(0, 0.02).
    """
    classname = m.__class__.__name__
    if classname.find('Conv') != -1:
        nn.init.normal_(m.weight.data, 0.0, 0.02)
    elif classname.find('BatchNorm') != -1:
        nn.init.normal_(m.weight.data, 1.0, 0.02)
        nn.init.constant_(m.bias.data, 0)

# Apply initialization
G = DCGANGenerator(latent_dim=100, channels=1).to(device)
D = DCGANDiscriminator(channels=1).to(device)
G.apply(weights_init)
D.apply(weights_init)

# Training follows same pattern as vanilla GAN but with images
# kept as 4D tensors (batch, channels, height, width)
```

> **Pro Tip**: For DCGAN, the learning rate of 0.0002 with Adam betas=(0.5, 0.999) is almost universally recommended. Don't change it unless you have a good reason.

---

## 5. Conditional GAN (cGAN)

### What It Is
A **Conditional GAN** allows you to control *what* the generator produces by feeding it a condition (like a class label). Instead of "generate any face," you can say "generate a face of a woman with glasses."

### How It Works

Both G and D receive the condition (class label, text, image):
- **Generator**: $G(z, c)$ — generates sample conditioned on class $c$
- **Discriminator**: $D(x, c)$ — evaluates if $x$ is real *for that specific class*

$$\min_G \max_D V(D, G) = \mathbb{E}_{x,c}[\log D(x, c)] + \mathbb{E}_{z,c}[\log(1 - D(G(z, c), c))]$$

### Code Example: Conditional GAN

```python
import torch
import torch.nn as nn
import torch.optim as optim

class ConditionalGenerator(nn.Module):
    def __init__(self, latent_dim=100, num_classes=10, img_dim=784):
        super(ConditionalGenerator, self).__init__()
        self.label_embedding = nn.Embedding(num_classes, num_classes)
        
        # Input: noise + embedded label
        self.model = nn.Sequential(
            nn.Linear(latent_dim + num_classes, 256),
            nn.LeakyReLU(0.2),
            nn.BatchNorm1d(256),
            nn.Linear(256, 512),
            nn.LeakyReLU(0.2),
            nn.BatchNorm1d(512),
            nn.Linear(512, 1024),
            nn.LeakyReLU(0.2),
            nn.BatchNorm1d(1024),
            nn.Linear(1024, img_dim),
            nn.Tanh(),
        )
    
    def forward(self, z, labels):
        # Embed labels and concatenate with noise
        label_embed = self.label_embedding(labels)
        gen_input = torch.cat([z, label_embed], dim=1)
        return self.model(gen_input)


class ConditionalDiscriminator(nn.Module):
    def __init__(self, num_classes=10, img_dim=784):
        super(ConditionalDiscriminator, self).__init__()
        self.label_embedding = nn.Embedding(num_classes, num_classes)
        
        # Input: image + embedded label
        self.model = nn.Sequential(
            nn.Linear(img_dim + num_classes, 1024),
            nn.LeakyReLU(0.2),
            nn.Dropout(0.3),
            nn.Linear(1024, 512),
            nn.LeakyReLU(0.2),
            nn.Dropout(0.3),
            nn.Linear(512, 256),
            nn.LeakyReLU(0.2),
            nn.Linear(256, 1),
            nn.Sigmoid(),
        )
    
    def forward(self, x, labels):
        label_embed = self.label_embedding(labels)
        disc_input = torch.cat([x, label_embed], dim=1)
        return self.model(disc_input)


# ============================================================
# Training Conditional GAN
# ============================================================
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
latent_dim = 100
num_classes = 10

G = ConditionalGenerator(latent_dim, num_classes).to(device)
D = ConditionalDiscriminator(num_classes).to(device)
optimizer_G = optim.Adam(G.parameters(), lr=2e-4, betas=(0.5, 0.999))
optimizer_D = optim.Adam(D.parameters(), lr=2e-4, betas=(0.5, 0.999))
criterion = nn.BCELoss()

from torchvision import datasets, transforms
from torch.utils.data import DataLoader

transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize([0.5], [0.5]),
])
train_loader = DataLoader(
    datasets.MNIST('./data', train=True, transform=transform, download=True),
    batch_size=128, shuffle=True, drop_last=True
)

for epoch in range(50):
    for real_images, real_labels in train_loader:
        batch_size = real_images.size(0)
        real_images = real_images.view(batch_size, -1).to(device)
        real_labels = real_labels.to(device)
        
        valid = torch.ones(batch_size, 1).to(device)
        fake = torch.zeros(batch_size, 1).to(device)
        
        # ─── Train Discriminator ───
        optimizer_D.zero_grad()
        d_real = D(real_images, real_labels)
        d_loss_real = criterion(d_real, valid)
        
        z = torch.randn(batch_size, latent_dim).to(device)
        gen_labels = torch.randint(0, num_classes, (batch_size,)).to(device)
        fake_images = G(z, gen_labels).detach()
        d_fake = D(fake_images, gen_labels)
        d_loss_fake = criterion(d_fake, fake)
        
        d_loss = (d_loss_real + d_loss_fake) / 2
        d_loss.backward()
        optimizer_D.step()
        
        # ─── Train Generator ───
        optimizer_G.zero_grad()
        z = torch.randn(batch_size, latent_dim).to(device)
        gen_labels = torch.randint(0, num_classes, (batch_size,)).to(device)
        fake_images = G(z, gen_labels)
        d_output = D(fake_images, gen_labels)
        g_loss = criterion(d_output, valid)
        g_loss.backward()
        optimizer_G.step()

# ============================================================
# Generate Specific Digits
# ============================================================
G.eval()
with torch.no_grad():
    # Generate all digits 0-9
    fig, axes = plt.subplots(10, 10, figsize=(10, 10))
    for digit in range(10):
        z = torch.randn(10, latent_dim).to(device)
        labels = torch.full((10,), digit, dtype=torch.long).to(device)
        generated = G(z, labels).view(-1, 28, 28).cpu()
        for j in range(10):
            axes[digit, j].imshow(generated[j], cmap='gray')
            axes[digit, j].axis('off')
    plt.suptitle('Conditional GAN: Controlled Digit Generation')
    plt.tight_layout()
    plt.show()
```

---

## 6. Wasserstein GAN (WGAN)

### What It Is
WGAN replaces the JS divergence with the **Wasserstein distance** (Earth Mover's Distance) as the training objective. This fixes many GAN training problems: mode collapse, training instability, and lack of meaningful loss metric.

### Why Standard GANs Fail (The Problem)

When real and generated distributions don't overlap (common early in training), JS divergence is **constant** → gradients are zero → no learning signal.

**Wasserstein distance** always provides a smooth gradient, even when distributions don't overlap:

$$W(p_{data}, p_g) = \inf_{\gamma \in \Pi(p_{data}, p_g)} \mathbb{E}_{(x,y) \sim \gamma}[\|x - y\|]$$

Intuition: minimum "cost" to transform one distribution into another (like moving piles of earth).

### Key Changes from Standard GAN

| Standard GAN | WGAN |
|-------------|------|
| Discriminator outputs probability | **Critic** outputs unbounded score |
| Sigmoid activation in D | No sigmoid (linear output) |
| BCE Loss | Wasserstein loss |
| May not correlate with quality | Loss directly indicates quality |
| Mode collapse common | Much more stable |

### WGAN Loss

**Critic loss** (maximize):
$$\mathcal{L}_C = \mathbb{E}_{x \sim p_{data}}[C(x)] - \mathbb{E}_{z \sim p_z}[C(G(z))]$$

**Generator loss** (minimize):
$$\mathcal{L}_G = -\mathbb{E}_{z \sim p_z}[C(G(z))]$$

### Lipschitz Constraint

WGAN requires the critic to be **1-Lipschitz** (gradients ≤ 1 everywhere).

**Methods to enforce:**
1. **Weight clipping** (original WGAN): Clip weights to [-c, c] — simple but crude
2. **Gradient penalty** (WGAN-GP): Add penalty when gradient norm ≠ 1 — preferred

**Gradient Penalty:**
$$\mathcal{L}_{GP} = \lambda \mathbb{E}_{\hat{x}}[(\|\nabla_{\hat{x}}C(\hat{x})\|_2 - 1)^2]$$

where $\hat{x} = \epsilon x_{real} + (1-\epsilon) x_{fake}$ (interpolation), $\lambda = 10$.

### Code Example: WGAN-GP

```python
import torch
import torch.nn as nn
import torch.optim as optim
import torch.autograd as autograd

class WGANCritic(nn.Module):
    """
    Critic (not discriminator!) — outputs unbounded score.
    No Sigmoid at the end. No BatchNorm (use LayerNorm instead).
    """
    def __init__(self, img_dim=784):
        super(WGANCritic, self).__init__()
        self.model = nn.Sequential(
            nn.Linear(img_dim, 512),
            nn.LeakyReLU(0.2),
            nn.LayerNorm(512),        # LayerNorm, not BatchNorm!
            nn.Linear(512, 256),
            nn.LeakyReLU(0.2),
            nn.LayerNorm(256),
            nn.Linear(256, 1),        # No sigmoid! Unbounded output.
        )
    
    def forward(self, x):
        return self.model(x)


class WGANGenerator(nn.Module):
    """Same as vanilla GAN generator"""
    def __init__(self, latent_dim=100, img_dim=784):
        super(WGANGenerator, self).__init__()
        self.model = nn.Sequential(
            nn.Linear(latent_dim, 256),
            nn.LeakyReLU(0.2),
            nn.BatchNorm1d(256),
            nn.Linear(256, 512),
            nn.LeakyReLU(0.2),
            nn.BatchNorm1d(512),
            nn.Linear(512, 1024),
            nn.LeakyReLU(0.2),
            nn.BatchNorm1d(1024),
            nn.Linear(1024, img_dim),
            nn.Tanh(),
        )
    
    def forward(self, z):
        return self.model(z)


def compute_gradient_penalty(critic, real_data, fake_data, device):
    """
    Compute gradient penalty for WGAN-GP.
    Interpolate between real and fake, compute critic gradient,
    penalize if gradient norm deviates from 1.
    """
    batch_size = real_data.size(0)
    
    # Random interpolation coefficient
    epsilon = torch.rand(batch_size, 1).to(device)
    
    # Interpolate between real and fake
    interpolated = (epsilon * real_data + (1 - epsilon) * fake_data)
    interpolated.requires_grad_(True)
    
    # Critic score on interpolated
    critic_interpolated = critic(interpolated)
    
    # Compute gradients w.r.t. interpolated data
    gradients = autograd.grad(
        outputs=critic_interpolated,
        inputs=interpolated,
        grad_outputs=torch.ones_like(critic_interpolated),
        create_graph=True,   # Need second-order gradients
        retain_graph=True,
    )[0]
    
    # Gradient penalty: (||grad||_2 - 1)^2
    gradients = gradients.view(batch_size, -1)
    gradient_norm = gradients.norm(2, dim=1)
    gradient_penalty = ((gradient_norm - 1) ** 2).mean()
    
    return gradient_penalty


# ============================================================
# WGAN-GP Training Loop
# ============================================================
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
latent_dim = 100
lambda_gp = 10       # Gradient penalty weight
n_critic = 5         # Train critic 5 times per generator step

G = WGANGenerator(latent_dim).to(device)
C = WGANCritic().to(device)

# WGAN uses lower learning rate and no momentum (betas=(0, 0.9))
optimizer_G = optim.Adam(G.parameters(), lr=1e-4, betas=(0.0, 0.9))
optimizer_C = optim.Adam(C.parameters(), lr=1e-4, betas=(0.0, 0.9))

from torchvision import datasets, transforms
from torch.utils.data import DataLoader

transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize([0.5], [0.5]),
])
train_loader = DataLoader(
    datasets.MNIST('./data', train=True, transform=transform, download=True),
    batch_size=64, shuffle=True, drop_last=True
)

for epoch in range(50):
    for batch_idx, (real_images, _) in enumerate(train_loader):
        real_images = real_images.view(real_images.size(0), -1).to(device)
        batch_size = real_images.size(0)
        
        # ─── Train Critic (n_critic times per G step) ───
        for _ in range(n_critic):
            optimizer_C.zero_grad()
            
            z = torch.randn(batch_size, latent_dim).to(device)
            fake_images = G(z).detach()
            
            # Wasserstein loss: maximize E[C(real)] - E[C(fake)]
            # → minimize -(E[C(real)] - E[C(fake)])
            critic_real = C(real_images).mean()
            critic_fake = C(fake_images).mean()
            
            # Gradient penalty
            gp = compute_gradient_penalty(C, real_images, fake_images, device)
            
            # Critic loss
            c_loss = critic_fake - critic_real + lambda_gp * gp
            c_loss.backward()
            optimizer_C.step()
        
        # ─── Train Generator ───
        optimizer_G.zero_grad()
        z = torch.randn(batch_size, latent_dim).to(device)
        fake_images = G(z)
        
        # Generator wants to maximize critic score on fakes
        g_loss = -C(fake_images).mean()
        g_loss.backward()
        optimizer_G.step()
    
    if (epoch + 1) % 10 == 0:
        # Wasserstein distance estimate (should decrease over training)
        w_distance = critic_real.item() - critic_fake.item()
        print(f"Epoch [{epoch+1}/50] | C Loss: {c_loss.item():.4f} | "
              f"G Loss: {g_loss.item():.4f} | W-dist: {w_distance:.4f}")
```

> **Key Insight**: In WGAN, the critic loss directly estimates the Wasserstein distance between real and generated distributions. Unlike standard GAN where loss is meaningless, WGAN's loss **correlates with sample quality** — lower loss = better images.

---

## 7. StyleGAN

### What It Is
**StyleGAN** (by NVIDIA) generates ultra-high-quality images by separating high-level style (pose, identity) from low-level details (freckles, hair). It introduced the **mapping network** and **style injection** concepts.

### Key Innovations

```
Standard GAN:       z ──────────────────────────▶ Generator ──▶ Image

StyleGAN:           z ──▶ Mapping Network ──▶ w ──▶ Style Injection at EVERY layer
                              (8 FC layers)           │
                                                      ▼
                              Constant Input ──▶ Synthesis Network ──▶ Image
                              (learned 4x4)     (progressive growing)
```

1. **Mapping Network**: $z → w$ (8 fully-connected layers). Disentangles the latent space.
2. **AdaIN (Adaptive Instance Normalization)**: Injects style at every layer
3. **Progressive Growing**: Start at 4×4, gradually increase to 1024×1024
4. **Style Mixing**: Mix styles from different latents at different layers
5. **Noise Injection**: Add per-pixel noise for stochastic details

### AdaIN (Style Injection)

$$AdaIN(x_i, y) = y_{s,i} \frac{x_i - \mu(x_i)}{\sigma(x_i)} + y_{b,i}$$

where $y_s$ (scale) and $y_b$ (bias) come from the style vector $w$.

### StyleGAN Conceptual Code

```python
import torch
import torch.nn as nn

class MappingNetwork(nn.Module):
    """Maps z to w (disentangled latent space)"""
    def __init__(self, latent_dim=512, w_dim=512, num_layers=8):
        super(MappingNetwork, self).__init__()
        layers = []
        for i in range(num_layers):
            layers.append(nn.Linear(latent_dim if i == 0 else w_dim, w_dim))
            layers.append(nn.LeakyReLU(0.2))
        self.mapping = nn.Sequential(*layers)
    
    def forward(self, z):
        # Normalize z (as in paper)
        z = z / (z.norm(dim=1, keepdim=True) + 1e-8)
        return self.mapping(z)


class AdaIN(nn.Module):
    """Adaptive Instance Normalization - injects style"""
    def __init__(self, channels, w_dim=512):
        super(AdaIN, self).__init__()
        self.instance_norm = nn.InstanceNorm2d(channels)
        # Affine transform from w → scale and bias
        self.style_scale = nn.Linear(w_dim, channels)
        self.style_bias = nn.Linear(w_dim, channels)
    
    def forward(self, x, w):
        # Normalize feature maps
        x = self.instance_norm(x)
        # Get style parameters from w
        scale = self.style_scale(w).unsqueeze(-1).unsqueeze(-1)
        bias = self.style_bias(w).unsqueeze(-1).unsqueeze(-1)
        # Apply style
        return scale * x + bias


class StyleBlock(nn.Module):
    """One block of the StyleGAN synthesis network"""
    def __init__(self, in_channels, out_channels, w_dim=512):
        super(StyleBlock, self).__init__()
        self.conv = nn.Conv2d(in_channels, out_channels, 3, padding=1)
        self.adain = AdaIN(out_channels, w_dim)
        self.noise_scale = nn.Parameter(torch.zeros(1, out_channels, 1, 1))
        self.activation = nn.LeakyReLU(0.2)
    
    def forward(self, x, w):
        x = self.conv(x)
        # Add noise for stochastic variation (freckles, hair strands)
        noise = torch.randn(x.size(0), 1, x.size(2), x.size(3)).to(x.device)
        x = x + self.noise_scale * noise
        x = self.activation(x)
        x = self.adain(x, w)  # Inject style
        return x


class SimplifiedStyleGAN(nn.Module):
    """Simplified StyleGAN for understanding the concept"""
    def __init__(self, latent_dim=512, w_dim=512):
        super(SimplifiedStyleGAN, self).__init__()
        
        self.mapping = MappingNetwork(latent_dim, w_dim)
        
        # Learned constant input (4x4)
        self.constant = nn.Parameter(torch.randn(1, 512, 4, 4))
        
        # Synthesis blocks (progressive resolution)
        self.blocks = nn.ModuleList([
            StyleBlock(512, 512, w_dim),   # 4x4
            StyleBlock(512, 256, w_dim),   # 8x8
            StyleBlock(256, 128, w_dim),   # 16x16
            StyleBlock(128, 64, w_dim),    # 32x32
        ])
        
        self.upsample = nn.Upsample(scale_factor=2, mode='bilinear', 
                                     align_corners=False)
        self.to_rgb = nn.Conv2d(64, 3, 1)  # Final 1x1 conv to RGB
    
    def forward(self, z):
        w = self.mapping(z)  # z → w (disentangled)
        
        # Start from learned constant
        x = self.constant.expand(z.size(0), -1, -1, -1)
        
        # Apply style at each resolution
        for block in self.blocks:
            x = self.upsample(x)
            x = block(x, w)  # Style injection at every layer!
        
        return torch.tanh(self.to_rgb(x))
    
    def style_mix(self, z1, z2, crossover_layer=2):
        """Mix styles from two different latent codes"""
        w1 = self.mapping(z1)
        w2 = self.mapping(z2)
        
        x = self.constant.expand(z1.size(0), -1, -1, -1)
        
        for i, block in enumerate(self.blocks):
            x = self.upsample(x)
            # Use w1 for early layers (coarse features)
            # Use w2 for later layers (fine details)
            w = w1 if i < crossover_layer else w2
            x = block(x, w)
        
        return torch.tanh(self.to_rgb(x))
```

### StyleGAN Versions Comparison

| Feature | StyleGAN | StyleGAN2 | StyleGAN3 |
|---------|----------|-----------|-----------|
| Year | 2018 | 2019 | 2021 |
| Resolution | 1024×1024 | 1024×1024 | 1024×1024 |
| Normalization | AdaIN | Weight demodulation | Continuous signals |
| Artifacts | Droplet artifacts | Fixed droplets | Fixed aliasing |
| Key Innovation | Mapping network + style | Better normalization | Alias-free ops |
| FID (FFHQ) | 4.40 | 2.84 | 2.79 |

---

## 8. GAN Training Tricks and Best Practices

### The Fundamental Challenge

GAN training is a **two-player game** — not standard optimization. Both networks must improve together. If one dominates, training collapses.

### Essential Training Tricks

#### 1. Label Smoothing
```python
# Instead of hard labels (0.0 and 1.0):
real_labels = torch.ones(batch_size, 1) * 0.9   # "Soft" real: 0.9
fake_labels = torch.zeros(batch_size, 1) + 0.1  # "Soft" fake: 0.1
# This prevents D from becoming too confident
```

#### 2. Feature Matching
```python
# Match statistics of intermediate D features instead of D output
def feature_matching_loss(D, real_images, fake_images):
    """Force G to match feature statistics of real data"""
    # Get intermediate features (hook into D's layers)
    real_features = D.get_features(real_images)  # Intermediate layer
    fake_features = D.get_features(fake_images)
    return torch.mean((real_features.mean(0) - fake_features.mean(0)) ** 2)
```

#### 3. Spectral Normalization
```python
# Constrains D's Lipschitz constant by normalizing weight matrices
# by their spectral norm (largest singular value)
from torch.nn.utils import spectral_norm

class StableDiscriminator(nn.Module):
    def __init__(self):
        super().__init__()
        self.model = nn.Sequential(
            spectral_norm(nn.Linear(784, 512)),  # Spectral norm!
            nn.LeakyReLU(0.2),
            spectral_norm(nn.Linear(512, 256)),
            nn.LeakyReLU(0.2),
            spectral_norm(nn.Linear(256, 1)),
        )
    
    def forward(self, x):
        return self.model(x)
```

#### 4. Two Timescale Update Rule (TTUR)
```python
# Different learning rates for G and D
optimizer_G = optim.Adam(G.parameters(), lr=1e-4, betas=(0.0, 0.9))
optimizer_D = optim.Adam(D.parameters(), lr=4e-4, betas=(0.0, 0.9))
# D gets higher LR — helps it keep up with G
```

#### 5. Progressive Training
```python
# Start training at low resolution, gradually increase
# Epoch 1-10:  Generate 4×4 images
# Epoch 11-20: Generate 8×8 images (add layers)
# Epoch 21-30: Generate 16×16 images
# ... up to target resolution
# Helps stabilize training for high-resolution GANs
```

### Mode Collapse — The Biggest Problem

**What it is**: Generator produces only a few different outputs, ignoring variety in real data.

**Example**: A GAN trained on all digits only generates 3's and 7's.

**Solutions**:
| Technique | How It Helps |
|-----------|-------------|
| Minibatch discrimination | D sees batches, can detect lack of variety |
| Unrolled GAN | G considers future D updates |
| WGAN-GP | More stable loss landscape |
| Diversity penalty | Penalize similar outputs in a batch |
| Multiple generators | Ensemble of specialized generators |

```python
# Minibatch Discrimination
class MinibatchDiscrimination(nn.Module):
    def __init__(self, in_features, out_features, kernel_dims=5):
        super().__init__()
        self.T = nn.Parameter(torch.randn(in_features, out_features, kernel_dims))
    
    def forward(self, x):
        # x: (batch, features)
        matrices = torch.mm(x, self.T.view(x.size(1), -1))
        matrices = matrices.view(-1, self.T.size(1), self.T.size(2))
        
        # Compute L1 distance between all pairs in batch
        # This allows D to see if all samples look the same
        diff = matrices.unsqueeze(0) - matrices.unsqueeze(1)
        abs_diff = torch.sum(torch.abs(diff), dim=3)
        minibatch_features = torch.sum(torch.exp(-abs_diff), dim=0)
        
        return torch.cat([x, minibatch_features], dim=1)
```

### GAN Evaluation Metrics

| Metric | What It Measures | Lower = Better? |
|--------|-----------------|----------------|
| **FID** (Fréchet Inception Distance) | Distance between real and fake feature distributions | Yes |
| **IS** (Inception Score) | Quality + diversity | No (higher = better) |
| **KID** (Kernel Inception Distance) | Like FID but unbiased for small samples | Yes |
| **LPIPS** | Perceptual similarity | Depends on task |

```python
# FID Calculation (simplified concept)
from scipy.linalg import sqrtm
import numpy as np

def calculate_fid(real_features, fake_features):
    """
    FID = ||mu_r - mu_f||² + Tr(Σ_r + Σ_f - 2*(Σ_r*Σ_f)^0.5)
    
    Lower FID = generated images more similar to real images
    FID = 0 means identical distributions
    """
    mu_r = np.mean(real_features, axis=0)
    mu_f = np.mean(fake_features, axis=0)
    sigma_r = np.cov(real_features, rowvar=False)
    sigma_f = np.cov(fake_features, rowvar=False)
    
    # Mean difference
    diff = mu_r - mu_f
    
    # Matrix square root
    covmean = sqrtm(sigma_r @ sigma_f)
    if np.iscomplexobj(covmean):
        covmean = covmean.real
    
    fid = diff @ diff + np.trace(sigma_r + sigma_f - 2 * covmean)
    return fid
```

---

## 9. GAN Applications

### Image-to-Image Translation (pix2pix)

```python
# pix2pix: Paired image translation
# Input: sketch → Output: realistic photo
# Uses U-Net generator + PatchGAN discriminator

class UNetGenerator(nn.Module):
    """U-Net with skip connections for image translation"""
    def __init__(self, in_channels=3, out_channels=3):
        super(UNetGenerator, self).__init__()
        
        # Encoder (downsampling)
        self.enc1 = self._enc_block(in_channels, 64, normalize=False)
        self.enc2 = self._enc_block(64, 128)
        self.enc3 = self._enc_block(128, 256)
        self.enc4 = self._enc_block(256, 512)
        
        # Bottleneck
        self.bottleneck = nn.Sequential(
            nn.Conv2d(512, 512, 4, 2, 1),
            nn.ReLU(True),
        )
        
        # Decoder (upsampling) with skip connections
        self.dec4 = self._dec_block(512, 512, dropout=True)
        self.dec3 = self._dec_block(1024, 256)    # 1024 = 512 + 512 skip
        self.dec2 = self._dec_block(512, 128)     # 512 = 256 + 256 skip
        self.dec1 = self._dec_block(256, 64)      # 256 = 128 + 128 skip
        
        self.final = nn.Sequential(
            nn.ConvTranspose2d(128, out_channels, 4, 2, 1),
            nn.Tanh(),
        )
    
    def _enc_block(self, in_c, out_c, normalize=True):
        layers = [nn.Conv2d(in_c, out_c, 4, 2, 1)]
        if normalize:
            layers.append(nn.BatchNorm2d(out_c))
        layers.append(nn.LeakyReLU(0.2))
        return nn.Sequential(*layers)
    
    def _dec_block(self, in_c, out_c, dropout=False):
        layers = [
            nn.ConvTranspose2d(in_c, out_c, 4, 2, 1),
            nn.BatchNorm2d(out_c),
            nn.ReLU(True),
        ]
        if dropout:
            layers.append(nn.Dropout(0.5))
        return nn.Sequential(*layers)
    
    def forward(self, x):
        # Encoder
        e1 = self.enc1(x)     # (64, H/2, W/2)
        e2 = self.enc2(e1)    # (128, H/4, W/4)
        e3 = self.enc3(e2)    # (256, H/8, W/8)
        e4 = self.enc4(e3)    # (512, H/16, W/16)
        
        # Bottleneck
        b = self.bottleneck(e4)  # (512, H/32, W/32)
        
        # Decoder with skip connections (concatenation)
        d4 = self.dec4(b)
        d4 = torch.cat([d4, e4], dim=1)  # Skip connection!
        d3 = self.dec3(d4)
        d3 = torch.cat([d3, e3], dim=1)
        d2 = self.dec2(d3)
        d2 = torch.cat([d2, e2], dim=1)
        d1 = self.dec1(d2)
        d1 = torch.cat([d1, e1], dim=1)
        
        return self.final(d1)


class PatchGANDiscriminator(nn.Module):
    """
    PatchGAN: classifies NxN patches as real/fake.
    More parameter efficient than classifying whole image.
    Captures high-frequency structure.
    """
    def __init__(self, in_channels=6):  # 6 = input_img + target/generated
        super(PatchGANDiscriminator, self).__init__()
        
        self.model = nn.Sequential(
            nn.Conv2d(in_channels, 64, 4, 2, 1),
            nn.LeakyReLU(0.2),
            
            nn.Conv2d(64, 128, 4, 2, 1),
            nn.BatchNorm2d(128),
            nn.LeakyReLU(0.2),
            
            nn.Conv2d(128, 256, 4, 2, 1),
            nn.BatchNorm2d(256),
            nn.LeakyReLU(0.2),
            
            nn.Conv2d(256, 1, 4, 1, 1),  # Output: patch-level predictions
            # No sigmoid — using LSGAN or WGAN loss
        )
    
    def forward(self, input_img, target_img):
        # Concatenate input and target along channel dimension
        x = torch.cat([input_img, target_img], dim=1)
        return self.model(x)
```

### CycleGAN (Unpaired Image Translation)

```
Domain A (Horses)                        Domain B (Zebras)
      │                                        │
      ▼                                        ▼
 G_A→B: Horse → Zebra                    G_B→A: Zebra → Horse
      │                                        │
      ▼                                        ▼
 Cycle Consistency:  Horse → Zebra → Horse ≈ Original Horse
                     Zebra → Horse → Zebra ≈ Original Zebra
```

**Cycle Consistency Loss:**
$$\mathcal{L}_{cycle} = \|G_{B→A}(G_{A→B}(x_A)) - x_A\|_1 + \|G_{A→B}(G_{B→A}(x_B)) - x_B\|_1$$

---

## 10. Common Mistakes

### Mistake 1: D Too Strong, G Can't Learn
**Problem**: D loss → 0 immediately, G loss stays high.
**Solution**: 
- Use label smoothing (real=0.9, fake=0.1)
- Add dropout to D
- Train D less frequently (e.g., every other step)
- Use spectral normalization

### Mistake 2: Mode Collapse
**Problem**: Generator only produces a few types of outputs.
**Solution**:
- Switch to WGAN-GP
- Add minibatch discrimination
- Use feature matching loss
- Increase latent dimension

### Mistake 3: Not Using Proper Normalization
**Problem**: Training explodes or produces artifacts.
**Solution**:
- Use BatchNorm in G (except output layer)
- Use LayerNorm or SpectralNorm in D (not BatchNorm with WGAN-GP!)
- Never use BatchNorm in D's first layer

### Mistake 4: Wrong Data Range
**Problem**: Using [0, 1] data with Tanh output or vice versa.
**Solution**: 
- If G uses Tanh → normalize data to [-1, 1]
- If G uses Sigmoid → keep data in [0, 1]
- **Always match generator output range to data range!**

### Mistake 5: Not Detaching Fake Images When Training D
**Problem**: Gradients from D flow into G when they shouldn't.
```python
# WRONG: gradients flow to G when training D
fake_images = G(z)
d_loss = criterion(D(fake_images), fake_labels)

# CORRECT: detach stops gradients to G
fake_images = G(z).detach()
d_loss = criterion(D(fake_images), fake_labels)
```

### Mistake 6: Using SGD Instead of Adam
**Problem**: SGD has trouble with the adversarial landscape.
**Solution**: Use Adam with betas=(0.5, 0.999) for standard GAN, or (0.0, 0.9) for WGAN.

### Mistake 7: Ignoring FID/IS During Training
**Problem**: GAN loss doesn't indicate quality.
**Solution**: Periodically compute FID on generated samples. Loss going down doesn't mean images are getting better!

---

## 11. Interview Questions

### Conceptual

**Q1: Explain the GAN minimax game. What happens at equilibrium?**
> **A**: G tries to fool D, D tries to distinguish real from fake. At Nash equilibrium, G generates data indistinguishable from real (p_g = p_data), and D outputs 0.5 for everything (random guessing).

**Q2: What is mode collapse and how do you detect/fix it?**
> **A**: Mode collapse = generator only produces limited variety. Detect: check diversity of outputs, compute coverage metrics. Fix: WGAN-GP, minibatch discrimination, unrolled GANs, or diversity regularization.

**Q3: Why do we use LeakyReLU in the discriminator but ReLU in the generator?**
> **A**: LeakyReLU in D provides gradients everywhere (even negative region), important for stable learning signal back to G. ReLU in G works fine because it receives gradients from D, and sparse activations help with mode coverage.

**Q4: Explain why WGAN is more stable than vanilla GAN.**
> **A**: Standard GAN uses JS divergence which is constant (and has zero gradients) when distributions don't overlap — common early in training. WGAN uses Wasserstein distance which always provides meaningful gradients, and its loss correlates with sample quality.

**Q5: What's the difference between pix2pix and CycleGAN?**
> **A**: pix2pix requires **paired** training data (input-output pairs). CycleGAN works with **unpaired** data using cycle consistency loss — translate A→B→A should recover original. CycleGAN is more flexible but harder to train.

**Q6: How does the gradient penalty in WGAN-GP enforce the Lipschitz constraint?**
> **A**: It penalizes the critic when its gradient norm deviates from 1 on interpolated points between real and fake samples. This ensures the critic is 1-Lipschitz (bounded gradients), which is required for Wasserstein distance estimation.

**Q7: Why does StyleGAN use a mapping network?**
> **A**: The mapping network transforms z → w, where w lives in a more disentangled space. Direct z inputs are entangled (changing one dimension affects multiple visual attributes). The mapping network learns to "untangle" these factors.

### Coding

**Q8: Implement the Wasserstein loss for critic and generator.**
```python
# Critic loss (minimize this)
c_loss = D(fake).mean() - D(real).mean()  # + gradient_penalty

# Generator loss (minimize this)
g_loss = -D(fake).mean()
```

**Q9: How would you add gradient penalty to a WGAN?**
```python
# Interpolate between real and fake
eps = torch.rand(batch, 1).to(device)
interpolated = eps * real + (1 - eps) * fake
interpolated.requires_grad_(True)

# Get critic output and compute gradient
critic_out = D(interpolated)
gradients = torch.autograd.grad(critic_out, interpolated, 
    grad_outputs=torch.ones_like(critic_out),
    create_graph=True)[0]

# Penalty = (||grad|| - 1)^2
gp = ((gradients.norm(2, dim=1) - 1) ** 2).mean()
```

---

## 12. Quick Reference

### GAN Variants Comparison

| GAN Type | Key Innovation | Best For | Stability |
|----------|---------------|----------|-----------|
| Vanilla GAN | Original framework | Understanding concepts | Low |
| DCGAN | Conv + architectural rules | Image generation baseline | Medium |
| WGAN-GP | Wasserstein + gradient penalty | Stable training | High |
| cGAN | Conditional generation | Controlled output | Medium |
| pix2pix | Paired image translation | Supervised I2I | Medium |
| CycleGAN | Unpaired + cycle consistency | Unsupervised I2I | Medium |
| StyleGAN | Style-based generation | High-quality faces | High |
| ProGAN | Progressive growing | High-resolution | High |
| BigGAN | Large-scale + class conditional | ImageNet generation | Medium |

### Training Checklist

```
□ Normalize data to match G's output range (Tanh → [-1,1])
□ Use Adam optimizer with betas=(0.5, 0.999)
□ Learning rate: 2e-4 (standard) or 1e-4 (WGAN)
□ Use LeakyReLU(0.2) in D
□ Use BatchNorm in G (except output), LayerNorm/SpectralNorm in D
□ Detach fake images when training D
□ Monitor FID periodically (loss is unreliable)
□ Use label smoothing if D dominates
□ Try WGAN-GP if training is unstable
□ Save checkpoints frequently (GANs can collapse suddenly)
```

### Key Hyperparameters

| Parameter | Typical Value | Notes |
|-----------|--------------|-------|
| Latent dim | 100-512 | Higher for more complex data |
| Batch size | 32-128 | Larger helps D see more variety |
| LR (Adam) | 1e-4 to 2e-4 | Lower for WGAN |
| D steps per G step | 1 (standard), 5 (WGAN) | Balance D and G |
| Label smoothing | 0.9 real, 0.1 fake | Prevents D overconfidence |
| Gradient penalty λ | 10 | Standard for WGAN-GP |
| Adam betas | (0.5, 0.999) or (0.0, 0.9) | Lower β1 for GANs |

### Decision Guide

```
Need stable training?                    → WGAN-GP
Need high-quality faces?                 → StyleGAN
Need controlled generation?              → Conditional GAN
Need image translation (paired)?         → pix2pix
Need image translation (unpaired)?       → CycleGAN
Need super-resolution?                   → SRGAN / ESRGAN
Need text-to-image?                      → Use diffusion models instead
Starting out / learning?                 → DCGAN
```

---

> **Key Takeaway**: GANs are powerful but notoriously hard to train. Start with DCGAN for learning, use WGAN-GP for stability, and StyleGAN for quality. Always monitor FID — don't trust the loss curves. In 2024+, diffusion models have largely surpassed GANs for image generation quality, but GANs remain faster at inference and are still used for real-time applications.
