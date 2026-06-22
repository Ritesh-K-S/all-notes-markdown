# Chapter 06: Attention and Transformers

## Table of Contents
- [What is Attention?](#what-is-attention)
- [Why Attention and Transformers Matter](#why-attention-and-transformers-matter)
- [Evolution: From Seq2Seq to Attention](#evolution-from-seq2seq-to-attention)
- [Self-Attention Mechanism](#self-attention-mechanism)
- [Multi-Head Attention](#multi-head-attention)
- [Positional Encoding](#positional-encoding)
- [The Transformer Architecture](#the-transformer-architecture)
- [Training Transformers](#training-transformers)
- [Variants and Modern Developments](#variants-and-modern-developments)
- [Code Examples](#code-examples)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Attention?

Attention is a mechanism that allows a model to **focus on relevant parts** of the input when producing each part of the output — instead of trying to compress everything into a single fixed-size vector.

### Simple Analogy (For a 15-year-old)

Imagine you're translating a sentence from English to French:
- "The **cat** sat on the **mat**" → "Le **chat** est assis sur le **tapis**"

When translating "chat" (cat), you don't need to look at the whole sentence equally — you **focus** on the word "cat." When translating "tapis" (mat), you focus on "mat."

Attention is this ability to **selectively focus** on the most relevant parts of the input for each output token.

Without attention (Seq2Seq):
```
"The cat sat on the mat" → [single vector] → "Le chat est assis sur le tapis"
                            ↑ bottleneck!
```

With attention:
```
"The cat sat on the mat"
   ↑   ↑↑↑  ↑   ↑  ↑↑↑    ← Different words get different attention
   │   │││  │   │  │││       for each output token
   └───┘└┘──┘───┘──┘└┘───→ Each output token "looks back" at relevant input
```

---

## Why Attention and Transformers Matter

### The Revolution

| Before Transformers (2017) | After Transformers |
|---------------------------|-------------------|
| RNNs for sequences (slow, sequential) | Parallel processing |
| Struggle with long sequences | Handle 100K+ tokens |
| Task-specific architectures | One architecture for everything |
| Limited model sizes (~100M params) | GPT-4 has ~1.7T parameters |

### What Transformers Power Today

| Application | Model | Impact |
|------------|-------|--------|
| ChatGPT, Claude | GPT-4, Claude | Conversational AI |
| Google Search | BERT, T5 | Understanding queries |
| GitHub Copilot | Codex | Code generation |
| DALL-E, Midjourney | Vision Transformer | Image generation |
| AlphaFold | Transformer variant | Protein structure prediction |
| Self-driving cars | Vision Transformers | Scene understanding |

### Why Transformers Beat RNNs

```
RNN Processing (Sequential):
Token 1 → Token 2 → Token 3 → ... → Token N
                                      ↑ Must wait for all previous!
Time complexity: O(N) sequential steps

Transformer Processing (Parallel):
Token 1 ─┐
Token 2 ─┼─→ [All processed simultaneously] → Outputs
Token 3 ─┤
  ...    ─┤
Token N ─┘
Time complexity: O(1) sequential steps (but O(N²) compute per step)
```

---

## Evolution: From Seq2Seq to Attention

### The Bottleneck Problem (Recap)

In vanilla Seq2Seq, the encoder compresses the entire input into one vector:

$$\text{context} = h_T^{encoder}$$

For long sequences, this single vector can't capture everything → performance degrades.

### Bahdanau Attention (2014) — Additive Attention

**Key Idea**: Let the decoder look at ALL encoder hidden states, not just the last one.

```
Encoder states: [h₁, h₂, h₃, ..., hₙ]
                  ↓   ↓   ↓        ↓
                 α₁  α₂  α₃      αₙ   ← Attention weights (sum to 1)
                  ↓   ↓   ↓        ↓
Context:     c = α₁h₁ + α₂h₂ + α₃h₃ + ... + αₙhₙ  (weighted sum)
```

**How weights are computed:**

$$e_{ij} = v^T \tanh(W_1 s_{i-1} + W_2 h_j)$$
$$\alpha_{ij} = \frac{\exp(e_{ij})}{\sum_{k=1}^{N} \exp(e_{ik})}$$

Where:
- $s_{i-1}$ = decoder hidden state at previous step
- $h_j$ = encoder hidden state at position $j$
- $\alpha_{ij}$ = how much the decoder at step $i$ should "attend" to encoder position $j$

### Luong Attention (2015) — Multiplicative Attention

Simpler and faster:

$$e_{ij} = s_i^T W h_j \quad \text{(general)}$$
$$e_{ij} = s_i^T h_j \quad \text{(dot product, simplest)}$$

### Attention Types Comparison

| Type | Formula | Speed | Quality |
|------|---------|-------|---------|
| Additive (Bahdanau) | $v^T \tanh(W_1 s + W_2 h)$ | Slower | Better for small dims |
| Dot-product (Luong) | $s^T h$ | Fast | Needs same dimensions |
| Scaled dot-product | $\frac{s^T h}{\sqrt{d_k}}$ | Fast | Stable for large dims |

---

## Self-Attention Mechanism

### From Cross-Attention to Self-Attention

- **Cross-attention**: Query from decoder, Keys/Values from encoder (different sequences)
- **Self-attention**: Query, Keys, Values all from the **same** sequence

Self-attention lets each token attend to **every other token** in the same sequence:

```
Input: "The cat sat on the mat"
        ↕   ↕   ↕  ↕   ↕   ↕    ← Every token attends to every other token
       "The cat sat on the mat"

Q: "What am I looking for?"
K: "What do I contain?"
V: "What information do I provide if matched?"
```

### Query, Key, Value (QKV) Intuition

Think of it like a **search engine**:
- **Query (Q)**: Your search query ("I'm looking for the subject")
- **Key (K)**: The title/keywords of each document ("I'm a noun", "I'm a verb")
- **Value (V)**: The actual content of the document (the information to retrieve)

The **attention score** = how well a Query matches a Key.
The **output** = weighted sum of Values, based on those scores.

### Scaled Dot-Product Attention

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

Step by step:

```
1. Input: sequence of embeddings X = [x₁, x₂, ..., xₙ]  (shape: n × d_model)

2. Create Q, K, V via linear projections:
   Q = X · W_Q    (n × d_k)
   K = X · W_K    (n × d_k)
   V = X · W_V    (n × d_v)

3. Compute attention scores:
   scores = Q · K^T    (n × n matrix — every token vs every token)
   
4. Scale by √d_k:
   scores = scores / √d_k   (prevents softmax saturation)

5. Apply softmax (row-wise):
   weights = softmax(scores)   (n × n, each row sums to 1)

6. Weighted sum of values:
   output = weights · V    (n × d_v)
```

### Why Scale by $\sqrt{d_k}$?

When $d_k$ is large, dot products grow in magnitude (variance = $d_k$). This pushes softmax into regions with very small gradients:

$$\text{Var}(q \cdot k) = d_k \quad \text{(if q, k have unit variance)}$$

Dividing by $\sqrt{d_k}$ normalizes the variance to 1, keeping softmax in its useful range.

```
Without scaling (d_k = 512):
Dot products might be [-50, +80, -30, ...] → softmax → [0.0, 1.0, 0.0, ...]
                                                         ↑ Nearly one-hot! No gradient!

With scaling:
Dot products / √512 ≈ [-2.2, +3.5, -1.3, ...] → softmax → [0.01, 0.85, 0.03, ...]
                                                              ↑ Smooth distribution!
```

### Visual Example: Self-Attention in Action

```
Sentence: "The animal didn't cross the street because it was too wide"

What does "it" attend to?

Token:     The  animal  didn't  cross  the  street  because  it  was  too  wide
Attention: 0.02  0.08   0.01   0.03   0.02  0.72    0.02   0.01 0.02 0.01 0.06
                                              ↑
                                    "it" attends most to "street"!
                              (The model learned "it" refers to "street")
```

### Attention Complexity

| Operation | Time Complexity | Space Complexity |
|-----------|---------------|-----------------|
| QK^T computation | $O(n^2 \cdot d_k)$ | $O(n^2)$ |
| Softmax | $O(n^2)$ | $O(n^2)$ |
| weights × V | $O(n^2 \cdot d_v)$ | $O(n \cdot d_v)$ |
| **Total** | **$O(n^2 \cdot d)$** | **$O(n^2 + n \cdot d)$** |

> **The $O(n^2)$ bottleneck**: This quadratic scaling is why standard Transformers struggle with very long sequences (>4K tokens without optimizations).

---

## Multi-Head Attention

### Why Multiple Heads?

A single attention head can only learn one "type" of relationship. But language has many:
- Syntactic (subject-verb agreement)
- Semantic (coreference: "it" → "cat")
- Positional (adjacent words)
- Long-range dependencies

**Multi-head attention** runs multiple attention operations in parallel, each learning a different pattern.

### How It Works

```
Input X (n × d_model)
    │
    ├──→ Head 1: Attention(XW₁Q, XW₁K, XW₁V) → (n × d_v/h)
    ├──→ Head 2: Attention(XW₂Q, XW₂K, XW₂V) → (n × d_v/h)
    ├──→ Head 3: Attention(XW₃Q, XW₃K, XW₃V) → (n × d_v/h)
    │    ...
    └──→ Head h: Attention(XWhQ, XWhK, XWhV) → (n × d_v/h)
              │
              ├── Concatenate → (n × d_v)
              │
              └── Linear(W_O) → (n × d_model)
```

### Mathematical Formulation

$$\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, ..., \text{head}_h) W^O$$

Where each head:
$$\text{head}_i = \text{Attention}(QW_i^Q, KW_i^K, VW_i^V)$$

Dimensions:
- $W_i^Q \in \mathbb{R}^{d_{model} \times d_k}$ where $d_k = d_{model} / h$
- $W_i^K \in \mathbb{R}^{d_{model} \times d_k}$
- $W_i^V \in \mathbb{R}^{d_{model} \times d_v}$ where $d_v = d_{model} / h$
- $W^O \in \mathbb{R}^{hd_v \times d_{model}}$

### Parameter Efficiency

With $h$ heads, each head operates on $d_k = d_{model}/h$ dimensions.

Total compute = same as single-head attention on full $d_{model}$!

```
Single head (d_model=512):      Multi-head (8 heads, d_k=64 each):
Q: 512×512 = 262,144            Q: 8 × (512×64) = 262,144  ← Same!
K: 512×512 = 262,144            K: 8 × (512×64) = 262,144  ← Same!
V: 512×512 = 262,144            V: 8 × (512×64) = 262,144  ← Same!
Total: 786,432                   Total: 786,432 + W_O(512×512) = 1,048,576
```

### What Different Heads Learn

Research shows different heads specialize:
```
Head 1: Attends to adjacent tokens (local context)
Head 2: Attends to sentence-beginning tokens (subject tracking)
Head 3: Attends to previous occurrence of same word
Head 4: Attends to syntactic dependencies (subject-verb)
Head 5: Attends to semantically related words
...
```

---

## Positional Encoding

### The Problem

Self-attention is **permutation-invariant** — it treats the input as a SET, not a sequence:

$$\text{Attention}(\text{"cat sat mat"}) = \text{Attention}(\text{"mat cat sat"})$$

But word order matters! "Dog bites man" ≠ "Man bites dog"

### Sinusoidal Positional Encoding (Original Transformer)

Add position information directly to the input embeddings:

$$PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right)$$

$$PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)$$

Where:
- $pos$ = position in the sequence (0, 1, 2, ...)
- $i$ = dimension index (0, 1, 2, ..., $d_{model}/2$)

```
Final input to Transformer = Token Embedding + Positional Encoding

Embedding("cat") = [0.2, -0.5, 0.8, ...]     (learned, same for "cat" everywhere)
PE(position=3)   = [0.14, 0.99, 0.001, ...]  (fixed, unique for position 3)
Input            = [0.34, 0.49, 0.801, ...]   (sum)
```

### Why Sinusoids?

1. **Unique encoding** for each position
2. **Bounded values** (always between -1 and 1)
3. **Relative positions are capturable**: $PE(pos+k)$ can be expressed as a linear function of $PE(pos)$ — the model can learn to attend to relative positions
4. **Generalizes to longer sequences** than seen during training

### Visualization of Positional Encodings

```
Position →   0     1     2     3     4     5     ...
Dim 0:     sin(0) sin(1) sin(2) sin(3) sin(4) sin(5)    ← Fast oscillation
Dim 1:     cos(0) cos(1) cos(2) cos(3) cos(4) cos(5)
Dim 2:     sin(0) sin(.1) sin(.2) sin(.3) sin(.4) sin(.5) ← Slower
Dim 3:     cos(0) cos(.1) cos(.2) cos(.3) cos(.4) cos(.5)
...
Dim d-1:   sin(0) sin(.001) sin(.002) ...                  ← Very slow

Each position gets a unique "fingerprint" pattern!
```

### Learned Positional Embeddings (Alternative)

Used in BERT, GPT:
```python
self.pos_embedding = nn.Embedding(max_seq_len, d_model)
# Learned during training — more flexible but can't generalize beyond max_seq_len
```

### RoPE (Rotary Positional Embedding) — Modern Standard

Used in LLaMA, Mistral, GPT-NeoX. Encodes position by **rotating** the query/key vectors:

$$f(x_m, m) = x_m e^{im\theta}$$

Advantages:
- Relative position information encoded in dot products
- Better extrapolation to longer sequences
- No addition — applied as rotation to Q and K

---

## The Transformer Architecture

### Full Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        TRANSFORMER                               │
│                                                                   │
│  ┌──────────────────┐           ┌──────────────────────────┐    │
│  │     ENCODER       │           │        DECODER            │    │
│  │     (×N layers)   │           │        (×N layers)        │    │
│  │                    │           │                            │    │
│  │  ┌──────────────┐│           │  ┌──────────────────────┐ │    │
│  │  │ Feed-Forward  ││           │  │    Feed-Forward       │ │    │
│  │  │   Network     ││           │  │      Network          │ │    │
│  │  └──────┬───────┘│           │  └──────────┬───────────┘ │    │
│  │         │ Add&Norm ││           │             │ Add&Norm    │    │
│  │  ┌──────┴───────┐│           │  ┌──────────┴───────────┐ │    │
│  │  │  Multi-Head   ││    ┌─────→  │  Multi-Head           │ │    │
│  │  │  Self-Attn    ││    │     │  │  Cross-Attention      │ │    │
│  │  └──────┬───────┘│    │     │  │  (Q from decoder,     │ │    │
│  │         │ Add&Norm ││    │     │  │   K,V from encoder)  │ │    │
│  │         │          │◄───┘     │  └──────────┬───────────┘ │    │
│  │  ┌──────┴───────┐│           │             │ Add&Norm    │    │
│  │  │   Input       ││           │  ┌──────────┴───────────┐ │    │
│  │  │  Embedding    ││           │  │  Masked Multi-Head    │ │    │
│  │  │  + Pos Enc    ││           │  │  Self-Attention       │ │    │
│  │  └──────┬───────┘│           │  └──────────┬───────────┘ │    │
│  └─────────┼────────┘           │             │ Add&Norm    │    │
│            ↑                     │  ┌──────────┴───────────┐ │    │
│     Source Tokens                │  │  Output Embedding     │ │    │
│     "I love cats"                │  │  + Pos Enc            │ │    │
│                                  │  └──────────┬───────────┘ │    │
│                                  └─────────────┼─────────────┘    │
│                                               ↑                   │
│                                        Target Tokens               │
│                                     "<SOS> J'aime les"            │
│                                               ↓                   │
│                                  ┌─────────────────────────┐      │
│                                  │  Linear + Softmax       │      │
│                                  │  → Next token prediction │      │
│                                  └─────────────────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### Layer Components in Detail

#### 1. Add & Norm (Residual Connection + Layer Normalization)

$$\text{LayerNorm}(x + \text{Sublayer}(x))$$

```
Input x → Sublayer(x) → Add(x + Sublayer(x)) → LayerNorm → Output
          (attention     ↑ Residual connection!
           or FFN)
```

Why both?
- **Residual connection**: Ensures gradient flow, enables deep networks (like ResNet)
- **Layer Normalization**: Stabilizes training, normalizes across features (not batch)

#### 2. Feed-Forward Network (Position-wise)

$$\text{FFN}(x) = \max(0, xW_1 + b_1)W_2 + b_2$$

- Applied **independently** to each position (token)
- First linear expands dimensions (×4 typically): $d_{model} \to 4 \times d_{model}$
- ReLU (or GELU) activation
- Second linear projects back: $4 \times d_{model} \to d_{model}$

```
x (512) → Linear(512→2048) → ReLU → Linear(2048→512) → output (512)
```

#### 3. Masked Self-Attention (Decoder)

In the decoder, tokens can only attend to **previous** positions (can't peek at future!):

```
Attention mask (causal):
         Position  1  2  3  4  5
Token 1:          [1, 0, 0, 0, 0]  ← Can only see itself
Token 2:          [1, 1, 0, 0, 0]  ← Can see positions 1-2
Token 3:          [1, 1, 1, 0, 0]  ← Can see positions 1-3
Token 4:          [1, 1, 1, 1, 0]
Token 5:          [1, 1, 1, 1, 1]  ← Can see all previous

Applied by setting future positions to -∞ before softmax:
scores[future] = -∞ → softmax(-∞) = 0
```

### Original Transformer Hyperparameters ("Attention Is All You Need")

| Parameter | Value |
|-----------|-------|
| $d_{model}$ (embedding dimension) | 512 |
| $h$ (number of attention heads) | 8 |
| $d_k = d_v = d_{model}/h$ | 64 |
| $d_{ff}$ (FFN inner dimension) | 2048 |
| Number of encoder layers $N$ | 6 |
| Number of decoder layers $N$ | 6 |
| Dropout | 0.1 |
| Max sequence length | 512 |
| Total parameters | ~65M |

---

## Training Transformers

### Learning Rate Schedule: Warmup + Decay

The original paper uses a special schedule:

$$lr = d_{model}^{-0.5} \cdot \min(step^{-0.5}, step \cdot warmup\_steps^{-1.5})$$

```
Learning Rate
     ↑
     │        /\
     │       /  \
     │      /    \
     │     /      \──────
     │    /               \──────
     │   /                        \──────
     │  /
     │ /
     └────────────────────────────────────→ Steps
     0    warmup         training
          (4000 steps)
```

Why warmup? Early in training, model weights are random → attention weights are ~uniform → gradients are noisy. Small LR lets the model stabilize before aggressive learning.

### Label Smoothing

Instead of hard targets [0, 0, 1, 0, 0], use soft targets [0.025, 0.025, 0.9, 0.025, 0.025]:

$$y_{smooth} = (1 - \epsilon) \cdot y_{hot} + \epsilon / K$$

Where $\epsilon = 0.1$ (typical), $K$ = vocabulary size. Improves generalization and BLEU scores.

### Key Training Techniques

| Technique | Purpose |
|-----------|---------|
| Warmup scheduling | Stable early training |
| Label smoothing | Better generalization |
| Gradient accumulation | Simulate larger batches |
| Mixed precision (fp16/bf16) | 2× speed, half memory |
| Gradient clipping | Prevent instability |
| Weight tying | Share embedding + output weights |

---

## Variants and Modern Developments

### Encoder-Only Models (BERT Family)

```
Use: Classification, NER, question answering
Training: Masked Language Modeling (predict [MASK] tokens)
Examples: BERT, RoBERTa, ALBERT, DeBERTa
```

```
Input:  "The [MASK] sat on the [MASK]"
Output: Predict "cat" and "mat"

Key: Bidirectional — sees all tokens (no causal mask)
```

### Decoder-Only Models (GPT Family)

```
Use: Text generation, chat, code generation
Training: Next token prediction (autoregressive)
Examples: GPT-2/3/4, LLaMA, Mistral, Claude
```

```
Input:  "The cat sat on"
Output: Predict "the" (next token)

Key: Causal mask — only sees past tokens
```

### Encoder-Decoder Models (T5 Family)

```
Use: Translation, summarization, any seq-to-seq
Training: Span corruption (mask spans, predict them)
Examples: T5, BART, mBART, FLAN-T5
```

### Architecture Comparison

| Model Type | Attention | Training Objective | Best For |
|-----------|-----------|-------------------|----------|
| Encoder-only | Bidirectional | MLM | Understanding/classification |
| Decoder-only | Causal (left-to-right) | Next token | Generation |
| Encoder-Decoder | Bi (enc) + Causal (dec) | Denoising | Seq-to-seq tasks |

### Efficient Attention Variants

| Method | Complexity | Key Idea |
|--------|-----------|----------|
| Standard | $O(n^2)$ | Full attention matrix |
| Sparse (Longformer) | $O(n)$ | Local window + global tokens |
| Linear (Linformer) | $O(n)$ | Low-rank approximation of attention |
| Flash Attention | $O(n^2)$ time, $O(n)$ memory | IO-aware tiling (no materialization) |
| Multi-Query Attention | $O(n^2)$ | Share K,V across heads (faster inference) |
| Grouped-Query Attention | $O(n^2)$ | Share K,V among groups of heads |

### Flash Attention

Not an approximation — computes **exact** attention but with optimized GPU memory access:

```
Standard Attention:
Q, K, V → compute QK^T (n×n matrix in HBM!) → softmax → multiply by V
                       ↑ This is the memory bottleneck!

Flash Attention:
Process attention in tiles that fit in GPU SRAM (fast memory)
Never materializes the full n×n matrix in slow HBM memory
Result: 2-4× faster, uses O(n) memory instead of O(n²)
```

---

## Code Examples

### Self-Attention from Scratch

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class ScaledDotProductAttention(nn.Module):
    """Implement attention from scratch to understand every detail"""
    
    def __init__(self):
        super().__init__()
    
    def forward(self, Q, K, V, mask=None):
        """
        Args:
            Q: (batch, num_heads, seq_len, d_k)
            K: (batch, num_heads, seq_len, d_k)
            V: (batch, num_heads, seq_len, d_v)
            mask: (batch, 1, 1, seq_len) or (batch, 1, seq_len, seq_len)
        Returns:
            output: (batch, num_heads, seq_len, d_v)
            attention_weights: (batch, num_heads, seq_len, seq_len)
        """
        d_k = Q.size(-1)
        
        # Step 1: Compute attention scores
        # Q @ K^T: (batch, heads, seq_q, d_k) @ (batch, heads, d_k, seq_k)
        #        = (batch, heads, seq_q, seq_k)
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(d_k)
        
        # Step 2: Apply mask (if provided)
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        
        # Step 3: Softmax to get attention weights
        attention_weights = F.softmax(scores, dim=-1)
        
        # Step 4: Weighted sum of values
        output = torch.matmul(attention_weights, V)
        
        return output, attention_weights


# Demo
batch, seq_len, d_model = 2, 5, 8
Q = torch.randn(batch, 1, seq_len, d_model)  # 1 head for simplicity
K = torch.randn(batch, 1, seq_len, d_model)
V = torch.randn(batch, 1, seq_len, d_model)

attn = ScaledDotProductAttention()
output, weights = attn(Q, K, V)

print(f"Output shape: {output.shape}")      # (2, 1, 5, 8)
print(f"Attention weights shape: {weights.shape}")  # (2, 1, 5, 5)
print(f"Weights sum per query: {weights.sum(-1)}")  # All 1.0
```

### Multi-Head Attention Implementation

```python
class MultiHeadAttention(nn.Module):
    """Complete Multi-Head Attention implementation"""
    
    def __init__(self, d_model, num_heads, dropout=0.1):
        super().__init__()
        assert d_model % num_heads == 0, "d_model must be divisible by num_heads"
        
        self.d_model = d_model
        self.num_heads = num_heads
        self.d_k = d_model // num_heads  # Dimension per head
        
        # Linear projections for Q, K, V (all heads computed together)
        self.W_q = nn.Linear(d_model, d_model)  # Projects to all heads at once
        self.W_k = nn.Linear(d_model, d_model)
        self.W_v = nn.Linear(d_model, d_model)
        self.W_o = nn.Linear(d_model, d_model)  # Output projection
        
        self.attention = ScaledDotProductAttention()
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, query, key, value, mask=None):
        """
        Args:
            query: (batch, seq_len, d_model)
            key:   (batch, seq_len, d_model)  [can differ from query for cross-attn]
            value: (batch, seq_len, d_model)
            mask:  Optional attention mask
        """
        batch_size = query.size(0)
        
        # 1. Linear projections: (batch, seq_len, d_model)
        Q = self.W_q(query)
        K = self.W_k(key)
        V = self.W_v(value)
        
        # 2. Reshape to (batch, num_heads, seq_len, d_k)
        #    Split d_model into num_heads × d_k
        Q = Q.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        K = K.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        V = V.view(batch_size, -1, self.num_heads, self.d_k).transpose(1, 2)
        
        # 3. Apply attention
        attn_output, attn_weights = self.attention(Q, K, V, mask)
        
        # 4. Concatenate heads: (batch, seq_len, d_model)
        attn_output = attn_output.transpose(1, 2).contiguous()
        attn_output = attn_output.view(batch_size, -1, self.d_model)
        
        # 5. Final linear projection
        output = self.W_o(attn_output)
        output = self.dropout(output)
        
        return output, attn_weights


# Test
mha = MultiHeadAttention(d_model=512, num_heads=8)
x = torch.randn(2, 10, 512)  # batch=2, seq=10, dim=512
output, weights = mha(x, x, x)  # Self-attention (Q=K=V=x)
print(f"Output: {output.shape}")   # (2, 10, 512)
print(f"Weights: {weights.shape}") # (2, 8, 10, 10)
```

### Complete Transformer Encoder Layer

```python
class TransformerEncoderLayer(nn.Module):
    """One layer of the Transformer encoder"""
    
    def __init__(self, d_model, num_heads, d_ff, dropout=0.1):
        super().__init__()
        
        # Multi-head self-attention
        self.self_attention = MultiHeadAttention(d_model, num_heads, dropout)
        self.norm1 = nn.LayerNorm(d_model)
        
        # Position-wise feed-forward network
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff),      # Expand: 512 → 2048
            nn.GELU(),                      # Modern activation (smoother than ReLU)
            nn.Dropout(dropout),
            nn.Linear(d_ff, d_model),      # Contract: 2048 → 512
            nn.Dropout(dropout)
        )
        self.norm2 = nn.LayerNorm(d_model)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, x, mask=None):
        """
        Args:
            x: (batch, seq_len, d_model)
            mask: Optional padding mask
        """
        # Sub-layer 1: Multi-head self-attention + residual + norm
        attn_output, _ = self.self_attention(x, x, x, mask)
        x = self.norm1(x + self.dropout(attn_output))  # Pre-norm variant: norm(x) + attn
        
        # Sub-layer 2: FFN + residual + norm
        ffn_output = self.ffn(x)
        x = self.norm2(x + ffn_output)
        
        return x


class TransformerEncoder(nn.Module):
    """Stack of N encoder layers"""
    
    def __init__(self, vocab_size, d_model=512, num_heads=8, 
                 d_ff=2048, num_layers=6, max_len=512, dropout=0.1):
        super().__init__()
        
        self.d_model = d_model
        self.embedding = nn.Embedding(vocab_size, d_model)
        self.pos_encoding = self._sinusoidal_encoding(max_len, d_model)
        self.dropout = nn.Dropout(dropout)
        
        self.layers = nn.ModuleList([
            TransformerEncoderLayer(d_model, num_heads, d_ff, dropout)
            for _ in range(num_layers)
        ])
        self.norm = nn.LayerNorm(d_model)
    
    def _sinusoidal_encoding(self, max_len, d_model):
        """Create sinusoidal positional encodings"""
        pe = torch.zeros(max_len, d_model)
        position = torch.arange(0, max_len, dtype=torch.float).unsqueeze(1)
        div_term = torch.exp(torch.arange(0, d_model, 2).float() * 
                            (-math.log(10000.0) / d_model))
        
        pe[:, 0::2] = torch.sin(position * div_term)  # Even dimensions
        pe[:, 1::2] = torch.cos(position * div_term)  # Odd dimensions
        
        return nn.Parameter(pe.unsqueeze(0), requires_grad=False)  # (1, max_len, d_model)
    
    def forward(self, x, mask=None):
        """
        Args:
            x: (batch, seq_len) token indices
        """
        seq_len = x.size(1)
        
        # Embed tokens + add positional encoding
        x = self.embedding(x) * math.sqrt(self.d_model)  # Scale embedding
        x = x + self.pos_encoding[:, :seq_len, :]
        x = self.dropout(x)
        
        # Pass through encoder layers
        for layer in self.layers:
            x = layer(x, mask)
        
        return self.norm(x)


# Test
encoder = TransformerEncoder(vocab_size=30000, d_model=512, num_heads=8,
                              d_ff=2048, num_layers=6)
tokens = torch.randint(0, 30000, (2, 50))  # batch=2, seq=50
output = encoder(tokens)
print(f"Encoder output: {output.shape}")  # (2, 50, 512)
print(f"Parameters: {sum(p.numel() for p in encoder.parameters()):,}")
```

### Complete Transformer for Classification (BERT-style)

```python
class TransformerClassifier(nn.Module):
    """BERT-like model for text classification"""
    
    def __init__(self, vocab_size, num_classes, d_model=256, 
                 num_heads=8, d_ff=1024, num_layers=4,
                 max_len=512, dropout=0.1):
        super().__init__()
        
        self.encoder = TransformerEncoder(
            vocab_size, d_model, num_heads, d_ff, 
            num_layers, max_len, dropout
        )
        
        # Classification head (use [CLS] token representation)
        self.classifier = nn.Sequential(
            nn.Linear(d_model, d_model),
            nn.Tanh(),
            nn.Dropout(dropout),
            nn.Linear(d_model, num_classes)
        )
    
    def forward(self, input_ids, attention_mask=None):
        """
        Args:
            input_ids: (batch, seq_len)
            attention_mask: (batch, seq_len) — 1 for real tokens, 0 for padding
        """
        # Create attention mask for padding
        if attention_mask is not None:
            # Expand mask for multi-head attention
            mask = attention_mask.unsqueeze(1).unsqueeze(2)  # (batch, 1, 1, seq_len)
        else:
            mask = None
        
        # Encode
        encoder_output = self.encoder(input_ids, mask)  # (batch, seq_len, d_model)
        
        # Use first token ([CLS]) for classification
        cls_output = encoder_output[:, 0, :]  # (batch, d_model)
        
        # Classify
        logits = self.classifier(cls_output)
        return logits


# ─── Training Example ──────────────────────────────────────
model = TransformerClassifier(vocab_size=30000, num_classes=4, 
                               d_model=256, num_heads=8, num_layers=4)

print(f"Model parameters: {sum(p.numel() for p in model.parameters()):,}")

# Simulated batch
input_ids = torch.randint(1, 30000, (16, 128))  # batch=16, seq=128
attention_mask = torch.ones(16, 128)
attention_mask[:, 100:] = 0  # Last 28 tokens are padding

logits = model(input_ids, attention_mask)
print(f"Output logits: {logits.shape}")  # (16, 4)

# Training
criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.AdamW(model.parameters(), lr=5e-5, weight_decay=0.01)

labels = torch.randint(0, 4, (16,))
loss = criterion(logits, labels)
loss.backward()
optimizer.step()
print(f"Loss: {loss.item():.4f}")
```

### Causal (Decoder) Transformer for Text Generation

```python
class CausalTransformerBlock(nn.Module):
    """Decoder block with causal (masked) self-attention"""
    
    def __init__(self, d_model, num_heads, d_ff, dropout=0.1):
        super().__init__()
        self.attention = MultiHeadAttention(d_model, num_heads, dropout)
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_ff),
            nn.GELU(),
            nn.Dropout(dropout),
            nn.Linear(d_ff, d_model),
            nn.Dropout(dropout)
        )
        self.norm1 = nn.LayerNorm(d_model)
        self.norm2 = nn.LayerNorm(d_model)
    
    def forward(self, x, causal_mask):
        # Pre-norm architecture (GPT-style, more stable for deep models)
        normed = self.norm1(x)
        attn_out, _ = self.attention(normed, normed, normed, causal_mask)
        x = x + attn_out
        
        normed = self.norm2(x)
        ffn_out = self.ffn(normed)
        x = x + ffn_out
        
        return x


class GPTModel(nn.Module):
    """Simplified GPT-style language model"""
    
    def __init__(self, vocab_size, d_model=512, num_heads=8,
                 d_ff=2048, num_layers=6, max_len=1024, dropout=0.1):
        super().__init__()
        
        self.token_embedding = nn.Embedding(vocab_size, d_model)
        self.position_embedding = nn.Embedding(max_len, d_model)
        self.dropout = nn.Dropout(dropout)
        
        self.blocks = nn.ModuleList([
            CausalTransformerBlock(d_model, num_heads, d_ff, dropout)
            for _ in range(num_layers)
        ])
        
        self.norm = nn.LayerNorm(d_model)
        self.lm_head = nn.Linear(d_model, vocab_size, bias=False)
        
        # Weight tying: share embedding and output weights
        self.lm_head.weight = self.token_embedding.weight
        
        # Create causal mask (lower triangular)
        self.register_buffer('causal_mask', 
                            torch.tril(torch.ones(max_len, max_len))
                            .unsqueeze(0).unsqueeze(0))
    
    def forward(self, input_ids):
        """
        Args:
            input_ids: (batch, seq_len)
        Returns:
            logits: (batch, seq_len, vocab_size)
        """
        batch, seq_len = input_ids.shape
        
        # Token + positional embeddings
        positions = torch.arange(seq_len, device=input_ids.device).unsqueeze(0)
        x = self.token_embedding(input_ids) + self.position_embedding(positions)
        x = self.dropout(x)
        
        # Causal mask for this sequence length
        mask = self.causal_mask[:, :, :seq_len, :seq_len]
        
        # Pass through transformer blocks
        for block in self.blocks:
            x = block(x, mask)
        
        x = self.norm(x)
        
        # Project to vocabulary
        logits = self.lm_head(x)  # (batch, seq_len, vocab_size)
        return logits
    
    @torch.no_grad()
    def generate(self, input_ids, max_new_tokens=100, temperature=1.0, top_k=50):
        """Autoregressive text generation"""
        self.eval()
        
        for _ in range(max_new_tokens):
            # Get logits for the last position
            logits = self(input_ids)[:, -1, :]  # (batch, vocab_size)
            
            # Apply temperature
            logits = logits / temperature
            
            # Top-k filtering
            if top_k > 0:
                values, _ = torch.topk(logits, top_k)
                min_value = values[:, -1].unsqueeze(-1)
                logits[logits < min_value] = float('-inf')
            
            # Sample from distribution
            probs = F.softmax(logits, dim=-1)
            next_token = torch.multinomial(probs, num_samples=1)
            
            # Append to sequence
            input_ids = torch.cat([input_ids, next_token], dim=1)
        
        return input_ids


# Test
gpt = GPTModel(vocab_size=10000, d_model=256, num_heads=8, 
               d_ff=1024, num_layers=4, max_len=512)

print(f"GPT Parameters: {sum(p.numel() for p in gpt.parameters()):,}")

# Forward pass
tokens = torch.randint(0, 10000, (2, 32))
logits = gpt(tokens)
print(f"Logits: {logits.shape}")  # (2, 32, 10000)

# Generate
prompt = torch.randint(0, 10000, (1, 5))  # 5-token prompt
generated = gpt.generate(prompt, max_new_tokens=20, temperature=0.8)
print(f"Generated sequence length: {generated.shape[1]}")  # 25 (5 + 20)
```

### Using HuggingFace Transformers (Production)

```python
from transformers import (
    AutoTokenizer, 
    AutoModelForSequenceClassification,
    AutoModelForCausalLM,
    Trainer, 
    TrainingArguments
)
import torch

# ─── BERT for Classification ──────────────────────────────
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
model = AutoModelForSequenceClassification.from_pretrained(
    "bert-base-uncased", num_labels=3
)

# Tokenize text
texts = ["This movie was amazing!", "Terrible waste of time."]
inputs = tokenizer(texts, padding=True, truncation=True, 
                   max_length=128, return_tensors="pt")

# Forward pass
outputs = model(**inputs)
predictions = torch.argmax(outputs.logits, dim=-1)
print(f"Predictions: {predictions}")

# ─── GPT-2 for Text Generation ────────────────────────────
tokenizer = AutoTokenizer.from_pretrained("gpt2")
model = AutoModelForCausalLM.from_pretrained("gpt2")

prompt = "The future of artificial intelligence is"
input_ids = tokenizer.encode(prompt, return_tensors="pt")

# Generate with various strategies
output = model.generate(
    input_ids,
    max_new_tokens=50,
    temperature=0.7,
    top_k=50,
    top_p=0.95,           # Nucleus sampling
    do_sample=True,
    repetition_penalty=1.2,
    no_repeat_ngram_size=3
)

generated_text = tokenizer.decode(output[0], skip_special_tokens=True)
print(generated_text)

# ─── Fine-tuning with HuggingFace Trainer ─────────────────
training_args = TrainingArguments(
    output_dir="./results",
    num_train_epochs=3,
    per_device_train_batch_size=16,
    per_device_eval_batch_size=64,
    warmup_steps=500,
    weight_decay=0.01,
    logging_dir="./logs",
    logging_steps=100,
    eval_strategy="epoch",
    save_strategy="epoch",
    load_best_model_at_end=True,
    fp16=True,  # Mixed precision
    gradient_accumulation_steps=2,  # Effective batch = 16 * 2 = 32
)

# trainer = Trainer(
#     model=model,
#     args=training_args,
#     train_dataset=train_dataset,
#     eval_dataset=eval_dataset,
# )
# trainer.train()
```

---

## Common Mistakes

### 1. Forgetting the Causal Mask in Decoder

```python
# ❌ WRONG: No mask → model can attend to future tokens (data leakage!)
output = self.attention(x, x, x, mask=None)

# ✅ CORRECT: Apply causal mask
seq_len = x.size(1)
causal_mask = torch.tril(torch.ones(seq_len, seq_len)).unsqueeze(0).unsqueeze(0)
output = self.attention(x, x, x, mask=causal_mask)
```

### 2. Not Scaling Embeddings by $\sqrt{d_{model}}$

```python
# ❌ WRONG: Positional encodings may dominate embeddings
x = self.embedding(tokens) + self.pos_encoding

# ✅ CORRECT: Scale embedding to match positional encoding magnitude
x = self.embedding(tokens) * math.sqrt(self.d_model) + self.pos_encoding
# Embedding values are typically small after init; scaling brings them
# to the same order of magnitude as positional encodings
```

### 3. Wrong Mask Shape for Padding

```python
# ❌ WRONG: Mask shape doesn't broadcast correctly
padding_mask = (input_ids != pad_token_id)  # (batch, seq_len)
# Can't directly use in attention: needs (batch, 1, 1, seq_len) or similar

# ✅ CORRECT: Reshape for broadcasting in multi-head attention
padding_mask = (input_ids != pad_token_id).unsqueeze(1).unsqueeze(2)
# Now: (batch, 1, 1, seq_len) — broadcasts across heads and query positions
```

### 4. Not Using Pre-Norm for Deep Transformers

```python
# ❌ Post-norm (original paper) — harder to train for deep models (>12 layers)
x = self.norm(x + self.attention(x))

# ✅ Pre-norm (GPT-2, modern practice) — much more stable
x = x + self.attention(self.norm(x))
```

### 5. Ignoring Attention Masking for Variable-Length Inputs

```python
# ❌ WRONG: Padding tokens participate in attention
logits = model(padded_input_ids)

# ✅ CORRECT: Pass attention mask to zero out padding
attention_mask = (input_ids != tokenizer.pad_token_id).long()
logits = model(input_ids, attention_mask=attention_mask)
```

### 6. Using Wrong Learning Rate

```python
# ❌ WRONG: Standard LR too high for Transformers
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)

# ✅ CORRECT: Use smaller LR with warmup
optimizer = torch.optim.AdamW(model.parameters(), lr=5e-5, weight_decay=0.01)
# + Add learning rate warmup scheduler!
scheduler = torch.optim.lr_scheduler.OneCycleLR(
    optimizer, max_lr=5e-5, total_steps=total_training_steps,
    pct_start=0.1  # 10% warmup
)
```

### 7. KV-Cache Mistake During Generation

```python
# ❌ WRONG: Recompute attention for ALL tokens at every generation step
for i in range(max_tokens):
    logits = model(full_sequence)  # O(n²) at every step!

# ✅ CORRECT: Use KV-cache — only process the new token
past_key_values = None
for i in range(max_tokens):
    logits, past_key_values = model(
        new_token_only, 
        past_key_values=past_key_values,
        use_cache=True
    )
# Each step is O(n) instead of O(n²)!
```

---

## Interview Questions

### Conceptual Questions

**Q1: Explain the self-attention mechanism step by step.**

1. Input embeddings are projected into Q, K, V matrices via learned linear layers
2. Compute attention scores: $QK^T$ (dot product of every query with every key)
3. Scale by $\sqrt{d_k}$ to prevent softmax saturation
4. Apply softmax to get attention weights (each row sums to 1)
5. Multiply weights by V to get weighted combination of values
6. Apply output projection

**Q2: Why do we scale by $\sqrt{d_k}$ in attention? What happens without it?**

Dot products of Q and K have variance proportional to $d_k$. For large $d_k$ (e.g., 64), scores become very large → softmax produces near-one-hot distributions → almost zero gradients. Scaling by $\sqrt{d_k}$ normalizes the variance to ~1, keeping softmax in a region with useful gradients.

**Q3: What's the difference between self-attention and cross-attention?**

- **Self-attention**: Q, K, V all come from the same sequence. Used in encoder and masked decoder.
- **Cross-attention**: Q from decoder, K and V from encoder output. Used in encoder-decoder models to let decoder "look at" the source.

**Q4: Why does the Transformer use positional encoding? How does it work?**

Self-attention is permutation-invariant — it doesn't know word order. Positional encodings inject position information by adding unique signals to each position. Sinusoidal encodings use sine/cosine at different frequencies so each position has a unique pattern, and relative positions can be learned.

**Q5: Explain the difference between encoder-only, decoder-only, and encoder-decoder Transformers.**

| Type | Attention | Training | Use Case | Example |
|------|-----------|----------|----------|---------|
| Encoder-only | Bidirectional | MLM | Understanding | BERT |
| Decoder-only | Causal | Next token | Generation | GPT |
| Enc-Dec | Bi + Causal | Denoising | Seq2seq | T5 |

**Q6: What is Multi-Head Attention and why is it useful?**

Instead of one attention operation, we run multiple (e.g., 8) in parallel with different learned projections. Each head can learn different relationships (syntax, semantics, position). Total computation is the same as single-head on full dimensions because each head operates on $d_{model}/h$ dimensions.

**Q7: What is the computational complexity of self-attention? How do methods like Flash Attention help?**

Self-attention is $O(n^2 \cdot d)$ in time and $O(n^2)$ in memory for the attention matrix. Flash Attention doesn't reduce asymptotic complexity but avoids materializing the $n \times n$ matrix in slow GPU memory (HBM). By computing attention in tiles that fit in fast SRAM, it achieves 2-4× speedup and $O(n)$ memory.

**Q8: Explain the causal mask in decoder. Why is it necessary?**

The causal mask prevents each position from attending to future positions. During training, we process the entire target sequence at once (for parallelism), but each position should only "see" preceding tokens — otherwise the model just copies the answer. It's implemented by setting future positions to $-\infty$ before softmax.

**Q9: What is KV-Cache? Why is it critical for inference?**

During autoregressive generation, each new token needs to attend to all previous tokens. Without caching, we'd recompute K and V for all previous positions at every step — $O(n^2)$ total. KV-Cache stores computed K, V from previous steps so each new step only computes K, V for the new token — reducing to $O(n)$ per step.

**Q10: Compare Pre-Norm vs Post-Norm in Transformers.**

```
Post-Norm (original): x = LayerNorm(x + Sublayer(x))
Pre-Norm (modern):    x = x + Sublayer(LayerNorm(x))
```

Pre-Norm is more stable for deep models (gradient magnitude is more consistent across layers), allows training without warmup, and is used in GPT-2+. Post-Norm can sometimes achieve slightly better final performance but requires careful tuning.

### Coding Questions

**Q11: Implement a causal mask for a sequence of length n:**

```python
def create_causal_mask(seq_len):
    """Returns a lower-triangular mask (1 = attend, 0 = block)"""
    mask = torch.tril(torch.ones(seq_len, seq_len))
    return mask.unsqueeze(0).unsqueeze(0)  # Add batch and head dims

# Example
mask = create_causal_mask(5)
print(mask[0, 0])
# tensor([[1., 0., 0., 0., 0.],
#         [1., 1., 0., 0., 0.],
#         [1., 1., 1., 0., 0.],
#         [1., 1., 1., 1., 0.],
#         [1., 1., 1., 1., 1.]])
```

**Q12: Calculate the number of parameters in a Transformer encoder layer.**

For $d_{model} = 512$, $h = 8$, $d_{ff} = 2048$:

```
Multi-Head Attention:
  W_Q: 512 × 512 = 262,144
  W_K: 512 × 512 = 262,144
  W_V: 512 × 512 = 262,144
  W_O: 512 × 512 = 262,144
  Biases: 4 × 512 = 2,048
  Subtotal: 1,050,624

Feed-Forward:
  W_1: 512 × 2048 = 1,048,576
  b_1: 2048
  W_2: 2048 × 512 = 1,048,576
  b_2: 512
  Subtotal: 2,099,712

Layer Norms (×2):
  2 × (512 + 512) = 2,048

Total per layer: ~3.15M
Total model (6 layers): ~19M + embeddings
```

---

## Quick Reference

### Transformer Cheat Sheet

| Component | Purpose | Shape Transform |
|-----------|---------|----------------|
| Token Embedding | Word → vector | (B, L) → (B, L, d) |
| Positional Encoding | Add position info | (B, L, d) → (B, L, d) |
| Self-Attention | Token interactions | (B, L, d) → (B, L, d) |
| FFN | Non-linear transform | (B, L, d) → (B, L, d) |
| Layer Norm | Stabilize training | (B, L, d) → (B, L, d) |
| LM Head | Predict next token | (B, L, d) → (B, L, V) |

### Attention Variants

| Name | Formula | Used In |
|------|---------|---------|
| Scaled Dot-Product | $\text{softmax}(QK^T/\sqrt{d_k})V$ | All Transformers |
| Multi-Head | Parallel heads + concat + project | All Transformers |
| Multi-Query | Shared K,V across heads | PaLM, Falcon |
| Grouped-Query | K,V shared within groups | LLaMA 2, Mistral |
| Cross-Attention | Q from one seq, K/V from another | Encoder-Decoder |
| Causal/Masked | Lower triangular mask | Decoders/GPT |

### Key Formulas

| Formula | Meaning |
|---------|---------|
| $\text{Attn} = \text{softmax}(QK^T/\sqrt{d_k})V$ | Core attention |
| $PE_{(pos,2i)} = \sin(pos/10000^{2i/d})$ | Positional encoding |
| $\text{FFN}(x) = \text{GELU}(xW_1+b_1)W_2+b_2$ | Feed-forward |
| Params per layer: $\approx 12d^2$ | Quick estimate |
| FLOPs per layer: $\approx 2nd^2 + 2n^2d$ | Compute budget |

### Model Sizes (Reference)

| Model | Params | Layers | d_model | Heads | Context |
|-------|--------|--------|---------|-------|---------|
| BERT-base | 110M | 12 | 768 | 12 | 512 |
| BERT-large | 340M | 24 | 1024 | 16 | 512 |
| GPT-2 | 1.5B | 48 | 1600 | 25 | 1024 |
| GPT-3 | 175B | 96 | 12288 | 96 | 2048 |
| LLaMA-7B | 7B | 32 | 4096 | 32 | 4096 |
| LLaMA-70B | 70B | 80 | 8192 | 64 | 4096 |

### When to Use What

| Task | Architecture | Why |
|------|-------------|-----|
| Text classification | BERT / encoder-only | Bidirectional context |
| Text generation | GPT / decoder-only | Autoregressive |
| Translation | T5 / encoder-decoder | Different I/O lengths |
| Embeddings/Search | BERT + mean pooling | Dense representations |
| Few-shot learning | Large GPT | In-context learning |
| Fine-tuning on small data | BERT-base + classifier | Less prone to overfit |

### Training Hyperparameters (Rules of Thumb)

| Parameter | Typical Value | Notes |
|-----------|--------------|-------|
| Learning rate | 1e-4 to 5e-5 | Lower for fine-tuning |
| Warmup | 5-10% of steps | Critical for stability |
| Weight decay | 0.01-0.1 | AdamW |
| Batch size | 256-4096 tokens | Use gradient accumulation |
| Dropout | 0.1 | In attention and FFN |
| Adam β₁, β₂ | 0.9, 0.999 (or 0.95) | Standard |
| Gradient clip | 1.0 | Prevent spikes |

---

> **Pro Tips for Production:**
> 1. **Flash Attention** — Always use it (PyTorch 2.0+ has `torch.nn.functional.scaled_dot_product_attention`)
> 2. **KV-Cache** — Essential for fast inference. Without it, generation is $O(n^2)$ per token
> 3. **Mixed Precision (bf16)** — Standard for training. Better numerical stability than fp16
> 4. **Gradient Checkpointing** — Trade compute for memory (recompute activations during backward)
> 5. **Weight Tying** — Share input embedding with output projection (saves ~30% params for small models)
> 6. **Rotary Embeddings (RoPE)** — Modern standard for positional encoding (better extrapolation)
> 7. **Grouped-Query Attention** — Reduce KV-cache memory by 4-8× with minimal quality loss
> 8. **SwiGLU FFN** — Replace ReLU/GELU FFN with gated variant (LLaMA, Mistral use this)
