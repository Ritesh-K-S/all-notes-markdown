# Chapter 05: RNN and Sequence Models

## Table of Contents
- [What are Sequence Models?](#what-are-sequence-models)
- [Why Sequence Models Matter](#why-sequence-models-matter)
- [Vanilla RNN](#vanilla-rnn)
- [The Vanishing Gradient Problem](#the-vanishing-gradient-problem)
- [LSTM — Long Short-Term Memory](#lstm--long-short-term-memory)
- [GRU — Gated Recurrent Unit](#gru--gated-recurrent-unit)
- [Bidirectional RNNs](#bidirectional-rnns)
- [Sequence-to-Sequence Models](#sequence-to-sequence-models)
- [Code Examples](#code-examples)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What are Sequence Models?

Sequence models are neural networks designed to process **ordered data** where the position/order of elements matters. Unlike CNNs (spatial patterns) or MLPs (independent features), sequence models maintain a **memory** of what came before.

### Simple Analogy (For a 15-year-old)

Reading a sentence is sequential — the meaning of "bank" depends on whether you previously read "river" or "money." Your brain keeps track of context as you read word by word. An RNN does the same thing:
- It reads one element at a time
- It maintains a "memory" (hidden state) of what it has seen so far
- It uses that memory to interpret the current element

---

## Why Sequence Models Matter

### Types of Sequential Data

| Data Type | Example | Why Order Matters |
|-----------|---------|-------------------|
| Text | "The cat sat on the mat" | Word meaning depends on context |
| Time Series | Stock prices, weather | Future depends on past patterns |
| Audio/Speech | Waveforms, spectrograms | Phonemes form words over time |
| Video | Frame sequences | Actions unfold over time |
| DNA | ATCG sequences | Gene function depends on sequence |
| Music | Note sequences | Melody requires temporal structure |

### Input/Output Configurations

```
One-to-One:       One-to-Many:      Many-to-One:      Many-to-Many:     Many-to-Many:
(Standard NN)     (Image Caption)   (Sentiment)       (Translation)     (Named Entity)

  ┌─┐              ┌─┐ ┌─┐ ┌─┐      ┌─┐              ┌─┐ ┌─┐ ┌─┐       ┌─┐ ┌─┐ ┌─┐
  │O│              │O│ │O│ │O│      │O│              │O│ │O│ │O│       │O│ │O│ │O│
  └┬┘              └┬┘ └┬┘ └┬┘      └┬┘              └┬┘ └┬┘ └┬┘       └┬┘ └┬┘ └┬┘
  ┌┴┐              ┌┴───┴───┴┐      ┌┴───┴───┴┐      ┌┴───┴───┴┐       ┌┴┐ ┌┴┐ ┌┴┐
  │H│              │   RNN   │      │   RNN   │      │ Decoder │       │H│→│H│→│H│
  └┬┘              └────┬────┘      └┬───┬───┬┘      └────┬────┘       └┬┘ └┬┘ └┬┘
  ┌┴┐              ┌────┴────┐      ┌┴┐ ┌┴┐ ┌┴┐      ┌────┴────┐       ┌┴┐ ┌┴┐ ┌┴┐
  │I│              │    I    │      │I│ │I│ │I│      │ Encoder │       │I│ │I│ │I│
  └─┘              └─────────┘      └─┘ └─┘ └─┘      └┬───┬───┬┘       └─┘ └─┘ └─┘
                                                       ┌┴┐ ┌┴┐ ┌┴┐
                                                       │I│ │I│ │I│
                                                       └─┘ └─┘ └─┘
```

---

## Vanilla RNN

### Architecture

```
Unrolled RNN through time:

      y₁        y₂        y₃        y₄
      ↑         ↑         ↑         ↑
   ┌──┴──┐  ┌──┴──┐  ┌──┴──┐  ┌──┴──┐
   │     │  │     │  │     │  │     │
h₀→│  h₁ │→│  h₂ │→│  h₃ │→│  h₄ │→ h₄
   │     │  │     │  │     │  │     │
   └──┬──┘  └──┬──┘  └──┬──┘  └──┬──┘
      ↑         ↑         ↑         ↑
      x₁        x₂        x₃        x₄

Each box is the SAME network (shared weights!)
```

### Mathematical Formulation

At each time step $t$:

$$h_t = \tanh(W_{hh} \cdot h_{t-1} + W_{xh} \cdot x_t + b_h)$$

$$y_t = W_{hy} \cdot h_t + b_y$$

Where:
- $x_t \in \mathbb{R}^d$ — input at time $t$ (e.g., word embedding)
- $h_t \in \mathbb{R}^H$ — hidden state at time $t$ (the "memory")
- $y_t$ — output at time $t$
- $W_{xh} \in \mathbb{R}^{H \times d}$ — input-to-hidden weights
- $W_{hh} \in \mathbb{R}^{H \times H}$ — hidden-to-hidden weights (recurrence!)
- $W_{hy} \in \mathbb{R}^{O \times H}$ — hidden-to-output weights

> **Key Insight**: The same $W_{xh}$, $W_{hh}$, $W_{hy}$ are shared across ALL time steps. This is **parameter sharing** — similar to how CNN filters share weights across spatial positions.

### Why tanh?

- Output range is $[-1, 1]$ — prevents hidden state from exploding
- Zero-centered (unlike sigmoid) — better gradient flow
- Squashes large values — acts as a "memory regulator"

### Backpropagation Through Time (BPTT)

To train an RNN, we "unroll" it through time and apply standard backpropagation:

```
Loss = L₁ + L₂ + L₃ + ... + Lₜ

Gradient of L w.r.t. W_hh requires chain rule through ALL time steps:

∂L/∂W_hh = Σₜ ∂Lₜ/∂W_hh

∂Lₜ/∂h₁ = ∂Lₜ/∂hₜ · ∂hₜ/∂hₜ₋₁ · ... · ∂h₂/∂h₁
           = ∂Lₜ/∂hₜ · ∏ᵢ₌₂ᵗ ∂hᵢ/∂hᵢ₋₁

Each term ∂hᵢ/∂hᵢ₋₁ = W_hh^T · diag(1 - h²ᵢ)
```

---

## The Vanishing Gradient Problem

### Why Vanilla RNNs Fail on Long Sequences

The gradient at time step 1 from a loss at time step $T$ is:

$$\frac{\partial L_T}{\partial h_1} = \frac{\partial L_T}{\partial h_T} \prod_{t=2}^{T} \frac{\partial h_t}{\partial h_{t-1}}$$

Each factor $\frac{\partial h_t}{\partial h_{t-1}} = W_{hh}^T \cdot \text{diag}(\text{tanh}'(z_t))$

Since $\text{tanh}'(x) \leq 1$ and if $\|W_{hh}\| < 1$:

$$\left\|\prod_{t=2}^{T} \frac{\partial h_t}{\partial h_{t-1}}\right\| \leq \lambda_{max}^{T-1} \to 0 \text{ as } T \to \infty$$

**The gradient vanishes exponentially** — the network can't learn long-range dependencies!

### Visual Intuition

```
Sequence: "The cat, which was sitting on the warm mat in the living room, was ___"
                                                                            ↑
           ← Need to remember "cat" from 15 words ago to predict "sleeping"

Gradient signal strength over time:
Step 1: ████████████ (strong - recent)
Step 5: ██████       (weakening)
Step 10: ██          (very weak)
Step 15: ░           (essentially zero - can't learn!)
```

### Exploding Gradient (The Flip Side)

If $\|W_{hh}\| > 1$, gradients EXPLODE exponentially → NaN losses, unstable training.

**Solution for exploding**: Gradient clipping
```python
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=5.0)
```

**Solution for vanishing**: LSTM / GRU (architectural change needed!)

---

## LSTM — Long Short-Term Memory

### The Big Idea

LSTM solves the vanishing gradient by introducing a **cell state** ($C_t$) — a "highway" that allows information to flow unchanged across many time steps. **Gates** control what information to add or remove.

### Analogy

Think of LSTM as a conveyor belt in a factory:
- **Cell state** = the conveyor belt (information flows freely)
- **Forget gate** = worker who removes defective items
- **Input gate** = worker who adds new items
- **Output gate** = inspector who decides what to ship

### Architecture Diagram

```
                    ┌───────────────────────────────────────────┐
                    │              LSTM Cell                      │
                    │                                            │
   Cₜ₋₁ ─────────────→[×]────────────→[+]────────────────→ Cₜ
                    │    ↑              ↑                        │
                    │    fₜ            iₜ ⊙ C̃ₜ                  │
                    │    │              │   │                    │
                    │ ┌──┴──┐       ┌──┴──┐ ┌──┴──┐            │
                    │ │  σ  │       │  σ  │ │tanh │            │
                    │ └──┬──┘       └──┬──┘ └──┬──┘            │
                    │    │              │       │                │
   hₜ₋₁ ──┬────────────┼──────────────┼───────┼──→[×]──→ hₜ   │
           │        │   │              │       │     ↑          │
           │        │   └──────┬───────┘       │   ┌─┴──┐      │
           │        │          │               │   │ σ  │ ← oₜ │
           │        │    ┌─────┴─────┐         │   └─┬──┘      │
           └────────────→│ [hₜ₋₁;xₜ] │←───────────┘  │ tanh(Cₜ)│
                    │    └───────────┘             │   │         │
                    │                              │   └────→hₜ  │
   xₜ ─────────────┘                              │             │
                    └───────────────────────────────────────────┘

Gates:
  fₜ = Forget gate (what to remove from cell state)
  iₜ = Input gate (what new info to store)
  oₜ = Output gate (what to output)
  C̃ₜ = Candidate values (new information to potentially add)
```

### Mathematical Formulation

Given input $x_t$, previous hidden state $h_{t-1}$, and previous cell state $C_{t-1}$:

**Step 1: Forget Gate** — What to throw away from cell state
$$f_t = \sigma(W_f \cdot [h_{t-1}, x_t] + b_f)$$

**Step 2: Input Gate** — What new information to store
$$i_t = \sigma(W_i \cdot [h_{t-1}, x_t] + b_i)$$
$$\tilde{C}_t = \tanh(W_C \cdot [h_{t-1}, x_t] + b_C)$$

**Step 3: Update Cell State** — Apply forget + add new
$$C_t = f_t \odot C_{t-1} + i_t \odot \tilde{C}_t$$

**Step 4: Output Gate** — What to output from cell state
$$o_t = \sigma(W_o \cdot [h_{t-1}, x_t] + b_o)$$
$$h_t = o_t \odot \tanh(C_t)$$

Where:
- $\sigma$ = sigmoid function (outputs 0–1, acts as a "gate")
- $\odot$ = element-wise multiplication
- $[h_{t-1}, x_t]$ = concatenation of previous hidden state and current input

### Why LSTM Solves Vanishing Gradients

The cell state update: $C_t = f_t \odot C_{t-1} + i_t \odot \tilde{C}_t$

Gradient of $C_t$ w.r.t. $C_{t-1}$:
$$\frac{\partial C_t}{\partial C_{t-1}} = f_t$$

If $f_t \approx 1$ (forget gate open), the gradient flows **unchanged** — like a highway! The network can "choose" to remember information for hundreds of time steps.

```
Gradient flow comparison:

Vanilla RNN:  ∂hₜ/∂h₁ = W^(T-1) × tanh'(...) → vanishes/explodes

LSTM:         ∂Cₜ/∂C₁ = f₂ × f₃ × ... × fₜ  → stays near 1 if gates open
              (forget gates are typically 0.9-0.99)
```

### LSTM Parameter Count

For input size $d$ and hidden size $H$:
$$\text{Parameters} = 4 \times [(H + d) \times H + H] = 4 \times [H^2 + Hd + H]$$

The "4×" comes from: forget gate + input gate + cell candidate + output gate

**Example**: Input=100, Hidden=256:
$4 \times (256^2 + 256 \times 100 + 256) = 4 \times 91,392 = 365,568$ parameters

---

## GRU — Gated Recurrent Unit

### How GRU Simplifies LSTM

GRU merges the cell state and hidden state, and combines forget/input gates into a single "update gate."

```
LSTM has:                    GRU has:
- 3 gates (f, i, o)         - 2 gates (r, z)
- Cell state + Hidden state  - Only Hidden state
- More parameters            - ~25% fewer parameters
- Slightly better on long    - Faster training
  sequences                  - Often comparable accuracy
```

### Architecture

```
            ┌──────────────────────────────────┐
            │            GRU Cell                │
            │                                   │
hₜ₋₁ ─────────→[×(1-zₜ)]──→[+]───────────→ hₜ
            │       ↑          ↑                │
            │       │        [×zₜ]              │
            │       │          ↑                │
            │    ┌──┴──┐    ┌──┴──┐            │
            │    │ hₜ₋₁│    │ h̃ₜ  │            │
            │    └─────┘    └──┬──┘            │
            │                  │                │
            │             ┌────┴────┐           │
            │             │  tanh   │           │
            │             └────┬────┘           │
            │         [rₜ⊙hₜ₋₁; xₜ]           │
            │              ↑                    │
            │           ┌──┴──┐                 │
            │           │  σ  │ ← rₜ (reset)   │
            │           └─────┘                 │
            │              ↑                    │
            │        [hₜ₋₁; xₜ]                │
            └──────────────────────────────────┘
```

### Mathematical Formulation

**Update Gate** — How much of the past to keep:
$$z_t = \sigma(W_z \cdot [h_{t-1}, x_t] + b_z)$$

**Reset Gate** — How much of the past to forget when computing candidate:
$$r_t = \sigma(W_r \cdot [h_{t-1}, x_t] + b_r)$$

**Candidate Hidden State** — New information:
$$\tilde{h}_t = \tanh(W_h \cdot [r_t \odot h_{t-1}, x_t] + b_h)$$

**Final Hidden State** — Interpolation between old and new:
$$h_t = (1 - z_t) \odot h_{t-1} + z_t \odot \tilde{h}_t$$

> **Key Insight**: When $z_t \approx 0$, the hidden state is simply copied ($h_t \approx h_{t-1}$). When $z_t \approx 1$, it's completely replaced by the candidate.

### LSTM vs GRU Comparison

| Aspect | LSTM | GRU |
|--------|------|-----|
| Gates | 3 (forget, input, output) | 2 (reset, update) |
| States | Hidden + Cell | Hidden only |
| Parameters | 4(H² + Hd + H) | 3(H² + Hd + H) |
| Training speed | Slower | ~25% faster |
| Long sequences | Slightly better | Comparable |
| When to use | Long dependencies, larger datasets | Shorter sequences, limited compute |
| Default choice | ✅ Start here | Good alternative |

---

## Bidirectional RNNs

### Motivation

Standard RNNs only look **backwards** (past → present). But in many tasks, future context matters too!

**Example**: "He said he would ___ tomorrow" — knowing what comes after helps predict the blank.

### Architecture

```
Forward:   h₁→  → h₂→  → h₃→  → h₄→
            ↑       ↑       ↑       ↑
            x₁      x₂      x₃      x₄
            ↓       ↓       ↓       ↓
Backward:  h₁←  ← h₂←  ← h₃←  ← h₄←

Output at each step: hₜ = [h→ₜ ; h←ₜ]  (concatenation)
```

$$\overrightarrow{h_t} = \text{RNN}(\overrightarrow{h_{t-1}}, x_t)$$
$$\overleftarrow{h_t} = \text{RNN}(\overleftarrow{h_{t+1}}, x_t)$$
$$h_t = [\overrightarrow{h_t}; \overleftarrow{h_t}]$$

### When to Use Bidirectional

| Use Case | Bidirectional? | Reason |
|----------|---------------|--------|
| Text classification | ✅ Yes | Full sentence is available |
| Named entity recognition | ✅ Yes | Context on both sides helps |
| Machine translation (encoder) | ✅ Yes | Encoder sees full input |
| Language modeling | ❌ No | Can't peek at future! |
| Real-time speech | ❌ No | Future audio not available |
| Machine translation (decoder) | ❌ No | Generate left to right |

> **Rule of Thumb**: Use bidirectional when the **entire input sequence** is available at inference time.

---

## Sequence-to-Sequence Models

### The Encoder-Decoder Architecture

For tasks where input and output sequences have **different lengths** (translation, summarization):

```
Encoder:                              Decoder:
"I love cats"                         "J'aime les chats"

 ┌───┐  ┌───┐  ┌───┐                ┌───┐  ┌───┐  ┌───┐  ┌───┐
 │ h₁│→│ h₂│→│ h₃│──Context──→│ s₁│→│ s₂│→│ s₃│→│ s₄│
 └─┬─┘  └─┬─┘  └─┬─┘   Vector      └─┬─┘  └─┬─┘  └─┬─┘  └─┬─┘
   ↑       ↑       ↑                   ↑       ↑       ↑       ↑
  "I"   "love"  "cats"              <SOS>   "J'"  "aime"  "les"
                                       ↓       ↓       ↓       ↓
                                     "J'"   "aime"  "les"  "chats"
```

### The Bottleneck Problem

The entire input sequence is compressed into a **single fixed-size vector** (context vector). For long sentences, this is insufficient — like trying to memorize a whole book in one sentence.

**Solution**: Attention mechanism (covered in Chapter 06).

### Teacher Forcing

During training, we can feed the **correct** previous token (not the model's prediction) to the decoder. This:
- Speeds up convergence
- Prevents error accumulation

```
Without teacher forcing:          With teacher forcing:
Decoder input: model's outputs    Decoder input: ground truth
"<SOS>" → "Je" → "love" → ...   "<SOS>" → "J'" → "aime" → ...
          (wrong! error cascades)          (correct! stable training)
```

> **Warning**: Teacher forcing causes **exposure bias** — at inference, the model never sees its own mistakes during training. Mitigation: scheduled sampling (gradually reduce teacher forcing ratio).

### Beam Search (Decoding Strategy)

Greedy decoding picks the most probable word at each step. Beam search keeps top-$k$ candidates:

```
Beam width = 3:

Step 1:  "The" (0.5)    "A" (0.3)     "My" (0.2)
              ↓               ↓              ↓
Step 2:  "The cat" (0.3) "A dog" (0.2) "The man" (0.15) ...
              ↓               ↓              ↓
Step 3:  Pick top 3 overall → continue until <EOS>
```

---

## Code Examples

### Vanilla RNN from Scratch

```python
import torch
import torch.nn as nn
import numpy as np

class VanillaRNN(nn.Module):
    """RNN implemented manually to show the internals"""
    
    def __init__(self, input_size, hidden_size, output_size):
        super(VanillaRNN, self).__init__()
        self.hidden_size = hidden_size
        
        # Weight matrices
        self.W_xh = nn.Linear(input_size, hidden_size)    # Input → Hidden
        self.W_hh = nn.Linear(hidden_size, hidden_size)   # Hidden → Hidden
        self.W_hy = nn.Linear(hidden_size, output_size)   # Hidden → Output
    
    def forward(self, x, h_prev=None):
        """
        Args:
            x: Input sequence (batch, seq_len, input_size)
            h_prev: Initial hidden state (batch, hidden_size)
        Returns:
            outputs: All outputs (batch, seq_len, output_size)
            h: Final hidden state (batch, hidden_size)
        """
        batch_size, seq_len, _ = x.shape
        
        # Initialize hidden state to zeros if not provided
        if h_prev is None:
            h_prev = torch.zeros(batch_size, self.hidden_size, device=x.device)
        
        outputs = []
        h = h_prev
        
        for t in range(seq_len):
            # Core RNN computation at each time step
            h = torch.tanh(self.W_xh(x[:, t, :]) + self.W_hh(h))
            y = self.W_hy(h)
            outputs.append(y)
        
        outputs = torch.stack(outputs, dim=1)  # (batch, seq_len, output_size)
        return outputs, h

# Test
rnn = VanillaRNN(input_size=10, hidden_size=64, output_size=5)
x = torch.randn(32, 20, 10)  # batch=32, seq_len=20, features=10
outputs, final_h = rnn(x)
print(f"Outputs shape: {outputs.shape}")    # (32, 20, 5)
print(f"Final hidden: {final_h.shape}")     # (32, 64)
```

### LSTM for Sentiment Analysis

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.nn.utils.rnn import pack_padded_sequence, pad_packed_sequence

class SentimentLSTM(nn.Module):
    """LSTM-based sentiment analysis model"""
    
    def __init__(self, vocab_size, embed_dim, hidden_dim, output_dim,
                 n_layers=2, dropout=0.3, bidirectional=True, pad_idx=0):
        super(SentimentLSTM, self).__init__()
        
        # Embedding layer: converts word indices → dense vectors
        self.embedding = nn.Embedding(vocab_size, embed_dim, padding_idx=pad_idx)
        
        # LSTM layer(s)
        self.lstm = nn.LSTM(
            input_size=embed_dim,
            hidden_size=hidden_dim,
            num_layers=n_layers,
            batch_first=True,        # Input shape: (batch, seq, features)
            dropout=dropout if n_layers > 1 else 0,  # Dropout between layers
            bidirectional=bidirectional
        )
        
        # Classification head
        direction_factor = 2 if bidirectional else 1
        self.fc = nn.Sequential(
            nn.Dropout(dropout),
            nn.Linear(hidden_dim * direction_factor, 64),
            nn.ReLU(),
            nn.Dropout(dropout / 2),
            nn.Linear(64, output_dim)
        )
    
    def forward(self, text, lengths):
        """
        Args:
            text: (batch, max_seq_len) - padded token indices
            lengths: (batch,) - actual lengths before padding
        """
        # Embed tokens: (batch, seq_len) → (batch, seq_len, embed_dim)
        embedded = self.embedding(text)
        
        # Pack padded sequences for efficient processing
        # This tells LSTM to ignore padding tokens
        packed = pack_padded_sequence(embedded, lengths.cpu(),
                                      batch_first=True, enforce_sorted=False)
        
        # LSTM forward pass
        packed_output, (hidden, cell) = self.lstm(packed)
        # hidden shape: (n_layers * n_directions, batch, hidden_dim)
        
        # For bidirectional: concatenate final forward and backward hidden states
        # Take the last layer's hidden states
        if self.lstm.bidirectional:
            # hidden[-2] = last layer forward, hidden[-1] = last layer backward
            hidden_cat = torch.cat([hidden[-2], hidden[-1]], dim=1)
        else:
            hidden_cat = hidden[-1]
        
        # Classify
        output = self.fc(hidden_cat)
        return output

# ─── Usage Example ─────────────────────────────────────────
VOCAB_SIZE = 25000
EMBED_DIM = 300
HIDDEN_DIM = 256
OUTPUT_DIM = 1  # Binary sentiment (positive/negative)

model = SentimentLSTM(VOCAB_SIZE, EMBED_DIM, HIDDEN_DIM, OUTPUT_DIM)
print(f"Parameters: {sum(p.numel() for p in model.parameters()):,}")

# Sample input
batch_texts = torch.randint(0, VOCAB_SIZE, (16, 50))  # 16 sentences, max 50 tokens
batch_lengths = torch.randint(10, 50, (16,))           # Actual lengths

output = model(batch_texts, batch_lengths)
print(f"Output shape: {output.shape}")  # (16, 1)
```

### GRU for Time Series Forecasting

```python
import torch
import torch.nn as nn
import numpy as np

class TimeSeriesGRU(nn.Module):
    """GRU model for multi-step time series forecasting"""
    
    def __init__(self, input_features, hidden_size, num_layers, 
                 forecast_horizon, dropout=0.2):
        super(TimeSeriesGRU, self).__init__()
        
        self.gru = nn.GRU(
            input_size=input_features,
            hidden_size=hidden_size,
            num_layers=num_layers,
            batch_first=True,
            dropout=dropout if num_layers > 1 else 0
        )
        
        # Predict multiple future steps at once
        self.fc = nn.Sequential(
            nn.Linear(hidden_size, hidden_size // 2),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(hidden_size // 2, forecast_horizon)
        )
    
    def forward(self, x):
        """
        Args:
            x: (batch, lookback_window, input_features)
        Returns:
            predictions: (batch, forecast_horizon)
        """
        # Process sequence
        gru_out, hidden = self.gru(x)
        
        # Use only the last time step's output
        last_output = gru_out[:, -1, :]  # (batch, hidden_size)
        
        # Predict future values
        predictions = self.fc(last_output)
        return predictions

# ─── Data Preparation Example ──────────────────────────────
def create_sequences(data, lookback=30, horizon=7):
    """Create sliding window sequences for time series"""
    X, y = [], []
    for i in range(len(data) - lookback - horizon + 1):
        X.append(data[i:i+lookback])
        y.append(data[i+lookback:i+lookback+horizon])
    return np.array(X), np.array(y)

# Simulated stock price data
np.random.seed(42)
prices = np.cumsum(np.random.randn(1000) * 0.5 + 0.01)  # Random walk
prices = (prices - prices.mean()) / prices.std()  # Normalize!

X, y = create_sequences(prices.reshape(-1, 1), lookback=30, horizon=7)
print(f"Input sequences: {X.shape}")   # (963, 30, 1)
print(f"Target values: {y.shape}")     # (963, 7, 1)

# Convert to tensors
X_tensor = torch.FloatTensor(X)
y_tensor = torch.FloatTensor(y.squeeze())

# Model
model = TimeSeriesGRU(input_features=1, hidden_size=64, 
                      num_layers=2, forecast_horizon=7)

# Training
criterion = nn.MSELoss()
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)

for epoch in range(100):
    model.train()
    predictions = model(X_tensor)
    loss = criterion(predictions, y_tensor)
    
    optimizer.zero_grad()
    loss.backward()
    torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)  # Clip gradients!
    optimizer.step()
    
    if (epoch + 1) % 20 == 0:
        print(f"Epoch {epoch+1}, Loss: {loss.item():.6f}")
```

### Sequence-to-Sequence with Attention (Preview)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class Encoder(nn.Module):
    """Bidirectional LSTM encoder"""
    
    def __init__(self, vocab_size, embed_dim, hidden_dim, n_layers, dropout):
        super(Encoder, self).__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, n_layers,
                           batch_first=True, bidirectional=True, dropout=dropout)
        # Project bidirectional hidden to decoder size
        self.fc_hidden = nn.Linear(hidden_dim * 2, hidden_dim)
        self.fc_cell = nn.Linear(hidden_dim * 2, hidden_dim)
    
    def forward(self, src):
        # src: (batch, src_len)
        embedded = self.embedding(src)  # (batch, src_len, embed_dim)
        outputs, (hidden, cell) = self.lstm(embedded)
        # outputs: (batch, src_len, hidden*2) — all encoder outputs
        
        # Combine bidirectional hidden states for decoder initialization
        # hidden: (n_layers*2, batch, hidden) → (n_layers, batch, hidden)
        hidden = torch.cat([hidden[0::2], hidden[1::2]], dim=2)
        cell = torch.cat([cell[0::2], cell[1::2]], dim=2)
        hidden = torch.tanh(self.fc_hidden(hidden))
        cell = torch.tanh(self.fc_cell(cell))
        
        return outputs, hidden, cell


class Attention(nn.Module):
    """Bahdanau (additive) attention"""
    
    def __init__(self, encoder_dim, decoder_dim):
        super(Attention, self).__init__()
        self.attn = nn.Linear(encoder_dim + decoder_dim, decoder_dim)
        self.v = nn.Linear(decoder_dim, 1, bias=False)
    
    def forward(self, decoder_hidden, encoder_outputs):
        """
        Args:
            decoder_hidden: (batch, decoder_dim) - current decoder state
            encoder_outputs: (batch, src_len, encoder_dim) - all encoder states
        Returns:
            attention_weights: (batch, src_len) - soft alignment
        """
        src_len = encoder_outputs.shape[1]
        
        # Repeat decoder hidden for each source position
        decoder_hidden = decoder_hidden.unsqueeze(1).repeat(1, src_len, 1)
        
        # Compute attention scores
        energy = torch.tanh(self.attn(torch.cat([decoder_hidden, encoder_outputs], dim=2)))
        attention = self.v(energy).squeeze(2)  # (batch, src_len)
        
        return F.softmax(attention, dim=1)


class Decoder(nn.Module):
    """LSTM decoder with attention"""
    
    def __init__(self, vocab_size, embed_dim, hidden_dim, n_layers,
                 dropout, encoder_dim):
        super(Decoder, self).__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.attention = Attention(encoder_dim, hidden_dim)
        self.lstm = nn.LSTM(embed_dim + encoder_dim, hidden_dim,
                           n_layers, batch_first=True, dropout=dropout)
        self.fc_out = nn.Linear(hidden_dim + encoder_dim + embed_dim, vocab_size)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, input_token, hidden, cell, encoder_outputs):
        """One decoding step"""
        # input_token: (batch, 1)
        embedded = self.dropout(self.embedding(input_token))  # (batch, 1, embed)
        
        # Compute attention weights
        attn_weights = self.attention(hidden[-1], encoder_outputs)  # (batch, src_len)
        
        # Context vector (weighted sum of encoder outputs)
        context = torch.bmm(attn_weights.unsqueeze(1), encoder_outputs)  # (batch, 1, enc_dim)
        
        # Combine embedding + context as LSTM input
        lstm_input = torch.cat([embedded, context], dim=2)
        output, (hidden, cell) = self.lstm(lstm_input, (hidden, cell))
        
        # Predict next token
        prediction = self.fc_out(torch.cat([output, context, embedded], dim=2))
        return prediction.squeeze(1), hidden, cell, attn_weights


# ─── Full Seq2Seq Model ────────────────────────────────────
class Seq2Seq(nn.Module):
    def __init__(self, encoder, decoder, device):
        super(Seq2Seq, self).__init__()
        self.encoder = encoder
        self.decoder = decoder
        self.device = device
    
    def forward(self, src, trg, teacher_forcing_ratio=0.5):
        """
        Args:
            src: (batch, src_len) - source sequence
            trg: (batch, trg_len) - target sequence
            teacher_forcing_ratio: probability of using ground truth
        """
        batch_size, trg_len = trg.shape
        trg_vocab_size = self.decoder.fc_out.out_features
        
        outputs = torch.zeros(batch_size, trg_len, trg_vocab_size).to(self.device)
        
        # Encode
        encoder_outputs, hidden, cell = self.encoder(src)
        
        # First decoder input is <SOS> token
        input_token = trg[:, 0:1]  # (batch, 1)
        
        for t in range(1, trg_len):
            prediction, hidden, cell, _ = self.decoder(
                input_token, hidden, cell, encoder_outputs
            )
            outputs[:, t] = prediction
            
            # Teacher forcing: use ground truth or model's prediction
            teacher_force = torch.rand(1).item() < teacher_forcing_ratio
            top1 = prediction.argmax(1, keepdim=True)
            input_token = trg[:, t:t+1] if teacher_force else top1
        
        return outputs
```

### Practical: Character-Level Language Model

```python
import torch
import torch.nn as nn
import string

class CharRNN(nn.Module):
    """Character-level language model using LSTM"""
    
    def __init__(self, vocab_size, embed_dim=64, hidden_dim=256, n_layers=2):
        super(CharRNN, self).__init__()
        self.hidden_dim = hidden_dim
        self.n_layers = n_layers
        
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, n_layers,
                           batch_first=True, dropout=0.2)
        self.fc = nn.Linear(hidden_dim, vocab_size)
    
    def forward(self, x, hidden=None):
        embedded = self.embedding(x)
        output, hidden = self.lstm(embedded, hidden)
        logits = self.fc(output)
        return logits, hidden
    
    def generate(self, start_char, char_to_idx, idx_to_char, 
                 length=200, temperature=0.8):
        """Generate text character by character"""
        self.eval()
        hidden = None
        input_idx = torch.tensor([[char_to_idx[start_char]]])
        result = [start_char]
        
        with torch.no_grad():
            for _ in range(length):
                logits, hidden = self(input_idx, hidden)
                
                # Temperature controls randomness
                # Low temp (0.2) = conservative, high temp (1.5) = creative
                logits = logits[:, -1, :] / temperature
                probs = torch.softmax(logits, dim=-1)
                
                # Sample from distribution (not greedy!)
                next_idx = torch.multinomial(probs, num_samples=1)
                result.append(idx_to_char[next_idx.item()])
                input_idx = next_idx
        
        return ''.join(result)

# ─── Training Example ──────────────────────────────────────
# Sample text corpus
text = "To be or not to be, that is the question. " * 100

# Build vocabulary
chars = sorted(set(text))
char_to_idx = {c: i for i, c in enumerate(chars)}
idx_to_char = {i: c for c, i in char_to_idx.items()}
vocab_size = len(chars)

# Create sequences
seq_length = 50
encoded = [char_to_idx[c] for c in text]

# Prepare training data
X_data, y_data = [], []
for i in range(0, len(encoded) - seq_length):
    X_data.append(encoded[i:i+seq_length])
    y_data.append(encoded[i+1:i+seq_length+1])  # Shifted by 1

X = torch.tensor(X_data[:1000])  # Limit for demo
y = torch.tensor(y_data[:1000])

# Train
model = CharRNN(vocab_size)
criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters(), lr=0.003)

for epoch in range(50):
    model.train()
    logits, _ = model(X)
    loss = criterion(logits.view(-1, vocab_size), y.view(-1))
    
    optimizer.zero_grad()
    loss.backward()
    torch.nn.utils.clip_grad_norm_(model.parameters(), 5.0)
    optimizer.step()
    
    if (epoch + 1) % 10 == 0:
        print(f"Epoch {epoch+1}, Loss: {loss.item():.4f}")
        # Generate sample
        sample = model.generate('T', char_to_idx, idx_to_char, length=50)
        print(f"  Generated: {sample}\n")
```

---

## Common Mistakes

### 1. Not Handling Variable-Length Sequences Properly

```python
# ❌ WRONG: Padding tokens contribute to the output
outputs, (h, c) = lstm(padded_sequences)
# The hidden state includes information from padding!

# ✅ CORRECT: Use pack_padded_sequence
from torch.nn.utils.rnn import pack_padded_sequence, pad_packed_sequence

packed = pack_padded_sequence(embedded, lengths.cpu(), 
                              batch_first=True, enforce_sorted=False)
packed_out, (h, c) = lstm(packed)
outputs, _ = pad_packed_sequence(packed_out, batch_first=True)
```

### 2. Forgetting to Detach Hidden States (Truncated BPTT)

```python
# ❌ WRONG: Hidden state carries computation graph from all previous batches
# → Memory grows unbounded!
hidden = model.init_hidden()
for batch in data_loader:
    output, hidden = model(batch, hidden)
    loss.backward()

# ✅ CORRECT: Detach hidden state between batches
hidden = model.init_hidden()
for batch in data_loader:
    hidden = tuple(h.detach() for h in hidden)  # Break computation graph
    output, hidden = model(batch, hidden)
    loss.backward()
```

### 3. Wrong Hidden State Dimensions

```python
# LSTM hidden state shape: (num_layers * num_directions, batch, hidden_size)
# NOT (batch, hidden_size)!

# For 2-layer bidirectional LSTM with batch=16, hidden=256:
# hidden shape: (2*2, 16, 256) = (4, 16, 256)

# To get the final hidden state for classification:
# Forward last layer: hidden[2]  (index = num_layers * 1)
# Backward last layer: hidden[3] (index = num_layers * 2 - 1)
final_hidden = torch.cat([hidden[-2], hidden[-1]], dim=1)  # (batch, hidden*2)
```

### 4. Not Clipping Gradients

```python
# ❌ WRONG: Gradients can explode in RNNs
loss.backward()
optimizer.step()

# ✅ CORRECT: Always clip gradients for RNN training
loss.backward()
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=5.0)
optimizer.step()
```

### 5. Using Bidirectional for Autoregressive Tasks

```python
# ❌ WRONG: Using bidirectional for next-word prediction (data leakage!)
model = nn.LSTM(input_size, hidden_size, bidirectional=True)
# The backward pass can "see" future tokens!

# ✅ CORRECT: Use unidirectional for generation/prediction tasks
model = nn.LSTM(input_size, hidden_size, bidirectional=False)
```

### 6. Incorrect Input Shape

```python
# PyTorch LSTM expects: (batch, seq_len, features) with batch_first=True
#                    OR: (seq_len, batch, features) with batch_first=False

# ❌ WRONG: Passing (batch, features) without seq dimension
x = torch.randn(32, 100)
lstm(x)  # Error!

# ✅ CORRECT: Add sequence dimension
x = torch.randn(32, 1, 100)  # (batch, seq_len=1, features)
lstm(x)
```

---

## Interview Questions

### Conceptual Questions

**Q1: Explain the vanishing gradient problem in RNNs and how LSTM solves it.**

In vanilla RNNs, gradients are multiplied by $W_{hh}$ at each time step during BPTT. If $\|W_{hh}\| < 1$, gradients vanish exponentially. LSTM introduces a cell state with additive updates: $C_t = f_t \odot C_{t-1} + i_t \odot \tilde{C}_t$. The gradient through the cell state is $\prod f_t$ — since forget gates are typically close to 1, gradients flow undiminished over long sequences.

**Q2: What's the difference between LSTM and GRU? When would you choose one over the other?**

| Aspect | LSTM | GRU |
|--------|------|-----|
| Gates | 3 (forget, input, output) | 2 (reset, update) |
| Memory | Separate cell state | Only hidden state |
| Params | ~33% more than GRU | Fewer |

Choose LSTM for: longer sequences, larger datasets, when accuracy is critical.
Choose GRU for: shorter sequences, limited compute, faster experimentation.

**Q3: Explain teacher forcing. What's its advantage and disadvantage?**

Teacher forcing feeds ground-truth tokens as decoder input during training instead of the model's own predictions.
- **Advantage**: Faster, more stable training
- **Disadvantage**: Exposure bias — model never sees its own errors during training, so at inference it can't recover from mistakes

**Q4: What is the "bottleneck problem" in Seq2Seq models?**

The encoder compresses the entire input sequence into a single fixed-size vector. For long sequences, this vector can't capture all information → performance degrades. Solution: Attention mechanism (lets decoder look at all encoder states, not just the last one).

**Q5: How does gradient clipping work? Why is it necessary for RNNs?**

Gradient clipping scales the gradient if its norm exceeds a threshold:
```
if ||g|| > max_norm:
    g = g × (max_norm / ||g||)
```
It's necessary because RNN gradients can explode due to repeated multiplication by $W_{hh}$. It prevents NaN losses while preserving gradient direction.

**Q6: What's the difference between stateful and stateless RNNs?**

- **Stateless**: Hidden state reset to zero for each batch (standard behavior)
- **Stateful**: Hidden state from the end of one batch is passed as initial state to the next batch. Used when sequences span multiple batches (e.g., very long documents).

**Q7: How many parameters does an LSTM layer have?**

For input size $d$ and hidden size $H$:
$$4 \times [(H + d) \times H + H] = 4H^2 + 4Hd + 4H$$

The "4" accounts for the four weight matrices (forget, input, cell candidate, output gates).

**Q8: Explain Bidirectional RNNs. When can't you use them?**

Bidirectional RNNs process sequences in both forward and backward directions, capturing both past and future context. You **cannot** use them for:
- Language generation (next token prediction)
- Real-time streaming applications
- Any task where future information isn't available at inference

### Coding Questions

**Q9: Implement a simple RNN cell in pure Python/NumPy:**

```python
import numpy as np

def rnn_cell(x_t, h_prev, W_xh, W_hh, b_h):
    """Single RNN cell computation"""
    h_t = np.tanh(W_xh @ x_t + W_hh @ h_prev + b_h)
    return h_t

# Initialize
hidden_size, input_size = 64, 32
W_xh = np.random.randn(hidden_size, input_size) * 0.01
W_hh = np.random.randn(hidden_size, hidden_size) * 0.01
b_h = np.zeros(hidden_size)

# Process sequence
sequence = np.random.randn(10, input_size)  # 10 time steps
h = np.zeros(hidden_size)

for t in range(len(sequence)):
    h = rnn_cell(sequence[t], h, W_xh, W_hh, b_h)
    print(f"Step {t}: h norm = {np.linalg.norm(h):.4f}")
```

**Q10: What's wrong with this training loop?**
```python
for epoch in range(100):
    for batch in dataloader:
        output, hidden = model(batch, hidden)
        loss = criterion(output, targets)
        loss.backward()
        optimizer.step()
```

Answer: Multiple issues:
1. `optimizer.zero_grad()` is missing → gradients accumulate
2. `hidden` is not detached → computation graph grows forever → OOM
3. No gradient clipping → potential gradient explosion

---

## Quick Reference

### RNN Family Cheat Sheet

| Model | Year | Hidden State | Gates | Key Strength |
|-------|------|-------------|-------|-------------|
| Vanilla RNN | 1986 | $h_t = \tanh(Wx + Wh + b)$ | 0 | Simple, fast |
| LSTM | 1997 | $h_t$ + $C_t$ (cell) | 3 (f, i, o) | Long-range dependencies |
| GRU | 2014 | $h_t$ only | 2 (r, z) | Fast LSTM alternative |
| Bi-LSTM | — | Forward + Backward | 3×2 | Full context |

### Dimensions Reference (PyTorch)

```python
# LSTM / GRU input/output shapes (batch_first=True):
# Input:   (batch, seq_len, input_size)
# Output:  (batch, seq_len, hidden_size * num_directions)
# Hidden:  (num_layers * num_directions, batch, hidden_size)
# Cell:    (num_layers * num_directions, batch, hidden_size) [LSTM only]
```

### Parameter Count Formulas

| Model | Parameters per Layer |
|-------|---------------------|
| Vanilla RNN | $(H + d) \times H + H$ |
| LSTM | $4 \times [(H + d) \times H + H]$ |
| GRU | $3 \times [(H + d) \times H + H]$ |

### When to Use What

| Task | Architecture | Notes |
|------|-------------|-------|
| Sentiment Analysis | Bi-LSTM + attention | Both directions matter |
| Machine Translation | Seq2Seq + attention | Different length I/O |
| Language Modeling | Unidirectional LSTM | Can't see future |
| Time Series | GRU / LSTM | Consider Transformers too |
| Named Entity Recognition | Bi-LSTM + CRF | CRF ensures valid tag sequences |
| Speech Recognition | Deep Bi-LSTM | CTC loss for alignment |

### Key Hyperparameters

| Parameter | Typical Range | Notes |
|-----------|--------------|-------|
| Hidden size | 128–512 | Larger = more capacity, slower |
| Num layers | 1–4 | Diminishing returns after 3 |
| Dropout | 0.2–0.5 | Between layers, not within |
| Sequence length | 50–500 | Longer = more memory needed |
| Learning rate | 1e-3 to 5e-4 | Use Adam or AdamW |
| Gradient clip | 1.0–5.0 | Always clip for RNNs! |

---

> **The State of RNNs in 2024+:**
> - Transformers have largely **replaced** RNNs for NLP tasks
> - RNNs still useful for: streaming data, low-latency inference, small models
> - New architectures (Mamba, RWKV) combine RNN efficiency with Transformer quality
> - Understanding RNNs/LSTMs is still essential for interviews and foundational knowledge
> - Many production systems still use LSTMs (especially in time series and edge devices)
