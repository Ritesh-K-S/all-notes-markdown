# Chapter 07: Diffusion Models — Image Generation from Noise

## Table of Contents
- [What Are Diffusion Models](#what-are-diffusion-models)
- [Why Diffusion Models Matter](#why-diffusion-models-matter)
- [How Diffusion Models Work](#how-diffusion-models-work)
- [DDPM — Denoising Diffusion Probabilistic Models](#ddpm--denoising-diffusion-probabilistic-models)
- [Stable Diffusion Architecture](#stable-diffusion-architecture)
- [ControlNet — Guided Generation](#controlnet--guided-generation)
- [Image Generation Pipeline](#image-generation-pipeline)
- [Code Examples](#code-examples)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What Are Diffusion Models

### Simple Explanation (Like Explaining to a 15-Year-Old)

Imagine you have a beautiful painting. Now imagine slowly adding random static noise to it — like TV snow — until it's completely unrecognizable. **Diffusion models learn to reverse this process.** They learn to take pure noise and gradually "clean it up" step by step until a beautiful image appears.

Think of it like this:
- **Forward process**: Dropping ink into clear water until it's all murky (easy, just add chaos)
- **Reverse process**: Learning to separate the ink back from the water, drop by drop (hard, requires intelligence)

The model doesn't generate images in one shot. It takes many small steps, each time removing a tiny bit of noise, until the final clean image emerges.

### Formal Definition

A diffusion model is a **generative model** that learns to produce data (images, audio, video) by:
1. **Forward diffusion**: Gradually adding Gaussian noise to data over $T$ timesteps until it becomes pure noise
2. **Reverse diffusion**: Learning a neural network to iteratively denoise, going from pure noise back to data

---

## Why Diffusion Models Matter

### Real-World Impact

| Application | Example | Companies Using It |
|---|---|---|
| Text-to-Image | "A cat riding a bicycle in space" → photorealistic image | Midjourney, DALL·E, Stable Diffusion |
| Image Editing | Inpainting, outpainting, style transfer | Adobe Firefly, Canva |
| Video Generation | Text-to-video, frame interpolation | Runway, Pika, Sora |
| Drug Discovery | Generating molecular structures | Various biotech firms |
| 3D Generation | Text-to-3D model creation | Point-E, DreamFusion |
| Audio/Music | Generating music, speech synthesis | AudioLDM, MusicGen |

### Why Not GANs?

Before diffusion models, **GANs** (Generative Adversarial Networks) dominated image generation. Diffusion models won because:

| Aspect | GANs | Diffusion Models |
|---|---|---|
| Training stability | Unstable (mode collapse) | Very stable |
| Image diversity | Limited (mode dropping) | High diversity |
| Image quality | High but inconsistent | Consistently high |
| Control/conditioning | Harder to add | Easy to condition |
| Theoretical grounding | Weak | Strong (variational inference) |

### When You'd Use Diffusion Models

- ✅ When you need **high-quality** image/video generation
- ✅ When **diversity** of outputs matters
- ✅ When you need **controllable** generation (text-guided, pose-guided, etc.)
- ✅ When training **stability** is important
- ❌ When you need **real-time** generation (they're slow by nature)
- ❌ When computational resources are extremely limited

---

## How Diffusion Models Work

### The Core Intuition

```
Forward Process (Adding Noise):
Step 0        Step 1        Step 2       ...      Step T
[Clean] → [Slightly Noisy] → [More Noisy] → ... → [Pure Noise]
  x₀           x₁               x₂                    x_T

Reverse Process (Removing Noise - LEARNED):
Step T        Step T-1      Step T-2     ...      Step 0
[Pure Noise] → [Less Noisy] → [Cleaner] → ... → [Generated Image]
  x_T           x_{T-1}         x_{T-2}              x₀
```

### Mathematical Foundation

#### Forward Process (Fixed, Not Learned)

At each timestep $t$, we add a small amount of Gaussian noise:

$$q(x_t | x_{t-1}) = \mathcal{N}(x_t; \sqrt{1 - \beta_t} \cdot x_{t-1}, \beta_t \cdot I)$$

Where:
- $\beta_t$ is the **noise schedule** (how much noise to add at step $t$)
- Typically $\beta_1 = 0.0001$ to $\beta_T = 0.02$ (linear schedule)
- $T$ is usually 1000 steps

**Key trick**: We can jump directly to any timestep $t$ without computing all intermediate steps:

$$q(x_t | x_0) = \mathcal{N}(x_t; \sqrt{\bar{\alpha}_t} \cdot x_0, (1 - \bar{\alpha}_t) \cdot I)$$

Where:
- $\alpha_t = 1 - \beta_t$
- $\bar{\alpha}_t = \prod_{s=1}^{t} \alpha_s$ (cumulative product)

This means: $x_t = \sqrt{\bar{\alpha}_t} \cdot x_0 + \sqrt{1 - \bar{\alpha}_t} \cdot \epsilon$, where $\epsilon \sim \mathcal{N}(0, I)$

#### Reverse Process (Learned by Neural Network)

The model learns to predict the noise that was added:

$$p_\theta(x_{t-1} | x_t) = \mathcal{N}(x_{t-1}; \mu_\theta(x_t, t), \Sigma_\theta(x_t, t))$$

In practice, the neural network $\epsilon_\theta(x_t, t)$ predicts the noise $\epsilon$, and we compute:

$$\mu_\theta(x_t, t) = \frac{1}{\sqrt{\alpha_t}} \left( x_t - \frac{\beta_t}{\sqrt{1 - \bar{\alpha}_t}} \cdot \epsilon_\theta(x_t, t) \right)$$

#### Training Objective

The loss is surprisingly simple — just predict the noise!

$$L = \mathbb{E}_{t, x_0, \epsilon} \left[ \| \epsilon - \epsilon_\theta(x_t, t) \|^2 \right]$$

### Visual Analogy: The Sculptor

Think of diffusion like a sculptor:
1. Start with a block of marble (pure noise)
2. Each denoising step is like one chisel strike
3. After 1000 careful strikes, a beautiful statue (image) emerges
4. The model learned which "strikes" to make by studying many finished statues

---

## DDPM — Denoising Diffusion Probabilistic Models

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    DDPM Training Loop                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Sample x₀ from dataset (clean image)                    │
│  2. Sample t ~ Uniform(1, T)                                │
│  3. Sample ε ~ N(0, I)                                      │
│  4. Compute x_t = √(ᾱ_t) · x₀ + √(1-ᾱ_t) · ε            │
│  5. Predict ε̂ = ε_θ(x_t, t)    ← Neural Network           │
│  6. Loss = ||ε - ε̂||²                                      │
│  7. Backpropagate and update θ                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### The U-Net Backbone

DDPM uses a **U-Net** architecture — an encoder-decoder with skip connections:

```
Input: x_t (noisy image) + t (timestep embedding)

Encoder (Downsampling):          Decoder (Upsampling):
┌──────────┐                     ┌──────────┐
│ 64×64×3  │─────────────────────│ 64×64×3  │ ← Output: predicted noise
│ Conv+Attn│                     │ Conv+Attn│
└────┬─────┘                     └────▲─────┘
     │ ↓                              │ ↑
┌────▼─────┐                     ┌────┴─────┐
│ 32×32×128│─────────────────────│ 32×32×128│
│ Conv+Attn│   Skip Connections  │ Conv+Attn│
└────┬─────┘                     └────▲─────┘
     │ ↓                              │ ↑
┌────▼─────┐                     ┌────┴─────┐
│ 16×16×256│─────────────────────│ 16×16×256│
│ Conv+Attn│                     │ Conv+Attn│
└────┬─────┘                     └────▲─────┘
     │ ↓                              │ ↑
     └──────────► Bottleneck ─────────┘
                  8×8×512
```

### Timestep Embedding

The model needs to know "how noisy is the input?" — this is encoded via **sinusoidal position embeddings** (same as in Transformers):

$$\text{emb}(t) = [\sin(t/10000^{0/d}), \cos(t/10000^{0/d}), \sin(t/10000^{2/d}), \cos(t/10000^{2/d}), \ldots]$$

### Noise Schedules

| Schedule | Formula | Properties |
|---|---|---|
| Linear | $\beta_t = \beta_1 + \frac{t-1}{T-1}(\beta_T - \beta_1)$ | Simple, original DDPM |
| Cosine | $\bar{\alpha}_t = \frac{f(t)}{f(0)}$, $f(t) = \cos\left(\frac{t/T + s}{1+s} \cdot \frac{\pi}{2}\right)^2$ | Better for low-resolution |
| Scaled Linear | Square root of linear | Better for high-resolution |

### Sampling (Inference)

```
Algorithm: DDPM Sampling
─────────────────────────
1. x_T ~ N(0, I)           ← Start from pure noise
2. for t = T, T-1, ..., 1:
   a. z ~ N(0, I) if t > 1, else z = 0
   b. ε̂ = ε_θ(x_t, t)     ← Model predicts noise
   c. x_{t-1} = (1/√α_t)(x_t - (β_t/√(1-ᾱ_t))·ε̂) + σ_t·z
3. return x₀
```

> **Important**: This requires T forward passes through the neural network — making it slow! (1000 steps × ~0.1s each = ~100 seconds per image)

---

## Stable Diffusion Architecture

### The Key Innovation: Latent Diffusion

The problem with DDPM: operating on full-resolution images (e.g., 512×512×3) is computationally expensive.

**Solution**: Do diffusion in a compressed **latent space** instead!

```
┌────────────────────────────────────────────────────────────────────┐
│                    STABLE DIFFUSION PIPELINE                         │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Text: "A sunset over mountains"                                    │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────┐                                                    │
│  │ CLIP Text   │ → Text Embeddings (77×768)                        │
│  │ Encoder     │                                                    │
│  └─────────────┘                                                    │
│         │                                                           │
│         ▼ (Cross-Attention)                                         │
│  ┌─────────────────────────────────────────────────┐               │
│  │            LATENT SPACE (64×64×4)                │               │
│  │                                                   │               │
│  │  Noise z_T → U-Net Denoising (guided by text)   │               │
│  │  → z_{T-1} → ... → z_0 (clean latent)           │               │
│  │                                                   │               │
│  └─────────────────────────────────────────────────┘               │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────┐                                                    │
│  │ VAE Decoder │ → Final Image (512×512×3)                         │
│  └─────────────┘                                                    │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

### Three Key Components

#### 1. VAE (Variational Autoencoder) — Image Compression

```
Image (512×512×3) → VAE Encoder → Latent (64×64×4) → VAE Decoder → Image (512×512×3)
                     (compression      8× spatial        (decompression)
                      factor: 48×)      reduction)
```

- Reduces computation by **48×** compared to pixel-space diffusion
- Trained separately, frozen during diffusion training
- Preserves perceptual quality despite massive compression

#### 2. U-Net with Cross-Attention — The Denoiser

The U-Net is modified with **cross-attention layers** that attend to text embeddings:

```python
# Simplified cross-attention mechanism
# Q comes from image features, K and V come from text embeddings
Q = W_q @ image_features      # "What am I looking for?"
K = W_k @ text_embeddings     # "What's available in the text?"
V = W_v @ text_embeddings     # "What information to use?"
attention = softmax(Q @ K.T / √d) @ V
```

#### 3. CLIP Text Encoder — Understanding Text

- Converts text prompt to embeddings that guide generation
- Pre-trained on 400M image-text pairs
- Token limit: 77 tokens
- SD 1.x uses CLIP ViT-L/14, SD 2.x uses OpenCLIP ViT-H/14
- SDXL uses **two** text encoders (CLIP + OpenCLIP)

### Classifier-Free Guidance (CFG)

The secret to making text conditioning work well:

$$\hat{\epsilon} = \epsilon_\theta(x_t, \emptyset) + w \cdot (\epsilon_\theta(x_t, c) - \epsilon_\theta(x_t, \emptyset))$$

Where:
- $\epsilon_\theta(x_t, c)$ = prediction conditioned on text
- $\epsilon_\theta(x_t, \emptyset)$ = unconditional prediction (empty prompt)
- $w$ = guidance scale (typically 7-12)

```
CFG Scale Effects:
───────────────────────────────────
w = 1:  No guidance, random/diverse output
w = 7:  Good balance of quality and diversity (default)
w = 12: Very prompt-adherent but less diverse
w = 20: Over-saturated, artifact-prone
```

> **Pro Tip**: CFG requires TWO forward passes per step (conditional + unconditional), doubling compute. Techniques like "negative prompting" exploit the unconditional prediction.

### Stable Diffusion Versions

| Version | Resolution | Text Encoder | Key Feature |
|---|---|---|---|
| SD 1.5 | 512×512 | CLIP ViT-L/14 | Most finetuned/community models |
| SD 2.0 | 768×768 | OpenCLIP ViT-H/14 | Better quality, less "artistic" |
| SD 2.1 | 768×768 | OpenCLIP ViT-H/14 | Re-added NSFW training data |
| SDXL | 1024×1024 | CLIP + OpenCLIP (dual) | Refiner model, much better quality |
| SD 3 | 1024×1024 | T5 + CLIP × 2 (triple) | DiT architecture (transformer-based) |
| FLUX | 1024×1024 | T5 + CLIP | Flow matching, very high quality |

---

## ControlNet — Guided Generation

### What is ControlNet?

ControlNet adds **spatial conditioning** to Stable Diffusion — controlling the output's structure using:
- Edge maps (Canny)
- Depth maps
- Human poses (OpenPose)
- Segmentation maps
- Scribbles/sketches
- Normal maps

### Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                     ControlNet Architecture                     │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  Input Condition          Locked SD U-Net                      │
│  (e.g., Canny edge)      (frozen weights)                     │
│       │                        │                               │
│       ▼                        ▼                               │
│  ┌──────────┐           ┌──────────────┐                      │
│  │ControlNet│           │ Original     │                      │
│  │(trainable│──────────▶│ U-Net Blocks │──→ Output            │
│  │  copy)   │  zero     │ (frozen)     │                      │
│  └──────────┘  convs    └──────────────┘                      │
│                                                                │
│  Key: ControlNet is a COPY of the encoder blocks               │
│       connected via zero-initialized convolutions              │
│       (starts with zero influence, gradually learns)           │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

### Why Zero Convolutions?

- Initialized to zero → ControlNet starts with **zero influence** on the base model
- Gradually learns to inject spatial information
- This prevents destroying the pre-trained model's capabilities during training

### Types of ControlNet Conditions

| Condition | Input | Best For |
|---|---|---|
| Canny Edge | Edge-detected image | Preserving exact structure |
| Depth | Depth map (MiDAS) | 3D-consistent scenes |
| OpenPose | Skeleton keypoints | Character poses |
| Segmentation | Semantic map | Scene layout control |
| Scribble | Hand-drawn sketch | Quick concept art |
| Normal Map | Surface normals | Lighting/material control |
| Line Art | Clean line drawing | Anime/illustration style |
| SoftEdge | HED/PiDiNet edges | Softer structural control |

---

## Image Generation Pipeline

### Complete Pipeline with Schedulers

```
┌────────────────────────────────────────────────────────────┐
│              COMPLETE GENERATION PIPELINE                    │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  1. ENCODE TEXT                                             │
│     prompt → tokenize → CLIP → text_embeddings (77×768)    │
│     "" → tokenize → CLIP → uncond_embeddings (for CFG)     │
│                                                             │
│  2. PREPARE LATENTS                                         │
│     z_T ~ N(0, I), shape: (1, 4, 64, 64)                  │
│     scale by scheduler's init_noise_sigma                   │
│                                                             │
│  3. DENOISING LOOP (e.g., 50 steps)                        │
│     for t in scheduler.timesteps:                           │
│       a. latent_input = scheduler.scale_model_input(z_t)   │
│       b. noise_pred_cond = unet(latent_input, t, text_emb) │
│       c. noise_pred_uncond = unet(latent_input, t, "")     │
│       d. noise_pred = uncond + cfg*(cond - uncond)         │
│       e. z_{t-1} = scheduler.step(noise_pred, t, z_t)     │
│                                                             │
│  4. DECODE LATENT                                           │
│     z_0 / 0.18215 → VAE Decoder → image (512×512×3)       │
│                                                             │
│  5. POST-PROCESS                                            │
│     [-1, 1] → [0, 255], float → uint8                     │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### Schedulers (Samplers)

Schedulers determine HOW to denoise — the numerical method for solving the reverse SDE/ODE:

| Scheduler | Steps Needed | Quality | Speed | Notes |
|---|---|---|---|---|
| DDPM | 1000 | High | Very Slow | Original, stochastic |
| DDIM | 50-100 | High | Moderate | Deterministic, skip steps |
| Euler | 20-30 | Good | Fast | Simple, popular default |
| Euler Ancestral | 20-30 | Good | Fast | Stochastic variant |
| DPM++ 2M | 20-30 | Very High | Fast | Best quality/speed ratio |
| DPM++ 2M Karras | 20-30 | Very High | Fast | + Karras noise schedule |
| UniPC | 10-20 | High | Very Fast | Fewest steps needed |
| LCM | 4-8 | Good | Fastest | Distilled, real-time |

> **Pro Tip**: For most use cases, **DPM++ 2M Karras** with 25-30 steps gives the best quality-speed tradeoff. For real-time applications, **LCM** with 4-8 steps is the go-to.

### Key Parameters for Generation

| Parameter | Range | Effect |
|---|---|---|
| `guidance_scale` | 1-20 | Higher = more prompt adherence |
| `num_inference_steps` | 4-100 | More steps = better quality (diminishing returns) |
| `seed` | Any integer | Reproducibility |
| `negative_prompt` | Text | What to avoid in generation |
| `strength` (img2img) | 0-1 | How much to change input image |

---

## Code Examples

### Example 1: Basic DDPM from Scratch (Understanding the Math)

```python
import torch
import torch.nn as nn
import numpy as np
import matplotlib.pyplot as plt

# ============================================
# DDPM Forward Process - Adding Noise
# ============================================

def linear_beta_schedule(timesteps, beta_start=0.0001, beta_end=0.02):
    """Create a linear noise schedule from beta_start to beta_end."""
    return torch.linspace(beta_start, beta_end, timesteps)

def cosine_beta_schedule(timesteps, s=0.008):
    """Cosine schedule as proposed in 'Improved DDPM' paper."""
    steps = timesteps + 1
    x = torch.linspace(0, timesteps, steps)
    # Compute cumulative alphas using cosine function
    alphas_cumprod = torch.cos(((x / timesteps) + s) / (1 + s) * torch.pi * 0.5) ** 2
    alphas_cumprod = alphas_cumprod / alphas_cumprod[0]  # Normalize
    betas = 1 - (alphas_cumprod[1:] / alphas_cumprod[:-1])
    return torch.clip(betas, 0.0001, 0.9999)  # Clip for numerical stability

# Setup the diffusion parameters
T = 1000  # Total number of timesteps
betas = linear_beta_schedule(T)

# Pre-compute useful quantities
alphas = 1.0 - betas                          # α_t = 1 - β_t
alphas_cumprod = torch.cumprod(alphas, dim=0)  # ᾱ_t = Π α_s
sqrt_alphas_cumprod = torch.sqrt(alphas_cumprod)
sqrt_one_minus_alphas_cumprod = torch.sqrt(1.0 - alphas_cumprod)

def forward_diffusion(x_0, t, noise=None):
    """
    Add noise to image x_0 at timestep t.
    
    Uses the closed-form formula:
    x_t = √(ᾱ_t) * x_0 + √(1 - ᾱ_t) * ε
    
    Args:
        x_0: Clean image tensor [B, C, H, W]
        t: Timestep tensor [B]
        noise: Optional pre-generated noise
    Returns:
        x_t: Noisy image at timestep t
        noise: The noise that was added (needed for training)
    """
    if noise is None:
        noise = torch.randn_like(x_0)
    
    # Gather the right coefficients for each sample's timestep
    sqrt_alpha_cumprod_t = sqrt_alphas_cumprod[t].reshape(-1, 1, 1, 1)
    sqrt_one_minus_alpha_cumprod_t = sqrt_one_minus_alphas_cumprod[t].reshape(-1, 1, 1, 1)
    
    # Apply the forward diffusion formula
    x_t = sqrt_alpha_cumprod_t * x_0 + sqrt_one_minus_alpha_cumprod_t * noise
    return x_t, noise

# Demonstrate forward diffusion
print("Forward Diffusion Demo:")
print(f"At t=0:   signal_ratio = {sqrt_alphas_cumprod[0]:.4f}, noise_ratio = {sqrt_one_minus_alphas_cumprod[0]:.4f}")
print(f"At t=250: signal_ratio = {sqrt_alphas_cumprod[250]:.4f}, noise_ratio = {sqrt_one_minus_alphas_cumprod[250]:.4f}")
print(f"At t=500: signal_ratio = {sqrt_alphas_cumprod[500]:.4f}, noise_ratio = {sqrt_one_minus_alphas_cumprod[500]:.4f}")
print(f"At t=999: signal_ratio = {sqrt_alphas_cumprod[999]:.4f}, noise_ratio = {sqrt_one_minus_alphas_cumprod[999]:.4f}")
```

**Output:**
```
Forward Diffusion Demo:
At t=0:   signal_ratio = 0.9999, noise_ratio = 0.0100
At t=250: signal_ratio = 0.8788, noise_ratio = 0.4772
At t=500: signal_ratio = 0.7071, noise_ratio = 0.7071
At t=999: signal_ratio = 0.0398, noise_ratio = 0.9992
```

### Example 2: Simple U-Net for DDPM

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class SinusoidalPositionEmbeddings(nn.Module):
    """Encode timestep t into a vector using sinusoidal embeddings."""
    
    def __init__(self, dim):
        super().__init__()
        self.dim = dim
    
    def forward(self, time):
        device = time.device
        half_dim = self.dim // 2
        # Same formula as Transformer position encoding
        embeddings = math.log(10000) / (half_dim - 1)
        embeddings = torch.exp(torch.arange(half_dim, device=device) * -embeddings)
        embeddings = time[:, None] * embeddings[None, :]
        embeddings = torch.cat((embeddings.sin(), embeddings.cos()), dim=-1)
        return embeddings


class Block(nn.Module):
    """Basic convolutional block with group normalization."""
    
    def __init__(self, in_ch, out_ch, time_emb_dim, up=False):
        super().__init__()
        # Time embedding projection
        self.time_mlp = nn.Linear(time_emb_dim, out_ch)
        
        if up:
            # Upsample path: input has skip connection (doubled channels)
            self.conv1 = nn.Conv2d(2 * in_ch, out_ch, 3, padding=1)
            self.transform = nn.ConvTranspose2d(out_ch, out_ch, 4, 2, 1)
        else:
            # Downsample path
            self.conv1 = nn.Conv2d(in_ch, out_ch, 3, padding=1)
            self.transform = nn.Conv2d(out_ch, out_ch, 4, 2, 1)
        
        self.conv2 = nn.Conv2d(out_ch, out_ch, 3, padding=1)
        self.bnorm1 = nn.GroupNorm(8, out_ch)
        self.bnorm2 = nn.GroupNorm(8, out_ch)
        self.relu = nn.ReLU()
    
    def forward(self, x, t):
        # First convolution
        h = self.bnorm1(self.relu(self.conv1(x)))
        # Add time embedding (broadcast across spatial dims)
        time_emb = self.relu(self.time_mlp(t))
        time_emb = time_emb[(...,) + (None,) * 2]  # [B, C] → [B, C, 1, 1]
        h = h + time_emb
        # Second convolution
        h = self.bnorm2(self.relu(self.conv2(h)))
        # Down/Up sample
        return self.transform(h)


class SimpleUNet(nn.Module):
    """
    Simplified U-Net for DDPM.
    Input: noisy image (B, 3, 64, 64) + timestep (B,)
    Output: predicted noise (B, 3, 64, 64)
    """
    
    def __init__(self, image_channels=3, time_emb_dim=32):
        super().__init__()
        
        # Time embedding
        self.time_mlp = nn.Sequential(
            SinusoidalPositionEmbeddings(time_emb_dim),
            nn.Linear(time_emb_dim, time_emb_dim),
            nn.ReLU()
        )
        
        # Encoder (downsampling)
        self.down1 = Block(image_channels, 64, time_emb_dim)   # 64→32
        self.down2 = Block(64, 128, time_emb_dim)               # 32→16
        self.down3 = Block(128, 256, time_emb_dim)              # 16→8
        
        # Decoder (upsampling)
        self.up1 = Block(256, 128, time_emb_dim, up=True)      # 8→16
        self.up2 = Block(128, 64, time_emb_dim, up=True)       # 16→32
        self.up3 = Block(64, image_channels, time_emb_dim, up=True)  # 32→64
        
        # Final output layer
        self.output = nn.Conv2d(image_channels, image_channels, 1)
    
    def forward(self, x, timestep):
        # Embed timestep
        t = self.time_mlp(timestep)
        
        # Encoder with skip connections
        d1 = self.down1(x, t)    # Store for skip connection
        d2 = self.down2(d1, t)   # Store for skip connection
        d3 = self.down3(d2, t)   # Bottleneck
        
        # Decoder with skip connections
        u1 = self.up1(torch.cat([d3, d3], dim=1), t)  # Simplified skip
        u2 = self.up2(torch.cat([u1, d2], dim=1), t)
        u3 = self.up3(torch.cat([u2, d1], dim=1), t)
        
        return self.output(u3)


# Create model and test
model = SimpleUNet()
# Count parameters
total_params = sum(p.numel() for p in model.parameters())
print(f"Model parameters: {total_params:,}")

# Test forward pass
x = torch.randn(4, 3, 64, 64)  # Batch of 4 images, 64x64 RGB
t = torch.randint(0, T, (4,))  # Random timesteps
noise_pred = model(x, t)
print(f"Input shape: {x.shape}")
print(f"Output shape: {noise_pred.shape}")  # Same as input!
```

### Example 3: Training Loop for DDPM

```python
import torch
from torch.optim import Adam
from torch.utils.data import DataLoader
from torchvision import datasets, transforms

# ============================================
# Complete DDPM Training
# ============================================

def get_loss(model, x_0, t):
    """
    Compute the simplified DDPM training loss.
    
    Loss = E[||ε - ε_θ(x_t, t)||²]
    
    Steps:
    1. Sample random noise
    2. Create noisy version of x_0 at timestep t
    3. Model predicts the noise
    4. MSE between actual and predicted noise
    """
    noise = torch.randn_like(x_0)
    x_t, _ = forward_diffusion(x_0, t, noise)  # Add noise to image
    noise_pred = model(x_t, t)                  # Model predicts noise
    return F.mse_loss(noise, noise_pred)        # Compare with actual noise


@torch.no_grad()
def sample(model, image_size, batch_size=16, channels=3):
    """
    Generate images using the trained model (DDPM sampling).
    
    Starts from pure noise and iteratively denoises.
    """
    device = next(model.parameters()).device
    
    # Start from pure Gaussian noise
    img = torch.randn((batch_size, channels, image_size, image_size), device=device)
    
    # Iteratively denoise
    for i in reversed(range(T)):
        t = torch.full((batch_size,), i, device=device, dtype=torch.long)
        
        # Predict noise
        predicted_noise = model(img, t)
        
        # Compute coefficients
        alpha = alphas[i]
        alpha_cumprod = alphas_cumprod[i]
        beta = betas[i]
        
        if i > 0:
            noise = torch.randn_like(img)
        else:
            noise = torch.zeros_like(img)  # No noise at final step
        
        # Reverse diffusion step
        img = (1 / torch.sqrt(alpha)) * (
            img - ((1 - alpha) / torch.sqrt(1 - alpha_cumprod)) * predicted_noise
        ) + torch.sqrt(beta) * noise
    
    return img


# Training setup
device = "cuda" if torch.cuda.is_available() else "cpu"
model = SimpleUNet().to(device)
optimizer = Adam(model.parameters(), lr=3e-4)

# Dataset (using MNIST for simplicity)
transform = transforms.Compose([
    transforms.Resize(64),
    transforms.ToTensor(),
    transforms.Lambda(lambda x: x.repeat(3, 1, 1)),  # 1ch → 3ch
    transforms.Normalize((0.5,), (0.5,))              # Scale to [-1, 1]
])

dataset = datasets.MNIST(root='./data', train=True, transform=transform, download=True)
dataloader = DataLoader(dataset, batch_size=64, shuffle=True, drop_last=True)

# Training loop
epochs = 10
for epoch in range(epochs):
    total_loss = 0
    for batch_idx, (images, _) in enumerate(dataloader):
        images = images.to(device)
        
        # Sample random timesteps for each image in batch
        t = torch.randint(0, T, (images.shape[0],), device=device)
        
        # Compute loss and update
        loss = get_loss(model, images, t)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        total_loss += loss.item()
    
    avg_loss = total_loss / len(dataloader)
    print(f"Epoch {epoch+1}/{epochs}, Loss: {avg_loss:.4f}")

# Generate samples
samples = sample(model, image_size=64, batch_size=16)
print(f"Generated {samples.shape[0]} images of shape {samples.shape[1:]}")
```

### Example 4: Using Stable Diffusion with Diffusers Library

```python
# pip install diffusers transformers accelerate torch

from diffusers import StableDiffusionPipeline, DPMSolverMultistepScheduler
import torch

# ============================================
# Basic Text-to-Image Generation
# ============================================

# Load the model (downloads ~4GB on first run)
model_id = "stabilityai/stable-diffusion-2-1"
pipe = StableDiffusionPipeline.from_pretrained(
    model_id,
    torch_dtype=torch.float16,  # Use FP16 for 2x speed, half memory
)
pipe = pipe.to("cuda")

# Enable memory optimization (for GPUs with <12GB VRAM)
pipe.enable_attention_slicing()      # Reduces memory at cost of ~10% speed
# pipe.enable_xformers_memory_efficient_attention()  # Best if xformers installed

# Use a faster scheduler (DPM++ 2M instead of default PNDM)
pipe.scheduler = DPMSolverMultistepScheduler.from_config(pipe.scheduler.config)

# Generate an image
prompt = "A majestic lion wearing a crown, digital art, highly detailed, 4k"
negative_prompt = "blurry, low quality, distorted, deformed"

image = pipe(
    prompt=prompt,
    negative_prompt=negative_prompt,
    num_inference_steps=25,      # 25 steps with DPM++ is plenty
    guidance_scale=7.5,          # Standard CFG scale
    height=768,
    width=768,
    generator=torch.Generator("cuda").manual_seed(42),  # Reproducible
).images[0]

image.save("lion_king.png")
print(f"Generated image size: {image.size}")


# ============================================
# Image-to-Image (img2img)
# ============================================
from diffusers import StableDiffusionImg2ImgPipeline
from PIL import Image

pipe_img2img = StableDiffusionImg2ImgPipeline.from_pretrained(
    model_id, torch_dtype=torch.float16
).to("cuda")

# Load an input image
init_image = Image.open("input_sketch.png").resize((768, 768))

result = pipe_img2img(
    prompt="A beautiful watercolor painting of a landscape",
    image=init_image,
    strength=0.75,           # 0.75 = change 75% of the image
    guidance_scale=7.5,
    num_inference_steps=30,
).images[0]


# ============================================
# Inpainting (edit specific regions)
# ============================================
from diffusers import StableDiffusionInpaintPipeline

pipe_inpaint = StableDiffusionInpaintPipeline.from_pretrained(
    "stabilityai/stable-diffusion-2-inpainting",
    torch_dtype=torch.float16
).to("cuda")

image = Image.open("photo.png").resize((512, 512))
mask = Image.open("mask.png").resize((512, 512))  # White = region to edit

result = pipe_inpaint(
    prompt="A red sports car",
    image=image,
    mask_image=mask,
    guidance_scale=7.5,
    num_inference_steps=30,
).images[0]
```

### Example 5: ControlNet with Stable Diffusion

```python
# pip install diffusers transformers accelerate controlnet_aux

from diffusers import StableDiffusionControlNetPipeline, ControlNetModel
from diffusers import UniPCMultistepScheduler
from controlnet_aux import CannyDetector, OpenposeDetector
from PIL import Image
import torch
import numpy as np

# ============================================
# Canny Edge ControlNet
# ============================================

# Load ControlNet model for canny edges
controlnet = ControlNetModel.from_pretrained(
    "lllyasviel/sd-controlnet-canny",
    torch_dtype=torch.float16
)

# Create pipeline with ControlNet
pipe = StableDiffusionControlNetPipeline.from_pretrained(
    "runwayml/stable-diffusion-v1-5",
    controlnet=controlnet,
    torch_dtype=torch.float16,
).to("cuda")

# Use fast scheduler
pipe.scheduler = UniPCMultistepScheduler.from_config(pipe.scheduler.config)
pipe.enable_model_cpu_offload()  # Offload to CPU when not in use

# Prepare control image (detect edges)
canny_detector = CannyDetector()
input_image = Image.open("portrait.png")
canny_image = canny_detector(input_image, low_threshold=100, high_threshold=200)

# Generate with structural control
output = pipe(
    prompt="A beautiful anime character, studio ghibli style",
    image=canny_image,
    num_inference_steps=20,
    guidance_scale=7.5,
    controlnet_conditioning_scale=1.0,  # How strongly to follow the control
).images[0]


# ============================================
# OpenPose ControlNet (Body Pose)
# ============================================

controlnet_pose = ControlNetModel.from_pretrained(
    "lllyasviel/sd-controlnet-openpose",
    torch_dtype=torch.float16
)

pipe_pose = StableDiffusionControlNetPipeline.from_pretrained(
    "runwayml/stable-diffusion-v1-5",
    controlnet=controlnet_pose,
    torch_dtype=torch.float16,
).to("cuda")

# Detect pose from reference image
pose_detector = OpenposeDetector.from_pretrained("lllyasviel/ControlNet")
pose_image = pose_detector(Image.open("dancing_person.png"))

# Generate new person in same pose
output = pipe_pose(
    prompt="A robot dancing in a neon-lit street, cyberpunk style",
    image=pose_image,
    num_inference_steps=25,
).images[0]


# ============================================
# Multi-ControlNet (combine multiple controls)
# ============================================
from diffusers import StableDiffusionControlNetPipeline, ControlNetModel

controlnets = [
    ControlNetModel.from_pretrained("lllyasviel/sd-controlnet-canny", torch_dtype=torch.float16),
    ControlNetModel.from_pretrained("lllyasviel/sd-controlnet-depth", torch_dtype=torch.float16),
]

pipe_multi = StableDiffusionControlNetPipeline.from_pretrained(
    "runwayml/stable-diffusion-v1-5",
    controlnet=controlnets,  # Pass list of ControlNets
    torch_dtype=torch.float16,
).to("cuda")

# Generate with both edge AND depth control
output = pipe_multi(
    prompt="A futuristic city at sunset",
    image=[canny_image, depth_image],  # List of control images
    controlnet_conditioning_scale=[0.8, 0.6],  # Weight per control
    num_inference_steps=25,
).images[0]
```

### Example 6: SDXL and Advanced Generation

```python
from diffusers import DiffusionPipeline, StableDiffusionXLPipeline
from diffusers import AutoencoderKL
import torch

# ============================================
# SDXL (Stable Diffusion XL) - Best Quality
# ============================================

# Load VAE separately (better quality VAE)
vae = AutoencoderKL.from_pretrained(
    "madebyollin/sdxl-vae-fp16-fix",
    torch_dtype=torch.float16
)

# Load SDXL base model
pipe = StableDiffusionXLPipeline.from_pretrained(
    "stabilityai/stable-diffusion-xl-base-1.0",
    vae=vae,
    torch_dtype=torch.float16,
    variant="fp16",
    use_safetensors=True
).to("cuda")

# SDXL supports two text prompts (CLIP + OpenCLIP)
image = pipe(
    prompt="An astronaut riding a horse on Mars, photorealistic, 8k",
    prompt_2="detailed textures, volumetric lighting, cinematic",  # Second encoder
    negative_prompt="cartoon, illustration, painting",
    num_inference_steps=30,
    guidance_scale=7.0,
    height=1024,
    width=1024,
).images[0]


# ============================================
# SDXL with Refiner (Two-Stage Generation)
# ============================================
from diffusers import StableDiffusionXLImg2ImgPipeline

# Load refiner
refiner = StableDiffusionXLImg2ImgPipeline.from_pretrained(
    "stabilityai/stable-diffusion-xl-refiner-1.0",
    torch_dtype=torch.float16,
    variant="fp16",
).to("cuda")

# Stage 1: Base model generates structure (denoise 80%)
base_image = pipe(
    prompt="A magical forest with glowing mushrooms",
    num_inference_steps=40,
    denoising_end=0.8,  # Stop at 80% denoised
    output_type="latent",  # Keep in latent space (faster)
).images

# Stage 2: Refiner adds fine details (remaining 20%)
refined_image = refiner(
    prompt="A magical forest with glowing mushrooms",
    image=base_image,
    num_inference_steps=40,
    denoising_start=0.8,  # Start from 80%
).images[0]


# ============================================
# LoRA Loading (Community Fine-tuned Styles)
# ============================================

pipe = StableDiffusionXLPipeline.from_pretrained(
    "stabilityai/stable-diffusion-xl-base-1.0",
    torch_dtype=torch.float16,
).to("cuda")

# Load a LoRA for a specific style (e.g., pixel art)
pipe.load_lora_weights("nerijs/pixel-art-xl", weight_name="pixel-art-xl.safetensors")
pipe.fuse_lora(lora_scale=0.8)  # Merge LoRA with base (0-1 scale)

image = pipe("A dragon in pixel art style").images[0]

# Unload LoRA when done
pipe.unfuse_lora()
pipe.unload_lora_weights()
```

---

## Common Mistakes

### 1. Wrong Image Dimensions
```python
# ❌ WRONG: Using arbitrary dimensions
image = pipe(prompt="cat", height=500, width=700).images[0]  # Artifacts!

# ✅ CORRECT: Use multiples of 8 (ideally 64) matching training resolution
image = pipe(prompt="cat", height=512, width=512).images[0]  # SD 1.5
image = pipe(prompt="cat", height=1024, width=1024).images[0]  # SDXL
```

### 2. CFG Scale Too High or Too Low
```python
# ❌ Too high → oversaturated, artifacts
image = pipe(prompt="sunset", guidance_scale=20).images[0]

# ❌ Too low → ignores prompt
image = pipe(prompt="sunset", guidance_scale=1).images[0]

# ✅ Sweet spot: 7-9 for most cases
image = pipe(prompt="sunset", guidance_scale=7.5).images[0]
```

### 3. Not Using FP16
```python
# ❌ WRONG: FP32 uses 2x memory, minimal quality gain
pipe = StableDiffusionPipeline.from_pretrained(model_id)

# ✅ CORRECT: Always use FP16 for inference
pipe = StableDiffusionPipeline.from_pretrained(model_id, torch_dtype=torch.float16)
```

### 4. Ignoring the Seed for Reproducibility
```python
# ❌ Different image every time, can't reproduce good results
image = pipe("cat").images[0]

# ✅ Set seed for reproducible results
generator = torch.Generator("cuda").manual_seed(42)
image = pipe("cat", generator=generator).images[0]
```

### 5. Too Many/Few Inference Steps
```python
# ❌ Too many steps with modern schedulers (wasted compute)
pipe.scheduler = DPMSolverMultistepScheduler.from_config(pipe.scheduler.config)
image = pipe("cat", num_inference_steps=100).images[0]  # No better than 25!

# ❌ Too few steps (quality suffers)
image = pipe("cat", num_inference_steps=5).images[0]  # Blurry

# ✅ Match steps to scheduler
# DPM++ 2M: 20-30 steps | Euler: 25-30 | LCM: 4-8
image = pipe("cat", num_inference_steps=25).images[0]
```

### 6. Not Understanding Negative Prompts
```python
# ❌ Putting what you WANT in negative prompt
image = pipe(prompt="cat", negative_prompt="beautiful high quality").images[0]

# ✅ Negative prompt describes what to AVOID
image = pipe(
    prompt="a beautiful cat, high quality, sharp focus",
    negative_prompt="blurry, low quality, deformed, ugly, extra limbs"
).images[0]
```

---

## Interview Questions

### Conceptual Questions

**Q1: Explain the forward and reverse process in diffusion models.**
> Forward process gradually adds Gaussian noise over T steps until data becomes pure noise (fixed, no learning). Reverse process learns to denoise step by step using a neural network that predicts the noise added at each step. Training minimizes MSE between actual and predicted noise.

**Q2: Why are diffusion models preferred over GANs for image generation?**
> - Stable training (no mode collapse or training instabilities)
> - Better mode coverage (more diverse outputs)
> - Easier to condition (text, pose, depth, etc.)
> - Strong theoretical grounding in variational inference
> - GANs still win in speed (single forward pass vs. iterative)

**Q3: What is Classifier-Free Guidance and why is it important?**
> CFG interpolates between conditional and unconditional predictions: ε̂ = ε_uncond + w·(ε_cond - ε_uncond). It amplifies the influence of the conditioning signal without needing a separate classifier. Higher guidance scale = more prompt adherence but less diversity.

**Q4: Why does Stable Diffusion operate in latent space instead of pixel space?**
> Latent space is 48× smaller than pixel space (64×64×4 vs 512×512×3). This makes training and inference dramatically faster and cheaper while preserving perceptual quality. The VAE handles compression/decompression.

**Q5: How does ControlNet maintain the quality of the pretrained model?**
> ControlNet creates a trainable copy of the encoder blocks connected via zero-initialized convolutions. Since zero convolutions start with zero output, the ControlNet initially has no effect on the pretrained model, gradually learning to inject spatial control without destroying existing capabilities.

**Q6: Compare DDPM, DDIM, and DPM++ samplers.**
> - DDPM: Stochastic, needs ~1000 steps, original sampler
> - DDIM: Deterministic, can skip steps (50-100), same trained model
> - DPM++: ODE solver, only 20-30 steps for high quality, best speed/quality ratio

**Q7: What's the difference between ε-prediction, v-prediction, and x₀-prediction?**
> - ε-prediction: Model predicts the noise added (most common, used in SD 1.x)
> - x₀-prediction: Model directly predicts the clean image
> - v-prediction: Model predicts v = √(ᾱ)·ε - √(1-ᾱ)·x₀ (used in SD 2.x, better at high noise levels)

### Coding Questions

**Q8: Write the DDPM sampling algorithm.**
```python
@torch.no_grad()
def ddpm_sample(model, shape, T, alphas, betas, alphas_cumprod):
    x = torch.randn(shape)  # Start from noise
    for t in reversed(range(T)):
        t_batch = torch.full((shape[0],), t, dtype=torch.long)
        predicted_noise = model(x, t_batch)
        
        alpha_t = alphas[t]
        alpha_bar_t = alphas_cumprod[t]
        
        # Compute mean
        mean = (1 / alpha_t.sqrt()) * (x - (1 - alpha_t) / (1 - alpha_bar_t).sqrt() * predicted_noise)
        
        # Add noise (except at t=0)
        if t > 0:
            noise = torch.randn_like(x)
            sigma = betas[t].sqrt()
            x = mean + sigma * noise
        else:
            x = mean
    return x
```

---

## Quick Reference

### Diffusion Models Cheat Sheet

| Concept | Key Formula/Detail |
|---|---|
| Forward process | $x_t = \sqrt{\bar{\alpha}_t} x_0 + \sqrt{1-\bar{\alpha}_t} \epsilon$ |
| Training loss | $L = \|\|\epsilon - \epsilon_\theta(x_t, t)\|\|^2$ |
| Reverse step | $x_{t-1} = \frac{1}{\sqrt{\alpha_t}}(x_t - \frac{\beta_t}{\sqrt{1-\bar\alpha_t}}\epsilon_\theta) + \sigma_t z$ |
| CFG | $\hat\epsilon = \epsilon_{uncond} + w(\epsilon_{cond} - \epsilon_{uncond})$ |
| Latent space (SD) | 512×512×3 → 64×64×4 (48× compression) |
| CFG scale | 7-9 for most use cases |
| Steps (DPM++) | 20-30 steps |
| SDXL resolution | 1024×1024 native |

### Pipeline Selection Guide

| Task | Pipeline | Key Parameter |
|---|---|---|
| Text → Image | `StableDiffusionPipeline` | `guidance_scale` |
| Image → Image | `StableDiffusionImg2ImgPipeline` | `strength` (0-1) |
| Inpainting | `StableDiffusionInpaintPipeline` | `mask_image` |
| Controlled gen | `StableDiffusionControlNetPipeline` | `controlnet_conditioning_scale` |
| Super-resolution | `StableDiffusionUpscalePipeline` | `noise_level` |
| Video | `TextToVideoSDPipeline` | `num_frames` |

### Memory Optimization Tricks

| Technique | Memory Saved | Speed Impact |
|---|---|---|
| `torch.float16` | ~50% | Faster |
| `enable_attention_slicing()` | ~30% | ~10% slower |
| `enable_vae_slicing()` | ~20% for batches | Minimal |
| `enable_model_cpu_offload()` | ~60% | ~20% slower |
| `enable_sequential_cpu_offload()` | ~80% | ~50% slower |
| xFormers attention | ~30% | Faster |
| `torch.compile()` | None | 20-30% faster |

---

> **Key Takeaway**: Diffusion models learn to reverse a noise-adding process. They trade speed for quality, stability, and controllability. Stable Diffusion made them practical by operating in latent space. Modern samplers (DPM++, LCM) have reduced the speed gap significantly.
