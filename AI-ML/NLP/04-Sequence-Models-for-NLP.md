# Chapter 04: Sequence Models for NLP

## Table of Contents
- [Introduction to Sequence Models](#introduction-to-sequence-models)
- [Recurrent Neural Networks (RNN)](#recurrent-neural-networks-rnn)
- [Long Short-Term Memory (LSTM)](#long-short-term-memory-lstm)
- [Gated Recurrent Unit (GRU)](#gated-recurrent-unit-gru)
- [Bidirectional RNNs](#bidirectional-rnns)
- [Language Modeling](#language-modeling)
- [Sequence-to-Sequence (Seq2Seq)](#sequence-to-sequence-seq2seq)
- [Practical Implementations](#practical-implementations)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Introduction to Sequence Models

### What It Is
Sequence models are neural networks designed to handle **ordered data** — where the position and order of elements matter. Think of reading a sentence: "Dog bites man" means something completely different from "Man bites dog." Sequence models understand this ordering.

### Why It Matters
- **Text is inherently sequential** — words derive meaning from context and position
- **Powers**: machine translation, chatbots, autocomplete, speech recognition, sentiment analysis
- **Before transformers dominated**, RNNs/LSTMs were the backbone of all NLP systems
- **Still used** in edge devices, real-time systems, and resource-constrained environments

### The Core Problem
Traditional neural networks (feedforward) treat each input independently. They can't "remember" what came before. For NLP, this is catastrophic:

```
Input: "The cat sat on the ___"
Feedforward NN: No idea what came before → random guess
Sequence Model: Remembers "cat", "sat", "on" → predicts "mat" or "floor"
```

---

## Recurrent Neural Networks (RNN)

### What It Is
An RNN is a neural network with a "memory loop." At each time step, it takes the current input AND its previous output (hidden state) to produce a new output. It's like reading a book — you remember what you've read so far.

### Why It Matters
- First architecture that could handle variable-length sequences
- Foundation for LSTM, GRU, and conceptually even Transformers
- Still useful for understanding sequential processing

### How It Works

#### The Core Intuition
Imagine you're watching a movie. At each scene, your understanding depends on:
1. What's happening NOW (current input)
2. What you remember from BEFORE (hidden state)

An RNN works exactly like this.

#### Architecture Diagram

```
        Output(t)
           ↑
    ┌──────────────┐
    │   Hidden     │
    │   State (t)  │──────────────┐
    └──────────────┘              │
           ↑                      │
    ┌──────┴──────┐               │ (loop back)
    │   Combine   │               │
    └──────┬──────┘               │
      ↑         ↑                 │
  Input(t)   Hidden(t-1) ←───────┘
```

#### Unrolled Through Time

```
  h₀ ──→ [RNN Cell] ──→ h₁ ──→ [RNN Cell] ──→ h₂ ──→ [RNN Cell] ──→ h₃
              ↑                      ↑                      ↑
             x₁                     x₂                     x₃
          ("The")               ("cat")                ("sat")
```

#### Mathematical Formulation

At each time step $t$:

$$h_t = \tanh(W_{hh} \cdot h_{t-1} + W_{xh} \cdot x_t + b_h)$$

$$y_t = W_{hy} \cdot h_t + b_y$$

Where:
- $h_t$ = hidden state at time $t$ (the "memory")
- $x_t$ = input at time $t$
- $W_{hh}$ = hidden-to-hidden weight matrix (how to use memory)
- $W_{xh}$ = input-to-hidden weight matrix (how to process input)
- $W_{hy}$ = hidden-to-output weight matrix
- $b_h, b_y$ = bias terms
- $\tanh$ = activation function (squashes values between -1 and 1)

### Code Example: Basic RNN from Scratch

```python
import numpy as np

class SimpleRNN:
    """
    A basic RNN implementation from scratch to understand the mechanics.
    """
    def __init__(self, input_size, hidden_size, output_size):
        # Initialize weight matrices with small random values
        # Xavier initialization for better training
        self.Wxh = np.random.randn(hidden_size, input_size) * 0.01   # input → hidden
        self.Whh = np.random.randn(hidden_size, hidden_size) * 0.01  # hidden → hidden
        self.Why = np.random.randn(output_size, hidden_size) * 0.01  # hidden → output
        self.bh = np.zeros((hidden_size, 1))   # hidden bias
        self.by = np.zeros((output_size, 1))   # output bias
        self.hidden_size = hidden_size
    
    def forward(self, inputs, h_prev):
        """
        Process a sequence of inputs one by one.
        
        Args:
            inputs: list of input vectors (one per time step)
            h_prev: initial hidden state (usually zeros)
        
        Returns:
            outputs: list of output vectors
            hidden_states: list of hidden states (useful for backprop)
        """
        outputs = []
        hidden_states = [h_prev]
        
        for x in inputs:
            # Core RNN equation: combine input and previous hidden state
            h_next = np.tanh(
                self.Wxh @ x +          # process current input
                self.Whh @ h_prev +     # incorporate memory
                self.bh                  # add bias
            )
            
            # Compute output from hidden state
            y = self.Why @ h_next + self.by
            
            outputs.append(y)
            hidden_states.append(h_next)
            h_prev = h_next  # update memory for next step
        
        return outputs, hidden_states

# Demo: Character-level prediction
rnn = SimpleRNN(input_size=4, hidden_size=8, output_size=4)
h0 = np.zeros((8, 1))  # initial hidden state is zeros

# Simulate 3 time steps with random inputs
inputs = [np.random.randn(4, 1) for _ in range(3)]
outputs, states = rnn.forward(inputs, h0)

print(f"Number of outputs: {len(outputs)}")
print(f"Output shape at each step: {outputs[0].shape}")
print(f"Hidden state shape: {states[1].shape}")
# Output:
# Number of outputs: 3
# Output shape at each step: (4, 1)
# Hidden state shape: (8, 1)
```

### Code Example: RNN with PyTorch

```python
import torch
import torch.nn as nn

class TextRNN(nn.Module):
    """
    RNN for text classification (e.g., sentiment analysis).
    """
    def __init__(self, vocab_size, embed_dim, hidden_dim, output_dim, 
                 n_layers=1, dropout=0.5):
        super().__init__()
        
        # Embedding layer: converts word indices → dense vectors
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        
        # RNN layer: processes sequence
        self.rnn = nn.RNN(
            input_size=embed_dim,
            hidden_size=hidden_dim,
            num_layers=n_layers,      # stacked RNN layers
            dropout=dropout if n_layers > 1 else 0,
            batch_first=True          # input shape: (batch, seq, features)
        )
        
        # Fully connected layer: hidden state → output classes
        self.fc = nn.Linear(hidden_dim, output_dim)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, text):
        """
        Args:
            text: tensor of word indices, shape (batch_size, seq_length)
        Returns:
            predictions: shape (batch_size, output_dim)
        """
        # text shape: (batch_size, seq_length)
        embedded = self.dropout(self.embedding(text))
        # embedded shape: (batch_size, seq_length, embed_dim)
        
        output, hidden = self.rnn(embedded)
        # output shape: (batch_size, seq_length, hidden_dim) — all hidden states
        # hidden shape: (n_layers, batch_size, hidden_dim) — final hidden state
        
        # Use the last hidden state for classification
        hidden = self.dropout(hidden[-1])  # shape: (batch_size, hidden_dim)
        
        return self.fc(hidden)

# Usage
model = TextRNN(
    vocab_size=10000,
    embed_dim=100,
    hidden_dim=256,
    output_dim=2,       # binary classification
    n_layers=2
)

# Simulate input: batch of 32 sentences, each 50 words
fake_input = torch.randint(0, 10000, (32, 50))
output = model(fake_input)
print(f"Output shape: {output.shape}")  # (32, 2)
```

### The Vanishing Gradient Problem

> **Critical Concept**: RNNs suffer from vanishing gradients — as sequences get longer, gradients during backpropagation shrink exponentially, making it impossible to learn long-range dependencies.

```
Sentence: "The cat, which was sitting on the mat in the living room, was happy."
                                                                    ↑
RNN trying to connect "cat" ────────────────────────────────────── "was"
         (subject)                    many steps away              (verb)

Gradient after 10+ steps: 0.9^10 = 0.35 (still ok)
Gradient after 50+ steps: 0.9^50 = 0.005 (basically zero!)
```

**Why it happens mathematically:**

During backpropagation through time (BPTT):

$$\frac{\partial L}{\partial h_0} = \frac{\partial L}{\partial h_T} \cdot \prod_{t=1}^{T} \frac{\partial h_t}{\partial h_{t-1}}$$

Each $\frac{\partial h_t}{\partial h_{t-1}}$ involves multiplying by $W_{hh}$ and the derivative of $\tanh$ (which is ≤ 1). After many multiplications, this product vanishes.

**This is exactly why LSTM was invented.**

---

## Long Short-Term Memory (LSTM)

### What It Is
LSTM is an RNN variant with a sophisticated gating mechanism that controls what to **remember**, what to **forget**, and what to **output**. Think of it as an RNN with a notepad — it can write notes, erase notes, and choose which notes to read.

### Why It Matters
- **Solves the vanishing gradient problem** through a cell state that acts as a "highway"
- **Can learn dependencies spanning 100+ time steps**
- **Industry workhorse** before Transformers: Google Translate (2016), Siri, Alexa
- **Still preferred** for real-time/streaming applications and small datasets

### How It Works

#### The Analogy: A Smart Secretary
Imagine a secretary managing information flow:
1. **Forget Gate**: "Should I throw away any old notes?" (delete irrelevant info)
2. **Input Gate**: "What new info should I write down?" (add relevant new info)
3. **Cell State**: "My notepad" (long-term memory)
4. **Output Gate**: "What should I tell my boss right now?" (relevant output)

#### Architecture Diagram

```
                          Cell State (Cₜ₋₁) ─────────────────────────→ Cell State (Cₜ)
                               │                    ↑           ↑              │
                               │              ×(forget)    +(new info)         │
                               │                    │           │              │
                               ↓                    │           │              ↓
    ┌─────────────────────────────────────────────────────────────────────────────┐
    │                                                                             │
    │   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐            │
    │   │  Forget  │    │  Input   │    │   Cell   │    │  Output  │            │
    │   │   Gate   │    │   Gate   │    │ Candidate│    │   Gate   │            │
    │   │  σ(fₜ)   │    │  σ(iₜ)   │    │ tanh(c̃ₜ) │    │  σ(oₜ)   │            │
    │   └──────────┘    └──────────┘    └──────────┘    └──────────┘            │
    │        ↑                ↑               ↑               ↑                  │
    │        └────────────────┴───────────────┴───────────────┘                  │
    │                              │                                              │
    │                    [hₜ₋₁ ; xₜ] (concatenated)                              │
    │                                                                             │
    └─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ↓
                                   Output (hₜ)
```

#### Mathematical Formulation

**Step 1: Forget Gate** — What old information to discard

$$f_t = \sigma(W_f \cdot [h_{t-1}, x_t] + b_f)$$

Output: values between 0 (forget completely) and 1 (keep everything)

**Step 2: Input Gate** — What new information to store

$$i_t = \sigma(W_i \cdot [h_{t-1}, x_t] + b_i)$$

$$\tilde{C}_t = \tanh(W_C \cdot [h_{t-1}, x_t] + b_C)$$

**Step 3: Update Cell State** — The key equation

$$C_t = f_t \odot C_{t-1} + i_t \odot \tilde{C}_t$$

> **This is why LSTM solves vanishing gradients!** The gradient flows through $C_t$ via addition (not multiplication), creating a "gradient highway."

**Step 4: Output Gate** — What to output

$$o_t = \sigma(W_o \cdot [h_{t-1}, x_t] + b_o)$$

$$h_t = o_t \odot \tanh(C_t)$$

Where:
- $\sigma$ = sigmoid function (outputs 0-1, acts as a "gate")
- $\odot$ = element-wise multiplication
- $[h_{t-1}, x_t]$ = concatenation of previous hidden state and current input

### Code Example: LSTM from Scratch

```python
import numpy as np

def sigmoid(x):
    """Sigmoid activation — squashes to (0, 1) — perfect for gates."""
    return 1 / (1 + np.exp(-np.clip(x, -500, 500)))  # clip for numerical stability

def tanh(x):
    """Tanh activation — squashes to (-1, 1) — for cell state candidates."""
    return np.tanh(x)

class LSTMCell:
    """
    Single LSTM cell implementation showing exact gate mechanics.
    """
    def __init__(self, input_size, hidden_size):
        self.hidden_size = hidden_size
        concat_size = input_size + hidden_size
        
        # All four gates share similar weight structure
        # In practice, they're often combined into one big matrix for efficiency
        self.Wf = np.random.randn(hidden_size, concat_size) * 0.01  # forget gate
        self.Wi = np.random.randn(hidden_size, concat_size) * 0.01  # input gate
        self.Wc = np.random.randn(hidden_size, concat_size) * 0.01  # cell candidate
        self.Wo = np.random.randn(hidden_size, concat_size) * 0.01  # output gate
        
        # Bias: forget gate bias initialized to 1 (important trick!)
        self.bf = np.ones((hidden_size, 1))   # Start by remembering everything
        self.bi = np.zeros((hidden_size, 1))
        self.bc = np.zeros((hidden_size, 1))
        self.bo = np.zeros((hidden_size, 1))
    
    def forward(self, x, h_prev, c_prev):
        """
        Single step of LSTM computation.
        
        Args:
            x: current input (input_size, 1)
            h_prev: previous hidden state (hidden_size, 1)
            c_prev: previous cell state (hidden_size, 1)
        
        Returns:
            h_next: new hidden state
            c_next: new cell state
        """
        # Concatenate input and previous hidden state
        concat = np.vstack([h_prev, x])  # shape: (input_size + hidden_size, 1)
        
        # Gate computations
        f = sigmoid(self.Wf @ concat + self.bf)     # Forget gate
        i = sigmoid(self.Wi @ concat + self.bi)     # Input gate
        c_tilde = tanh(self.Wc @ concat + self.bc)  # Cell candidate
        o = sigmoid(self.Wo @ concat + self.bo)     # Output gate
        
        # Update cell state: forget old + add new
        c_next = f * c_prev + i * c_tilde
        
        # Compute hidden state (output)
        h_next = o * tanh(c_next)
        
        return h_next, c_next

# Demo
cell = LSTMCell(input_size=10, hidden_size=20)
h = np.zeros((20, 1))
c = np.zeros((20, 1))

# Process 5 time steps
for t in range(5):
    x = np.random.randn(10, 1)
    h, c = cell.forward(x, h, c)
    print(f"Step {t}: h_mean={h.mean():.4f}, c_mean={c.mean():.4f}")
```

### Code Example: LSTM for Sentiment Analysis (PyTorch)

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset

class SentimentLSTM(nn.Module):
    """
    Production-ready LSTM for binary sentiment classification.
    """
    def __init__(self, vocab_size, embed_dim, hidden_dim, output_dim,
                 n_layers, bidirectional, dropout, pad_idx):
        super().__init__()
        
        self.embedding = nn.Embedding(
            vocab_size, embed_dim, padding_idx=pad_idx
        )
        
        self.lstm = nn.LSTM(
            input_size=embed_dim,
            hidden_size=hidden_dim,
            num_layers=n_layers,
            bidirectional=bidirectional,  # process forward AND backward
            dropout=dropout if n_layers > 1 else 0,
            batch_first=True
        )
        
        # If bidirectional, hidden_dim is doubled
        fc_input_dim = hidden_dim * 2 if bidirectional else hidden_dim
        
        self.fc = nn.Linear(fc_input_dim, output_dim)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, text, text_lengths):
        """
        Args:
            text: (batch_size, max_seq_len) padded sequences
            text_lengths: (batch_size,) actual lengths before padding
        """
        embedded = self.dropout(self.embedding(text))
        
        # Pack padded sequences for efficient computation
        # This tells LSTM to ignore padding tokens
        packed = nn.utils.rnn.pack_padded_sequence(
            embedded, text_lengths.cpu(), 
            batch_first=True, enforce_sorted=False
        )
        
        packed_output, (hidden, cell) = self.lstm(packed)
        
        # For bidirectional: concatenate final forward and backward hidden states
        # hidden shape: (n_layers * n_directions, batch, hidden_dim)
        if self.lstm.bidirectional:
            hidden = torch.cat([hidden[-2], hidden[-1]], dim=1)
        else:
            hidden = hidden[-1]
        
        hidden = self.dropout(hidden)
        return self.fc(hidden)

# Initialize model
model = SentimentLSTM(
    vocab_size=25000,
    embed_dim=300,
    hidden_dim=256,
    output_dim=1,        # binary: positive/negative
    n_layers=2,
    bidirectional=True,
    dropout=0.5,
    pad_idx=0
)

# Count parameters
total_params = sum(p.numel() for p in model.parameters())
print(f"Total parameters: {total_params:,}")
# Output: Total parameters: ~8.4M (depending on exact config)

# Training loop skeleton
optimizer = optim.Adam(model.parameters(), lr=1e-3)
criterion = nn.BCEWithLogitsLoss()

def train_epoch(model, dataloader, optimizer, criterion):
    model.train()
    epoch_loss = 0
    
    for batch_text, batch_lengths, batch_labels in dataloader:
        optimizer.zero_grad()
        predictions = model(batch_text, batch_lengths).squeeze(1)
        loss = criterion(predictions, batch_labels.float())
        loss.backward()
        
        # Gradient clipping — ESSENTIAL for RNNs/LSTMs
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        
        optimizer.step()
        epoch_loss += loss.item()
    
    return epoch_loss / len(dataloader)
```

> **Pro Tip**: Always use gradient clipping with RNNs/LSTMs. Without it, exploding gradients can destroy your model in a single update. `clip_grad_norm_` with `max_norm=1.0` or `5.0` is standard.

---

## Gated Recurrent Unit (GRU)

### What It Is
GRU is a simplified version of LSTM with only **2 gates** instead of 3. It merges the forget and input gates into a single "update gate" and combines cell state with hidden state. Think of it as LSTM's younger, faster sibling.

### Why It Matters
- **Fewer parameters** = faster training, less overfitting on small datasets
- **Comparable performance** to LSTM on most tasks
- **Simpler to implement and debug**
- **Use when**: you have limited data or need faster training; LSTM and GRU perform similarly — GRU is the simpler default

### How It Works

#### GRU vs LSTM Comparison

```
LSTM:  3 gates (forget, input, output) + cell state + hidden state
GRU:   2 gates (reset, update) + hidden state only

LSTM parameters: 4 × (hidden² + hidden×input + hidden)
GRU parameters:  3 × (hidden² + hidden×input + hidden)
                 → 25% fewer parameters!
```

#### Architecture

```
    ┌────────────────────────────────────────────────┐
    │                                                │
    │  ┌──────────┐    ┌──────────┐                 │
    │  │  Reset   │    │  Update  │                 │
    │  │  Gate rₜ │    │  Gate zₜ │                 │
    │  │  σ(...)  │    │  σ(...)  │                 │
    │  └────┬─────┘    └────┬─────┘                 │
    │       │               │                        │
    │       ↓               ↓                        │
    │  ┌─────────┐    ┌──────────┐                  │
    │  │Candidate│    │  Linear  │                  │
    │  │  h̃ₜ    │    │Interpolat│                  │
    │  └────┬────┘    │   ion    │                  │
    │       │         └────┬─────┘                  │
    │       └──────────────┘                         │
    │              │                                  │
    │              ↓                                  │
    │         hₜ (output)                            │
    └────────────────────────────────────────────────┘
```

#### Mathematical Formulation

**Update Gate** — How much of the past to keep:

$$z_t = \sigma(W_z \cdot [h_{t-1}, x_t] + b_z)$$

**Reset Gate** — How much of the past to forget when computing candidate:

$$r_t = \sigma(W_r \cdot [h_{t-1}, x_t] + b_r)$$

**Candidate Hidden State** — New content:

$$\tilde{h}_t = \tanh(W_h \cdot [r_t \odot h_{t-1}, x_t] + b_h)$$

**Final Hidden State** — Interpolate between old and new:

$$h_t = (1 - z_t) \odot h_{t-1} + z_t \odot \tilde{h}_t$$

> **Key Insight**: The update gate $z_t$ acts as BOTH the forget gate and input gate. When $z_t = 0$, the hidden state stays unchanged (perfect memory). When $z_t = 1$, it completely overwrites with new info.

### Code Example: GRU in PyTorch

```python
import torch
import torch.nn as nn

class TextGRU(nn.Module):
    """
    GRU-based text classifier — drop-in replacement for LSTM version.
    """
    def __init__(self, vocab_size, embed_dim, hidden_dim, output_dim,
                 n_layers=2, bidirectional=True, dropout=0.3, pad_idx=0):
        super().__init__()
        
        self.embedding = nn.Embedding(vocab_size, embed_dim, padding_idx=pad_idx)
        
        self.gru = nn.GRU(
            input_size=embed_dim,
            hidden_size=hidden_dim,
            num_layers=n_layers,
            bidirectional=bidirectional,
            dropout=dropout if n_layers > 1 else 0,
            batch_first=True
        )
        
        # Bidirectional doubles the hidden dimension
        self.fc = nn.Linear(hidden_dim * 2 if bidirectional else hidden_dim, output_dim)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, text, text_lengths):
        embedded = self.dropout(self.embedding(text))
        
        packed = nn.utils.rnn.pack_padded_sequence(
            embedded, text_lengths.cpu(),
            batch_first=True, enforce_sorted=False
        )
        
        packed_output, hidden = self.gru(packed)
        # GRU returns only hidden (no cell state — simpler!)
        # hidden: (n_layers * n_directions, batch, hidden_dim)
        
        if self.gru.bidirectional:
            hidden = torch.cat([hidden[-2], hidden[-1]], dim=1)
        else:
            hidden = hidden[-1]
        
        return self.fc(self.dropout(hidden))

# Compare parameter counts
lstm_model = nn.LSTM(input_size=300, hidden_size=256, num_layers=2, batch_first=True)
gru_model = nn.GRU(input_size=300, hidden_size=256, num_layers=2, batch_first=True)

lstm_params = sum(p.numel() for p in lstm_model.parameters())
gru_params = sum(p.numel() for p in gru_model.parameters())

print(f"LSTM parameters: {lstm_params:,}")   # ~1,142,784
print(f"GRU parameters:  {gru_params:,}")    # ~857,088
print(f"GRU is {(1 - gru_params/lstm_params)*100:.1f}% smaller")  # ~25% smaller
```

### LSTM vs GRU: When to Use What

| Criterion | LSTM | GRU |
|-----------|------|-----|
| Parameters | More (4 gates) | Fewer (3 gates, ~25% less) |
| Training Speed | Slower | Faster |
| Long sequences (>500 tokens) | Better | Slightly worse |
| Small datasets | Overfits more | Better generalization |
| Interpretability | Cell state is inspectable | Simpler to understand |
| Default choice | Complex tasks, large data | Simple tasks, limited data |
| Memory usage | Higher | Lower |

> **Rule of Thumb**: Start with GRU. Switch to LSTM only if GRU underperforms on your specific task, especially for very long sequences.

---

## Bidirectional RNNs

### What It Is
A bidirectional RNN processes the sequence in **both directions** — forward (left→right) and backward (right→left) — then combines the results. It's like reading a sentence twice: once normally and once backwards.

### Why It Matters

Consider this sentence:
```
"He said 'Teddy Roosevelt was a great president'"
"He said 'Teddy bears are on sale'"
```

To understand what "Teddy" refers to, you need FUTURE context (what comes after). A unidirectional RNN only sees the past. A bidirectional RNN sees both.

### How It Works

```
Forward:   h₁→  h₂→  h₃→  h₄→  h₅→
Input:     x₁   x₂   x₃   x₄   x₅
Backward:  h₁←  h₂←  h₃←  h₄←  h₅←

Final output at each step: [h_t→ ; h_t←] (concatenation)
```

### Code Example

```python
import torch
import torch.nn as nn

# PyTorch makes bidirectional trivial — just set the flag
bilstm = nn.LSTM(
    input_size=100,
    hidden_size=128,
    num_layers=2,
    bidirectional=True,   # ← This is all you need!
    batch_first=True
)

# Input: batch of 16 sequences, each 30 tokens, embedding dim 100
x = torch.randn(16, 30, 100)
output, (hidden, cell) = bilstm(x)

print(f"Output shape: {output.shape}")   # (16, 30, 256) — hidden_dim × 2!
print(f"Hidden shape: {hidden.shape}")   # (4, 16, 128) — n_layers × 2, batch, hidden

# To get final representation for classification:
# Concatenate last forward state and last backward state
forward_last = hidden[-2]   # last layer, forward direction
backward_last = hidden[-1]  # last layer, backward direction
final = torch.cat([forward_last, backward_last], dim=1)  # (16, 256)
```

> **When NOT to use bidirectional**: Real-time/streaming tasks (text generation, live translation) where you only have access to past context. You can't look at the future if it hasn't happened yet!

---

## Language Modeling

### What It Is
Language modeling is the task of **predicting the next word** (or the probability of a sequence of words). It's the foundation of autocomplete, text generation, and pre-training for NLP.

### Why It Matters
- **Foundation of GPT, BERT, and all modern LLMs**
- Powers autocomplete (Gmail Smart Compose, phone keyboards)
- Enables text generation, machine translation, summarization
- Self-supervised: needs NO labeled data — just raw text!

### Types of Language Models

| Type | Task | Example | Used In |
|------|------|---------|---------|
| Autoregressive (Causal) | Predict next token | "The cat sat on the ___" | GPT, text generation |
| Masked | Predict masked token | "The [MASK] sat on the mat" | BERT, understanding |
| Sequence-to-Sequence | Map one sequence to another | "Hello" → "Bonjour" | Translation, summarization |

### Mathematical Foundation

$$P(w_1, w_2, ..., w_n) = \prod_{t=1}^{n} P(w_t | w_1, ..., w_{t-1})$$

The chain rule: probability of a sentence = product of each word's probability given all previous words.

**Perplexity** — How to evaluate a language model:

$$PPL = \exp\left(-\frac{1}{N}\sum_{i=1}^{N} \log P(w_i | w_{<i})\right)$$

Lower perplexity = better model. A perplexity of 20 means the model is "as confused as if it had to choose between 20 equally likely words at each step."

### Code Example: Character-Level Language Model

```python
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np

class CharLSTM(nn.Module):
    """
    Character-level language model using LSTM.
    Learns to generate text one character at a time.
    """
    def __init__(self, vocab_size, embed_dim=64, hidden_dim=128, n_layers=2):
        super().__init__()
        self.hidden_dim = hidden_dim
        self.n_layers = n_layers
        
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, n_layers, 
                           batch_first=True, dropout=0.2)
        self.fc = nn.Linear(hidden_dim, vocab_size)
    
    def forward(self, x, hidden=None):
        embedded = self.embedding(x)
        output, hidden = self.lstm(embedded, hidden)
        logits = self.fc(output)  # predict next character at every position
        return logits, hidden
    
    def generate(self, start_char_idx, char_to_idx, idx_to_char, 
                 length=200, temperature=0.8):
        """
        Generate text character by character.
        
        Args:
            temperature: controls randomness (lower = more predictable)
        """
        self.eval()
        hidden = None
        current = torch.tensor([[start_char_idx]])
        generated = [idx_to_char[start_char_idx]]
        
        with torch.no_grad():
            for _ in range(length):
                logits, hidden = self(current, hidden)
                
                # Apply temperature scaling
                # High temp → more random, Low temp → more deterministic
                logits = logits[0, -1] / temperature
                probs = torch.softmax(logits, dim=0)
                
                # Sample from the distribution
                next_idx = torch.multinomial(probs, 1).item()
                generated.append(idx_to_char[next_idx])
                current = torch.tensor([[next_idx]])
        
        return ''.join(generated)

# Prepare data (Shakespeare example)
text = "To be or not to be, that is the question. " * 100  # repeated for demo
chars = sorted(set(text))
char_to_idx = {c: i for i, c in enumerate(chars)}
idx_to_char = {i: c for i, c in enumerate(chars)}
vocab_size = len(chars)

# Encode text
encoded = torch.tensor([char_to_idx[c] for c in text])

# Create training sequences
seq_length = 50
sequences = []
targets = []
for i in range(0, len(encoded) - seq_length):
    sequences.append(encoded[i:i+seq_length])
    targets.append(encoded[i+1:i+seq_length+1])

X = torch.stack(sequences[:1000])  # limit for demo
Y = torch.stack(targets[:1000])

# Train
model = CharLSTM(vocab_size)
optimizer = optim.Adam(model.parameters(), lr=0.003)
criterion = nn.CrossEntropyLoss()

for epoch in range(5):
    model.train()
    # Process in batches
    batch_size = 64
    total_loss = 0
    
    for i in range(0, len(X), batch_size):
        batch_x = X[i:i+batch_size]
        batch_y = Y[i:i+batch_size]
        
        logits, _ = model(batch_x)
        # Reshape for cross-entropy: (batch*seq, vocab) vs (batch*seq,)
        loss = criterion(logits.view(-1, vocab_size), batch_y.view(-1))
        
        optimizer.zero_grad()
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
        optimizer.step()
        total_loss += loss.item()
    
    print(f"Epoch {epoch+1}, Loss: {total_loss/(len(X)//batch_size):.4f}")

# Generate text
generated = model.generate(
    start_char_idx=char_to_idx['T'],
    char_to_idx=char_to_idx,
    idx_to_char=idx_to_char,
    length=100,
    temperature=0.7
)
print(f"\nGenerated: {generated}")
```

---

## Sequence-to-Sequence (Seq2Seq)

### What It Is
Seq2Seq is an architecture that transforms one sequence into another. It uses an **encoder** to compress the input sequence into a fixed representation, and a **decoder** to generate the output sequence. Think of it as a translator who reads the full sentence first, then writes the translation.

### Why It Matters
- **Machine Translation**: English → French
- **Text Summarization**: Long article → Short summary
- **Chatbots**: Question → Answer
- **Code Generation**: Description → Code
- **Historical significance**: Led directly to the Attention mechanism and Transformers

### How It Works

#### Architecture

```
Encoder:
  "How are you" → [h₁] → [h₂] → [h₃] → Context Vector (c)
                                                  │
Decoder:                                          ↓
                                         [d₁] → [d₂] → [d₃] → [d₄]
                                       "Comment"  "allez"  "vous"  "<EOS>"
```

#### The Bottleneck Problem
The entire input sentence must be compressed into a single fixed-size vector (context). For long sentences, this is like trying to remember a whole book in one sentence — information is lost.

> **This bottleneck is exactly why Attention was invented** (covered in Chapter 05).

### Code Example: Seq2Seq for Translation

```python
import torch
import torch.nn as nn
import random

class Encoder(nn.Module):
    """
    Encodes source sequence into a context vector.
    The final hidden state captures the "meaning" of the input.
    """
    def __init__(self, input_dim, embed_dim, hidden_dim, n_layers, dropout):
        super().__init__()
        self.embedding = nn.Embedding(input_dim, embed_dim)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, n_layers, 
                           dropout=dropout, batch_first=True)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, src):
        """
        Args:
            src: source tokens (batch_size, src_len)
        Returns:
            hidden, cell: context vectors for decoder initialization
        """
        embedded = self.dropout(self.embedding(src))
        outputs, (hidden, cell) = self.lstm(embedded)
        # We only need the final hidden/cell states
        return hidden, cell


class Decoder(nn.Module):
    """
    Generates target sequence one token at a time.
    At each step, takes previous output + hidden state → next word.
    """
    def __init__(self, output_dim, embed_dim, hidden_dim, n_layers, dropout):
        super().__init__()
        self.embedding = nn.Embedding(output_dim, embed_dim)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, n_layers,
                           dropout=dropout, batch_first=True)
        self.fc = nn.Linear(hidden_dim, output_dim)
        self.dropout = nn.Dropout(dropout)
        self.output_dim = output_dim
    
    def forward(self, input_token, hidden, cell):
        """
        Single decoding step.
        
        Args:
            input_token: (batch_size, 1) — previous predicted/target token
            hidden: decoder hidden state
            cell: decoder cell state
        Returns:
            prediction: (batch_size, output_vocab_size)
            hidden, cell: updated states
        """
        embedded = self.dropout(self.embedding(input_token))
        output, (hidden, cell) = self.lstm(embedded, (hidden, cell))
        prediction = self.fc(output.squeeze(1))
        return prediction, hidden, cell


class Seq2Seq(nn.Module):
    """
    Complete Seq2Seq model combining encoder and decoder.
    Implements teacher forcing for training.
    """
    def __init__(self, encoder, decoder, device):
        super().__init__()
        self.encoder = encoder
        self.decoder = decoder
        self.device = device
    
    def forward(self, src, trg, teacher_forcing_ratio=0.5):
        """
        Args:
            src: source sequence (batch_size, src_len)
            trg: target sequence (batch_size, trg_len)
            teacher_forcing_ratio: probability of using actual target vs prediction
        
        Teacher Forcing Explained:
        - With TF: Feed actual target word as next input (faster learning)
        - Without TF: Feed model's own prediction (more realistic)
        - During training: mix both (typically 0.5)
        - During inference: always 0 (no target available)
        """
        batch_size = src.shape[0]
        trg_len = trg.shape[1]
        trg_vocab_size = self.decoder.output_dim
        
        # Store all decoder outputs
        outputs = torch.zeros(batch_size, trg_len, trg_vocab_size).to(self.device)
        
        # Encode: compress source into context vectors
        hidden, cell = self.encoder(src)
        
        # First decoder input is <SOS> token
        input_token = trg[:, 0:1]  # (batch_size, 1)
        
        for t in range(1, trg_len):
            prediction, hidden, cell = self.decoder(input_token, hidden, cell)
            outputs[:, t] = prediction
            
            # Teacher forcing: use actual target or predicted token?
            use_teacher_forcing = random.random() < teacher_forcing_ratio
            
            if use_teacher_forcing:
                input_token = trg[:, t:t+1]  # use actual next word
            else:
                input_token = prediction.argmax(dim=1, keepdim=True)  # use prediction
        
        return outputs

# Build the model
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

INPUT_DIM = 8000    # source vocabulary size (e.g., English)
OUTPUT_DIM = 6000   # target vocabulary size (e.g., French)
EMBED_DIM = 256
HIDDEN_DIM = 512
N_LAYERS = 2
DROPOUT = 0.5

encoder = Encoder(INPUT_DIM, EMBED_DIM, HIDDEN_DIM, N_LAYERS, DROPOUT)
decoder = Decoder(OUTPUT_DIM, EMBED_DIM, HIDDEN_DIM, N_LAYERS, DROPOUT)
model = Seq2Seq(encoder, decoder, device).to(device)

# Parameter count
total = sum(p.numel() for p in model.parameters())
print(f"Total parameters: {total:,}")  # ~14.7M

# Training setup
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
criterion = nn.CrossEntropyLoss(ignore_index=0)  # ignore padding token

# Simulated training step
src = torch.randint(1, INPUT_DIM, (32, 20)).to(device)   # batch of English sentences
trg = torch.randint(1, OUTPUT_DIM, (32, 25)).to(device)  # batch of French translations

output = model(src, trg, teacher_forcing_ratio=0.5)
# output shape: (32, 25, 6000) — predictions for each position

# Compute loss (skip first token which is <SOS>)
loss = criterion(
    output[:, 1:].contiguous().view(-1, OUTPUT_DIM),
    trg[:, 1:].contiguous().view(-1)
)
print(f"Loss: {loss.item():.4f}")
```

### Teacher Forcing: Critical Training Technique

```
Without Teacher Forcing (Exposure Bias Problem):
Step 1: Model predicts "Le" (correct ✓)
Step 2: Input "Le" → predicts "chat" (wrong ✗, should be "chien")
Step 3: Input "chat" → predicts "noir" (error compounds!)
→ One mistake cascades into many more

With Teacher Forcing:
Step 1: Model predicts "Le" (correct ✓)  
Step 2: Input actual target "chien" → predicts "est" (can learn from mistakes)
Step 3: Input actual target "est" → predicts correctly
→ Faster convergence, but model never learns to recover from errors

Best Practice: Scheduled Sampling
- Start with high teacher forcing (0.9)
- Gradually reduce to 0.5 or lower during training
- Model learns to predict AND recover from errors
```

---

## Practical Implementations

### Named Entity Recognition with BiLSTM

```python
import torch
import torch.nn as nn

class BiLSTM_NER(nn.Module):
    """
    Bidirectional LSTM for Named Entity Recognition.
    Predicts a tag for EVERY token in the sequence.
    """
    def __init__(self, vocab_size, tagset_size, embed_dim=100, 
                 hidden_dim=128, n_layers=1, dropout=0.3):
        super().__init__()
        
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.lstm = nn.LSTM(
            embed_dim, hidden_dim, n_layers,
            bidirectional=True, batch_first=True, dropout=dropout
        )
        # Output a tag for each token
        self.fc = nn.Linear(hidden_dim * 2, tagset_size)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, x):
        """
        Args:
            x: (batch_size, seq_len) token indices
        Returns:
            tag_scores: (batch_size, seq_len, tagset_size)
        """
        embedded = self.dropout(self.embedding(x))
        lstm_out, _ = self.lstm(embedded)
        tag_scores = self.fc(self.dropout(lstm_out))
        return tag_scores

# Example tag set for NER
tags = ['O', 'B-PER', 'I-PER', 'B-ORG', 'I-ORG', 'B-LOC', 'I-LOC']
# O = Outside any entity
# B-PER = Beginning of a Person name
# I-PER = Inside a Person name (continuation)

model = BiLSTM_NER(vocab_size=5000, tagset_size=len(tags))

# Simulate: "John Smith works at Google in London"
x = torch.randint(0, 5000, (1, 7))
output = model(x)
predictions = output.argmax(dim=-1)
print(f"Predicted tags: {[tags[i] for i in predictions[0].tolist()]}")
```

### Text Generation with Temperature Sampling

```python
def generate_with_strategies(model, start_tokens, max_length=50, 
                             strategy='temperature', **kwargs):
    """
    Different text generation strategies.
    
    Strategies:
    - greedy: always pick highest probability (boring, repetitive)
    - temperature: scale logits then sample (control randomness)
    - top_k: sample from top K candidates only
    - top_p (nucleus): sample from smallest set whose prob sum > p
    """
    model.eval()
    tokens = start_tokens.clone()
    
    with torch.no_grad():
        for _ in range(max_length):
            logits, _ = model(tokens.unsqueeze(0))
            next_logits = logits[0, -1]  # last position predictions
            
            if strategy == 'greedy':
                next_token = next_logits.argmax()
            
            elif strategy == 'temperature':
                temp = kwargs.get('temperature', 1.0)
                scaled = next_logits / temp
                probs = torch.softmax(scaled, dim=0)
                next_token = torch.multinomial(probs, 1)
            
            elif strategy == 'top_k':
                k = kwargs.get('k', 50)
                top_k_logits, top_k_indices = next_logits.topk(k)
                probs = torch.softmax(top_k_logits, dim=0)
                idx = torch.multinomial(probs, 1)
                next_token = top_k_indices[idx]
            
            elif strategy == 'top_p':
                p = kwargs.get('p', 0.9)
                sorted_logits, sorted_indices = next_logits.sort(descending=True)
                cumulative_probs = torch.softmax(sorted_logits, dim=0).cumsum(dim=0)
                
                # Remove tokens with cumulative prob > p
                sorted_indices_to_remove = cumulative_probs > p
                sorted_indices_to_remove[1:] = sorted_indices_to_remove[:-1].clone()
                sorted_indices_to_remove[0] = False
                
                sorted_logits[sorted_indices_to_remove] = float('-inf')
                probs = torch.softmax(sorted_logits, dim=0)
                idx = torch.multinomial(probs, 1)
                next_token = sorted_indices[idx]
            
            tokens = torch.cat([tokens, next_token.view(1)])
    
    return tokens

# Temperature effects:
# temp = 0.1 → very deterministic (almost greedy)
# temp = 1.0 → standard sampling
# temp = 2.0 → very random (creative but potentially nonsensical)
```

---

## Common Mistakes

### 1. Not Using Gradient Clipping
```python
# ❌ WRONG — gradients can explode
loss.backward()
optimizer.step()

# ✅ CORRECT — always clip for RNNs/LSTMs
loss.backward()
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
optimizer.step()
```

### 2. Forgetting to Pack Padded Sequences
```python
# ❌ WRONG — LSTM wastes computation on padding tokens
output, hidden = lstm(padded_batch)

# ✅ CORRECT — tells LSTM to skip padding
packed = nn.utils.rnn.pack_padded_sequence(
    padded_batch, lengths, batch_first=True, enforce_sorted=False
)
packed_output, hidden = lstm(packed)
output, _ = nn.utils.rnn.pad_packed_sequence(packed_output, batch_first=True)
```

### 3. Wrong Hidden State Handling for Bidirectional
```python
# ❌ WRONG — only takes one direction
hidden = hidden[-1]  # misses backward information!

# ✅ CORRECT — concatenate both directions
hidden = torch.cat([hidden[-2], hidden[-1]], dim=1)
```

### 4. Not Detaching Hidden State in TBPTT
```python
# ❌ WRONG — memory leak for long sequences (graph keeps growing)
for batch in dataloader:
    output, hidden = model(batch, hidden)
    loss.backward()

# ✅ CORRECT — detach to prevent backprop through entire history
for batch in dataloader:
    hidden = tuple(h.detach() for h in hidden)  # break computation graph
    output, hidden = model(batch, hidden)
    loss.backward()
```

### 5. Using Bidirectional for Generation Tasks
```python
# ❌ WRONG — can't use future context during generation!
decoder = nn.LSTM(embed_dim, hidden_dim, bidirectional=True)

# ✅ CORRECT — decoder must be unidirectional (causal)
decoder = nn.LSTM(embed_dim, hidden_dim, bidirectional=False)
# Encoder can be bidirectional, decoder NEVER
```

### 6. Incorrect Teacher Forcing During Inference
```python
# ❌ WRONG — using teacher forcing at test time (target doesn't exist!)
output = model(src, trg, teacher_forcing_ratio=0.5)

# ✅ CORRECT — no teacher forcing at inference
output = model(src, trg, teacher_forcing_ratio=0.0)
# Or better: implement a separate inference method
```

---

## Interview Questions

### Conceptual Questions

**Q1: Explain the vanishing gradient problem in RNNs and how LSTM solves it.**

> **Answer**: During backpropagation through time, gradients are multiplied by the recurrent weight matrix at each step. If eigenvalues < 1, gradients shrink exponentially (vanish); if > 1, they explode. LSTM solves this with the **cell state** — a linear path where gradients flow via addition (not multiplication). The forget gate allows gradients to flow unchanged when $f_t = 1$, creating a "gradient highway."

**Q2: What's the difference between LSTM and GRU? When would you choose one over the other?**

> **Answer**: GRU has 2 gates (reset, update) vs LSTM's 3 (forget, input, output). GRU merges cell/hidden state. Use GRU for: smaller datasets, faster training, simpler models. Use LSTM for: very long sequences, when you need fine-grained memory control. In practice, performance is often similar — GRU is a good default.

**Q3: Explain teacher forcing. What problem does it cause?**

> **Answer**: Teacher forcing feeds actual target tokens as decoder input during training instead of model predictions. **Problem**: "exposure bias" — at inference time, the model only sees its own (potentially wrong) predictions, causing error accumulation. **Solutions**: scheduled sampling (gradually reduce TF ratio), reinforcement learning fine-tuning, or beam search at inference.

**Q4: Why is the Seq2Seq bottleneck a problem? How is it solved?**

> **Answer**: The encoder compresses the entire input into a single fixed-size vector. For long sequences, this loses information. **Solution**: Attention mechanism — allows the decoder to "look back" at all encoder hidden states and focus on relevant parts at each decoding step.

**Q5: What is perplexity and how do you interpret it?**

> **Answer**: Perplexity is $2^{H}$ (or $e^{H}$ for natural log) where $H$ is cross-entropy loss. It represents the effective vocabulary size the model is choosing from. PPL of 50 means the model is as uncertain as picking uniformly from 50 words. Lower = better. A perfect model has PPL = 1.

### Coding Questions

**Q6: Implement a simple RNN cell forward pass.**
(See "Basic RNN from Scratch" code above)

**Q7: How do you handle variable-length sequences in PyTorch?**
> Use `pack_padded_sequence` before the RNN and `pad_packed_sequence` after. This ensures padding tokens don't affect hidden states.

**Q8: What's the output shape of a bidirectional LSTM with 2 layers and hidden_size=256?**
> - `output`: (batch, seq_len, 512) — concatenated forward + backward
> - `hidden`: (4, batch, 256) — 2 layers × 2 directions
> - `cell`: (4, batch, 256) — same as hidden

---

## Quick Reference

### Architecture Comparison

| Feature | Vanilla RNN | LSTM | GRU |
|---------|-------------|------|-----|
| Gates | 0 | 3 (forget, input, output) | 2 (reset, update) |
| States | Hidden only | Hidden + Cell | Hidden only |
| Parameters (for hidden=256, input=100) | ~91K | ~364K | ~273K |
| Handles long sequences | ❌ | ✅ | ✅ (slightly worse than LSTM) |
| Training speed | Fast | Slowest | Medium |
| When to use | Never (for production) | Long sequences, complex tasks | Default choice, limited data |

### Key Equations Cheat Sheet

| Model | Key Equation | Purpose |
|-------|-------------|---------|
| RNN | $h_t = \tanh(W_{hh}h_{t-1} + W_{xh}x_t)$ | Update hidden state |
| LSTM | $C_t = f_t \odot C_{t-1} + i_t \odot \tilde{C}_t$ | Update cell state (gradient highway) |
| GRU | $h_t = (1-z_t) \odot h_{t-1} + z_t \odot \tilde{h}_t$ | Interpolate old/new |
| Seq2Seq | Encoder→Context→Decoder | Sequence transformation |

### PyTorch Quick Reference

```python
# RNN
nn.RNN(input_size, hidden_size, num_layers, batch_first=True)

# LSTM  
nn.LSTM(input_size, hidden_size, num_layers, batch_first=True, bidirectional=True)

# GRU
nn.GRU(input_size, hidden_size, num_layers, batch_first=True)

# Pack padded sequences (ALWAYS use with variable-length inputs)
packed = nn.utils.rnn.pack_padded_sequence(x, lengths, batch_first=True, enforce_sorted=False)
output, hidden = lstm(packed)
output, lengths = nn.utils.rnn.pad_packed_sequence(output, batch_first=True)

# Gradient clipping (ALWAYS use with RNNs)
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
```

### Hyperparameter Guidelines

| Parameter | Typical Range | Notes |
|-----------|--------------|-------|
| Hidden size | 128-512 | Larger = more capacity but slower |
| Num layers | 1-3 | >3 rarely helps, use dropout between |
| Dropout | 0.2-0.5 | Apply to embeddings and between layers |
| Learning rate | 1e-3 to 1e-4 | Use Adam optimizer |
| Gradient clip | 1.0-5.0 | Essential for training stability |
| Embedding dim | 100-300 | Use pretrained (GloVe/Word2Vec) if possible |
| Teacher forcing | 0.5-1.0 | Reduce over training (scheduled sampling) |

---

*Next Chapter: [05-Attention-and-Transformers-NLP](05-Attention-and-Transformers-NLP.md) — How attention eliminates the sequence bottleneck*
