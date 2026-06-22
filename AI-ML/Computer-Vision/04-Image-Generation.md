# Chapter 04: Image Generation

## Table of Contents
- [4.1 What is Image Generation?](#41-what-is-image-generation)
- [4.2 Generative Adversarial Networks (GANs)](#42-generative-adversarial-networks-gans)
- [4.3 GAN Variants and Applications](#43-gan-variants-and-applications)
- [4.4 Variational Autoencoders (VAEs)](#44-variational-autoencoders-vaes)
- [4.5 Diffusion Models — The Revolution](#45-diffusion-models--the-revolution)
- [4.6 Stable Diffusion — Architecture Deep Dive](#46-stable-diffusion--architecture-deep-dive)
- [4.7 Image-to-Image Translation](#47-image-to-image-translation)
- [4.8 Super-Resolution](#48-super-resolution)
- [4.9 Image Inpainting](#49-image-inpainting)
- [4.10 Evaluation Metrics for Generation](#410-evaluation-metrics-for-generation)
- [4.11 Training Your Own Generative Model](#411-training-your-own-generative-model)
- [4.12 Ethical Considerations and Limitations](#412-ethical-considerations-and-limitations)
- [Quick Reference](#quick-reference)

---

## 4.1 What is Image Generation?

### What It Is
Image generation creates **new images that never existed before** using learned patterns from training data. The model learns the statistical distribution of images and can sample new ones from it.

Analogy: A human artist studies thousands of paintings, internalizes patterns (style, composition, color), then creates original works. Generative models do the same — learn patterns from data, then produce novel images.

### Why It Matters
- **Content creation:** Art, design, marketing materials at scale
- **Data augmentation:** Generate synthetic training data for rare cases
- **Medical imaging:** Create synthetic patient data (privacy-safe)
- **Gaming/Film:** Generate textures, environments, characters
- **Science:** Visualize molecules, simulate physical phenomena
- **Editing:** Inpainting, style transfer, super-resolution

### The Generative Model Landscape

```
                    Generative Models
                         │
          ┌──────────────┼──────────────────┐
          │              │                   │
        GANs          VAEs            Diffusion Models
     (2014-2021)    (2013-present)      (2020-present)
          │              │                   │
    ┌─────┴─────┐       │          ┌────────┴────────┐
    │           │        │          │                 │
 StyleGAN   CycleGAN   VQ-VAE    DDPM            Stable Diffusion
 ProGAN     Pix2Pix    DALL-E 1  DALL-E 2/3      Midjourney
                                  Imagen
```

### Comparison of Approaches

| Method | Training Stability | Quality | Diversity | Speed | Control |
|--------|-------------------|---------|-----------|-------|---------|
| GANs | Difficult | Very High | Mode collapse risk | Fast inference | Limited |
| VAEs | Stable | Medium | Good | Fast inference | Good (latent space) |
| Diffusion | Very Stable | Highest | Excellent | Slow inference | Excellent |
| Flow-based | Stable | Good | Good | Fast | Good |

---

## 4.2 Generative Adversarial Networks (GANs)

### What It Is
A GAN is two neural networks playing a **game against each other**:
- **Generator (G):** Creates fake images from random noise
- **Discriminator (D):** Tries to distinguish real images from fakes

They improve each other through competition — like a counterfeiter and a detective, each getting better at their job.

### Why It Matters
GANs produced the first convincing photorealistic generated images. StyleGAN (2019) generated faces so realistic that humans couldn't tell them from photos. Even though diffusion models now dominate, GAN concepts are foundational.

### How It Works

```
                    Random Noise z ~ N(0,1)
                          │
                          ↓
                   ┌──────────────┐
                   │  Generator G │
                   │  (Neural Net)│
                   └──────┬───────┘
                          │
                     Fake Image
                          │
                          ↓
Real Images ───→  ┌──────────────┐  ───→ Real/Fake?
                  │Discriminator D│
                  │  (Neural Net) │
                  └───────────────┘
                          │
                    Loss (Binary CE)
                          │
              ┌───────────┴───────────┐
              ↓                       ↓
         Update D                 Update G
    (better at detecting)     (better at fooling)
```

**GAN Loss (Minimax Game):**

$$\min_G \max_D \; \mathbb{E}_{x \sim p_{data}}[\log D(x)] + \mathbb{E}_{z \sim p_z}[\log(1 - D(G(z)))]$$

- $D$ wants to **maximize**: correctly classify reals (D(x)→1) and fakes (D(G(z))→0)
- $G$ wants to **minimize**: fool D into thinking fakes are real (D(G(z))→1)

**Training dynamics:**
```
Epoch 1:   G produces noise        → D easily says "fake"
Epoch 10:  G produces blurry blobs → D still catches it
Epoch 100: G produces faces        → D struggles
Epoch 500: G produces photorealistic → D can't tell (D→0.5)
```

### Code Examples

```python
import torch
import torch.nn as nn

# === Simple DCGAN (Deep Convolutional GAN) ===

class Generator(nn.Module):
    """Maps latent vector z to an image."""
    def __init__(self, latent_dim=100, img_channels=3, feature_maps=64):
        super().__init__()
        self.net = nn.Sequential(
            # Input: (B, latent_dim, 1, 1)
            # Upsample progressively: 1×1 → 4×4 → 8×8 → 16×16 → 32×32 → 64×64
            self._block(latent_dim, feature_maps * 8, 4, 1, 0),  # 4×4
            self._block(feature_maps * 8, feature_maps * 4, 4, 2, 1),  # 8×8
            self._block(feature_maps * 4, feature_maps * 2, 4, 2, 1),  # 16×16
            self._block(feature_maps * 2, feature_maps, 4, 2, 1),  # 32×32
            # Final layer: no batch norm, use Tanh for [-1, 1] output
            nn.ConvTranspose2d(feature_maps, img_channels, 4, 2, 1),  # 64×64
            nn.Tanh(),  # Output in [-1, 1]
        )
    
    def _block(self, in_ch, out_ch, kernel, stride, padding):
        return nn.Sequential(
            nn.ConvTranspose2d(in_ch, out_ch, kernel, stride, padding, bias=False),
            nn.BatchNorm2d(out_ch),
            nn.ReLU(True),
        )
    
    def forward(self, z):
        return self.net(z)

class Discriminator(nn.Module):
    """Classifies images as real or fake."""
    def __init__(self, img_channels=3, feature_maps=64):
        super().__init__()
        self.net = nn.Sequential(
            # Input: (B, 3, 64, 64)
            # Downsample: 64×64 → 32×32 → 16×16 → 8×8 → 4×4 → 1×1
            nn.Conv2d(img_channels, feature_maps, 4, 2, 1),  # 32×32
            nn.LeakyReLU(0.2, True),
            self._block(feature_maps, feature_maps * 2, 4, 2, 1),  # 16×16
            self._block(feature_maps * 2, feature_maps * 4, 4, 2, 1),  # 8×8
            self._block(feature_maps * 4, feature_maps * 8, 4, 2, 1),  # 4×4
            nn.Conv2d(feature_maps * 8, 1, 4, 1, 0),  # 1×1
            nn.Sigmoid(),  # Output probability [0, 1]
        )
    
    def _block(self, in_ch, out_ch, kernel, stride, padding):
        return nn.Sequential(
            nn.Conv2d(in_ch, out_ch, kernel, stride, padding, bias=False),
            nn.BatchNorm2d(out_ch),
            nn.LeakyReLU(0.2, True),
        )
    
    def forward(self, img):
        return self.net(img).view(-1)

# === Training Loop ===
latent_dim = 100
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

G = Generator(latent_dim).to(device)
D = Discriminator().to(device)

# Key: use separate optimizers for G and D
opt_G = torch.optim.Adam(G.parameters(), lr=2e-4, betas=(0.5, 0.999))
opt_D = torch.optim.Adam(D.parameters(), lr=2e-4, betas=(0.5, 0.999))
criterion = nn.BCELoss()

# Training step
def train_step(real_images):
    batch_size = real_images.size(0)
    real_labels = torch.ones(batch_size, device=device)
    fake_labels = torch.zeros(batch_size, device=device)
    
    # === Train Discriminator ===
    z = torch.randn(batch_size, latent_dim, 1, 1, device=device)
    fake_images = G(z).detach()  # Don't backprop through G
    
    d_real = D(real_images)
    d_fake = D(fake_images)
    loss_D = criterion(d_real, real_labels) + criterion(d_fake, fake_labels)
    
    opt_D.zero_grad()
    loss_D.backward()
    opt_D.step()
    
    # === Train Generator ===
    z = torch.randn(batch_size, latent_dim, 1, 1, device=device)
    fake_images = G(z)
    d_fake = D(fake_images)
    # G wants D to think fakes are real → use real_labels
    loss_G = criterion(d_fake, real_labels)
    
    opt_G.zero_grad()
    loss_G.backward()
    opt_G.step()
    
    return loss_D.item(), loss_G.item()

# === Generate images after training ===
G.eval()
with torch.no_grad():
    z = torch.randn(16, latent_dim, 1, 1, device=device)
    generated = G(z)
    # generated shape: (16, 3, 64, 64), values in [-1, 1]
    # Convert to displayable: (generated + 1) / 2 → [0, 1]
```

### Common GAN Problems and Solutions

| Problem | Symptom | Solution |
|---------|---------|----------|
| Mode Collapse | G produces same image repeatedly | Wasserstein loss, spectral norm, diversity penalty |
| Training Instability | Loss oscillates wildly | WGAN-GP, progressive growing, lower LR |
| Vanishing Gradients | G can't improve (D too strong) | Non-saturating loss, label smoothing |
| Checkerboard Artifacts | Grid pattern in generated images | Use resize+conv instead of transposed conv |

---

## 4.3 GAN Variants and Applications

### Key GAN Architectures

#### StyleGAN (2019-2021) — Photorealistic Faces

**Key ideas:**
- Mapping network: $z \rightarrow w$ (intermediate latent space)
- Style injection at each resolution via AdaIN
- Progressive growing
- Noise injection for stochastic details

```python
# StyleGAN is complex to implement from scratch
# Use the official NVIDIA repo or pretrained models:
# pip install stylegan2-ada-pytorch

# === Generate faces with pretrained StyleGAN ===
# Conceptual (requires NVIDIA pretrained weights):
"""
import dnnlib
import legacy

with dnnlib.util.open_url('https://...stylegan2-ffhq.pkl') as f:
    G = legacy.load_network_pkl(f)['G_ema']

z = torch.randn(1, G.z_dim)
w = G.mapping(z, None)  # Map to W space
img = G.synthesis(w)     # Generate from W space
"""
```

#### Conditional GAN (cGAN)
Generate images conditioned on a class label:

$$\min_G \max_D \; \mathbb{E}[\log D(x|y)] + \mathbb{E}[\log(1 - D(G(z|y)|y))]$$

```python
class ConditionalGenerator(nn.Module):
    """Generator conditioned on class label."""
    def __init__(self, latent_dim=100, num_classes=10, img_channels=3):
        super().__init__()
        # Embed class label
        self.label_embed = nn.Embedding(num_classes, latent_dim)
        
        # Same architecture as before but input is z + label embedding
        self.net = nn.Sequential(
            # ... (same upsampling blocks)
        )
    
    def forward(self, z, labels):
        # Combine noise with label
        label_embed = self.label_embed(labels).unsqueeze(-1).unsqueeze(-1)
        combined = z + label_embed  # Or concatenate
        return self.net(combined)
```

#### Pix2Pix — Paired Image-to-Image Translation

Translates between paired image domains (sketch→photo, map→satellite, etc.):

```python
# === Pix2Pix conceptual architecture ===
# Generator: U-Net (image → image)
# Discriminator: PatchGAN (judges 70×70 patches)
# Loss: adversarial + L1 reconstruction

# Using the actual loss:
loss_G = adversarial_loss + lambda_L1 * L1_loss(generated, target)
# lambda_L1 = 100 (strong reconstruction signal)
```

#### CycleGAN — Unpaired Image Translation

Translates between domains WITHOUT paired examples (horse→zebra, summer→winter):

```
Domain A ──→ G_AB ──→ Domain B ──→ G_BA ──→ Domain A (should match original!)
                                                  ↑
                                          Cycle Consistency Loss
```

$$\mathcal{L}_{cycle} = \|G_{BA}(G_{AB}(x_A)) - x_A\|_1 + \|G_{AB}(G_{BA}(x_B)) - x_B\|_1$$

---

## 4.4 Variational Autoencoders (VAEs)

### What It Is
A VAE is an autoencoder that learns a **structured latent space** where you can sample new points and decode them into images. Unlike GANs which learn implicitly, VAEs learn an explicit probability distribution.

Analogy: If generating images is like being a chef, GANs learn by competition (cook-off), while VAEs learn by compression — squeezing images into a recipe (encoding) and recreating them (decoding). The recipes form an organized cookbook where nearby recipes produce similar dishes.

### Why It Matters
- **Structured latent space:** Smooth interpolation between images
- **Stable training:** No adversarial instability
- **Explicit density:** Can compute probabilities (useful for anomaly detection)
- **Foundation for modern models:** DALL-E 1, Stable Diffusion's VAE component

### How It Works

```
Input x ──→ Encoder ──→ μ, σ ──→ z = μ + σ·ε (reparameterization) ──→ Decoder ──→ x̂
              q(z|x)         ε ~ N(0,1)                                    p(x|z)
```

**VAE Loss (ELBO — Evidence Lower Bound):**

$$\mathcal{L} = \underbrace{\mathbb{E}_{q(z|x)}[\log p(x|z)]}_{\text{Reconstruction Loss}} - \underbrace{D_{KL}(q(z|x) \| p(z))}_{\text{KL Divergence (regularization)}}$$

- **Reconstruction loss:** Make decoded output match input (MSE or BCE)
- **KL divergence:** Force latent distribution to be close to standard normal $N(0,1)$

$$D_{KL} = -\frac{1}{2} \sum_{j=1}^{d} (1 + \log\sigma_j^2 - \mu_j^2 - \sigma_j^2)$$

**Why KL divergence matters:**
Without KL, the encoder could map each image to a unique point → no smooth interpolation. KL forces a smooth, continuous latent space where nearby points produce similar images.

### Code Examples

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class VAE(nn.Module):
    """Variational Autoencoder for 64×64 images."""
    def __init__(self, latent_dim=256, img_channels=3):
        super().__init__()
        self.latent_dim = latent_dim
        
        # Encoder: Image → (μ, log_σ²)
        self.encoder = nn.Sequential(
            nn.Conv2d(img_channels, 32, 4, 2, 1),   # 32×32
            nn.ReLU(),
            nn.Conv2d(32, 64, 4, 2, 1),              # 16×16
            nn.ReLU(),
            nn.Conv2d(64, 128, 4, 2, 1),             # 8×8
            nn.ReLU(),
            nn.Conv2d(128, 256, 4, 2, 1),            # 4×4
            nn.ReLU(),
            nn.Flatten(),                             # 256*4*4 = 4096
        )
        
        self.fc_mu = nn.Linear(4096, latent_dim)
        self.fc_logvar = nn.Linear(4096, latent_dim)
        
        # Decoder: z → Image
        self.decoder_input = nn.Linear(latent_dim, 4096)
        self.decoder = nn.Sequential(
            nn.ConvTranspose2d(256, 128, 4, 2, 1),   # 8×8
            nn.ReLU(),
            nn.ConvTranspose2d(128, 64, 4, 2, 1),    # 16×16
            nn.ReLU(),
            nn.ConvTranspose2d(64, 32, 4, 2, 1),     # 32×32
            nn.ReLU(),
            nn.ConvTranspose2d(32, img_channels, 4, 2, 1),  # 64×64
            nn.Sigmoid(),  # Output in [0, 1]
        )
    
    def encode(self, x):
        h = self.encoder(x)
        mu = self.fc_mu(h)
        logvar = self.fc_logvar(h)
        return mu, logvar
    
    def reparameterize(self, mu, logvar):
        """Reparameterization trick: z = μ + σ·ε, ε ~ N(0,1)"""
        std = torch.exp(0.5 * logvar)  # σ = exp(0.5 * log(σ²))
        eps = torch.randn_like(std)     # ε ~ N(0,1)
        return mu + std * eps           # Differentiable sampling!
    
    def decode(self, z):
        h = self.decoder_input(z)
        h = h.view(-1, 256, 4, 4)
        return self.decoder(h)
    
    def forward(self, x):
        mu, logvar = self.encode(x)
        z = self.reparameterize(mu, logvar)
        recon = self.decode(z)
        return recon, mu, logvar

def vae_loss(recon_x, x, mu, logvar, beta=1.0):
    """
    VAE loss = Reconstruction + β * KL divergence.
    β > 1: disentangled representations (β-VAE)
    β < 1: better reconstruction, less regularized latent
    """
    # Reconstruction loss (per pixel)
    recon_loss = F.mse_loss(recon_x, x, reduction='sum')
    
    # KL divergence: -0.5 * Σ(1 + log(σ²) - μ² - σ²)
    kl_loss = -0.5 * torch.sum(1 + logvar - mu.pow(2) - logvar.exp())
    
    return recon_loss + beta * kl_loss

# === Training ===
vae = VAE(latent_dim=256).to(device)
optimizer = torch.optim.Adam(vae.parameters(), lr=1e-3)

for epoch in range(100):
    for images, _ in dataloader:
        images = images.to(device)
        recon, mu, logvar = vae(images)
        loss = vae_loss(recon, images, mu, logvar)
        
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

# === Generation (sample from latent space) ===
with torch.no_grad():
    z = torch.randn(16, 256).to(device)  # Sample from N(0,1)
    generated = vae.decode(z)

# === Interpolation (smooth transition between images) ===
with torch.no_grad():
    mu1, _ = vae.encode(image1.unsqueeze(0))
    mu2, _ = vae.encode(image2.unsqueeze(0))
    
    # Linear interpolation in latent space
    alphas = torch.linspace(0, 1, 10)
    interpolations = []
    for alpha in alphas:
        z = (1 - alpha) * mu1 + alpha * mu2
        interpolations.append(vae.decode(z))
```

### VQ-VAE (Vector Quantized VAE)
Replaces continuous latent with **discrete codes** → sharper images, basis for DALL-E 1:

```python
# Conceptual: instead of continuous z, use a codebook
# z_continuous → find nearest codebook vector → z_discrete
# Loss adds: commitment_loss = ||z_e - sg[e]||² (sg = stop gradient)
```

---

## 4.5 Diffusion Models — The Revolution

### What It Is
Diffusion models generate images by **learning to reverse a noise-adding process**. Start with pure noise, then iteratively denoise it step by step until a clean image emerges.

Analogy: Imagine someone slowly adds static to a TV until the picture is completely gone (forward process). A diffusion model learns to reverse this — starting from pure static and recovering the original picture step by step.

### Why It Matters
- **State-of-the-art quality:** Best image quality of any generative method (as of 2024)
- **Stable training:** No adversarial instability (unlike GANs)
- **Mode coverage:** Captures full data diversity (no mode collapse)
- **Controllability:** Text conditioning, inpainting, editing — all natural
- **Foundation for:** DALL-E 2/3, Stable Diffusion, Midjourney, Imagen

### How It Works

**Forward Process (adding noise):**

Given a clean image $x_0$, gradually add Gaussian noise over $T$ timesteps:

$$q(x_t | x_{t-1}) = \mathcal{N}(x_t; \sqrt{1-\beta_t} \cdot x_{t-1}, \beta_t \cdot I)$$

Where $\beta_t$ is a noise schedule ($\beta_1 < \beta_2 < ... < \beta_T$).

After many steps, $x_T \approx \mathcal{N}(0, I)$ — pure noise.

**Key insight — closed form for any timestep:**

$$x_t = \sqrt{\bar{\alpha}_t} \cdot x_0 + \sqrt{1-\bar{\alpha}_t} \cdot \epsilon, \quad \epsilon \sim \mathcal{N}(0, I)$$

Where $\alpha_t = 1 - \beta_t$ and $\bar{\alpha}_t = \prod_{s=1}^t \alpha_s$.

**Reverse Process (denoising — what we learn):**

$$p_\theta(x_{t-1} | x_t) = \mathcal{N}(x_{t-1}; \mu_\theta(x_t, t), \Sigma_\theta(x_t, t))$$

The neural network predicts the noise $\epsilon_\theta(x_t, t)$ that was added.

**Training objective (simplified):**

$$\mathcal{L} = \mathbb{E}_{t, x_0, \epsilon}\left[\|\epsilon - \epsilon_\theta(x_t, t)\|^2\right]$$

Simply: predict the noise that was added!

```
Forward Process (known, no learning):
x₀ (clean) → x₁ (slightly noisy) → ... → x_T (pure noise)

Reverse Process (learned):
x_T (noise) → x_{T-1} (less noisy) → ... → x₀ (generated image!)

Training:
1. Take clean image x₀
2. Pick random timestep t
3. Add noise: x_t = √ᾱₜ · x₀ + √(1-ᾱₜ) · ε
4. Predict noise: ε̂ = model(x_t, t)
5. Loss = ||ε - ε̂||²
```

### Code Examples

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np

# === Noise Schedule ===
def linear_beta_schedule(timesteps, beta_start=1e-4, beta_end=0.02):
    """Linear noise schedule (original DDPM)."""
    return torch.linspace(beta_start, beta_end, timesteps)

def cosine_beta_schedule(timesteps, s=0.008):
    """Cosine noise schedule (better for high-res, used in improved DDPM)."""
    steps = timesteps + 1
    x = torch.linspace(0, timesteps, steps)
    alphas_cumprod = torch.cos(((x / timesteps) + s) / (1 + s) * torch.pi * 0.5) ** 2
    alphas_cumprod = alphas_cumprod / alphas_cumprod[0]
    betas = 1 - (alphas_cumprod[1:] / alphas_cumprod[:-1])
    return torch.clamp(betas, 0.0001, 0.9999)

# === Simple Diffusion Model ===
class SimpleDiffusion:
    """DDPM (Denoising Diffusion Probabilistic Model) implementation."""
    def __init__(self, timesteps=1000, device='cuda'):
        self.timesteps = timesteps
        self.device = device
        
        # Define schedule
        self.betas = linear_beta_schedule(timesteps).to(device)
        self.alphas = 1.0 - self.betas
        self.alphas_cumprod = torch.cumprod(self.alphas, dim=0)
        self.sqrt_alphas_cumprod = torch.sqrt(self.alphas_cumprod)
        self.sqrt_one_minus_alphas_cumprod = torch.sqrt(1.0 - self.alphas_cumprod)
        
        # For sampling
        self.alphas_cumprod_prev = F.pad(self.alphas_cumprod[:-1], (1, 0), value=1.0)
        self.posterior_variance = (self.betas * (1.0 - self.alphas_cumprod_prev) / 
                                   (1.0 - self.alphas_cumprod))
    
    def add_noise(self, x_0, t, noise=None):
        """Forward process: add noise to x_0 at timestep t."""
        if noise is None:
            noise = torch.randn_like(x_0)
        
        sqrt_alpha = self.sqrt_alphas_cumprod[t][:, None, None, None]
        sqrt_one_minus_alpha = self.sqrt_one_minus_alphas_cumprod[t][:, None, None, None]
        
        return sqrt_alpha * x_0 + sqrt_one_minus_alpha * noise
    
    def training_loss(self, model, x_0):
        """Compute training loss for one batch."""
        batch_size = x_0.shape[0]
        
        # Random timestep for each image in batch
        t = torch.randint(0, self.timesteps, (batch_size,), device=self.device)
        
        # Add noise
        noise = torch.randn_like(x_0)
        x_t = self.add_noise(x_0, t, noise)
        
        # Predict noise
        predicted_noise = model(x_t, t)
        
        # Simple MSE loss
        return F.mse_loss(predicted_noise, noise)
    
    @torch.no_grad()
    def sample(self, model, shape):
        """Generate images by iterative denoising (DDPM sampling)."""
        # Start from pure noise
        x = torch.randn(shape, device=self.device)
        
        for t in reversed(range(self.timesteps)):
            t_batch = torch.full((shape[0],), t, device=self.device, dtype=torch.long)
            
            # Predict noise
            predicted_noise = model(x, t_batch)
            
            # Compute denoised estimate
            alpha = self.alphas[t]
            alpha_cumprod = self.alphas_cumprod[t]
            beta = self.betas[t]
            
            # Mean of p(x_{t-1} | x_t)
            mean = (1 / torch.sqrt(alpha)) * (
                x - (beta / torch.sqrt(1 - alpha_cumprod)) * predicted_noise
            )
            
            # Add noise (except at t=0)
            if t > 0:
                noise = torch.randn_like(x)
                sigma = torch.sqrt(self.posterior_variance[t])
                x = mean + sigma * noise
            else:
                x = mean
        
        return x

# === U-Net for noise prediction (simplified) ===
class TimeEmbedding(nn.Module):
    """Sinusoidal time embedding (like positional encoding in transformers)."""
    def __init__(self, dim):
        super().__init__()
        self.dim = dim
    
    def forward(self, t):
        half_dim = self.dim // 2
        embeddings = torch.exp(
            -torch.arange(half_dim, device=t.device) * 
            (np.log(10000) / (half_dim - 1))
        )
        embeddings = t[:, None] * embeddings[None, :]
        return torch.cat([torch.sin(embeddings), torch.cos(embeddings)], dim=-1)

# In practice, the denoising model is a U-Net with:
# 1. Time embedding injected via addition/FiLM at each block
# 2. Self-attention layers at lower resolutions
# 3. ResNet-style blocks with GroupNorm

# === Training loop ===
diffusion = SimpleDiffusion(timesteps=1000)
model = UNetForDiffusion(...)  # U-Net with time conditioning
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4)

for epoch in range(epochs):
    for images, _ in dataloader:
        images = images.to(device)  # Normalized to [-1, 1]
        
        loss = diffusion.training_loss(model, images)
        
        optimizer.zero_grad()
        loss.backward()
        nn.utils.clip_grad_norm_(model.parameters(), 1.0)
        optimizer.step()

# === Generate images ===
generated = diffusion.sample(model, shape=(16, 3, 64, 64))
# generated is in [-1, 1], convert to [0, 1] for display
generated = (generated + 1) / 2
```

### Sampling Speed Improvements

| Method | Steps | Quality | Key Idea |
|--------|-------|---------|----------|
| DDPM | 1000 | Best | Original, very slow |
| DDIM | 50-100 | Very good | Deterministic, skip steps |
| DPM-Solver | 20-50 | Very good | ODE solver |
| Consistency Models | 1-2 | Good | Distill multi-step into single step |
| LCM (Latent Consistency) | 4-8 | Good | Distilled for speed |

```python
# === DDIM Sampling (much faster — 50 steps instead of 1000) ===
@torch.no_grad()
def ddim_sample(model, diffusion, shape, num_steps=50, eta=0.0):
    """DDIM deterministic sampling (eta=0) or stochastic (eta=1=DDPM)."""
    # Select subset of timesteps
    step_size = diffusion.timesteps // num_steps
    timesteps = list(range(0, diffusion.timesteps, step_size))[::-1]
    
    x = torch.randn(shape, device=diffusion.device)
    
    for i, t in enumerate(timesteps):
        t_batch = torch.full((shape[0],), t, device=diffusion.device, dtype=torch.long)
        predicted_noise = model(x, t_batch)
        
        alpha_cumprod_t = diffusion.alphas_cumprod[t]
        alpha_cumprod_prev = (diffusion.alphas_cumprod[timesteps[i+1]] 
                             if i < len(timesteps) - 1 else torch.tensor(1.0))
        
        # Predict x_0
        pred_x0 = (x - torch.sqrt(1 - alpha_cumprod_t) * predicted_noise) / \
                   torch.sqrt(alpha_cumprod_t)
        pred_x0 = pred_x0.clamp(-1, 1)
        
        # Direction pointing to x_t
        sigma = eta * torch.sqrt((1 - alpha_cumprod_prev) / (1 - alpha_cumprod_t) *
                                  (1 - alpha_cumprod_t / alpha_cumprod_prev))
        dir_xt = torch.sqrt(1 - alpha_cumprod_prev - sigma**2) * predicted_noise
        
        # x_{t-1}
        noise = torch.randn_like(x) if t > 0 else 0
        x = torch.sqrt(alpha_cumprod_prev) * pred_x0 + dir_xt + sigma * noise
    
    return x
```

---

## 4.6 Stable Diffusion — Architecture Deep Dive

### What It Is
Stable Diffusion performs diffusion in a **compressed latent space** rather than pixel space, making it fast enough for practical use. It combines a VAE (compression), a U-Net (denoising), and CLIP (text understanding).

### Why It Matters
- **Democratized AI art:** First high-quality open-source text-to-image model
- **Practical speed:** Generates 512×512 images in seconds (vs hours for pixel-space diffusion)
- **Extensible:** ControlNet, LoRA, inpainting, img2img — huge ecosystem
- **Foundation:** Architecture used by SDXL, Stable Diffusion 3, Flux, etc.

### How It Works

```
Text Prompt: "A cat sitting on a throne, digital art"
         │
         ↓
┌─────────────────┐
│  Text Encoder   │  (CLIP ViT-L/14 — frozen)
│  "cat" → [emb]  │  Output: (77, 768) token embeddings
└────────┬────────┘
         │ Cross-Attention
         ↓
┌─────────────────┐      ┌─────────────────┐
│  Latent Noise   │  →   │  U-Net Denoiser │  ← Timestep t
│  z_T ~ N(0,1)   │      │  (predicts noise)│
│  (4, 64, 64)    │      │  Iterates T steps│
└─────────────────┘      └────────┬────────┘
                                   │
                              z_0 (denoised latent)
                                   │
                                   ↓
                          ┌─────────────────┐
                          │  VAE Decoder    │  (converts latent → pixels)
                          │  (4,64,64)→     │
                          │  (3,512,512)    │
                          └─────────────────┘
                                   │
                                   ↓
                             Final Image (512×512)
```

**Three components:**

1. **VAE:** Compresses 512×512×3 → 64×64×4 (64× compression!) and decodes back
2. **U-Net:** Operates on the 64×64×4 latent, conditioned on text + timestep
3. **Text Encoder (CLIP):** Converts text prompt to embeddings for conditioning

**Classifier-Free Guidance (CFG):**

$$\epsilon_{guided} = \epsilon_{uncond} + w \cdot (\epsilon_{cond} - \epsilon_{uncond})$$

Where $w$ = guidance scale (typically 7-12). Higher = more adherent to prompt but less diverse.

### Code Examples

```python
# === Using Stable Diffusion with diffusers library ===
from diffusers import StableDiffusionPipeline, DPMSolverMultistepScheduler
import torch

# Load model (downloads ~4GB first time)
pipe = StableDiffusionPipeline.from_pretrained(
    "stabilityai/stable-diffusion-2-1",
    torch_dtype=torch.float16,  # Half precision for speed
)
pipe = pipe.to("cuda")

# Use faster scheduler
pipe.scheduler = DPMSolverMultistepScheduler.from_config(pipe.scheduler.config)

# === Text-to-Image ===
prompt = "A majestic lion in a field of sunflowers, oil painting, detailed"
negative_prompt = "blurry, low quality, deformed, ugly"

image = pipe(
    prompt=prompt,
    negative_prompt=negative_prompt,
    num_inference_steps=25,      # More steps = better quality
    guidance_scale=7.5,           # CFG scale (7-12 typical)
    height=512,
    width=512,
).images[0]

image.save("generated.png")

# === Image-to-Image (modify existing image) ===
from diffusers import StableDiffusionImg2ImgPipeline
from PIL import Image

pipe_img2img = StableDiffusionImg2ImgPipeline.from_pretrained(
    "stabilityai/stable-diffusion-2-1",
    torch_dtype=torch.float16,
).to("cuda")

init_image = Image.open("photo.jpg").resize((512, 512))
image = pipe_img2img(
    prompt="Same scene but in winter with snow",
    image=init_image,
    strength=0.75,  # 0=no change, 1=complete regeneration
    guidance_scale=7.5,
).images[0]

# === Inpainting (fill in masked regions) ===
from diffusers import StableDiffusionInpaintPipeline

pipe_inpaint = StableDiffusionInpaintPipeline.from_pretrained(
    "stabilityai/stable-diffusion-2-inpainting",
    torch_dtype=torch.float16,
).to("cuda")

image = Image.open("photo.jpg").resize((512, 512))
mask = Image.open("mask.png").resize((512, 512))  # White = area to fill

result = pipe_inpaint(
    prompt="A beautiful garden with flowers",
    image=image,
    mask_image=mask,
    guidance_scale=7.5,
).images[0]

# === ControlNet (precise structural control) ===
from diffusers import StableDiffusionControlNetPipeline, ControlNetModel
import cv2
import numpy as np

# Load ControlNet for edge guidance
controlnet = ControlNetModel.from_pretrained(
    "lllyasviel/control_v11p_sd15_canny",
    torch_dtype=torch.float16,
)

pipe_controlnet = StableDiffusionControlNetPipeline.from_pretrained(
    "runwayml/stable-diffusion-v1-5",
    controlnet=controlnet,
    torch_dtype=torch.float16,
).to("cuda")

# Generate Canny edges as control signal
image = cv2.imread("photo.jpg")
edges = cv2.Canny(image, 100, 200)
control_image = Image.fromarray(edges)

result = pipe_controlnet(
    prompt="A beautiful anime scene",
    image=control_image,
    num_inference_steps=20,
).images[0]

# === LoRA (Low-Rank Adaptation — fine-tune with minimal resources) ===
# Train a LoRA on custom images (e.g., your face, a specific style)
# Then load it as an adapter:
pipe.load_lora_weights("path/to/lora_weights")
image = pipe("portrait in custom_style style").images[0]
pipe.unload_lora_weights()
```

> **Pro Tip:** For production use, enable attention slicing and VAE slicing to reduce VRAM usage: `pipe.enable_attention_slicing()`, `pipe.enable_vae_slicing()`. This allows generating on GPUs with as little as 4GB VRAM.

---

## 4.7 Image-to-Image Translation

### What It Is
Converting an image from one domain to another while preserving structure. Examples: sketch→photo, day→night, horse→zebra, low-res→high-res.

### Approaches Summary

| Method | Requires Paired Data? | Example Use |
|--------|----------------------|-------------|
| Pix2Pix | Yes | Sketch → Photo |
| CycleGAN | No | Horse → Zebra |
| UNIT | No | Day → Night |
| Stable Diffusion img2img | No (text-guided) | Style transfer |
| InstructPix2Pix | No (instruction-guided) | "Make it winter" |

### Code Examples

```python
# === Style Transfer with Neural Style Transfer (classic approach) ===
import torch
import torchvision.models as models
import torchvision.transforms as T

def neural_style_transfer(content_img, style_img, num_steps=300, 
                          style_weight=1e6, content_weight=1):
    """
    Gatys et al. (2015) Neural Style Transfer.
    Optimizes the generated image to match:
    - Content of content_img (from later CNN layers)
    - Style of style_img (from Gram matrices of early layers)
    """
    # Load VGG-19 features
    vgg = models.vgg19(weights='IMAGENET1K_V1').features.eval()
    
    # Content layers (deep = semantic structure)
    content_layers = ['conv4_2']
    # Style layers (multiple scales of texture)
    style_layers = ['conv1_1', 'conv2_1', 'conv3_1', 'conv4_1', 'conv5_1']
    
    # Initialize generated image (start from content)
    generated = content_img.clone().requires_grad_(True)
    optimizer = torch.optim.Adam([generated], lr=0.01)
    
    for step in range(num_steps):
        # Extract features for content, style, and generated
        # Compute content loss (MSE between feature maps)
        # Compute style loss (MSE between Gram matrices)
        # Update generated image
        pass  # (full implementation omitted for brevity)
    
    return generated

# === Fast Style Transfer (feed-forward network, real-time) ===
# Train a network to apply a specific style in one forward pass
# Much faster than optimization-based NST (~1000× speedup)

# === Modern approach: Stable Diffusion for style transfer ===
from diffusers import StableDiffusionImg2ImgPipeline

pipe = StableDiffusionImg2ImgPipeline.from_pretrained(
    "stabilityai/stable-diffusion-2-1", torch_dtype=torch.float16
).to("cuda")

result = pipe(
    prompt="in the style of Van Gogh's Starry Night",
    image=init_image,
    strength=0.6,  # Lower = more faithful to original structure
    guidance_scale=7.5,
).images[0]
```

---

## 4.8 Super-Resolution

### What It Is
Super-resolution (SR) increases image resolution by generating plausible high-frequency details that didn't exist in the low-resolution input. It's "CSI enhance" but real.

### Why It Matters
- **Surveillance:** Enhance low-res CCTV footage
- **Medical:** Improve resolution of scans without re-scanning
- **Old media:** Upscale old photos/videos
- **Gaming:** DLSS uses SR for real-time frame upscaling

### Approaches

| Method | Type | Speed | Quality |
|--------|------|-------|---------|
| Bicubic | Interpolation | Instant | Blurry |
| SRCNN | CNN | Fast | Decent |
| ESRGAN | GAN-based | Medium | Very sharp |
| Real-ESRGAN | GAN + degradation | Medium | Best for real photos |
| SwinIR | Transformer | Slow | State-of-art |
| Stable Diffusion Upscaler | Diffusion | Slow | Highest quality |

### Code Examples

```python
# === Using Real-ESRGAN (practical, production-ready) ===
# pip install realesrgan

from realesrgan import RealESRGANer
from basicsr.archs.rrdbnet_arch import RRDBNet
import cv2

# Setup model (4× upscaling)
model = RRDBNet(num_in_ch=3, num_out_ch=3, num_feat=64, 
                num_block=23, num_grow_ch=32, scale=4)

upsampler = RealESRGANer(
    scale=4,
    model_path='RealESRGAN_x4plus.pth',
    model=model,
    tile=400,       # Process in tiles to save memory
    tile_pad=10,
    pre_pad=0,
    half=True,      # FP16 for speed
)

# Upscale
img = cv2.imread('low_res.jpg', cv2.IMREAD_UNCHANGED)
output, _ = upsampler.enhance(img, outscale=4)
cv2.imwrite('high_res.png', output)

# === Simple SRCNN (to understand the concept) ===
class SRCNN(nn.Module):
    """Super-Resolution CNN (Dong et al., 2014)."""
    def __init__(self):
        super().__init__()
        # Patch extraction and representation
        self.conv1 = nn.Conv2d(3, 64, 9, padding=4)
        # Non-linear mapping
        self.conv2 = nn.Conv2d(64, 32, 5, padding=2)
        # Reconstruction
        self.conv3 = nn.Conv2d(32, 3, 5, padding=4)
        self.relu = nn.ReLU(inplace=True)
    
    def forward(self, x):
        # Input: bicubic-upscaled low-res image
        x = self.relu(self.conv1(x))
        x = self.relu(self.conv2(x))
        x = self.conv3(x)
        return x

# === Stable Diffusion 4× Upscaler ===
from diffusers import StableDiffusionUpscalePipeline

pipe = StableDiffusionUpscalePipeline.from_pretrained(
    "stabilityai/stable-diffusion-x4-upscaler",
    torch_dtype=torch.float16,
).to("cuda")

low_res = Image.open("low_res.png")  # e.g., 128×128
upscaled = pipe(
    prompt="high quality, detailed",
    image=low_res,
).images[0]  # 512×512 with generated details
```

---

## 4.9 Image Inpainting

### What It Is
Inpainting fills in **missing or masked regions** of an image with plausible content. Like having an AI artist paint over a hole in a canvas, matching the surrounding context.

### Why It Matters
- **Photo editing:** Remove unwanted objects (people, watermarks)
- **Restoration:** Repair damaged/corrupted images
- **Content creation:** Extend image boundaries (outpainting)
- **Medical:** Fill in missing scan data

### Code Examples

```python
# === OpenCV inpainting (classical, fast) ===
import cv2
import numpy as np

image = cv2.imread('damaged.jpg')
mask = cv2.imread('mask.png', cv2.IMREAD_GRAYSCALE)  # White = area to fill

# Method 1: Navier-Stokes based (good for small regions)
result_ns = cv2.inpaint(image, mask, inpaintRadius=3, flags=cv2.INPAINT_NS)

# Method 2: Fast Marching Method (good for thin scratches)
result_telea = cv2.inpaint(image, mask, inpaintRadius=3, flags=cv2.INPAINT_TELEA)

# === Deep learning inpainting with LaMa ===
# LaMa (Large Mask Inpainting) — handles large missing regions
# pip install simple-lama-inpainting
from simple_lama_inpainting import SimpleLama

lama = SimpleLama()
result = lama(image, mask)  # Both as PIL Images

# === Stable Diffusion Inpainting (best quality) ===
from diffusers import StableDiffusionInpaintPipeline
from PIL import Image

pipe = StableDiffusionInpaintPipeline.from_pretrained(
    "stabilityai/stable-diffusion-2-inpainting",
    torch_dtype=torch.float16,
).to("cuda")

image = Image.open("photo.jpg").resize((512, 512))
mask = Image.open("mask.png").resize((512, 512))

result = pipe(
    prompt="seamless continuation of the background",
    image=image,
    mask_image=mask,
    num_inference_steps=25,
    guidance_scale=7.5,
).images[0]
result.save("inpainted.png")

# === Outpainting (extend image boundaries) ===
# Extend canvas and inpaint the new regions
def outpaint(image, direction='right', extend_pixels=256):
    """Extend image in a given direction."""
    w, h = image.size
    
    if direction == 'right':
        new_image = Image.new('RGB', (w + extend_pixels, h), (128, 128, 128))
        new_image.paste(image, (0, 0))
        mask = Image.new('L', (w + extend_pixels, h), 0)
        mask.paste(255, (w, 0, w + extend_pixels, h))  # White = fill area
    
    # Use inpainting pipeline on extended image + mask
    return pipe(prompt="natural continuation", image=new_image, 
                mask_image=mask).images[0]
```

---

## 4.10 Evaluation Metrics for Generation

### What It Is
Evaluating generative models is uniquely challenging — there's no single "correct" output. We need metrics that capture both quality (do images look real?) and diversity (does the model generate varied outputs?).

### Core Metrics

#### FID (Fréchet Inception Distance) — The Standard

Measures the distance between the distribution of generated images and real images in Inception-v3 feature space.

$$FID = \|\mu_r - \mu_g\|^2 + \text{Tr}(\Sigma_r + \Sigma_g - 2(\Sigma_r \Sigma_g)^{1/2})$$

- Lower FID = better (0 = identical distributions)
- Captures both quality AND diversity
- Needs ~10K+ generated images for reliability

#### IS (Inception Score)

$$IS = \exp(\mathbb{E}[D_{KL}(p(y|x) \| p(y))])$$

- Higher IS = better
- Measures: sharp class predictions + diverse outputs
- Weakness: doesn't compare against real data

#### CLIP Score (for text-to-image)

$$\text{CLIP Score} = \cos(E_{img}(x), E_{text}(prompt))$$

Measures how well the generated image matches the text prompt.

### Code Examples

```python
# === FID Calculation ===
# pip install pytorch-fid
# Command line:
# python -m pytorch_fid path/to/real_images path/to/generated_images

# Or programmatically:
from pytorch_fid import fid_score

fid_value = fid_score.calculate_fid_given_paths(
    ['path/to/real_images', 'path/to/generated_images'],
    batch_size=50,
    device='cuda',
    dims=2048,  # Inception feature dimension
)
print(f"FID: {fid_value:.2f}")

# === CLIP Score ===
import clip
from PIL import Image

model, preprocess = clip.load("ViT-B/32", device="cuda")

def clip_score(image, text):
    """Compute CLIP similarity between image and text."""
    image_input = preprocess(image).unsqueeze(0).to("cuda")
    text_input = clip.tokenize([text]).to("cuda")
    
    with torch.no_grad():
        image_features = model.encode_image(image_input)
        text_features = model.encode_text(text_input)
    
    # Cosine similarity
    similarity = torch.cosine_similarity(image_features, text_features)
    return similarity.item()

score = clip_score(Image.open("generated.png"), "A cat on a throne")
print(f"CLIP Score: {score:.3f}")

# === LPIPS (Perceptual Similarity) ===
# pip install lpips
import lpips

loss_fn = lpips.LPIPS(net='alex').to('cuda')

# Lower LPIPS = more similar (opposite of metrics above)
img1 = load_and_preprocess(img1_path)  # Normalized to [-1, 1]
img2 = load_and_preprocess(img2_path)
distance = loss_fn(img1, img2)
print(f"LPIPS distance: {distance.item():.4f}")
```

### Metric Reference

| Metric | Measures | Range | Better | Min Samples |
|--------|----------|-------|--------|-------------|
| FID | Quality + Diversity | 0–∞ | Lower | ~10K |
| IS | Quality + Diversity | 1–∞ | Higher | ~5K |
| CLIP Score | Text-Image alignment | -1 to 1 | Higher | Any |
| LPIPS | Perceptual similarity | 0–1 | Lower | Pairs |
| PSNR | Pixel-level similarity | 0–∞ dB | Higher | Pairs |
| SSIM | Structural similarity | -1 to 1 | Higher | Pairs |

---

## 4.11 Training Your Own Generative Model

### When to Train vs Use Pretrained

| Scenario | Approach | Why |
|----------|---------|-----|
| General image generation | Use Stable Diffusion as-is | Already excellent |
| Specific style | Fine-tune with LoRA | Efficient, ~1 hour |
| Specific subject (your face) | DreamBooth or Textual Inversion | ~20 images needed |
| Specific domain (medical, satellite) | Full fine-tune or train from scratch | Domain too different |
| 64×64 experiments | Train small diffusion model | Learning/research |

### Fine-tuning Stable Diffusion with LoRA

```python
# === LoRA fine-tuning (Low-Rank Adaptation) ===
# Uses ~100-1000× less memory than full fine-tuning
# Trains in ~1-2 hours on a single GPU

# Using the diffusers library training script:
"""
accelerate launch train_text_to_image_lora.py \
  --pretrained_model_name_or_path="stabilityai/stable-diffusion-2-1" \
  --dataset_name="path/to/your/images" \
  --resolution=512 \
  --train_batch_size=1 \
  --gradient_accumulation_steps=4 \
  --max_train_steps=1000 \
  --learning_rate=1e-4 \
  --lr_scheduler="cosine" \
  --output_dir="./lora-model" \
  --rank=4  # LoRA rank (4-16 typical)
"""

# === DreamBooth (personalize with ~5-20 images) ===
"""
accelerate launch train_dreambooth.py \
  --pretrained_model_name_or_path="stabilityai/stable-diffusion-2-1" \
  --instance_data_dir="path/to/subject/images" \
  --instance_prompt="a photo of sks person" \
  --class_data_dir="path/to/class/images" \
  --class_prompt="a photo of a person" \
  --resolution=512 \
  --train_batch_size=1 \
  --max_train_steps=800 \
  --learning_rate=5e-6 \
  --with_prior_preservation --prior_loss_weight=1.0
"""

# === Textual Inversion (learn a new "word" for your concept) ===
"""
accelerate launch textual_inversion.py \
  --pretrained_model_name_or_path="stabilityai/stable-diffusion-2-1" \
  --train_data_dir="path/to/concept/images" \
  --learnable_property="object" \
  --placeholder_token="<my-concept>" \
  --initializer_token="dog" \
  --resolution=512 \
  --train_batch_size=1 \
  --max_train_steps=3000
"""
```

### Training a Small Diffusion Model from Scratch

```python
"""
Train a small unconditional diffusion model on a custom dataset.
Useful for learning and for small, domain-specific datasets.
"""
import torch
import torch.nn as nn
from torch.utils.data import DataLoader
from torchvision import datasets, transforms
from tqdm import tqdm

# === Configuration ===
class Config:
    image_size = 64
    channels = 3
    timesteps = 1000
    batch_size = 64
    epochs = 100
    lr = 2e-4
    device = 'cuda'

cfg = Config()

# === Dataset ===
transform = transforms.Compose([
    transforms.Resize(cfg.image_size),
    transforms.CenterCrop(cfg.image_size),
    transforms.RandomHorizontalFlip(),
    transforms.ToTensor(),
    transforms.Normalize([0.5], [0.5]),  # Scale to [-1, 1]
])

dataset = datasets.ImageFolder('path/to/images', transform=transform)
dataloader = DataLoader(dataset, batch_size=cfg.batch_size, shuffle=True, num_workers=4)

# === Model (simplified U-Net — use a proper one for real training) ===
# In practice, use: from diffusers import UNet2DModel
from diffusers import UNet2DModel

model = UNet2DModel(
    sample_size=cfg.image_size,
    in_channels=cfg.channels,
    out_channels=cfg.channels,
    layers_per_block=2,
    block_out_channels=(128, 256, 256, 512),
    down_block_types=(
        "DownBlock2D", "DownBlock2D", "AttnDownBlock2D", "AttnDownBlock2D",
    ),
    up_block_types=(
        "AttnUpBlock2D", "AttnUpBlock2D", "UpBlock2D", "UpBlock2D",
    ),
).to(cfg.device)

# === Noise scheduler ===
from diffusers import DDPMScheduler

scheduler = DDPMScheduler(num_train_timesteps=cfg.timesteps)

# === Training ===
optimizer = torch.optim.AdamW(model.parameters(), lr=cfg.lr)

for epoch in range(cfg.epochs):
    total_loss = 0
    for batch, _ in tqdm(dataloader):
        batch = batch.to(cfg.device)
        
        # Sample random timesteps
        t = torch.randint(0, cfg.timesteps, (batch.shape[0],), device=cfg.device).long()
        
        # Add noise
        noise = torch.randn_like(batch)
        noisy = scheduler.add_noise(batch, noise, t)
        
        # Predict noise
        pred_noise = model(noisy, t).sample
        
        # Loss
        loss = nn.functional.mse_loss(pred_noise, noise)
        
        optimizer.zero_grad()
        loss.backward()
        nn.utils.clip_grad_norm_(model.parameters(), 1.0)
        optimizer.step()
        
        total_loss += loss.item()
    
    avg_loss = total_loss / len(dataloader)
    print(f"Epoch {epoch+1}/{cfg.epochs} | Loss: {avg_loss:.4f}")

# === Generate ===
from diffusers import DDPMPipeline

pipeline = DDPMPipeline(unet=model, scheduler=scheduler)
images = pipeline(batch_size=8, num_inference_steps=1000).images
```

---

## 4.12 Ethical Considerations and Limitations

### Key Concerns

| Issue | Description | Mitigation |
|-------|-------------|-----------|
| Deepfakes | Generating fake images of real people | Watermarking, detection tools |
| Bias | Models reflect training data biases | Diverse training data, bias auditing |
| Copyright | Generated images may resemble copyrighted works | Filtering, attribution |
| Misinformation | Fake news imagery | Provenance tracking, C2PA |
| Non-consensual content | NSFW generation of real people | Safety filters, legal frameworks |

### Technical Limitations

| Limitation | Details | Workaround |
|-----------|---------|-----------|
| Hands/fingers | Often generates wrong number of fingers | Negative prompts, inpainting, ControlNet |
| Text in images | Garbled text generation | Specialized text-rendering models |
| Consistency | Same character varies between generations | DreamBooth, IP-Adapter, consistent seeds |
| Counting | Can't reliably generate "exactly 3 cats" | ControlNet, region-specific prompting |
| Spatial relationships | "X to the left of Y" often fails | ControlNet, layout guidance |

---

## Interview Questions

1. **Q: Explain the GAN training procedure and what mode collapse is.**
   - A: GANs train two networks adversarially — Generator creates fakes, Discriminator detects them. Mode collapse occurs when the Generator finds a few outputs that fool the Discriminator and keeps producing only those, ignoring the full data diversity. Solutions: Wasserstein loss, minibatch discrimination, spectral normalization.

2. **Q: Why are diffusion models better than GANs for image generation?**
   - A: Training stability (no adversarial game), mode coverage (no collapse), controllability (easy conditioning via cross-attention), and quality. GANs are faster at inference but diffusion models produce more diverse, higher-fidelity images. GANs are still preferred when speed is critical.

3. **Q: Explain the reparameterization trick in VAEs and why it's needed.**
   - A: VAE's encoder outputs μ and σ, then we sample z ~ N(μ, σ²). But "sampling" is not differentiable — we can't backpropagate through randomness. The trick: z = μ + σ·ε where ε ~ N(0,1). Now the randomness is external, and the computation is differentiable with respect to μ and σ.

4. **Q: What is classifier-free guidance and why is it used?**
   - A: During training, randomly drop the conditioning (text) with some probability. At inference, compute both conditional and unconditional predictions, then extrapolate: ε_guided = ε_uncond + w·(ε_cond - ε_uncond). Higher w = stronger adherence to prompt. It improves quality dramatically without needing a separate classifier.

5. **Q: Why does Stable Diffusion work in latent space instead of pixel space?**
   - A: Pixel-space diffusion on 512×512×3 images is computationally prohibitive (millions of pixels to denoise iteratively). The VAE compresses to 64×64×4 (64× reduction), making diffusion 64× faster while preserving important image structure. The perceptual quality loss from compression is minimal.

6. **Q: Explain FID and its limitations.**
   - A: FID measures the Fréchet distance between real and generated image distributions in Inception-v3 feature space. Lower = better. Limitations: needs many samples (>10K), depends on Inception features (not perfect for all domains), doesn't capture per-image quality, and different resizing/preprocessing gives different scores.

7. **Q: What is LoRA and why is it preferred for fine-tuning diffusion models?**
   - A: LoRA (Low-Rank Adaptation) freezes the original model and adds small trainable matrices to attention layers: W' = W + BA where B∈R^(d×r), A∈R^(r×d), r << d. Benefits: trains 100-1000× fewer parameters, multiple LoRAs can be composed/switched, original model stays intact, and fine-tunes in 1-2 hours on consumer GPUs.

8. **Q: How would you evaluate a text-to-image model?**
   - A: Multi-metric approach: FID for overall quality/diversity, CLIP Score for text-image alignment, human evaluation for subjective quality, LPIPS for diversity between samples, and task-specific metrics. No single metric is sufficient. Human evaluation remains the gold standard but is expensive.

---

## Quick Reference

### Generation Method Selection

| Need | Method | Tool/Library |
|------|--------|-------------|
| Text → Image | Stable Diffusion | diffusers |
| Image → Image (style) | SD img2img / ControlNet | diffusers |
| Super-resolution | Real-ESRGAN / SD upscaler | realesrgan / diffusers |
| Inpainting | SD Inpainting / LaMa | diffusers |
| Face generation | StyleGAN | NVIDIA repos |
| Custom concept | DreamBooth / LoRA | diffusers |
| Video generation | AnimateDiff / SVD | diffusers |

### Stable Diffusion Parameters Guide

| Parameter | Range | Effect |
|-----------|-------|--------|
| guidance_scale | 1–20 | Higher = more prompt adherent, less diverse |
| num_inference_steps | 20–50 | More = better quality, slower |
| strength (img2img) | 0–1 | Higher = more change from original |
| negative_prompt | text | Describe what to avoid |
| seed | int | Fix for reproducibility |

### Model Size & Speed

| Model | Parameters | VRAM | Speed (512×512) |
|-------|-----------|------|-----------------|
| SD 1.5 | 860M | ~4GB (fp16) | ~3s (25 steps) |
| SD 2.1 | 865M | ~5GB (fp16) | ~3s (25 steps) |
| SDXL | 3.5B | ~7GB (fp16) | ~6s (25 steps) |
| SD 3 | 2B | ~12GB | ~4s (28 steps) |
| Flux.1 | 12B | ~24GB | ~10s |

### GAN vs VAE vs Diffusion

| Aspect | GAN | VAE | Diffusion |
|--------|-----|-----|-----------|
| Training | Unstable | Stable | Stable |
| Quality | High | Medium | Highest |
| Diversity | Mode collapse risk | Good | Excellent |
| Inference Speed | Fast (~ms) | Fast (~ms) | Slow (~s) |
| Controllability | Limited | Moderate | Excellent |
| Interpolation | Possible | Smooth | Possible |
| Likelihood | No | Yes (ELBO) | Yes (approx) |
| Best for 2024+ | Real-time (gaming) | Components | Primary choice |
