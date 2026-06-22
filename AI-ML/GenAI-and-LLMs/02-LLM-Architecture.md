# Chapter 02: LLM Architecture — Transformers Deep Dive

## Table of Contents
- [What is an LLM?](#what-is-an-llm)
- [The Transformer Architecture](#the-transformer-architecture)
- [Self-Attention Mechanism](#self-attention-mechanism)
- [Multi-Head Attention](#multi-head-attention)
- [Positional Encoding](#positional-encoding)
- [Feed-Forward Networks](#feed-forward-networks)
- [Layer Normalization & Residual Connections](#layer-normalization--residual-connections)
- [Major LLM Architectures](#major-llm-architectures)
- [Scaling Laws & Training](#scaling-laws--training)
- [Code Examples](#code-examples)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is an LLM?

### Simple Explanation
A Large Language Model is like a super-powered autocomplete. It's a neural network with billions of parameters that has read virtually the entire internet and can predict what text should come next — so well that it appears to "understand" and "reason." The "large" refers to both the model size (billions of parameters) and training data (trillions of tokens).

### Why It Matters
- LLMs are the engine behind ChatGPT, Claude, Gemini, and Copilot
- Understanding architecture helps you choose the right model for your use case
- Knowing internals lets you fine-tune, optimize, and debug effectively
- Interview favorite: "Explain how Transformers work"

### The Key Insight
Before Transformers, language models used RNNs/LSTMs which processed text **sequentially** (one word at a time). Transformers process all words **simultaneously** through attention, making them:
- Faster to train (parallelizable)
- Better at capturing long-range dependencies
- Scalable to massive sizes

---

## The Transformer Architecture

### Original Architecture ("Attention Is All You Need", 2017)

```
                    ┌─────────────────────┐
                    │    OUTPUT TOKENS     │
                    └──────────┬──────────┘
                               │
                    ┌──────────┴──────────┐
                    │   Linear + Softmax   │
                    └──────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              │         DECODER (Nx)            │
              │  ┌────────────────────────┐     │
              │  │ Feed-Forward Network    │     │
              │  ├────────────────────────┤     │
              │  │ Cross-Attention         │◄────┼──── From Encoder
              │  ├────────────────────────┤     │
              │  │ Masked Self-Attention   │     │
              │  └────────────────────────┘     │
              └────────────────┬────────────────┘
                               │
              ┌────────────────┴────────────────┐
              │         ENCODER (Nx)            │
              │  ┌────────────────────────┐     │
              │  │ Feed-Forward Network    │     │
              │  ├────────────────────────┤     │
              │  │ Self-Attention          │     │
              │  └────────────────────────┘     │
              └────────────────┬────────────────┘
                               │
                    ┌──────────┴──────────┐
                    │ Input Embeddings +   │
                    │ Positional Encoding  │
                    └──────────┬──────────┘
                               │
                    ┌──────────┴──────────┐
                    │    INPUT TOKENS      │
                    └─────────────────────┘
```

### Three Families of Transformer Models

| Architecture | Type | Examples | Best For |
|-------------|------|----------|----------|
| Encoder-Only | Bidirectional | BERT, RoBERTa, DeBERTa | Classification, NER, similarity |
| Decoder-Only | Autoregressive | GPT, LLaMA, Mistral | Text generation, chat, code |
| Encoder-Decoder | Seq2Seq | T5, BART, Flan-T5 | Translation, summarization |

```
ENCODER-ONLY (BERT):
  "The [MASK] sat on the mat"  →  Predict: "cat"
  (Sees all tokens simultaneously, bidirectional)

DECODER-ONLY (GPT):
  "The cat sat on the" → Predict: "mat"
  (Only sees previous tokens, left-to-right)

ENCODER-DECODER (T5):
  Encoder: "Translate English to French: Hello"
  Decoder: "Bonjour"
  (Encoder reads input, decoder generates output)
```

> **Modern Trend**: Decoder-only models dominate (GPT-4, Claude, LLaMA). Simpler architecture + scaling = better performance for generation tasks.

---

## Self-Attention Mechanism

### Why Attention?

Consider: "The animal didn't cross the street because **it** was too tired."

What does "it" refer to? The animal! To understand this, the model needs to connect "it" back to "animal" across several tokens. Attention provides this mechanism.

### The QKV Framework

Self-attention uses three projections of the same input:
- **Q (Query)**: "What am I looking for?"
- **K (Key)**: "What do I contain?"
- **V (Value)**: "What information do I provide?"

**Analogy**: Think of it like a library search:
- Q = Your search query ("books about Python")
- K = Index cards describing each book (metadata)
- V = The actual book content

You match your Query against all Keys to find relevant books, then retrieve their Values.

### Mathematical Formulation

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

Step by step:

1. **Compute attention scores**: $QK^T$ — how relevant is each position?
2. **Scale**: Divide by $\sqrt{d_k}$ — prevents extremely large values that make softmax saturate
3. **Softmax**: Convert to probabilities (sum to 1)
4. **Weighted sum**: Multiply by V to get output

### Worked Example

```
Input: "The cat sat"
(3 tokens, embedding dimension = 4 for simplicity)

Token embeddings:
  "The" = [1, 0, 1, 0]
  "cat" = [0, 1, 0, 1]  
  "sat" = [1, 1, 0, 0]

Step 1: Project to Q, K, V using learned weight matrices
  (In practice, Wq, Wk, Wv are learned parameters)
  
  Q = X · Wq    K = X · Wk    V = X · Wv

Step 2: Compute attention scores = Q · K^T
  
  Attention Matrix (3x3):
         "The"  "cat"  "sat"
  "The" [ 0.8    0.1    0.5 ]  ← "The" attends mostly to itself
  "cat" [ 0.2    0.7    0.3 ]  ← "cat" attends mostly to itself
  "sat" [ 0.4    0.5    0.6 ]  ← "sat" attends to all somewhat evenly

Step 3: Scale by √d_k and apply softmax (each row sums to 1)

Step 4: Multiply by V to get context-aware representations
  Output for "cat" = 0.2·V_The + 0.7·V_cat + 0.3·V_sat
  (Weighted combination of all positions!)
```

### Why Scale by √d_k?

Without scaling, when $d_k$ is large, dot products grow large → softmax outputs become very peaked (close to one-hot) → gradients vanish.

$$\text{If } q, k \sim \mathcal{N}(0, 1), \text{ then } q \cdot k \sim \mathcal{N}(0, d_k)$$

Dividing by $\sqrt{d_k}$ normalizes variance back to 1.

### Causal (Masked) Attention for Decoder Models

In GPT-style models, each token can only attend to **previous** tokens (can't peek at the future):

```
Attention Mask (for "The cat sat"):

         "The"  "cat"  "sat"
  "The" [  1      0      0  ]   ← Can only see "The"
  "cat" [  1      1      0  ]   ← Can see "The", "cat"
  "sat" [  1      1      1  ]   ← Can see all three

(0 = masked out, set to -infinity before softmax)
```

---

## Multi-Head Attention

### Why Multiple Heads?

Single attention can only learn one type of relationship at a time. Multiple heads let the model learn **different types of relationships simultaneously**:

- Head 1 might learn syntactic relationships (subject-verb)
- Head 2 might learn semantic relationships (pronoun resolution)
- Head 3 might learn positional patterns (adjacent words)

### How It Works

```
Input X (shape: seq_len × d_model)
    │
    ├──→ Head 1: Attention(XWq1, XWk1, XWv1) → output1 (seq_len × d_v)
    ├──→ Head 2: Attention(XWq2, XWk2, XWv2) → output2
    ├──→ Head 3: Attention(XWq3, XWk3, XWv3) → output3
    ...
    └──→ Head h: Attention(XWqh, XWkh, XWvh) → outputh
    
    Concatenate all heads → [output1; output2; ...; outputh]
    
    Project back → Concat · Wo → Final output (seq_len × d_model)
```

### Mathematical Formulation

$$\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, ..., \text{head}_h) W^O$$

where $\text{head}_i = \text{Attention}(QW_i^Q, KW_i^K, VW_i^V)$

### Dimensions

For a model with $d_{model} = 768$ and $h = 12$ heads:
- Each head operates on $d_k = d_v = d_{model} / h = 64$ dimensions
- Total compute is the same as single-head with full dimension
- But much more expressive!

| Model | d_model | Heads | d_head | Layers |
|-------|---------|-------|--------|--------|
| GPT-2 Small | 768 | 12 | 64 | 12 |
| GPT-2 Large | 1280 | 20 | 64 | 36 |
| GPT-3 | 12288 | 96 | 128 | 96 |
| LLaMA-2 7B | 4096 | 32 | 128 | 32 |
| LLaMA-2 70B | 8192 | 64 | 128 | 80 |

---

## Positional Encoding

### The Problem
Self-attention is **permutation invariant** — it doesn't inherently know word order! "Cat sat on mat" and "Mat sat on cat" would produce the same attention scores without positional information.

### Solution: Add Position Information

#### 1. Sinusoidal Positional Encoding (Original Transformer)

$$PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right)$$
$$PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)$$

**Why sinusoidal?**
- Each position gets a unique encoding
- Model can learn relative positions (PE_{pos+k} is a linear function of PE_pos)
- Generalizes to unseen sequence lengths

#### 2. Learned Positional Embeddings (GPT-2)
- Just learn a position embedding for each position (0, 1, 2, ..., max_seq_len)
- Simple but can't generalize beyond training length

#### 3. Rotary Position Embeddings (RoPE) — Modern Standard

Used by LLaMA, Mistral, GPT-NeoX, and most modern models.

**Key idea**: Encode position by *rotating* the query and key vectors:

$$f(x, m) = x \cdot e^{im\theta}$$

**Advantages**:
- Relative position information in attention scores
- Better extrapolation to longer sequences
- No separate positional embedding needed

#### 4. ALiBi (Attention with Linear Biases)
- Add a linear bias to attention scores based on distance
- Very simple: $\text{score}_{ij} = q_i \cdot k_j - m \cdot |i - j|$
- Where $m$ is a head-specific slope

### Comparison

| Method | Extrapolation | Compute | Used In |
|--------|--------------|---------|---------|
| Sinusoidal | Moderate | Minimal | Original Transformer |
| Learned | Poor | Minimal | GPT-2 |
| RoPE | Good | Minimal | LLaMA, Mistral, GPT-NeoX |
| ALiBi | Excellent | Minimal | MPT, BLOOM |

---

## Feed-Forward Networks

### What They Do
After attention aggregates information from all positions, the FFN processes each position **independently** — applying non-linear transformations to the aggregated representations.

### Standard FFN

$$\text{FFN}(x) = \text{GELU}(xW_1 + b_1)W_2 + b_2$$

- $W_1$: Projects from $d_{model}$ to $d_{ff}$ (typically $4 \times d_{model}$)
- GELU activation: Non-linearity
- $W_2$: Projects back from $d_{ff}$ to $d_{model}$

```
Input (d_model=768) → Expand (d_ff=3072) → GELU → Contract (d_model=768) → Output
```

### SwiGLU (Modern Standard — LLaMA, Mistral, GPT-4)

$$\text{SwiGLU}(x) = (\text{Swish}(xW_1) \odot xW_3)W_2$$

- Uses a gating mechanism ($\odot$ is element-wise multiplication)
- Empirically better than standard FFN
- $d_{ff} = \frac{8}{3} d_{model}$ (adjusted for extra parameters)

> **Key Insight**: FFN layers are where most of the model's "knowledge" is stored. Attention routes information; FFN processes it.

---

## Layer Normalization & Residual Connections

### Residual Connections (Skip Connections)

$$\text{output} = \text{LayerNorm}(x + \text{Sublayer}(x))$$

**Why?**
- Allows gradient flow in deep networks (solves vanishing gradient)
- Each layer only needs to learn the "residual" (what to add)
- Enables training of 100+ layer models

```
Input x ─────────────────────┐
    │                         │  (skip connection)
    ▼                         │
┌─────────┐                   │
│ Sublayer │  (Attention       │
│ (Attn/   │   or FFN)        │
│  FFN)    │                   │
└────┬────┘                   │
     │                         │
     ▼                         ▼
   ┌─┴─────────────────────────┐
   │         ADD                │
   └────────────┬───────────────┘
                │
                ▼
         ┌──────────┐
         │LayerNorm │
         └────┬─────┘
              │
              ▼
           Output
```

### Pre-Norm vs Post-Norm

**Post-Norm** (Original Transformer):
$$\text{output} = \text{LayerNorm}(x + \text{Sublayer}(x))$$

**Pre-Norm** (Modern, used by GPT-2+, LLaMA):
$$\text{output} = x + \text{Sublayer}(\text{LayerNorm}(x))$$

Pre-Norm is more stable to train (especially for deep models) and is now the standard.

### RMSNorm (LLaMA, Mistral)

Simplification of LayerNorm — only normalizes by RMS, no mean subtraction:

$$\text{RMSNorm}(x) = \frac{x}{\sqrt{\frac{1}{n}\sum x_i^2}} \cdot \gamma$$

- Faster than LayerNorm (saves compute)
- Empirically works just as well
- Used in all modern LLMs

---

## Major LLM Architectures

### GPT Family (OpenAI)

```
GPT Architecture:
┌─────────────────────────────────┐
│  Token Embedding + Positional    │
└──────────────┬──────────────────┘
               │
               ▼ (× N layers)
┌─────────────────────────────────┐
│  Masked Multi-Head Attention     │
│  + Residual + LayerNorm          │
├─────────────────────────────────┤
│  Feed-Forward Network            │
│  + Residual + LayerNorm          │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  Linear → Vocab Logits → Softmax│
└─────────────────────────────────┘
```

| Version | Params | Context | Key Innovation |
|---------|--------|---------|---------------|
| GPT-1 | 117M | 512 | Pre-training + fine-tuning paradigm |
| GPT-2 | 1.5B | 1024 | Zero-shot transfer |
| GPT-3 | 175B | 4096 | In-context learning, few-shot |
| GPT-4 | ~1.8T (MoE) | 128K | Multi-modal, reasoning |
| GPT-4o | Unknown | 128K | Unified multi-modal |

### BERT (Google)

**Bidirectional Encoder Representations from Transformers**

Training objectives:
1. **Masked Language Model (MLM)**: Mask 15% of tokens, predict them
2. **Next Sentence Prediction (NSP)**: Is sentence B the actual next sentence?

```
Input:  "The [MASK] sat on the [MASK]"
Output: "The  cat   sat on the  mat"
         (predicts masked tokens using FULL context)
```

### LLaMA Family (Meta)

Key innovations:
- **Pre-normalization** (RMSNorm)
- **RoPE** positional encoding
- **SwiGLU** activation
- **Grouped-Query Attention (GQA)** — reduced KV heads for efficiency

```
LLaMA Architecture Specifics:
- RMSNorm (instead of LayerNorm)
- SwiGLU FFN (instead of ReLU/GELU)
- RoPE (instead of learned positional)
- GQA with fewer KV heads (LLaMA-2 70B: 8 KV heads, 64 Q heads)
```

| Version | Sizes | Context | Training Data |
|---------|-------|---------|---------------|
| LLaMA-1 | 7B, 13B, 33B, 65B | 2048 | 1.4T tokens |
| LLaMA-2 | 7B, 13B, 70B | 4096 | 2T tokens |
| LLaMA-3 | 8B, 70B, 405B | 128K | 15T+ tokens |

### Mistral & Mixtral

**Mistral 7B** innovations:
- **Sliding Window Attention**: Each token attends to W previous tokens
- **GQA**: Grouped-Query Attention for efficiency
- Outperforms LLaMA-2 13B despite being smaller

**Mixtral 8x7B** (Mixture of Experts):
- 8 expert FFN networks, router selects 2 per token
- Total params: 46.7B, Active params: 12.9B (per token)
- Same quality as much larger dense models, cheaper inference

```
Mixture of Experts (MoE):
                    
Input → [Router] → selects 2 of 8 experts
              │
              ├─→ Expert 1 (selected) ──┐
              ├─→ Expert 2              │
              ├─→ Expert 3 (selected) ──┤── Weighted sum → Output
              ├─→ Expert 4              │
              ├─→ Expert 5              │
              ├─→ Expert 6              │
              ├─→ Expert 7              │
              └─→ Expert 8              │
                                        ▼
                                   Final Output
```

### Grouped-Query Attention (GQA)

```
Multi-Head Attention (MHA):       Grouped-Query Attention (GQA):
  Q: 32 heads                       Q: 32 heads
  K: 32 heads                       K: 8 heads (shared across Q groups)
  V: 32 heads                       V: 8 heads
  
  KV cache: 32 × 2 × d_head        KV cache: 8 × 2 × d_head
  (Expensive for long sequences!)   (4x less memory!)
```

**Multi-Query Attention (MQA)**: Even more extreme — 1 K head, 1 V head shared across all Q heads.

---

## Scaling Laws & Training

### Compute-Optimal Training (Chinchilla)

For a fixed compute budget $C$:
- Model size $N \propto C^{0.5}$
- Training tokens $D \propto C^{0.5}$
- Optimal ratio: ~20 tokens per parameter

| Model | Params | Training Tokens | Tokens/Param | Status |
|-------|--------|----------------|--------------|--------|
| GPT-3 | 175B | 300B | 1.7 | Under-trained |
| Chinchilla | 70B | 1.4T | 20 | Optimal |
| LLaMA-1 | 65B | 1.4T | 21.5 | Optimal |
| LLaMA-2 | 70B | 2T | 28.6 | Over-trained (intentional) |
| LLaMA-3 | 8B | 15T | 1875 | Heavily over-trained |

> **Modern insight**: For *inference-time efficiency*, over-training smaller models is preferred. You pay once to train, but serve millions of times.

### Training Infrastructure

```
Typical LLM Training Setup:
┌──────────────────────────────────────────────┐
│  Cluster: 1000-10000 GPUs (A100/H100)         │
│                                                │
│  Data Parallelism: Split batches across GPUs   │
│  Tensor Parallelism: Split layers across GPUs  │
│  Pipeline Parallelism: Split model stages      │
│  ZeRO Optimization: Distribute optimizer state │
│                                                │
│  Training Time: Weeks to months                │
│  Cost: $10M - $100M+ (GPT-4 level)            │
│  Power: Megawatts of electricity               │
└──────────────────────────────────────────────┘
```

### Training Stability Techniques

1. **Gradient Clipping**: Prevent exploding gradients
2. **Learning Rate Warmup**: Gradually increase LR at start
3. **Cosine LR Schedule**: Decay learning rate over training
4. **BF16/FP16 Training**: Mixed precision for efficiency
5. **Gradient Checkpointing**: Trade compute for memory

---

## Code Examples

### Example 1: Self-Attention from Scratch

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class SelfAttention(nn.Module):
    """Single-head self-attention mechanism from scratch."""
    
    def __init__(self, d_model: int, d_k: int):
        super().__init__()
        self.d_k = d_k
        
        # Learned projection matrices
        self.W_q = nn.Linear(d_model, d_k, bias=False)  # Query projection
        self.W_k = nn.Linear(d_model, d_k, bias=False)  # Key projection
        self.W_v = nn.Linear(d_model, d_k, bias=False)  # Value projection
    
    def forward(self, x, mask=None):
        """
        x: (batch_size, seq_len, d_model)
        mask: (seq_len, seq_len) - causal mask for decoder
        """
        # Project input to Q, K, V
        Q = self.W_q(x)  # (batch, seq_len, d_k)
        K = self.W_k(x)  # (batch, seq_len, d_k)
        V = self.W_v(x)  # (batch, seq_len, d_k)
        
        # Compute attention scores: Q · K^T / sqrt(d_k)
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        # scores shape: (batch, seq_len, seq_len)
        
        # Apply causal mask (for autoregressive models)
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        # Softmax to get attention weights (each row sums to 1)
        attention_weights = F.softmax(scores, dim=-1)
        
        # Weighted sum of values
        output = torch.matmul(attention_weights, V)  # (batch, seq_len, d_k)
        
        return output, attention_weights


# Demo
batch_size, seq_len, d_model, d_k = 1, 4, 8, 4
x = torch.randn(batch_size, seq_len, d_model)

# Create causal mask (lower triangular = can only attend to past)
mask = torch.tril(torch.ones(seq_len, seq_len))
print("Causal Mask:")
print(mask)

attn = SelfAttention(d_model, d_k)
output, weights = attn(x, mask)

print(f"\nInput shape: {x.shape}")
print(f"Output shape: {output.shape}")
print(f"\nAttention weights (how much each position attends to others):")
print(weights[0].detach().numpy().round(3))
```

### Example 2: Multi-Head Attention

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class MultiHeadAttention(nn.Module):
    """Multi-head attention as used in Transformers."""
    
    def __init__(self, d_model: int, num_heads: int):
        super().__init__()
        assert d_model % num_heads == 0, "d_model must be divisible by num_heads"
        
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads  # Dimension per head
        
        # Combined QKV projection (more efficient than separate)
        self.W_qkv = nn.Linear(d_model, 3 * d_model, bias=False)
        # Output projection
        self.W_o = nn.Linear(d_model, d_model, bias=False)
    
    def forward(self, x, mask=None):
        batch_size, seq_len, _ = x.shape
        
        # Project to Q, K, V simultaneously
        qkv = self.W_qkv(x)  # (batch, seq_len, 3 * d_model)
        
        # Split into Q, K, V
        qkv = qkv.reshape(batch_size, seq_len, 3, self.num_heads, self.d_k)
        qkv = qkv.permute(2, 0, 3, 1, 4)  # (3, batch, heads, seq_len, d_k)
        Q, K, V = qkv[0], qkv[1], qkv[2]
        
        # Scaled dot-product attention (per head)
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        attention_weights = F.softmax(scores, dim=-1)
        
        # Apply attention to values
        context = torch.matmul(attention_weights, V)
        # context: (batch, heads, seq_len, d_k)
        
        # Concatenate heads
        context = context.transpose(1, 2).reshape(batch_size, seq_len, self.d_model)
        
        # Final linear projection
        output = self.W_o(context)
        
        return output, attention_weights


# Demo
d_model, num_heads = 512, 8
mha = MultiHeadAttention(d_model, num_heads)

x = torch.randn(2, 10, d_model)  # batch=2, seq_len=10
mask = torch.tril(torch.ones(10, 10)).unsqueeze(0).unsqueeze(0)  # Causal mask

output, attn_weights = mha(x, mask)
print(f"Input: {x.shape}")
print(f"Output: {output.shape}")
print(f"Attention weights: {attn_weights.shape}")  # (batch, heads, seq, seq)
print(f"Each head learns different patterns!")
```

### Example 3: Complete Transformer Block

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class RMSNorm(nn.Module):
    """Root Mean Square Layer Normalization (used in LLaMA)."""
    
    def __init__(self, d_model: int, eps: float = 1e-6):
        super().__init__()
        self.eps = eps
        self.weight = nn.Parameter(torch.ones(d_model))
    
    def forward(self, x):
        # RMS normalization (no mean subtraction, unlike LayerNorm)
        rms = torch.sqrt(torch.mean(x ** 2, dim=-1, keepdim=True) + self.eps)
        return x / rms * self.weight


class SwiGLU(nn.Module):
    """SwiGLU Feed-Forward Network (used in LLaMA, Mistral)."""
    
    def __init__(self, d_model: int, d_ff: int = None):
        super().__init__()
        # Default FFN dimension: 8/3 * d_model (for SwiGLU)
        if d_ff is None:
            d_ff = int(8 / 3 * d_model)
            # Round to nearest multiple of 64 for efficiency
            d_ff = ((d_ff + 63) // 64) * 64
        
        self.w1 = nn.Linear(d_model, d_ff, bias=False)  # Gate projection
        self.w2 = nn.Linear(d_ff, d_model, bias=False)  # Down projection
        self.w3 = nn.Linear(d_model, d_ff, bias=False)  # Up projection
    
    def forward(self, x):
        # SwiGLU: (Swish(xW1) ⊙ xW3) W2
        return self.w2(F.silu(self.w1(x)) * self.w3(x))


class TransformerBlock(nn.Module):
    """A single Transformer decoder block (LLaMA-style)."""
    
    def __init__(self, d_model: int, num_heads: int, d_ff: int = None):
        super().__init__()
        self.attention = MultiHeadAttention(d_model, num_heads)
        self.ffn = SwiGLU(d_model, d_ff)
        self.norm1 = RMSNorm(d_model)  # Pre-attention norm
        self.norm2 = RMSNorm(d_model)  # Pre-FFN norm
    
    def forward(self, x, mask=None):
        # Pre-norm + Attention + Residual
        normed = self.norm1(x)
        attn_output, _ = self.attention(normed, mask)
        x = x + attn_output  # Residual connection
        
        # Pre-norm + FFN + Residual
        normed = self.norm2(x)
        ffn_output = self.ffn(normed)
        x = x + ffn_output  # Residual connection
        
        return x


# Build a mini GPT-style model
class MiniGPT(nn.Module):
    """Minimal GPT-like model for educational purposes."""
    
    def __init__(self, vocab_size, d_model, num_heads, num_layers, max_seq_len):
        super().__init__()
        self.token_embedding = nn.Embedding(vocab_size, d_model)
        self.position_embedding = nn.Embedding(max_seq_len, d_model)
        
        self.layers = nn.ModuleList([
            TransformerBlock(d_model, num_heads)
            for _ in range(num_layers)
        ])
        
        self.final_norm = RMSNorm(d_model)
        self.output_proj = nn.Linear(d_model, vocab_size, bias=False)
    
    def forward(self, token_ids):
        batch, seq_len = token_ids.shape
        
        # Get embeddings
        tok_emb = self.token_embedding(token_ids)
        pos_ids = torch.arange(seq_len, device=token_ids.device)
        pos_emb = self.position_embedding(pos_ids)
        x = tok_emb + pos_emb
        
        # Causal mask
        mask = torch.tril(torch.ones(seq_len, seq_len, device=x.device))
        mask = mask.unsqueeze(0).unsqueeze(0)
        
        # Pass through transformer layers
        for layer in self.layers:
            x = layer(x, mask)
        
        # Final norm and project to vocabulary
        x = self.final_norm(x)
        logits = self.output_proj(x)  # (batch, seq_len, vocab_size)
        
        return logits


# Instantiate and test
model = MiniGPT(
    vocab_size=32000,    # Typical LLaMA vocab size
    d_model=256,         # Small for demo
    num_heads=8,
    num_layers=4,
    max_seq_len=512
)

# Count parameters
total_params = sum(p.numel() for p in model.parameters())
print(f"Model parameters: {total_params:,}")  # ~8.5M for this mini model

# Forward pass
input_ids = torch.randint(0, 32000, (2, 64))  # batch=2, seq_len=64
logits = model(input_ids)
print(f"Input shape: {input_ids.shape}")
print(f"Output logits shape: {logits.shape}")  # (2, 64, 32000)

# Next token prediction: take logits at last position
next_token_logits = logits[:, -1, :]  # (2, 32000)
next_token = torch.argmax(next_token_logits, dim=-1)
print(f"Predicted next token IDs: {next_token}")
```

### Example 4: KV Cache for Fast Inference

```python
import torch
import torch.nn as nn
import time

class CachedAttention(nn.Module):
    """Attention with KV cache for efficient autoregressive generation."""
    
    def __init__(self, d_model, num_heads):
        super().__init__()
        self.num_heads = num_heads
        self.d_k = d_model // num_heads
        self.W_q = nn.Linear(d_model, d_model, bias=False)
        self.W_k = nn.Linear(d_model, d_model, bias=False)
        self.W_v = nn.Linear(d_model, d_model, bias=False)
        self.W_o = nn.Linear(d_model, d_model, bias=False)
    
    def forward(self, x, kv_cache=None):
        """
        During generation, x is just the NEW token (seq_len=1).
        We reuse cached K, V from previous tokens.
        """
        B, seq_len, _ = x.shape
        
        Q = self.W_q(x).view(B, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(x).view(B, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(x).view(B, seq_len, self.num_heads, self.d_k).transpose(1, 2)
        
        # Append to cache
        if kv_cache is not None:
            K = torch.cat([kv_cache[0], K], dim=2)  # Append new K
            V = torch.cat([kv_cache[1], V], dim=2)  # Append new V
        
        new_cache = (K, V)  # Save for next step
        
        # Attention: Q attends to ALL K (including cached)
        scores = torch.matmul(Q, K.transpose(-2, -1)) / (self.d_k ** 0.5)
        weights = torch.softmax(scores, dim=-1)
        output = torch.matmul(weights, V)
        
        output = output.transpose(1, 2).reshape(B, seq_len, -1)
        return self.W_o(output), new_cache


# Demonstrate speed benefit of KV caching
print("Without KV cache: Recompute attention for ALL tokens each step")
print("With KV cache: Only compute attention for NEW token, reuse past")
print()
print("Speed comparison (conceptual):")
print("  Generating 100 tokens without cache: O(100²) = 10,000 attention ops")
print("  Generating 100 tokens with cache:    O(100)  = 100 attention ops")
print("  → 100x speedup for this example!")
```

### Example 5: Loading and Using a Real LLM

```python
# Load and run inference with a real LLM using Hugging Face
# Install: pip install transformers torch accelerate

from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

# Load a small but capable model
model_name = "microsoft/phi-2"  # 2.7B params, good quality
# Alternative: "mistralai/Mistral-7B-Instruct-v0.2"

print(f"Loading {model_name}...")
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.float16,     # Half precision (saves ~50% memory)
    device_map="auto",              # Automatically use GPU if available
    trust_remote_code=True
)

# Generate text
prompt = "Explain the attention mechanism in transformers:\n"
inputs = tokenizer(prompt, return_tensors="pt").to(model.device)

# Generation with various parameters
with torch.no_grad():
    outputs = model.generate(
        **inputs,
        max_new_tokens=200,         # Limit output length
        temperature=0.7,            # Moderate creativity
        top_p=0.9,                  # Nucleus sampling
        do_sample=True,             # Enable sampling
        repetition_penalty=1.1,     # Discourage repetition
        pad_token_id=tokenizer.eos_token_id
    )

# Decode and print
generated_text = tokenizer.decode(outputs[0], skip_special_tokens=True)
print(generated_text)

# Check model size
param_count = sum(p.numel() for p in model.parameters())
print(f"\nModel size: {param_count / 1e9:.1f}B parameters")
print(f"Memory usage: ~{param_count * 2 / 1e9:.1f}GB (FP16)")
```

### Example 6: Visualizing Attention Patterns

```python
# Visualize what attention heads learn
# Install: pip install transformers matplotlib numpy

from transformers import AutoTokenizer, AutoModel
import torch
import numpy as np

# Load BERT (good for visualization since it's bidirectional)
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
model = AutoModel.from_pretrained("bert-base-uncased", output_attentions=True)

# Tokenize a sentence
text = "The cat sat on the mat because it was tired"
inputs = tokenizer(text, return_tensors="pt")
tokens = tokenizer.convert_ids_to_tokens(inputs["input_ids"][0])

# Get attention weights
with torch.no_grad():
    outputs = model(**inputs)

# outputs.attentions: tuple of (batch, heads, seq, seq) for each layer
# Let's look at layer 6, head 10 (often captures coreference)
layer_idx, head_idx = 5, 10
attention = outputs.attentions[layer_idx][0, head_idx].numpy()

print(f"Attention pattern for Layer {layer_idx+1}, Head {head_idx+1}:")
print(f"Tokens: {tokens}")
print(f"\nAttention matrix (rows attend to columns):")
print(f"{'':>10}", end="")
for t in tokens:
    print(f"{t:>8}", end="")
print()

for i, t in enumerate(tokens):
    print(f"{t:>10}", end="")
    for j in range(len(tokens)):
        val = attention[i, j]
        # Show as bar chart using characters
        bar = "█" * int(val * 10)
        print(f"{val:>8.3f}", end="")
    print()

# Pro Tip: Use BertViz library for interactive visualization
# pip install bertviz
# from bertviz import head_view
# head_view(attention, tokens)
```

---

## Common Mistakes

### 1. Confusing Parameters with Intelligence
❌ **Wrong**: "Bigger model = always better"
✅ **Right**: A well-trained 8B model (LLaMA-3 8B) can beat a poorly trained 65B model. Quality of training data and technique matter as much as size.

### 2. Ignoring KV Cache Memory
❌ **Wrong**: "I'll just use a 70B model with 128K context on my 24GB GPU"
✅ **Right**: KV cache for 70B model at 128K context ≈ 160GB. You need to account for KV cache memory, not just model weights.

```
KV Cache Memory = 2 × num_layers × num_kv_heads × d_head × seq_len × 2 bytes (FP16)
LLaMA-2 70B at 4K context: 2 × 80 × 8 × 128 × 4096 × 2 = ~1.3GB
LLaMA-2 70B at 128K context: 2 × 80 × 8 × 128 × 128000 × 2 = ~42GB
```

### 3. Misunderstanding Attention Complexity
❌ **Wrong**: "Attention is O(n) because it's just matrix multiplication"
✅ **Right**: Standard self-attention is **O(n²)** in sequence length. This is why long-context models need special techniques (FlashAttention, sliding window).

### 4. Not Understanding Pre-Norm vs Post-Norm
❌ **Wrong**: Applying LayerNorm in the wrong place when implementing custom models
✅ **Right**: Modern models use Pre-Norm (normalize before sublayer). It's more stable for deep networks.

### 5. Forgetting About Tokenization Artifacts
❌ **Wrong**: Assuming 1 word = 1 token
✅ **Right**: Average English is ~1.3 tokens/word. Code is often 2-3 tokens/word. Non-English can be 3-5x more tokens.

---

## Interview Questions

### Conceptual

**Q1: Walk me through how a Transformer generates one token.**
> 1. Input tokens are converted to embeddings + positional encoding
> 2. Passes through N transformer blocks (each: attention → FFN)
> 3. Final layer norm applied
> 4. Linear projection to vocabulary size → logits
> 5. Apply temperature, top-k/top-p sampling
> 6. Sample one token from the distribution
> 7. Append to sequence, repeat from step 1

**Q2: Why is self-attention O(n²) and what are the solutions?**
> The attention matrix is (seq_len × seq_len), so memory and compute scale quadratically. Solutions:
> - FlashAttention: IO-aware exact attention (no approximation, just faster)
> - Sliding Window: Each token only attends to W neighbors
> - Linear Attention: Approximate attention in O(n)
> - Sparse Attention: Only compute attention for selected pairs

**Q3: Explain Grouped-Query Attention (GQA) and why it's used.**
> In standard MHA, each head has its own Q, K, V. In GQA, multiple Q heads share the same K, V heads. This reduces the KV cache size (dominant memory cost during generation) by the grouping factor while maintaining most of the quality. LLaMA-2 70B uses 64 Q heads but only 8 KV heads.

**Q4: What is FlashAttention and why is it important?**
> FlashAttention computes exact attention without materializing the full N×N attention matrix in GPU HBM (high-bandwidth memory). It tiles the computation to use fast SRAM, achieving 2-4x speedup and enabling longer sequences. It's not an approximation — it gives identical results to standard attention, just faster.

**Q5: Explain the difference between model parallelism and data parallelism.**
> Data parallelism: Same model on multiple GPUs, different data batches. Gradient sync after each step.
> Tensor parallelism: Split individual layers across GPUs (e.g., split attention heads).
> Pipeline parallelism: Split model layers across GPUs (layers 1-20 on GPU1, 21-40 on GPU2).
> ZeRO: Distribute optimizer states and gradients (DeepSpeed).

### Technical

**Q6: Why do modern LLMs use RoPE instead of learned positional embeddings?**
> RoPE encodes relative position in the attention dot product naturally, allowing better length generalization. Learned embeddings are absolute and can't extrapolate beyond training length. RoPE also enables techniques like position interpolation and YaRN for extending context length post-training.

**Q7: What is the role of the FFN layers in a Transformer?**
> FFN layers act as the "memory" of the model — storing factual knowledge and learned patterns. Attention routes information between positions; FFN transforms the information at each position independently. Studies show that factual knowledge can be localized to specific FFN neurons.

**Q8: How does Mixture of Experts (MoE) work?**
> A router network selects k-of-N expert FFN networks for each token. Only selected experts are activated, reducing compute while maintaining capacity. Key challenges: load balancing (all experts should be used equally) and communication costs (experts may be on different GPUs).

---

## Quick Reference

### Architecture Comparison Table

| Component | Original Transformer | GPT-2 | LLaMA-2/3 | Mistral |
|-----------|---------------------|-------|-----------|---------|
| Normalization | Post-Norm (LayerNorm) | Pre-Norm (LayerNorm) | Pre-Norm (RMSNorm) | Pre-Norm (RMSNorm) |
| Activation | ReLU | GELU | SwiGLU | SwiGLU |
| Position | Sinusoidal | Learned | RoPE | RoPE |
| Attention | MHA | MHA | GQA | GQA + Sliding Window |
| Bias | Yes | Yes | No | No |

### Key Dimensions Formula

| Metric | Formula |
|--------|---------|
| Total Params (approx) | $12 \times L \times d^2$ (for decoder-only) |
| FFN hidden dim | $4d$ (standard) or $\frac{8}{3}d$ (SwiGLU) |
| Head dimension | $d_{head} = d_{model} / n_{heads}$ |
| KV Cache (per token) | $2 \times L \times n_{kv\_heads} \times d_{head} \times 2$ bytes |
| FLOPs per token | $\approx 2 \times N$ (N = total params) |
| Training FLOPs | $\approx 6 \times N \times D$ (D = training tokens) |

### Memory Requirements (FP16)

| Model Size | Weights | KV Cache (4K ctx) | Total (inference) |
|-----------|---------|-------------------|-------------------|
| 7B | 14 GB | ~1 GB | ~16 GB |
| 13B | 26 GB | ~1.6 GB | ~28 GB |
| 70B | 140 GB | ~5 GB | ~150 GB |
| 405B | 810 GB | ~30 GB | ~850 GB |

### Attention Variants Cheat Sheet

| Variant | Mechanism | Use Case |
|---------|-----------|----------|
| MHA | Full Q, K, V per head | Original, highest quality |
| MQA | Shared single K, V | Fastest inference |
| GQA | Grouped K, V heads | Balance quality/speed |
| Sliding Window | Local attention window | Long sequences |
| FlashAttention | IO-optimized exact attention | Always (drop-in replacement) |
| Ring Attention | Distribute across GPUs | Very long context |

---

## Summary

The Transformer architecture's power comes from:
1. **Self-attention**: Dynamic, content-based routing of information
2. **Parallel processing**: Train on thousands of GPUs simultaneously
3. **Scaling**: Predictable improvement with more compute/data
4. **Simplicity**: Same architecture works for text, code, images, audio

Modern LLMs (2024-2026) are decoder-only Transformers with:
- RMSNorm (pre-norm)
- RoPE positional encoding
- SwiGLU activation
- GQA for efficient inference
- Trained on 10T+ tokens with RLHF alignment
