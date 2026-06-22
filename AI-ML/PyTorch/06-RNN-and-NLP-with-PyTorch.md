# Chapter 06: RNN and NLP with PyTorch

## Table of Contents
- [1. Recurrent Neural Networks — Memory for Sequences](#1-recurrent-neural-networks)
- [2. RNN Building Blocks in PyTorch](#2-rnn-building-blocks)
- [3. LSTM — Long Short-Term Memory](#3-lstm)
- [4. GRU — Gated Recurrent Unit](#4-gru)
- [5. Text Preprocessing Pipeline](#5-text-preprocessing-pipeline)
- [6. Word Embeddings](#6-word-embeddings)
- [7. Text Classification with RNNs](#7-text-classification)
- [8. Sequence-to-Sequence Models](#8-sequence-to-sequence)
- [9. Packed Sequences — Handling Variable Lengths](#9-packed-sequences)
- [10. Bidirectional and Stacked RNNs](#10-bidirectional-and-stacked)
- [11. Common Mistakes](#11-common-mistakes)
- [12. Interview Questions](#12-interview-questions)
- [13. Quick Reference](#13-quick-reference)

---

## 1. Recurrent Neural Networks — Memory for Sequences {#1-recurrent-neural-networks}

### What It Is
An RNN is a neural network designed for **sequential data** — data where order matters. Unlike feedforward networks that process each input independently, RNNs maintain a **hidden state** (memory) that gets updated at each time step, carrying information from previous steps forward.

Think of reading a sentence: you understand each word in the context of all the words that came before it. "I went to the **bank** to deposit **money**" vs "I sat on the **bank** of the **river**." The word "bank" means different things depending on what came before. RNNs capture this sequential context.

### Why It Matters
- **Natural Language Processing**: text classification, machine translation, chatbots
- **Time Series**: stock prices, weather forecasting, sensor data
- **Speech**: recognition, generation, music composition
- **Sequential Decision Making**: reinforcement learning, video analysis

### How It Works

```
Standard Neural Network (no memory):
  Input₁ → Output₁     Input₂ → Output₂     Input₃ → Output₃
  (each processed independently)

RNN (with memory):
  Input₁ ──→ [h₁] ──→ Input₂ ──→ [h₂] ──→ Input₃ ──→ [h₃] → Output
              ↑ hidden    ↑ hidden    ↑ hidden
              state       state       state
              
  Each step uses BOTH the current input AND the previous hidden state
```

### The RNN Equation

$$h_t = \tanh(W_{ih} x_t + b_{ih} + W_{hh} h_{t-1} + b_{hh})$$

Where:
- $x_t$ = input at time step $t$
- $h_t$ = hidden state at time step $t$ (the "memory")
- $W_{ih}$ = input-to-hidden weights
- $W_{hh}$ = hidden-to-hidden weights (this is the recurrence!)
- $\tanh$ = activation function (squashes values to [-1, 1])

```
Unrolled RNN:

   x₁        x₂        x₃        x₄
    │          │          │          │
    ▼          ▼          ▼          ▼
┌───────┐  ┌───────┐  ┌───────┐  ┌───────┐
│       │→ │       │→ │       │→ │       │
│ RNN   │  │ RNN   │  │ RNN   │  │ RNN   │  (same weights!)
│ Cell  │→ │ Cell  │→ │ Cell  │→ │ Cell  │
│       │  │       │  │       │  │       │
└───┬───┘  └───┬───┘  └───┬───┘  └───┬───┘
    │          │          │          │
    ▼          ▼          ▼          ▼
   h₁         h₂         h₃         h₄
                                     │
                                     ▼
                                  Output
   
   All cells share the SAME weights (weight sharing)
   h₀ is usually initialized to zeros
```

---

## 2. RNN Building Blocks in PyTorch {#2-rnn-building-blocks}

### Basic RNN

```python
import torch
import torch.nn as nn

# ═══════════════════════════════════════════
# nn.RNN — Basic Recurrent Neural Network
# ═══════════════════════════════════════════
rnn = nn.RNN(
    input_size=10,      # Dimension of input at each time step
    hidden_size=20,     # Dimension of hidden state
    num_layers=1,       # Number of stacked RNN layers
    batch_first=True,   # Input shape: (batch, seq_len, features)
    nonlinearity='tanh',  # 'tanh' or 'relu'
    dropout=0,          # Dropout between layers (only if num_layers > 1)
    bidirectional=False # Process sequence in both directions
)

# Input: (batch_size, sequence_length, input_size)
x = torch.randn(32, 15, 10)  # 32 sequences, length 15, 10 features each

# Output
output, h_n = rnn(x)
# output: (32, 15, 20) — hidden state at EVERY time step
# h_n:    (1, 32, 20)  — hidden state at the LAST time step only

print(f"Output shape: {output.shape}")  # (32, 15, 20)
print(f"Hidden shape: {h_n.shape}")     # (1, 32, 20)

# Verify: last output equals last hidden state
print(torch.allclose(output[:, -1, :], h_n[0]))  # True!
```

### Understanding the Shapes

```
batch_first=True:

Input x:     (batch, seq_len, input_size)     = (32, 15, 10)
Output:      (batch, seq_len, hidden_size)    = (32, 15, 20)
h_n:         (num_layers, batch, hidden_size) = (1, 32, 20)

batch_first=False (default):

Input x:     (seq_len, batch, input_size)     = (15, 32, 10)
Output:      (seq_len, batch, hidden_size)    = (15, 32, 20)
h_n:         (num_layers, batch, hidden_size) = (1, 32, 20)
```

> **Pro Tip**: Always use `batch_first=True`. It's more intuitive and consistent with how data is typically organized. The default `batch_first=False` is a legacy choice.

### Passing Initial Hidden State

```python
# By default, h_0 = zeros
output, h_n = rnn(x)

# You can also pass an initial hidden state
h_0 = torch.zeros(1, 32, 20)  # (num_layers, batch, hidden_size)
output, h_n = rnn(x, h_0)

# This is useful when processing sequences in chunks
# or when you want to condition on some initial context
```

---

## 3. LSTM — Long Short-Term Memory {#3-lstm}

### What It Is
LSTM is a special type of RNN designed to solve the **vanishing gradient problem** — the inability of vanilla RNNs to learn long-range dependencies. It uses **gates** (small neural networks) to control what information to remember, forget, and output.

Think of LSTM as having a notebook (cell state) alongside your working memory (hidden state). The notebook can store important information for a long time, while gates decide what to write in, what to erase, and what to read from the notebook.

### Why LSTM over Vanilla RNN?
- Vanilla RNNs forget information after ~10-20 steps (gradient vanishes)
- LSTMs can remember information for hundreds of steps
- LSTMs are the standard for most sequence tasks (before Transformers)

### How LSTM Works

```
LSTM Cell:

                    cell state (C_t) — the "conveyor belt"
    ────────────────────────────────────────────────────────►
                  │           ↑              │
                  │      ┌────┴────┐         │
                  ▼      │  tanh   │         ▼
              ┌───────┐  │ (new    │    ┌────────┐
              │ Forget │  │  info)  │    │ Output │
              │ Gate   │  └────┬────┘    │ Gate   │
              │  σ     │       │         │  σ     │
              └───┬───┘  ┌────┴────┐    └────┬───┘
                  │      │  Input  │         │
                  │      │  Gate   │         │
                  │      │   σ     │         │
                  │      └────┬───┘         │
                  │           │              │
    ──────────────┴───────────┴──────────────┴──────────────
    h_{t-1}, x_t                                    h_t

    Forget Gate: "What old info should I throw away?"
    Input Gate:  "What new info should I write to memory?"
    Output Gate: "What part of memory should I output?"
```

### LSTM Equations

$$f_t = \sigma(W_f \cdot [h_{t-1}, x_t] + b_f) \quad \text{(Forget gate)}$$
$$i_t = \sigma(W_i \cdot [h_{t-1}, x_t] + b_i) \quad \text{(Input gate)}$$
$$\tilde{C}_t = \tanh(W_C \cdot [h_{t-1}, x_t] + b_C) \quad \text{(Candidate cell state)}$$
$$C_t = f_t \odot C_{t-1} + i_t \odot \tilde{C}_t \quad \text{(New cell state)}$$
$$o_t = \sigma(W_o \cdot [h_{t-1}, x_t] + b_o) \quad \text{(Output gate)}$$
$$h_t = o_t \odot \tanh(C_t) \quad \text{(Hidden state / output)}$$

Where $\sigma$ is the sigmoid function and $\odot$ is element-wise multiplication.

### LSTM in PyTorch

```python
# ═══════════════════════════════════════════
# nn.LSTM — Long Short-Term Memory
# ═══════════════════════════════════════════
lstm = nn.LSTM(
    input_size=10,      # Input dimension per time step
    hidden_size=20,     # Hidden state dimension
    num_layers=2,       # Stack 2 LSTM layers
    batch_first=True,   # (batch, seq, features)
    dropout=0.3,        # Dropout between LSTM layers (not on last layer)
    bidirectional=False
)

x = torch.randn(32, 15, 10)  # (batch=32, seq_len=15, features=10)

# LSTM returns TWO hidden states: h_n and c_n
output, (h_n, c_n) = lstm(x)

print(f"Output: {output.shape}")   # (32, 15, 20)  — all time steps
print(f"h_n: {h_n.shape}")        # (2, 32, 20)   — last hidden per layer
print(f"c_n: {c_n.shape}")        # (2, 32, 20)   — last cell state per layer

# h_n[0] = last hidden state of layer 0
# h_n[1] = last hidden state of layer 1 (= output[:, -1, :] for last layer)

# With initial states
h_0 = torch.zeros(2, 32, 20)  # (num_layers, batch, hidden)
c_0 = torch.zeros(2, 32, 20)
output, (h_n, c_n) = lstm(x, (h_0, c_0))
```

### LSTM for Classification

```python
class LSTMClassifier(nn.Module):
    """LSTM for sequence classification (e.g., sentiment analysis)."""
    
    def __init__(self, vocab_size, embed_dim, hidden_dim, output_dim,
                 n_layers=2, dropout=0.5, pad_idx=0):
        super().__init__()
        
        # Embedding layer: token indices → dense vectors
        self.embedding = nn.Embedding(
            num_embeddings=vocab_size,
            embedding_dim=embed_dim,
            padding_idx=pad_idx  # Embedding for padding token = zeros
        )
        
        # LSTM
        self.lstm = nn.LSTM(
            input_size=embed_dim,
            hidden_size=hidden_dim,
            num_layers=n_layers,
            batch_first=True,
            dropout=dropout if n_layers > 1 else 0,
            bidirectional=True  # Process forward AND backward
        )
        
        # Classifier
        # bidirectional doubles hidden_dim
        self.fc = nn.Linear(hidden_dim * 2, output_dim)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, text):
        # text: (batch, seq_len) — integer token indices
        
        embedded = self.dropout(self.embedding(text))
        # embedded: (batch, seq_len, embed_dim)
        
        output, (h_n, c_n) = self.lstm(embedded)
        # h_n: (num_layers * 2, batch, hidden_dim) — *2 for bidirectional
        
        # Concatenate last hidden state from forward and backward directions
        # h_n[-2] = forward direction last layer
        # h_n[-1] = backward direction last layer
        hidden = torch.cat((h_n[-2], h_n[-1]), dim=1)
        # hidden: (batch, hidden_dim * 2)
        
        hidden = self.dropout(hidden)
        logits = self.fc(hidden)
        return logits

# Usage
model = LSTMClassifier(
    vocab_size=10000,
    embed_dim=300,
    hidden_dim=256,
    output_dim=2,    # Binary classification (positive/negative)
    n_layers=2,
    dropout=0.5
)

# Test
x = torch.randint(0, 10000, (32, 100))  # 32 sentences, max 100 tokens
out = model(x)
print(f"Output: {out.shape}")  # (32, 2) — logits for 2 classes
```

---

## 4. GRU — Gated Recurrent Unit {#4-gru}

### What It Is
GRU is a simplified version of LSTM with **fewer gates** (2 instead of 3) and **no separate cell state**. It often performs similarly to LSTM but is faster to train.

### LSTM vs GRU

```
LSTM:  3 gates (forget, input, output) + cell state
       Parameters per unit: 4 × (input + hidden + 1) × hidden
       More expressive, better for very long sequences

GRU:   2 gates (reset, update) — NO cell state
       Parameters per unit: 3 × (input + hidden + 1) × hidden
       ~25% fewer parameters, faster training
       Often as good as LSTM in practice
```

### GRU Equations

$$z_t = \sigma(W_z \cdot [h_{t-1}, x_t]) \quad \text{(Update gate: how much of old state to keep)}$$
$$r_t = \sigma(W_r \cdot [h_{t-1}, x_t]) \quad \text{(Reset gate: how much of old state to use for new)}$$
$$\tilde{h}_t = \tanh(W \cdot [r_t \odot h_{t-1}, x_t]) \quad \text{(Candidate hidden state)}$$
$$h_t = (1 - z_t) \odot h_{t-1} + z_t \odot \tilde{h}_t \quad \text{(New hidden state)}$$

### GRU in PyTorch

```python
# ═══════════════════════════════════════════
# nn.GRU — Gated Recurrent Unit
# ═══════════════════════════════════════════
gru = nn.GRU(
    input_size=10,
    hidden_size=20,
    num_layers=2,
    batch_first=True,
    dropout=0.3,
    bidirectional=True
)

x = torch.randn(32, 15, 10)

# GRU returns only h_n (no c_n — no cell state!)
output, h_n = gru(x)

print(f"Output: {output.shape}")  # (32, 15, 40)  — 40 = 20*2 (bidirectional)
print(f"h_n: {h_n.shape}")       # (4, 32, 20)   — 4 = 2 layers * 2 directions
```

### GRU Classifier

```python
class GRUClassifier(nn.Module):
    """GRU-based text classifier."""
    
    def __init__(self, vocab_size, embed_dim, hidden_dim, output_dim,
                 n_layers=1, dropout=0.5):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.gru = nn.GRU(embed_dim, hidden_dim, n_layers,
                          batch_first=True, dropout=dropout if n_layers > 1 else 0,
                          bidirectional=True)
        self.fc = nn.Linear(hidden_dim * 2, output_dim)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, text):
        embedded = self.dropout(self.embedding(text))
        output, h_n = self.gru(embedded)
        
        # Use last hidden states from both directions
        hidden = torch.cat((h_n[-2], h_n[-1]), dim=1)
        return self.fc(self.dropout(hidden))
```

### When to Use What?

| Feature | Vanilla RNN | LSTM | GRU |
|---------|------------|------|-----|
| Parameters | Fewest | Most | Medium |
| Training Speed | Fastest | Slowest | Medium |
| Long Dependencies | Poor (10-20 steps) | Best | Good |
| Memory Usage | Lowest | Highest | Medium |
| Cell State | No | Yes (separate) | No (merged) |
| Gates | 0 | 3 | 2 |
| Best For | Short sequences, simple tasks | Long sequences, complex tasks | General purpose, efficiency |

> **Pro Tip**: Start with GRU. If performance is insufficient, try LSTM. If working with very long sequences (>500 tokens), consider Transformers instead.

---

## 5. Text Preprocessing Pipeline {#5-text-preprocessing-pipeline}

### What It Is
Before feeding text to a neural network, you need to convert words into numbers. The pipeline: raw text → tokens → numerical indices → padded tensor.

### Complete Text Preprocessing

```python
import torch
from torch.utils.data import Dataset, DataLoader
from collections import Counter
import re

class Vocabulary:
    """Build and manage a word-to-index vocabulary."""
    
    def __init__(self, max_size=None, min_freq=1):
        self.max_size = max_size
        self.min_freq = min_freq
        self.word2idx = {'<PAD>': 0, '<UNK>': 1, '<BOS>': 2, '<EOS>': 3}
        self.idx2word = {v: k for k, v in self.word2idx.items()}
        self.word_count = Counter()
    
    def build(self, texts):
        """Build vocabulary from a list of texts."""
        # Count word frequencies
        for text in texts:
            tokens = self.tokenize(text)
            self.word_count.update(tokens)
        
        # Filter by min_freq and max_size
        words = [w for w, c in self.word_count.most_common(self.max_size)
                 if c >= self.min_freq]
        
        # Add to vocabulary
        for word in words:
            if word not in self.word2idx:
                idx = len(self.word2idx)
                self.word2idx[word] = idx
                self.idx2word[idx] = word
        
        print(f"Vocabulary size: {len(self.word2idx)}")
        return self
    
    @staticmethod
    def tokenize(text):
        """Simple whitespace + lowercasing tokenizer."""
        text = text.lower()
        text = re.sub(r'[^a-zA-Z\s]', '', text)  # Remove punctuation
        return text.split()
    
    def encode(self, text, max_length=None):
        """Convert text to list of indices."""
        tokens = self.tokenize(text)
        indices = [self.word2idx.get(t, 1) for t in tokens]  # 1 = <UNK>
        
        if max_length:
            indices = indices[:max_length]  # Truncate
            indices += [0] * (max_length - len(indices))  # Pad
        
        return indices
    
    def decode(self, indices):
        """Convert indices back to text."""
        return ' '.join(self.idx2word.get(i, '<UNK>') for i in indices if i != 0)
    
    def __len__(self):
        return len(self.word2idx)


class TextDataset(Dataset):
    """PyTorch Dataset for text classification."""
    
    def __init__(self, texts, labels, vocab, max_length=256):
        self.texts = texts
        self.labels = labels
        self.vocab = vocab
        self.max_length = max_length
    
    def __len__(self):
        return len(self.texts)
    
    def __getitem__(self, idx):
        encoded = self.vocab.encode(self.texts[idx], self.max_length)
        return (
            torch.tensor(encoded, dtype=torch.long),
            torch.tensor(self.labels[idx], dtype=torch.long)
        )


# ═══════════════════════════════════════════
# USAGE EXAMPLE
# ═══════════════════════════════════════════
# Sample data
train_texts = [
    "This movie is absolutely fantastic and wonderful",
    "Terrible film I hated every minute of it",
    "Great acting and brilliant storyline",
    "Worst movie ever made do not watch",
    "A masterpiece of modern cinema",
    "Boring and predictable waste of time",
]
train_labels = [1, 0, 1, 0, 1, 0]  # 1=positive, 0=negative

# Build vocabulary from training data
vocab = Vocabulary(max_size=10000, min_freq=1)
vocab.build(train_texts)

# Create dataset and loader
dataset = TextDataset(train_texts, train_labels, vocab, max_length=20)
loader = DataLoader(dataset, batch_size=2, shuffle=True)

# Test
for batch_text, batch_labels in loader:
    print(f"Text shape: {batch_text.shape}")    # (2, 20)
    print(f"Labels shape: {batch_labels.shape}") # (2,)
    print(f"Sample text indices: {batch_text[0]}")
    print(f"Decoded: {vocab.decode(batch_text[0].tolist())}")
    break
```

---

## 6. Word Embeddings {#6-word-embeddings}

### What It Is
An embedding converts a discrete token (word index) into a dense, continuous vector. Instead of representing "cat" as index 42, it becomes a vector like [0.2, -0.5, 0.8, ...]. Similar words get similar vectors.

### Why It Matters
- Neural networks can't process raw integers meaningfully
- Embeddings capture **semantic relationships**: king - man + woman ≈ queen
- Pre-trained embeddings (Word2Vec, GloVe, FastText) give a massive head start

```
One-Hot Encoding (bad):          Embedding (good):
cat = [0,0,0,1,0,...,0]  →  cat = [0.2, -0.5, 0.8, 0.1]
dog = [0,0,1,0,0,...,0]  →  dog = [0.3, -0.4, 0.7, 0.2]  ← similar to cat!
car = [0,1,0,0,0,...,0]  →  car = [-0.8, 0.9, -0.1, 0.5] ← far from cat/dog

One-hot: 10,000-dim sparse vector (wasteful, no similarity)
Embedding: 300-dim dense vector (compact, captures meaning)
```

### nn.Embedding in PyTorch

```python
import torch.nn as nn

# ═══════════════════════════════════════════
# nn.Embedding — Lookup table for embeddings
# ═══════════════════════════════════════════
embedding = nn.Embedding(
    num_embeddings=10000,  # Vocabulary size
    embedding_dim=300,      # Dimension of embedding vectors
    padding_idx=0           # Index 0 (PAD) always maps to zeros
)

# Input: integer indices
indices = torch.tensor([[1, 2, 3, 0, 0],    # Sentence 1 (padded)
                        [4, 5, 6, 7, 8]])    # Sentence 2

embedded = embedding(indices)
print(embedded.shape)  # (2, 5, 300) — each word → 300-dim vector

# Padding embeddings are zeros
print(embedded[0, 3].sum())  # tensor(0.) — padding_idx=0 works!
```

### Loading Pretrained Embeddings (GloVe)

```python
import numpy as np

def load_glove_embeddings(glove_path, vocab, embed_dim=300):
    """
    Load pretrained GloVe embeddings into an Embedding layer.
    Download GloVe from: https://nlp.stanford.edu/projects/glove/
    """
    # Initialize with random embeddings
    embedding_matrix = np.random.normal(0, 0.1, (len(vocab), embed_dim))
    embedding_matrix[0] = 0  # PAD should be zeros
    
    # Load GloVe vectors
    found = 0
    with open(glove_path, 'r', encoding='utf-8') as f:
        for line in f:
            parts = line.strip().split()
            word = parts[0]
            if word in vocab.word2idx:
                idx = vocab.word2idx[word]
                vector = np.array(parts[1:], dtype=np.float32)
                embedding_matrix[idx] = vector
                found += 1
    
    print(f"Found {found}/{len(vocab)} words in GloVe")
    
    # Create Embedding layer with pretrained weights
    embedding = nn.Embedding.from_pretrained(
        torch.tensor(embedding_matrix, dtype=torch.float32),
        freeze=False,       # Fine-tune embeddings during training
        padding_idx=0
    )
    
    return embedding

# Usage:
# embedding = load_glove_embeddings('glove.6B.300d.txt', vocab, 300)
```

### Freeze vs Fine-Tune Embeddings

```python
# FREEZE: Don't update embeddings during training
# Use when: small dataset, pretrained embeddings cover your vocab well
embedding = nn.Embedding.from_pretrained(weights, freeze=True)
# Or: embedding.weight.requires_grad = False

# FINE-TUNE: Update embeddings during training (RECOMMENDED)
# Use when: domain-specific text, enough training data
embedding = nn.Embedding.from_pretrained(weights, freeze=False)

# PARTIAL FREEZE: Freeze pretrained, train only new words
# Advanced technique
for idx in pretrained_indices:
    embedding.weight.data[idx].requires_grad = False
```

---

## 7. Text Classification with RNNs {#7-text-classification}

### Complete Sentiment Analysis Pipeline

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, Dataset
import re
from collections import Counter

# ══════════════════════════════════════════════════════════════
# 1. DATA PREPARATION (simplified IMDB-style)
# ══════════════════════════════════════════════════════════════

# Assume we have training data:
# train_texts = ["great movie...", "terrible film...", ...]
# train_labels = [1, 0, ...]

# (Using synthetic data for runnable example)
import random
random.seed(42)

positive_words = ['great', 'amazing', 'wonderful', 'fantastic', 'excellent',
                  'love', 'perfect', 'beautiful', 'brilliant', 'outstanding']
negative_words = ['terrible', 'awful', 'horrible', 'bad', 'worst',
                  'hate', 'boring', 'waste', 'poor', 'disappointing']
neutral_words = ['the', 'a', 'is', 'was', 'it', 'this', 'movie', 'film',
                 'show', 'really', 'very', 'so', 'just', 'not', 'but']

def generate_review(label, length=20):
    words = []
    for _ in range(length):
        r = random.random()
        if r < 0.3:
            words.append(random.choice(positive_words if label == 1 else negative_words))
        else:
            words.append(random.choice(neutral_words))
    return ' '.join(words)

train_texts = [generate_review(i % 2) for i in range(2000)]
train_labels = [i % 2 for i in range(2000)]
val_texts = [generate_review(i % 2) for i in range(400)]
val_labels = [i % 2 for i in range(400)]

# ══════════════════════════════════════════════════════════════
# 2. VOCABULARY + DATASET
# ══════════════════════════════════════════════════════════════

class SimpleVocab:
    def __init__(self, texts, max_size=5000):
        counter = Counter()
        for text in texts:
            counter.update(text.lower().split())
        
        self.word2idx = {'<PAD>': 0, '<UNK>': 1}
        for word, _ in counter.most_common(max_size):
            self.word2idx[word] = len(self.word2idx)
    
    def encode(self, text, max_len=100):
        tokens = text.lower().split()[:max_len]
        indices = [self.word2idx.get(t, 1) for t in tokens]
        length = len(indices)
        indices += [0] * (max_len - length)  # Pad
        return indices, length
    
    def __len__(self):
        return len(self.word2idx)

class ReviewDataset(Dataset):
    def __init__(self, texts, labels, vocab, max_len=100):
        self.texts = texts
        self.labels = labels
        self.vocab = vocab
        self.max_len = max_len
    
    def __len__(self):
        return len(self.texts)
    
    def __getitem__(self, idx):
        indices, length = self.vocab.encode(self.texts[idx], self.max_len)
        return (torch.tensor(indices, dtype=torch.long),
                torch.tensor(length, dtype=torch.long),
                torch.tensor(self.labels[idx], dtype=torch.long))

vocab = SimpleVocab(train_texts)
train_dataset = ReviewDataset(train_texts, train_labels, vocab)
val_dataset = ReviewDataset(val_texts, val_labels, vocab)

train_loader = DataLoader(train_dataset, batch_size=64, shuffle=True)
val_loader = DataLoader(val_dataset, batch_size=64, shuffle=False)

# ══════════════════════════════════════════════════════════════
# 3. MODEL
# ══════════════════════════════════════════════════════════════

class SentimentLSTM(nn.Module):
    def __init__(self, vocab_size, embed_dim=128, hidden_dim=256,
                 output_dim=2, n_layers=2, dropout=0.5):
        super().__init__()
        
        self.embedding = nn.Embedding(vocab_size, embed_dim, padding_idx=0)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, n_layers,
                           batch_first=True, dropout=dropout,
                           bidirectional=True)
        
        # Attention layer — weight each time step by importance
        self.attention = nn.Linear(hidden_dim * 2, 1)
        
        self.fc = nn.Sequential(
            nn.Linear(hidden_dim * 2, hidden_dim),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(hidden_dim, output_dim)
        )
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, text, lengths):
        # text: (batch, seq_len)
        embedded = self.dropout(self.embedding(text))  # (batch, seq_len, embed_dim)
        
        # Pack padded sequences for efficient LSTM processing
        packed = nn.utils.rnn.pack_padded_sequence(
            embedded, lengths.cpu(), batch_first=True, enforce_sorted=False
        )
        packed_output, (h_n, c_n) = self.lstm(packed)
        output, _ = nn.utils.rnn.pad_packed_sequence(
            packed_output, batch_first=True
        )
        # output: (batch, seq_len, hidden_dim * 2)
        
        # Simple attention: learn which time steps matter most
        attn_weights = torch.softmax(self.attention(output).squeeze(-1), dim=1)
        # attn_weights: (batch, seq_len)
        
        # Weighted sum of all time steps
        context = torch.bmm(attn_weights.unsqueeze(1), output).squeeze(1)
        # context: (batch, hidden_dim * 2)
        
        logits = self.fc(context)
        return logits

# ══════════════════════════════════════════════════════════════
# 4. TRAINING
# ══════════════════════════════════════════════════════════════

device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = SentimentLSTM(len(vocab)).to(device)
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=1e-3)

print(f"Model parameters: {sum(p.numel() for p in model.parameters()):,}")

for epoch in range(10):
    # Train
    model.train()
    total_loss, correct, total = 0, 0, 0
    
    for text, lengths, labels in train_loader:
        text = text.to(device)
        lengths = lengths.to(device)
        labels = labels.to(device)
        
        outputs = model(text, lengths)
        loss = criterion(outputs, labels)
        
        optimizer.zero_grad()
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=5.0)
        optimizer.step()
        
        total_loss += loss.item() * text.size(0)
        _, predicted = outputs.max(1)
        total += labels.size(0)
        correct += predicted.eq(labels).sum().item()
    
    train_acc = 100. * correct / total
    
    # Validate
    model.eval()
    val_correct, val_total = 0, 0
    with torch.no_grad():
        for text, lengths, labels in val_loader:
            text, lengths, labels = text.to(device), lengths.to(device), labels.to(device)
            outputs = model(text, lengths)
            _, predicted = outputs.max(1)
            val_total += labels.size(0)
            val_correct += predicted.eq(labels).sum().item()
    
    val_acc = 100. * val_correct / val_total
    print(f"Epoch [{epoch+1}/10] Loss: {total_loss/total:.4f} | "
          f"Train Acc: {train_acc:.1f}% | Val Acc: {val_acc:.1f}%")
```

---

## 8. Sequence-to-Sequence Models {#8-sequence-to-sequence}

### What It Is
Seq2Seq models transform one sequence into another. Used for machine translation, summarization, chatbots, and any task where input and output are both sequences (potentially different lengths).

### Architecture

```
    Encoder                              Decoder
    
    "I love cats"                       "J'aime les chats"
    
     I    love   cats                    <BOS>   J'    aime   les
     │      │      │                       │      │      │      │
     ▼      ▼      ▼                       ▼      ▼      ▼      ▼
   ┌───┐  ┌───┐  ┌───┐    context     ┌───┐  ┌───┐  ┌───┐  ┌───┐
   │   │→ │   │→ │   │ ──────────▶    │   │→ │   │→ │   │→ │   │
   └───┘  └───┘  └───┘   vector      └───┘  └───┘  └───┘  └───┘
                           (h, c)       │      │      │      │
                                        ▼      ▼      ▼      ▼
                                       J'    aime   les   chats
```

### Seq2Seq Implementation

```python
class Encoder(nn.Module):
    """Encodes source sequence into a context vector."""
    
    def __init__(self, vocab_size, embed_dim, hidden_dim, n_layers, dropout):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, n_layers,
                           batch_first=True, dropout=dropout)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, src):
        # src: (batch, src_len)
        embedded = self.dropout(self.embedding(src))
        outputs, (hidden, cell) = self.lstm(embedded)
        return hidden, cell  # Context vector


class Decoder(nn.Module):
    """Decodes context vector into target sequence, one token at a time."""
    
    def __init__(self, vocab_size, embed_dim, hidden_dim, n_layers, dropout):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, n_layers,
                           batch_first=True, dropout=dropout)
        self.fc = nn.Linear(hidden_dim, vocab_size)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, input_token, hidden, cell):
        # input_token: (batch, 1) — single token
        embedded = self.dropout(self.embedding(input_token))
        output, (hidden, cell) = self.lstm(embedded, (hidden, cell))
        prediction = self.fc(output.squeeze(1))  # (batch, vocab_size)
        return prediction, hidden, cell


class Seq2Seq(nn.Module):
    """Complete Seq2Seq model with teacher forcing."""
    
    def __init__(self, encoder, decoder, device):
        super().__init__()
        self.encoder = encoder
        self.decoder = decoder
        self.device = device
    
    def forward(self, src, trg, teacher_forcing_ratio=0.5):
        # src: (batch, src_len)
        # trg: (batch, trg_len)
        
        batch_size = src.size(0)
        trg_len = trg.size(1)
        trg_vocab_size = self.decoder.fc.out_features
        
        # Store decoder outputs
        outputs = torch.zeros(batch_size, trg_len, trg_vocab_size).to(self.device)
        
        # Encode
        hidden, cell = self.encoder(src)
        
        # First input to decoder is <BOS> token
        input_token = trg[:, 0:1]  # (batch, 1)
        
        for t in range(1, trg_len):
            prediction, hidden, cell = self.decoder(input_token, hidden, cell)
            outputs[:, t] = prediction
            
            # Teacher forcing: use actual next token vs predicted token
            if random.random() < teacher_forcing_ratio:
                input_token = trg[:, t:t+1]  # Use ground truth
            else:
                input_token = prediction.argmax(1, keepdim=True)  # Use prediction
        
        return outputs

# Setup
VOCAB_SIZE = 5000
EMBED_DIM = 256
HIDDEN_DIM = 512
N_LAYERS = 2
DROPOUT = 0.5

device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

encoder = Encoder(VOCAB_SIZE, EMBED_DIM, HIDDEN_DIM, N_LAYERS, DROPOUT)
decoder = Decoder(VOCAB_SIZE, EMBED_DIM, HIDDEN_DIM, N_LAYERS, DROPOUT)
model = Seq2Seq(encoder, decoder, device).to(device)

# Training: gradually reduce teacher_forcing_ratio from 1.0 to 0.0
```

> **Teacher Forcing**: During training, we can feed the correct previous token (teacher forcing) or the model's own prediction. High teacher forcing = faster training but exposure bias. Low teacher forcing = slower but more robust at inference.

---

## 9. Packed Sequences — Handling Variable Lengths {#9-packed-sequences}

### What It Is
When sentences have different lengths, you pad them to the same length. But the LSTM wastes computation processing padding tokens. **Packed sequences** tell the LSTM to skip padding positions, making training faster and more correct.

### Why It Matters
- Without packing: LSTM processes padding, which adds noise to hidden states
- With packing: LSTM only processes actual tokens — cleaner gradients, faster training
- Essential for correct results when using the final hidden state

### How to Use Packed Sequences

```python
from torch.nn.utils.rnn import pack_padded_sequence, pad_packed_sequence

# ═══════════════════════════════════════════
# STEP BY STEP
# ═══════════════════════════════════════════

# Batch of 4 sentences with different lengths
sentences = [
    [1, 2, 3, 4, 5],      # length 5
    [6, 7, 8],              # length 3
    [9, 10, 11, 12],        # length 4
    [13, 14],               # length 2
]

# Pad to max length
max_len = max(len(s) for s in sentences)
padded = torch.zeros(4, max_len, dtype=torch.long)
lengths = []
for i, s in enumerate(sentences):
    padded[i, :len(s)] = torch.tensor(s)
    lengths.append(len(s))

lengths = torch.tensor(lengths)  # [5, 3, 4, 2]

print(f"Padded:\n{padded}")
# tensor([[ 1,  2,  3,  4,  5],
#         [ 6,  7,  8,  0,  0],   ← padded with 0
#         [ 9, 10, 11, 12,  0],   ← padded with 0
#         [13, 14,  0,  0,  0]])  ← padded with 0

# Embed
embedding = nn.Embedding(20, 8, padding_idx=0)
embedded = embedding(padded)  # (4, 5, 8)

# Pack — tell LSTM to ignore padding
packed = pack_padded_sequence(
    embedded,
    lengths.cpu(),         # Lengths must be on CPU
    batch_first=True,
    enforce_sorted=False   # Don't require descending length order
)

# Run LSTM on packed sequence
lstm = nn.LSTM(8, 16, batch_first=True)
packed_output, (h_n, c_n) = lstm(packed)

# Unpack back to padded tensor
output, output_lengths = pad_packed_sequence(
    packed_output,
    batch_first=True
)

print(f"Output shape: {output.shape}")         # (4, 5, 16)
print(f"Output lengths: {output_lengths}")     # tensor([5, 3, 4, 2])

# h_n now contains the CORRECT last hidden state for each sequence
# (not the hidden state at the padded position)
```

### Collate Function for Variable-Length Batches

```python
from torch.nn.utils.rnn import pad_sequence

def collate_fn(batch):
    """Custom collate function for variable-length sequences."""
    # batch is a list of (text_tensor, label) tuples
    texts, labels = zip(*batch)
    
    # Get lengths before padding
    lengths = torch.tensor([len(t) for t in texts])
    
    # Pad sequences to max length in this batch
    padded_texts = pad_sequence(texts, batch_first=True, padding_value=0)
    
    labels = torch.stack(labels)
    
    return padded_texts, lengths, labels

# Usage
loader = DataLoader(dataset, batch_size=32, collate_fn=collate_fn, shuffle=True)
```

---

## 10. Bidirectional and Stacked RNNs {#10-bidirectional-and-stacked}

### Bidirectional RNN

```
Forward:   x₁ → x₂ → x₃ → x₄  (reads left to right)
Backward:  x₁ ← x₂ ← x₃ ← x₄  (reads right to left)

Each time step gets context from BOTH past AND future.
Output at each step = [forward_hidden ; backward_hidden]
```

```python
# Bidirectional LSTM
bilstm = nn.LSTM(
    input_size=10,
    hidden_size=20,
    num_layers=1,
    batch_first=True,
    bidirectional=True  # ← This doubles the output dimension
)

x = torch.randn(32, 15, 10)
output, (h_n, c_n) = bilstm(x)

# Output: hidden_size * 2 (forward + backward concatenated)
print(f"Output: {output.shape}")  # (32, 15, 40)  ← 20*2=40
print(f"h_n: {h_n.shape}")       # (2, 32, 20)   ← [forward_last, backward_last]

# h_n[0] = last hidden of forward direction  (after seeing x₁ to x₁₅)
# h_n[1] = last hidden of backward direction (after seeing x₁₅ to x₁)

# For classification, concatenate both directions:
hidden = torch.cat((h_n[0], h_n[1]), dim=1)  # (32, 40)
```

### Stacked (Multi-Layer) RNNs

```
Layer 2:  [h₁²] → [h₂²] → [h₃²] → [h₄²]  (higher-level features)
             ↑       ↑       ↑       ↑
Layer 1:  [h₁¹] → [h₂¹] → [h₃¹] → [h₄¹]  (low-level features)
             ↑       ↑       ↑       ↑
             x₁      x₂      x₃      x₄
             
Output of layer 1 becomes input of layer 2.
Dropout is applied between layers (not within a layer).
```

```python
# Stacked bidirectional LSTM (common for NLP)
stacked_bilstm = nn.LSTM(
    input_size=300,     # e.g., GloVe embedding dim
    hidden_size=256,
    num_layers=3,       # 3 stacked layers
    batch_first=True,
    dropout=0.3,        # Applied between layers (not after last layer)
    bidirectional=True
)

x = torch.randn(32, 100, 300)  # (batch, seq_len, embed_dim)
output, (h_n, c_n) = stacked_bilstm(x)

# h_n shape: (num_layers * num_directions, batch, hidden_size)
print(f"h_n: {h_n.shape}")  # (6, 32, 256)  ← 3 layers × 2 directions

# To get the final hidden state for classification:
# Forward direction of last layer: h_n[-2]
# Backward direction of last layer: h_n[-1]
hidden = torch.cat((h_n[-2], h_n[-1]), dim=1)  # (32, 512)
```

### How Many Layers to Use?

| Layers | When to Use | Notes |
|--------|------------|-------|
| 1 | Simple tasks, small data | Fastest, least overfitting |
| 2 | Most NLP tasks | Good default |
| 3 | Complex tasks, large data | Diminishing returns above 3 |
| 4+ | Rare, only for very complex tasks | Need lots of regularization |

---

## 11. Common Mistakes {#11-common-mistakes}

| Mistake | Problem | Fix |
|---------|---------|-----|
| Not using `batch_first=True` | Confusing shape: (seq, batch, feat) | Always use `batch_first=True` |
| Forgetting `h_n` vs `output` distinction | Using wrong hidden state | `h_n` = last step only, `output` = all steps |
| Not packing padded sequences | LSTM processes padding tokens | Use `pack_padded_sequence` |
| Missing gradient clipping for RNNs | Exploding gradients → NaN loss | Use `clip_grad_norm_(params, 5.0)` |
| Applying softmax before `CrossEntropyLoss` | Double softmax | CE expects raw logits |
| Wrong `h_n` indexing for bidirectional | Using wrong direction's hidden | Forward last: `h_n[-2]`, Backward last: `h_n[-1]` |
| Not detaching hidden state for long sequences | Memory grows unboundedly | `h = h.detach()` when processing chunks |
| Using vanilla RNN for long sequences | Vanishing gradients | Use LSTM or GRU |
| `dropout` with `num_layers=1` | Warning: dropout has no effect | Only use dropout when `num_layers > 1` |
| Initializing embeddings poorly | Slow convergence | Use pretrained (GloVe) or Xavier init |

---

## 12. Interview Questions {#12-interview-questions}

**Q1: Why do vanilla RNNs suffer from vanishing gradients?**
> During backpropagation through time (BPTT), gradients are multiplied by the weight matrix at each step. If the largest eigenvalue of $W_{hh}$ is < 1, gradients shrink exponentially. After 10-20 steps, gradients are near zero, so early time steps can't learn. LSTM solves this with the cell state, which has an additive (not multiplicative) update path, allowing gradients to flow unchanged through the forget gate.

**Q2: Explain the forget gate in LSTM. Why is it important?**
> The forget gate $f_t = \sigma(W_f[h_{t-1}, x_t] + b_f)$ produces values between 0 and 1 for each element of the cell state. A value of 1 means "keep this information completely," while 0 means "forget completely." This selective forgetting is crucial — without it, the cell state would grow unboundedly with accumulated information. Interestingly, initializing the forget gate bias to a positive value (e.g., 1.0) helps training by defaulting to "remember" initially.

**Q3: When should you use bidirectional RNNs?**
> Use bidirectional when you have access to the entire sequence (classification, named entity recognition, encoding for translation). Do NOT use bidirectional for autoregressive generation (language models, text generation) where future tokens aren't available at inference time.

**Q4: What is teacher forcing and what is its drawback?**
> Teacher forcing feeds the ground truth previous token to the decoder during training (instead of the model's own prediction). Pro: much faster convergence. Con: exposure bias — at inference time, the model must use its own (potentially wrong) predictions, which it never saw during training. This can cause error accumulation. Solutions: scheduled sampling (gradually reduce teacher forcing), or sequence-level training.

**Q5: How do packed sequences improve training?**
> Without packing, the LSTM processes padding tokens, which: (1) wastes computation, (2) corrupts the hidden state at padded positions, (3) the "last" hidden state may actually be from a padding position. Packed sequences tell the LSTM to skip padding entirely, producing correct hidden states and saving 20-50% compute on variable-length batches.

**Q6: Compare LSTM vs GRU. When would you prefer one over the other?**
> LSTM: 3 gates + separate cell state → more expressive, better for very long dependencies, more parameters. GRU: 2 gates, no cell state → simpler, faster, fewer parameters, often equally effective. Use GRU as default for efficiency. Switch to LSTM if (1) sequences are very long (>500 tokens), (2) GRU performance plateaus, (3) you need fine-grained control over memory (cell state access). For most tasks, the difference is minimal — Transformers have largely replaced both.

**Q7: Why have Transformers replaced RNNs for most NLP tasks?**
> RNNs process tokens sequentially (can't parallelize), have limited memory for long sequences (despite LSTM), and struggle with very long-range dependencies. Transformers use self-attention to process all tokens in parallel, capture any-distance dependencies equally well, and scale much better on modern GPUs/TPUs. However, RNNs are still useful for streaming data, low-latency requirements, and when sequence length is very long (linear vs quadratic complexity).

---

## 13. Quick Reference {#13-quick-reference}

### RNN/LSTM/GRU Output Shapes

```python
# batch_first=True, bidirectional=False, num_layers=L

# RNN/GRU:
output: (batch, seq_len, hidden_size)
h_n:    (L, batch, hidden_size)

# LSTM:
output: (batch, seq_len, hidden_size)
h_n:    (L, batch, hidden_size)
c_n:    (L, batch, hidden_size)

# With bidirectional=True:
output: (batch, seq_len, hidden_size * 2)
h_n:    (L * 2, batch, hidden_size)
```

### Text Classification Template

```python
class TextClassifier(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_dim, num_classes,
                 n_layers=2, dropout=0.5, bidirectional=True):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim, padding_idx=0)
        self.rnn = nn.LSTM(embed_dim, hidden_dim, n_layers,
                          batch_first=True, dropout=dropout,
                          bidirectional=bidirectional)
        direction_factor = 2 if bidirectional else 1
        self.fc = nn.Linear(hidden_dim * direction_factor, num_classes)
        self.dropout = nn.Dropout(dropout)
    
    def forward(self, text, lengths):
        embedded = self.dropout(self.embedding(text))
        packed = nn.utils.rnn.pack_padded_sequence(
            embedded, lengths.cpu(), batch_first=True, enforce_sorted=False)
        _, (h_n, _) = self.rnn(packed)
        hidden = torch.cat((h_n[-2], h_n[-1]), dim=1)
        return self.fc(self.dropout(hidden))
```

### Key Functions

| Function | Purpose |
|----------|---------|
| `nn.Embedding(V, D)` | Lookup table: index → dense vector |
| `nn.LSTM(input, hidden, layers)` | LSTM layer |
| `nn.GRU(input, hidden, layers)` | GRU layer |
| `pack_padded_sequence()` | Pack padded batch for efficient RNN |
| `pad_packed_sequence()` | Unpack back to padded tensor |
| `pad_sequence()` | Pad list of tensors to same length |
| `clip_grad_norm_()` | Clip gradients (essential for RNNs) |

### Typical Hyperparameters

```python
# Sentiment Analysis / Text Classification
embed_dim = 300        # 300 if using GloVe, 128-256 otherwise
hidden_dim = 256       # 128-512 typical range
n_layers = 2           # 1-3 layers
dropout = 0.5          # 0.3-0.5 for RNNs
lr = 1e-3              # For Adam
grad_clip = 5.0        # Gradient clipping norm

# Seq2Seq / Translation
embed_dim = 256-512
hidden_dim = 512-1024
n_layers = 2-4
dropout = 0.3
lr = 1e-3 → 1e-4       # Reduce over time
teacher_forcing = 1.0 → 0.0  # Anneal during training
```

### Imports Cheat Sheet

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import Dataset, DataLoader
from torch.nn.utils.rnn import (
    pack_padded_sequence,
    pad_packed_sequence,
    pad_sequence
)
```

---

*Previous: [05-CNN-with-PyTorch](05-CNN-with-PyTorch.md) | Next: [07-Transfer-Learning-PyTorch](07-Transfer-Learning-PyTorch.md)*
