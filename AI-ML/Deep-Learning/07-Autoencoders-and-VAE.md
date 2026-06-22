# Chapter 07: Autoencoders and Variational Autoencoders (VAE)

## Table of Contents
- [1. Introduction to Autoencoders](#1-introduction-to-autoencoders)
- [2. Vanilla Autoencoder](#2-vanilla-autoencoder)
- [3. Denoising Autoencoder](#3-denoising-autoencoder)
- [4. Sparse Autoencoder](#4-sparse-autoencoder)
- [5. Variational Autoencoder (VAE)](#5-variational-autoencoder-vae)
- [6. Advanced VAE Variants](#6-advanced-vae-variants)
- [7. Latent Space Exploration](#7-latent-space-exploration)
- [8. Common Mistakes](#8-common-mistakes)
- [9. Interview Questions](#9-interview-questions)
- [10. Quick Reference](#10-quick-reference)

---

## 1. Introduction to Autoencoders

### What It Is
An **autoencoder** is a neural network that learns to compress data into a smaller representation (encoding) and then reconstruct it back (decoding). Think of it like a zip file — you compress a folder into a smaller file, then unzip it to get the original back. The network learns what's *important* about the data by being forced to squeeze it through a bottleneck.

### Why It Matters
- **Dimensionality Reduction**: Better than PCA for non-linear data
- **Feature Learning**: Automatically discovers meaningful features
- **Anomaly Detection**: Reconstructs "normal" data well, fails on anomalies
- **Generative Models**: VAEs can generate new, realistic data
- **Denoising**: Clean corrupted images/signals
- **Pre-training**: Initialize weights for downstream tasks

### Architecture Overview

```
Input (784)  →  Encoder  →  Latent Code (32)  →  Decoder  →  Output (784)
   x                           z                              x̂

┌─────────┐    ┌─────────┐    ┌───┐    ┌─────────┐    ┌─────────┐
│  Input  │───▶│ Encoder │───▶│ z │───▶│ Decoder │───▶│  Output │
│ (784)   │    │ 512→256 │    │32 │    │ 256→512 │    │  (784)  │
└─────────┘    └─────────┘    └───┘    └─────────┘    └─────────┘
                                ▲
                          Bottleneck
                       (Latent Space)
```

### The Core Idea

The network is trained to **reproduce its input as output**. The loss function measures how different the output is from the input:

$$\mathcal{L} = \|x - \hat{x}\|^2$$

Since the latent code $z$ is smaller than the input $x$, the network must learn a **compressed representation** that captures the most important features.

---

## 2. Vanilla Autoencoder

### What It Is
The simplest autoencoder — just an encoder network followed by a decoder network, trained to minimize reconstruction error. No special tricks, no probabilistic interpretation.

### How It Works

**Encoder**: Maps input $x$ to latent representation $z$
$$z = f_\theta(x) = \sigma(Wx + b)$$

**Decoder**: Maps latent $z$ back to reconstruction $\hat{x}$
$$\hat{x} = g_\phi(z) = \sigma(W'z + b')$$

**Loss Function** (MSE for continuous data):
$$\mathcal{L}_{reconstruction} = \frac{1}{n}\sum_{i=1}^{n}(x_i - \hat{x}_i)^2$$

**Loss Function** (Binary Cross-Entropy for binary/image data):
$$\mathcal{L}_{BCE} = -\frac{1}{n}\sum_{i=1}^{n}[x_i \log(\hat{x}_i) + (1-x_i)\log(1-\hat{x}_i)]$$

### Code Example: Vanilla Autoencoder with PyTorch

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms
from torch.utils.data import DataLoader
import matplotlib.pyplot as plt
import numpy as np

# ============================================================
# STEP 1: Define the Autoencoder Architecture
# ============================================================
class VanillaAutoencoder(nn.Module):
    def __init__(self, input_dim=784, hidden_dims=[512, 256], latent_dim=32):
        super(VanillaAutoencoder, self).__init__()
        
        # Encoder: progressively compresses the input
        encoder_layers = []
        prev_dim = input_dim
        for h_dim in hidden_dims:
            encoder_layers.append(nn.Linear(prev_dim, h_dim))
            encoder_layers.append(nn.ReLU())       # ReLU for non-linearity
            encoder_layers.append(nn.BatchNorm1d(h_dim))  # Stabilize training
            prev_dim = h_dim
        encoder_layers.append(nn.Linear(prev_dim, latent_dim))  # Final compression
        self.encoder = nn.Sequential(*encoder_layers)
        
        # Decoder: progressively expands back to original size
        decoder_layers = []
        prev_dim = latent_dim
        for h_dim in reversed(hidden_dims):
            decoder_layers.append(nn.Linear(prev_dim, h_dim))
            decoder_layers.append(nn.ReLU())
            decoder_layers.append(nn.BatchNorm1d(h_dim))
            prev_dim = h_dim
        decoder_layers.append(nn.Linear(prev_dim, input_dim))
        decoder_layers.append(nn.Sigmoid())  # Output between [0, 1] for images
        self.decoder = nn.Sequential(*decoder_layers)
    
    def encode(self, x):
        """Compress input to latent representation"""
        return self.encoder(x)
    
    def decode(self, z):
        """Reconstruct from latent representation"""
        return self.decoder(z)
    
    def forward(self, x):
        z = self.encode(x)       # Compress
        x_hat = self.decode(z)   # Reconstruct
        return x_hat, z          # Return both for analysis

# ============================================================
# STEP 2: Prepare Data (MNIST)
# ============================================================
transform = transforms.Compose([
    transforms.ToTensor(),  # Converts to [0, 1] range automatically
])

train_dataset = datasets.MNIST(root='./data', train=True, 
                               transform=transform, download=True)
test_dataset = datasets.MNIST(root='./data', train=False, 
                              transform=transform, download=True)

train_loader = DataLoader(train_dataset, batch_size=128, shuffle=True)
test_loader = DataLoader(test_dataset, batch_size=128, shuffle=False)

# ============================================================
# STEP 3: Training Loop
# ============================================================
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = VanillaAutoencoder(input_dim=784, latent_dim=32).to(device)
optimizer = optim.Adam(model.parameters(), lr=1e-3)
criterion = nn.MSELoss()  # Reconstruction loss

def train_epoch(model, loader, optimizer, criterion):
    model.train()
    total_loss = 0
    for batch_idx, (data, _) in enumerate(loader):
        # Flatten images: (batch, 1, 28, 28) → (batch, 784)
        data = data.view(data.size(0), -1).to(device)
        
        optimizer.zero_grad()
        reconstructed, latent = model(data)
        loss = criterion(reconstructed, data)  # Compare output with input
        loss.backward()
        optimizer.step()
        
        total_loss += loss.item()
    return total_loss / len(loader)

# Train for 20 epochs
for epoch in range(20):
    avg_loss = train_epoch(model, train_loader, optimizer, criterion)
    if (epoch + 1) % 5 == 0:
        print(f"Epoch [{epoch+1}/20], Loss: {avg_loss:.6f}")

# ============================================================
# STEP 4: Visualize Reconstructions
# ============================================================
model.eval()
with torch.no_grad():
    test_data = next(iter(test_loader))[0][:8]  # Get 8 test images
    flat_data = test_data.view(8, -1).to(device)
    reconstructed, _ = model(flat_data)
    
    fig, axes = plt.subplots(2, 8, figsize=(16, 4))
    for i in range(8):
        # Original
        axes[0, i].imshow(test_data[i].squeeze(), cmap='gray')
        axes[0, i].set_title('Original')
        axes[0, i].axis('off')
        # Reconstructed
        axes[1, i].imshow(reconstructed[i].cpu().view(28, 28), cmap='gray')
        axes[1, i].set_title('Reconstructed')
        axes[1, i].axis('off')
    plt.tight_layout()
    plt.savefig('autoencoder_reconstruction.png', dpi=100)
    plt.show()
```

### When to Use Vanilla Autoencoders
| Use Case | Why It Works |
|----------|-------------|
| Dimensionality reduction | Non-linear compression beats PCA |
| Feature extraction | Latent space captures meaningful patterns |
| Pre-training | Initialize deeper networks |
| Image compression | Learned compression for specific domains |

---

## 3. Denoising Autoencoder

### What It Is
A denoising autoencoder (DAE) intentionally **corrupts the input** and trains the network to recover the clean version. It's like giving someone a blurry photo and training them to produce the sharp original. This forces the network to learn robust features, not just memorize.

### Why It Matters
- Learns more **robust features** than vanilla AE
- Practical application in **image denoising**
- Acts as a **regularizer** — prevents trivial identity mapping
- Better **generalization** to unseen data

### How It Works

```
Clean Input x  →  Add Noise  →  Corrupted x̃  →  Encoder  →  z  →  Decoder  →  x̂
                                                                              ↕
                                                                     Loss = ||x - x̂||²
                                                              (compare with CLEAN input!)
```

The key insight: We corrupt the input but compute loss against the **original clean data**.

**Corruption types:**
- **Gaussian noise**: $\tilde{x} = x + \epsilon$, where $\epsilon \sim \mathcal{N}(0, \sigma^2)$
- **Masking noise**: Randomly zero out pixels (like dropout on input)
- **Salt-and-pepper noise**: Random pixels set to 0 or 1

### Code Example: Denoising Autoencoder

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms
from torch.utils.data import DataLoader
import matplotlib.pyplot as plt

class DenoisingAutoencoder(nn.Module):
    def __init__(self, input_dim=784, latent_dim=64):
        super(DenoisingAutoencoder, self).__init__()
        
        # Encoder with skip-connection-friendly dimensions
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, 512),
            nn.LeakyReLU(0.2),        # LeakyReLU avoids dead neurons
            nn.Dropout(0.2),           # Additional regularization
            nn.Linear(512, 256),
            nn.LeakyReLU(0.2),
            nn.Linear(256, latent_dim),
        )
        
        # Decoder mirrors the encoder
        self.decoder = nn.Sequential(
            nn.Linear(latent_dim, 256),
            nn.LeakyReLU(0.2),
            nn.Linear(256, 512),
            nn.LeakyReLU(0.2),
            nn.Linear(512, input_dim),
            nn.Sigmoid(),              # Output in [0, 1]
        )
    
    def forward(self, x):
        z = self.encoder(x)
        x_hat = self.decoder(z)
        return x_hat

def add_noise(images, noise_factor=0.3):
    """Add Gaussian noise to images"""
    noisy = images + noise_factor * torch.randn_like(images)
    # Clamp to valid pixel range [0, 1]
    return torch.clamp(noisy, 0., 1.)

# ============================================================
# Training the Denoising Autoencoder
# ============================================================
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = DenoisingAutoencoder(input_dim=784, latent_dim=64).to(device)
optimizer = optim.Adam(model.parameters(), lr=1e-3, weight_decay=1e-5)
criterion = nn.MSELoss()

transform = transforms.ToTensor()
train_dataset = datasets.MNIST('./data', train=True, transform=transform, download=True)
train_loader = DataLoader(train_dataset, batch_size=128, shuffle=True)

for epoch in range(30):
    model.train()
    total_loss = 0
    for data, _ in train_loader:
        data = data.view(data.size(0), -1).to(device)
        
        # KEY: Corrupt the input
        noisy_data = add_noise(data, noise_factor=0.4)
        
        optimizer.zero_grad()
        output = model(noisy_data)       # Feed noisy data
        loss = criterion(output, data)    # Compare with CLEAN data
        loss.backward()
        optimizer.step()
        
        total_loss += loss.item()
    
    if (epoch + 1) % 10 == 0:
        print(f"Epoch [{epoch+1}/30], Loss: {total_loss/len(train_loader):.6f}")

# ============================================================
# Visualize Denoising Results
# ============================================================
model.eval()
with torch.no_grad():
    test_data = next(iter(DataLoader(
        datasets.MNIST('./data', train=False, transform=transform),
        batch_size=8, shuffle=True
    )))[0].view(8, -1).to(device)
    
    noisy = add_noise(test_data, noise_factor=0.4)
    denoised = model(noisy)
    
    fig, axes = plt.subplots(3, 8, figsize=(16, 6))
    for i in range(8):
        axes[0, i].imshow(test_data[i].cpu().view(28, 28), cmap='gray')
        axes[0, i].set_title('Clean')
        axes[0, i].axis('off')
        
        axes[1, i].imshow(noisy[i].cpu().view(28, 28), cmap='gray')
        axes[1, i].set_title('Noisy')
        axes[1, i].axis('off')
        
        axes[2, i].imshow(denoised[i].cpu().view(28, 28), cmap='gray')
        axes[2, i].set_title('Denoised')
        axes[2, i].axis('off')
    plt.tight_layout()
    plt.show()
```

> **Pro Tip**: Start with noise_factor=0.3 and increase gradually. Too much noise makes the task impossible; too little doesn't force the network to learn robust features.

---

## 4. Sparse Autoencoder

### What It Is
A sparse autoencoder adds a **sparsity constraint** to the latent layer — most neurons should be inactive (close to 0) for any given input. Think of it like forcing a team where only a few specialists work on each task, rather than everyone doing a little bit of everything.

### Why It Matters
- Learns **disentangled features** — each neuron captures one concept
- Can have latent dimension **larger** than input (overcomplete) without learning identity
- Features are more **interpretable**
- Used extensively in **unsupervised feature learning**

### How It Works

The loss function adds a sparsity penalty:

$$\mathcal{L} = \mathcal{L}_{reconstruction} + \beta \cdot \mathcal{L}_{sparsity}$$

**KL-Divergence Sparsity** (most common):

Let $\hat{\rho}_j$ be the average activation of neuron $j$ over all training examples, and $\rho$ be the desired sparsity level (e.g., 0.05):

$$\mathcal{L}_{sparsity} = \sum_{j=1}^{s} KL(\rho \| \hat{\rho}_j) = \sum_{j=1}^{s} \left[ \rho \log\frac{\rho}{\hat{\rho}_j} + (1-\rho)\log\frac{1-\rho}{1-\hat{\rho}_j} \right]$$

**L1 Sparsity** (simpler alternative):
$$\mathcal{L}_{sparsity} = \sum_{j=1}^{s} |z_j|$$

### Code Example: Sparse Autoencoder

```python
import torch
import torch.nn as nn
import torch.optim as optim

class SparseAutoencoder(nn.Module):
    def __init__(self, input_dim=784, hidden_dim=256, latent_dim=128):
        super(SparseAutoencoder, self).__init__()
        
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, latent_dim),
            nn.Sigmoid(),  # Sigmoid so activations are in [0, 1] for KL
        )
        
        self.decoder = nn.Sequential(
            nn.Linear(latent_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, input_dim),
            nn.Sigmoid(),
        )
    
    def forward(self, x):
        z = self.encoder(x)
        x_hat = self.decoder(z)
        return x_hat, z

def kl_divergence_sparsity(rho, rho_hat):
    """
    Compute KL divergence between desired sparsity (rho) 
    and actual average activation (rho_hat).
    
    rho: target sparsity (e.g., 0.05 — want neurons active 5% of time)
    rho_hat: actual average activation per neuron over batch
    """
    # Add small epsilon to avoid log(0)
    rho_hat = torch.clamp(rho_hat, 1e-6, 1 - 1e-6)
    return (rho * torch.log(rho / rho_hat) + 
            (1 - rho) * torch.log((1 - rho) / (1 - rho_hat))).sum()

# ============================================================
# Training with Sparsity Constraint
# ============================================================
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = SparseAutoencoder(input_dim=784, latent_dim=128).to(device)
optimizer = optim.Adam(model.parameters(), lr=1e-3)
criterion = nn.MSELoss()

# Hyperparameters for sparsity
rho = 0.05      # Target sparsity: want ~5% of neurons active
beta = 0.5      # Weight of sparsity penalty (tune this!)

from torchvision import datasets, transforms
from torch.utils.data import DataLoader

train_loader = DataLoader(
    datasets.MNIST('./data', train=True, transform=transforms.ToTensor(), download=True),
    batch_size=128, shuffle=True
)

for epoch in range(20):
    total_loss = 0
    total_recon = 0
    total_sparse = 0
    
    for data, _ in train_loader:
        data = data.view(data.size(0), -1).to(device)
        
        optimizer.zero_grad()
        x_hat, z = model(data)
        
        # Reconstruction loss
        recon_loss = criterion(x_hat, data)
        
        # Sparsity loss: average activation per neuron across batch
        rho_hat = z.mean(dim=0)  # Shape: (latent_dim,)
        sparsity_loss = kl_divergence_sparsity(rho, rho_hat)
        
        # Total loss
        loss = recon_loss + beta * sparsity_loss
        loss.backward()
        optimizer.step()
        
        total_loss += loss.item()
        total_recon += recon_loss.item()
        total_sparse += sparsity_loss.item()
    
    if (epoch + 1) % 5 == 0:
        n = len(train_loader)
        print(f"Epoch [{epoch+1}/20] | Total: {total_loss/n:.4f} | "
              f"Recon: {total_recon/n:.4f} | Sparse: {total_sparse/n:.4f}")

# Check actual sparsity achieved
model.eval()
with torch.no_grad():
    sample = next(iter(train_loader))[0].view(128, -1).to(device)
    _, z = model(sample)
    avg_activation = z.mean().item()
    active_ratio = (z > 0.1).float().mean().item()
    print(f"\nAverage activation: {avg_activation:.4f} (target: {rho})")
    print(f"Fraction of active neurons (>0.1): {active_ratio:.4f}")
```

---

## 5. Variational Autoencoder (VAE)

### What It Is
A **Variational Autoencoder** doesn't just compress data — it learns a **probability distribution** over the latent space. Instead of encoding to a single point, it encodes to a mean and variance, then *samples* from that distribution. This makes the latent space smooth and continuous, enabling **generation of new data**.

**Analogy**: A vanilla AE is like memorizing where each city is on a map (discrete points). A VAE is like learning that cities follow a pattern — they cluster near rivers, coastlines, etc. — so you can predict where undiscovered cities might be.

### Why It Matters
- **Generate new data**: Sample from latent space → decode → new realistic sample
- **Smooth latent space**: Similar inputs map to nearby regions
- **Interpolation**: Smoothly morph between two data points
- **Principled framework**: Based on Bayesian inference
- **Foundation for modern generative AI**: Directly influenced diffusion models

### How It Works — The Math

#### The Generative Story

1. Sample latent variable: $z \sim p(z) = \mathcal{N}(0, I)$
2. Generate data from latent: $x \sim p_\theta(x|z)$

We want to learn $p_\theta(x)$ — the distribution of real data. But computing $p_\theta(x) = \int p_\theta(x|z)p(z)dz$ is intractable (can't compute the integral).

#### The Solution: Variational Inference

Introduce an **approximate posterior** $q_\phi(z|x)$ (the encoder) that approximates the true posterior $p_\theta(z|x)$.

**Evidence Lower Bound (ELBO)**:

$$\log p_\theta(x) \geq \mathbb{E}_{q_\phi(z|x)}[\log p_\theta(x|z)] - D_{KL}(q_\phi(z|x) \| p(z))$$

$$\mathcal{L}_{VAE} = \underbrace{-\mathbb{E}_{q_\phi(z|x)}[\log p_\theta(x|z)]}_{\text{Reconstruction Loss}} + \underbrace{D_{KL}(q_\phi(z|x) \| p(z))}_{\text{KL Divergence (Regularizer)}}$$

#### The Reparameterization Trick

**Problem**: We can't backpropagate through a random sampling operation.

**Solution**: Instead of sampling $z \sim \mathcal{N}(\mu, \sigma^2)$, reparameterize:

$$z = \mu + \sigma \odot \epsilon, \quad \epsilon \sim \mathcal{N}(0, I)$$

Now gradients flow through $\mu$ and $\sigma$ — the randomness is in $\epsilon$ which doesn't depend on parameters.

```
WITHOUT reparameterization:         WITH reparameterization:
                                    
x → Encoder → μ, σ                 x → Encoder → μ, σ
                ↓                                   ↓     ε ~ N(0,1)
         z ~ N(μ, σ²)  ← CAN'T     z = μ + σ·ε  ← CAN
         BACKPROP!                   BACKPROP through μ, σ!
                ↓                                   ↓
         Decoder → x̂                        Decoder → x̂
```

#### KL Divergence (Closed-Form for Gaussians)

When $q_\phi(z|x) = \mathcal{N}(\mu, \sigma^2)$ and $p(z) = \mathcal{N}(0, 1)$:

$$D_{KL} = -\frac{1}{2}\sum_{j=1}^{d}(1 + \log\sigma_j^2 - \mu_j^2 - \sigma_j^2)$$

### Code Example: Complete VAE Implementation

```python
import torch
import torch.nn as nn
import torch.optim as optim
import torch.nn.functional as F
from torchvision import datasets, transforms
from torch.utils.data import DataLoader
import matplotlib.pyplot as plt
import numpy as np

# ============================================================
# VAE Architecture
# ============================================================
class VAE(nn.Module):
    def __init__(self, input_dim=784, hidden_dim=512, latent_dim=20):
        super(VAE, self).__init__()
        
        # Encoder: input → hidden → (mu, log_var)
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, 256)
        self.fc_mu = nn.Linear(256, latent_dim)      # Mean of q(z|x)
        self.fc_logvar = nn.Linear(256, latent_dim)  # Log variance of q(z|x)
        
        # Decoder: z → hidden → output
        self.fc3 = nn.Linear(latent_dim, 256)
        self.fc4 = nn.Linear(256, hidden_dim)
        self.fc5 = nn.Linear(hidden_dim, input_dim)
        
        # Batch normalization for training stability
        self.bn1 = nn.BatchNorm1d(hidden_dim)
        self.bn2 = nn.BatchNorm1d(256)
        self.bn3 = nn.BatchNorm1d(256)
        self.bn4 = nn.BatchNorm1d(hidden_dim)
    
    def encode(self, x):
        """Encode input to latent distribution parameters"""
        h = F.relu(self.bn1(self.fc1(x)))
        h = F.relu(self.bn2(self.fc2(h)))
        mu = self.fc_mu(h)           # No activation — mu can be any value
        log_var = self.fc_logvar(h)  # Log variance (more numerically stable than σ)
        return mu, log_var
    
    def reparameterize(self, mu, log_var):
        """
        Reparameterization trick: z = mu + std * epsilon
        During training: sample epsilon from N(0, I)
        During inference: just use mu (no randomness)
        """
        if self.training:
            std = torch.exp(0.5 * log_var)  # σ = exp(0.5 * log(σ²))
            epsilon = torch.randn_like(std)  # ε ~ N(0, I)
            z = mu + std * epsilon
            return z
        else:
            return mu  # At inference, use the mean
    
    def decode(self, z):
        """Decode latent vector to reconstruction"""
        h = F.relu(self.bn3(self.fc3(z)))
        h = F.relu(self.bn4(self.fc4(h)))
        x_hat = torch.sigmoid(self.fc5(h))  # Sigmoid for [0,1] pixel values
        return x_hat
    
    def forward(self, x):
        mu, log_var = self.encode(x)
        z = self.reparameterize(mu, log_var)
        x_hat = self.decode(z)
        return x_hat, mu, log_var

# ============================================================
# VAE Loss Function
# ============================================================
def vae_loss(x, x_hat, mu, log_var, beta=1.0):
    """
    VAE loss = Reconstruction + β * KL Divergence
    
    Args:
        x: original input
        x_hat: reconstruction
        mu: latent mean
        log_var: latent log variance
        beta: weight for KL term (β-VAE when β > 1)
    """
    # Reconstruction loss (per sample, summed over features)
    recon_loss = F.binary_cross_entropy(x_hat, x, reduction='sum')
    
    # KL divergence: D_KL(N(mu, sigma) || N(0, 1))
    # = -0.5 * sum(1 + log(sigma^2) - mu^2 - sigma^2)
    kl_loss = -0.5 * torch.sum(1 + log_var - mu.pow(2) - log_var.exp())
    
    return recon_loss + beta * kl_loss, recon_loss, kl_loss

# ============================================================
# Training
# ============================================================
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = VAE(input_dim=784, hidden_dim=512, latent_dim=20).to(device)
optimizer = optim.Adam(model.parameters(), lr=1e-3)

transform = transforms.ToTensor()
train_loader = DataLoader(
    datasets.MNIST('./data', train=True, transform=transform, download=True),
    batch_size=128, shuffle=True
)

# Training with KL annealing (start with low beta, increase)
num_epochs = 30
for epoch in range(num_epochs):
    model.train()
    total_loss = 0
    total_recon = 0
    total_kl = 0
    
    # KL Annealing: gradually increase KL weight
    # Prevents "posterior collapse" (latent space being ignored)
    beta = min(1.0, epoch / 10.0)  # Linearly anneal from 0 to 1 over 10 epochs
    
    for data, _ in train_loader:
        data = data.view(data.size(0), -1).to(device)
        
        optimizer.zero_grad()
        x_hat, mu, log_var = model(data)
        loss, recon, kl = vae_loss(data, x_hat, mu, log_var, beta=beta)
        loss.backward()
        
        # Gradient clipping for stability
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        optimizer.step()
        
        total_loss += loss.item()
        total_recon += recon.item()
        total_kl += kl.item()
    
    n_samples = len(train_loader.dataset)
    if (epoch + 1) % 5 == 0:
        print(f"Epoch [{epoch+1}/{num_epochs}] | β={beta:.2f} | "
              f"Loss: {total_loss/n_samples:.2f} | "
              f"Recon: {total_recon/n_samples:.2f} | "
              f"KL: {total_kl/n_samples:.2f}")

# ============================================================
# Generate New Samples
# ============================================================
model.eval()
with torch.no_grad():
    # Sample from standard normal (the prior)
    z_samples = torch.randn(64, 20).to(device)
    generated = model.decode(z_samples)
    
    fig, axes = plt.subplots(8, 8, figsize=(10, 10))
    for i in range(64):
        axes[i//8, i%8].imshow(generated[i].cpu().view(28, 28), cmap='gray')
        axes[i//8, i%8].axis('off')
    plt.suptitle('VAE Generated Samples', fontsize=14)
    plt.tight_layout()
    plt.show()

# ============================================================
# Latent Space Interpolation
# ============================================================
def interpolate(model, x1, x2, n_steps=10):
    """Smoothly interpolate between two images in latent space"""
    model.eval()
    with torch.no_grad():
        # Encode both images
        mu1, _ = model.encode(x1.view(1, -1).to(device))
        mu2, _ = model.encode(x2.view(1, -1).to(device))
        
        # Linear interpolation in latent space
        alphas = torch.linspace(0, 1, n_steps)
        interpolations = []
        for alpha in alphas:
            z = (1 - alpha) * mu1 + alpha * mu2
            img = model.decode(z)
            interpolations.append(img.cpu().view(28, 28))
        
        # Plot
        fig, axes = plt.subplots(1, n_steps, figsize=(20, 2))
        for i, img in enumerate(interpolations):
            axes[i].imshow(img, cmap='gray')
            axes[i].axis('off')
        plt.suptitle('Latent Space Interpolation')
        plt.show()

# Interpolate between a '3' and a '7'
test_dataset = datasets.MNIST('./data', train=False, transform=transform)
idx_3 = next(i for i, (_, l) in enumerate(test_dataset) if l == 3)
idx_7 = next(i for i, (_, l) in enumerate(test_dataset) if l == 7)
interpolate(model, test_dataset[idx_3][0], test_dataset[idx_7][0])
```

### VAE vs Vanilla AE — Key Differences

| Aspect | Vanilla AE | VAE |
|--------|-----------|-----|
| Latent space | Deterministic point | Probability distribution |
| Can generate? | No (gaps in latent space) | Yes (smooth, continuous) |
| Loss function | Reconstruction only | Reconstruction + KL |
| Interpolation | May pass through empty regions | Smooth transitions |
| Theoretical basis | None (just compression) | Bayesian inference |
| Output quality | Sharp but can't generalize | Slightly blurry but generalizes |

---

## 6. Advanced VAE Variants

### β-VAE (Disentangled Representations)

**Idea**: Increase the KL weight ($\beta > 1$) to force more disentanglement in latent space.

$$\mathcal{L}_{\beta-VAE} = \mathcal{L}_{recon} + \beta \cdot D_{KL}$$

When $\beta > 1$: Each latent dimension captures ONE independent factor of variation (e.g., rotation, size, color separately).

```python
# β-VAE: Just change beta in the loss function
def beta_vae_loss(x, x_hat, mu, log_var, beta=4.0):
    """Higher beta = more disentanglement, worse reconstruction"""
    recon_loss = F.binary_cross_entropy(x_hat, x, reduction='sum')
    kl_loss = -0.5 * torch.sum(1 + log_var - mu.pow(2) - log_var.exp())
    return recon_loss + beta * kl_loss
```

### Conditional VAE (CVAE)

**Idea**: Condition generation on a label/class. "Generate a digit that looks like a 7."

```python
class ConditionalVAE(nn.Module):
    def __init__(self, input_dim=784, latent_dim=20, num_classes=10):
        super(ConditionalVAE, self).__init__()
        
        # Encoder takes input + one-hot class label
        self.fc1 = nn.Linear(input_dim + num_classes, 512)
        self.fc_mu = nn.Linear(512, latent_dim)
        self.fc_logvar = nn.Linear(512, latent_dim)
        
        # Decoder takes latent + one-hot class label
        self.fc3 = nn.Linear(latent_dim + num_classes, 512)
        self.fc4 = nn.Linear(512, input_dim)
        self.num_classes = num_classes
    
    def encode(self, x, c):
        """Encode input conditioned on class c"""
        # One-hot encode the class
        c_onehot = F.one_hot(c, self.num_classes).float()
        # Concatenate input with class label
        xc = torch.cat([x, c_onehot], dim=1)
        h = F.relu(self.fc1(xc))
        return self.fc_mu(h), self.fc_logvar(h)
    
    def decode(self, z, c):
        """Decode latent conditioned on class c"""
        c_onehot = F.one_hot(c, self.num_classes).float()
        zc = torch.cat([z, c_onehot], dim=1)
        h = F.relu(self.fc3(zc))
        return torch.sigmoid(self.fc4(h))
    
    def forward(self, x, c):
        mu, log_var = self.encode(x, c)
        z = self.reparameterize(mu, log_var)
        x_hat = self.decode(z, c)
        return x_hat, mu, log_var
    
    def reparameterize(self, mu, log_var):
        std = torch.exp(0.5 * log_var)
        eps = torch.randn_like(std)
        return mu + std * eps
    
    def generate(self, c, num_samples=1):
        """Generate samples of a specific class"""
        z = torch.randn(num_samples, 20).to(next(self.parameters()).device)
        c_tensor = torch.full((num_samples,), c, dtype=torch.long).to(z.device)
        return self.decode(z, c_tensor)
```

### VQ-VAE (Vector Quantized VAE)

**Idea**: Replace continuous latent space with a **discrete codebook**. Used in DALL-E, audio generation, etc.

```python
class VectorQuantizer(nn.Module):
    """
    Discrete latent space: maps continuous encoder output
    to nearest codebook vector.
    """
    def __init__(self, num_embeddings=512, embedding_dim=64, commitment_cost=0.25):
        super(VectorQuantizer, self).__init__()
        self.num_embeddings = num_embeddings
        self.embedding_dim = embedding_dim
        self.commitment_cost = commitment_cost
        
        # Codebook: learnable dictionary of latent vectors
        self.embedding = nn.Embedding(num_embeddings, embedding_dim)
        # Initialize uniformly
        self.embedding.weight.data.uniform_(-1/num_embeddings, 1/num_embeddings)
    
    def forward(self, z):
        """
        z: (batch, embedding_dim) continuous latent from encoder
        Returns: quantized z (nearest codebook vector), loss, indices
        """
        # Compute distances to all codebook vectors
        # |z - e|² = |z|² + |e|² - 2*z·e
        distances = (torch.sum(z**2, dim=1, keepdim=True) 
                    + torch.sum(self.embedding.weight**2, dim=1)
                    - 2 * torch.matmul(z, self.embedding.weight.t()))
        
        # Find nearest codebook vector
        encoding_indices = torch.argmin(distances, dim=1)
        z_q = self.embedding(encoding_indices)  # Quantized latent
        
        # Losses
        # Codebook loss: move codebook vectors towards encoder outputs
        codebook_loss = F.mse_loss(z_q, z.detach())
        # Commitment loss: encourage encoder to commit to codebook vectors
        commitment_loss = F.mse_loss(z_q.detach(), z)
        
        vq_loss = codebook_loss + self.commitment_cost * commitment_loss
        
        # Straight-through estimator: copy gradients from z_q to z
        z_q = z + (z_q - z).detach()
        
        return z_q, vq_loss, encoding_indices
```

---

## 7. Latent Space Exploration

### What It Is
The latent space is the compressed representation learned by the autoencoder. Understanding its structure reveals what the model has learned and enables applications like generation, interpolation, and anomaly detection.

### Visualizing Latent Space (2D VAE)

```python
import torch
import numpy as np
import matplotlib.pyplot as plt
from sklearn.manifold import TSNE

def visualize_latent_space(model, test_loader, device, method='tsne'):
    """Visualize the latent space colored by digit class"""
    model.eval()
    latents = []
    labels = []
    
    with torch.no_grad():
        for data, target in test_loader:
            data = data.view(data.size(0), -1).to(device)
            mu, _ = model.encode(data)
            latents.append(mu.cpu().numpy())
            labels.append(target.numpy())
    
    latents = np.concatenate(latents, axis=0)
    labels = np.concatenate(labels, axis=0)
    
    # If latent dim > 2, use t-SNE to reduce to 2D
    if latents.shape[1] > 2:
        print("Running t-SNE...")
        tsne = TSNE(n_components=2, random_state=42, perplexity=30)
        latents_2d = tsne.fit_transform(latents[:5000])  # Subset for speed
        labels_2d = labels[:5000]
    else:
        latents_2d = latents
        labels_2d = labels
    
    plt.figure(figsize=(10, 8))
    scatter = plt.scatter(latents_2d[:, 0], latents_2d[:, 1], 
                         c=labels_2d, cmap='tab10', alpha=0.6, s=5)
    plt.colorbar(scatter, label='Digit')
    plt.title('VAE Latent Space Visualization')
    plt.xlabel('Dimension 1')
    plt.ylabel('Dimension 2')
    plt.show()

def generate_manifold(model, device, n=20, latent_dim=2, digit_size=28):
    """
    Generate a 2D manifold of digits by sampling a grid in latent space.
    Only works well with latent_dim=2.
    """
    model.eval()
    figure = np.zeros((digit_size * n, digit_size * n))
    
    # Linearly sample from the latent space (ppf of normal distribution)
    from scipy.stats import norm
    grid_x = norm.ppf(np.linspace(0.05, 0.95, n))
    grid_y = norm.ppf(np.linspace(0.05, 0.95, n))
    
    with torch.no_grad():
        for i, yi in enumerate(grid_x):
            for j, xi in enumerate(grid_y):
                z = torch.tensor([[xi, yi]], dtype=torch.float32).to(device)
                if latent_dim > 2:
                    # Pad with zeros for higher dimensions
                    z = F.pad(z, (0, latent_dim - 2))
                decoded = model.decode(z)
                digit = decoded.cpu().view(digit_size, digit_size).numpy()
                figure[i * digit_size: (i + 1) * digit_size,
                       j * digit_size: (j + 1) * digit_size] = digit
    
    plt.figure(figsize=(12, 12))
    plt.imshow(figure, cmap='gray')
    plt.title('VAE Latent Space Manifold')
    plt.axis('off')
    plt.show()
```

### Anomaly Detection with Autoencoders

```python
def compute_anomaly_score(model, data, device):
    """
    Anomaly score = reconstruction error.
    Normal data reconstructs well (low error).
    Anomalies reconstruct poorly (high error).
    """
    model.eval()
    with torch.no_grad():
        data = data.view(data.size(0), -1).to(device)
        
        if hasattr(model, 'fc_mu'):  # VAE
            x_hat, mu, log_var = model(data)
            # For VAE: can also use negative ELBO as score
            recon_error = F.mse_loss(x_hat, data, reduction='none').sum(dim=1)
            kl = -0.5 * torch.sum(1 + log_var - mu.pow(2) - log_var.exp(), dim=1)
            score = recon_error + kl
        else:  # Regular AE
            x_hat = model(data)
            score = F.mse_loss(x_hat, data, reduction='none').sum(dim=1)
    
    return score.cpu().numpy()

# Example: Train on digits 0-8, detect digit 9 as anomaly
# (In practice: train on "normal" data, detect outliers)
```

---

## 8. Common Mistakes

### Mistake 1: Posterior Collapse in VAEs
**Problem**: The decoder ignores the latent code; KL goes to 0.
**Solution**: 
- Use KL annealing (gradually increase β from 0 to 1)
- Use free bits: allow minimum KL per dimension
- Use a weaker decoder (fewer layers/capacity)

### Mistake 2: Using Wrong Loss for Data Type
**Problem**: Using MSE loss for binary images or BCE for continuous data.
**Solution**:
- Binary/grayscale images (pixels in [0,1]): Binary Cross-Entropy
- Continuous data: MSE
- Color images (normalized): MSE or perceptual loss

### Mistake 3: Latent Dimension Too Large or Small
**Problem**: Too large → model memorizes (no compression). Too small → underfits.
**Solution**: Start with 10-50 for images, use reconstruction quality to tune.

### Mistake 4: Not Using Batch Normalization
**Problem**: Training instability, especially with deep encoders/decoders.
**Solution**: Add BatchNorm after linear/conv layers (before activation).

### Mistake 5: Forgetting to Clamp/Clip Noisy Data
**Problem**: Adding noise creates values outside [0,1] for images.
**Solution**: Always `torch.clamp(noisy_data, 0., 1.)`.

### Mistake 6: Not Switching Between Training/Eval Mode
**Problem**: BatchNorm and Dropout behave differently during inference.
**Solution**: Always call `model.eval()` before inference, `model.train()` before training.

### Mistake 7: Ignoring the KL Term Scale
**Problem**: KL loss is summed over dimensions; reconstruction is over pixels. They're on different scales.
**Solution**: Normalize both by batch size, or carefully tune β.

---

## 9. Interview Questions

### Conceptual Questions

**Q1: What's the difference between an autoencoder and PCA?**
> **A**: PCA finds linear projections; autoencoders learn non-linear mappings. A single-layer AE with linear activations and MSE loss is equivalent to PCA. Deep AEs capture non-linear manifolds that PCA misses.

**Q2: Why can't a vanilla autoencoder generate new data?**
> **A**: The latent space has gaps and irregular structure. Random points in latent space may not correspond to valid data. VAEs regularize the latent space to be continuous and smooth (via KL divergence to N(0,1)).

**Q3: Explain the reparameterization trick.**
> **A**: Instead of sampling z ~ N(μ, σ²) (non-differentiable), we compute z = μ + σ·ε where ε ~ N(0,1). This makes z a deterministic function of μ and σ (both learnable), allowing backpropagation through the sampling step.

**Q4: What is posterior collapse and how do you fix it?**
> **A**: When the decoder is powerful enough to model data without the latent code, the encoder learns q(z|x) ≈ p(z) = N(0,1), making KL=0 and latent code useless. Fix: KL annealing, weaker decoder, free bits, or δ-VAE.

**Q5: When would you use a denoising autoencoder over a vanilla one?**
> **A**: When you want more robust features (DAE is a regularizer), when you have noisy data in production, or when you need pre-training features for a downstream task.

**Q6: How do autoencoders perform anomaly detection?**
> **A**: Train on normal data only. At inference, compute reconstruction error. Normal samples reconstruct well (low error), anomalies reconstruct poorly (high error). Threshold the error for classification.

**Q7: What's the trade-off in β-VAE when increasing β?**
> **A**: Higher β → more disentangled latent space (each dim = one factor), but worse reconstruction quality. It's a spectrum between pure compression (β=0) and maximum disentanglement (β>>1).

### Coding Questions

**Q8: Implement the KL divergence loss for a VAE with diagonal Gaussian encoder.**
```python
def kl_divergence(mu, log_var):
    # KL(N(mu, sigma) || N(0, 1))
    return -0.5 * torch.sum(1 + log_var - mu.pow(2) - log_var.exp())
```

**Q9: How would you modify a VAE to generate specific classes (digits)?**
> **A**: Use a Conditional VAE — concatenate one-hot class labels to both encoder input and decoder input. At generation time, specify the desired class.

---

## 10. Quick Reference

| Model | Loss | Latent Space | Can Generate? | Best For |
|-------|------|-------------|---------------|----------|
| Vanilla AE | MSE/BCE | Deterministic | No | Compression, features |
| Denoising AE | MSE (clean target) | Deterministic | No | Robust features, denoising |
| Sparse AE | MSE + L1/KL sparsity | Sparse deterministic | No | Interpretable features |
| VAE | ELBO (Recon + KL) | Gaussian distribution | Yes | Generation, interpolation |
| β-VAE | Recon + β·KL (β>1) | Disentangled Gaussian | Yes | Disentangled factors |
| CVAE | ELBO + conditioning | Conditional Gaussian | Yes (controlled) | Class-specific generation |
| VQ-VAE | Recon + VQ loss | Discrete codebook | Yes | High-quality generation |

### Key Hyperparameters

| Parameter | Typical Range | Effect |
|-----------|--------------|--------|
| Latent dim | 2-256 | Lower = more compression, higher = more capacity |
| β (VAE) | 0.1-10 | Higher = more disentanglement |
| Noise factor (DAE) | 0.1-0.5 | Higher = harder task, more robust features |
| Sparsity ρ | 0.01-0.1 | Lower = sparser activations |
| Learning rate | 1e-4 to 1e-3 | Standard Adam LR |
| KL annealing epochs | 5-20 | Gradual increase prevents collapse |

### When to Use What

```
Need dimensionality reduction?          → Vanilla AE
Need robust features?                   → Denoising AE
Need interpretable features?            → Sparse AE
Need to generate new data?              → VAE
Need disentangled factors?              → β-VAE
Need class-controlled generation?       → Conditional VAE
Need high-quality discrete generation?  → VQ-VAE
Need anomaly detection?                 → Any AE (train on normal only)
```

---

> **Key Takeaway**: Autoencoders learn compressed representations by reconstructing input. VAEs add probabilistic structure to enable generation. The choice between variants depends on whether you need compression, generation, denoising, or interpretability.
