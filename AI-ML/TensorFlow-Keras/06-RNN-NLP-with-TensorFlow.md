# Chapter 06: RNN and NLP with TensorFlow/Keras

## Table of Contents
- [What are RNNs?](#what-are-rnns)
- [Why RNNs Matter for NLP](#why-rnns-matter-for-nlp)
- [How RNNs Work](#how-rnns-work)
- [Text Preprocessing in TensorFlow](#text-preprocessing-in-tensorflow)
- [Embedding Layer](#embedding-layer)
- [SimpleRNN in Keras](#simplernn-in-keras)
- [LSTM — Long Short-Term Memory](#lstm-long-short-term-memory)
- [GRU — Gated Recurrent Unit](#gru-gated-recurrent-unit)
- [Bidirectional RNNs](#bidirectional-rnns)
- [Text Classification — Complete Walkthrough](#text-classification-complete-walkthrough)
- [Sequence-to-Sequence Models](#sequence-to-sequence-models)
- [Text Generation](#text-generation)
- [Named Entity Recognition (NER)](#named-entity-recognition)
- [Advanced Techniques](#advanced-techniques)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What are RNNs?

### Simple Explanation

Imagine you're reading a sentence: **"The cat sat on the ___"**

Your brain doesn't process each word in isolation — it **remembers** what came before:
- "The" → OK, something is coming
- "The cat" → We're talking about a cat
- "The cat sat" → The cat is sitting
- "The cat sat on the" → What's it sitting on? Probably "mat", "floor", etc.

An **RNN (Recurrent Neural Network)** works the same way. It processes sequences **one element at a time**, maintaining a **hidden state** (memory) that carries information from previous steps.

```
Regular Neural Network:      RNN:
Input → Output               Input₁ → Output₁
(no memory)                        ↓ (memory)
                              Input₂ → Output₂
                                   ↓ (memory)
                              Input₃ → Output₃
```

### Technical Definition

An RNN is a neural network with **loops** that allow information to persist. At each time step $t$, it takes the current input $x_t$ and the previous hidden state $h_{t-1}$, and produces a new hidden state $h_t$ and optionally an output $y_t$.

---

## Why RNNs Matter for NLP

### Sequential Data is Everywhere

| Domain | Sequential Data | Task |
|--------|----------------|------|
| NLP | Text (sequences of words) | Sentiment analysis, translation |
| Speech | Audio waveforms | Speech recognition, synthesis |
| Finance | Stock prices over time | Price prediction |
| Music | Note sequences | Music generation |
| Biology | DNA/protein sequences | Gene prediction |
| IoT | Sensor readings over time | Anomaly detection |

### Why Not Use Dense Networks for Text?

```
Problem 1: Variable Length
- "Good movie" → 2 words
- "This is the best movie I have ever seen in my life" → 12 words
- Dense networks need FIXED input size

Problem 2: Word Order Matters
- "Dog bites man" ≠ "Man bites dog"
- Dense networks treat all inputs independently

Problem 3: Long-range Dependencies
- "The cat, which was sitting on the mat, purred"
- "purred" relates to "cat" — 8 words apart
- Dense networks can't capture this naturally
```

### RNN vs CNN vs Transformer for NLP

| Feature | RNN/LSTM | 1D CNN | Transformer |
|---------|----------|--------|-------------|
| Sequential processing | Yes (slow) | No (fast) | No (fast) |
| Long-range dependencies | LSTM: yes | Limited | Excellent |
| Training speed | Slow | Fast | Fast |
| Memory usage | Low | Low | High |
| Best for | Small sequences | Text classification | Everything (with data) |
| State of the art (2024+) | No | No | Yes |

> **Reality Check**: Transformers (BERT, GPT) have largely replaced RNNs for NLP. However, understanding RNNs is essential because: (1) They're still used in production for simpler/faster tasks, (2) They're fundamental to understanding attention and transformers, (3) They're excellent for time series and real-time streaming.

---

## How RNNs Work

### The Unrolled View

```
Folded (Compact):                Unrolled (Across Time):

        ┌───┐                    ┌───┐    ┌───┐    ┌───┐    ┌───┐
  x ──▶ │ A │ ──▶ h              │ A │──▶ │ A │──▶ │ A │──▶ │ A │──▶ h₄
        └─┬─┘                    └─▲─┘    └─▲─┘    └─▲─┘    └─▲─┘
          │                        │         │         │         │
          └──── (loop)            x₁        x₂        x₃        x₄
                                "The"     "movie"    "was"    "great"
```

### Mathematics of a Simple RNN

At each time step $t$:

$$h_t = \tanh(W_{xh} \cdot x_t + W_{hh} \cdot h_{t-1} + b_h)$$

$$y_t = W_{hy} \cdot h_t + b_y$$

Where:
- $x_t$ = input at time $t$ (e.g., word embedding)
- $h_t$ = hidden state at time $t$ (the "memory")
- $h_{t-1}$ = previous hidden state
- $W_{xh}$ = input-to-hidden weights
- $W_{hh}$ = hidden-to-hidden weights (the "recurrent" weights)
- $W_{hy}$ = hidden-to-output weights
- $b_h$, $b_y$ = biases

### The Vanishing Gradient Problem

```
Backpropagation through time (BPTT):

Gradients flow backward through each time step:
h₁₀ ← h₉ ← h₈ ← h₇ ← h₆ ← h₅ ← h₄ ← h₃ ← h₂ ← h₁

At each step, gradient is multiplied by W_hh:
grad × W_hh × W_hh × W_hh × W_hh × ... = grad × (W_hh)¹⁰

If |W_hh| < 1: gradients → 0     (VANISHING — can't learn long dependencies)
If |W_hh| > 1: gradients → ∞     (EXPLODING — training becomes unstable)
```

**Solutions**:
- **LSTM** / **GRU**: Gating mechanisms that control information flow
- **Gradient clipping**: Cap gradient magnitude
- **Skip connections**: Shortcut paths for gradients

---

## Text Preprocessing in TensorFlow

### The Text-to-Numbers Pipeline

```
Raw Text → Tokenize → Pad/Truncate → Embed → Model

"I love TensorFlow" → [4, 15, 237] → [4, 15, 237, 0, 0] → [[0.2, ...], ...] → RNN
```

### TextVectorization Layer (Recommended)

```python
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers
import numpy as np

# Sample text data
texts = [
    "I love this movie",
    "This movie is terrible",
    "Great acting and storyline",
    "Worst film I have ever seen",
    "Absolutely brilliant performance",
]
labels = [1, 0, 1, 0, 1]  # 1=positive, 0=negative

# Create TextVectorization layer
vectorizer = layers.TextVectorization(
    max_tokens=10000,            # Vocabulary size limit
    output_mode='int',           # Output integer indices
    output_sequence_length=50,   # Pad/truncate to this length
    standardize='lower_and_strip_punctuation',  # Lowercase + remove punctuation
)

# Adapt the vectorizer to your data (builds vocabulary)
vectorizer.adapt(texts)

# See the vocabulary
print("Vocabulary:", vectorizer.get_vocabulary()[:20])
# ['', '[UNK]', 'movie', 'this', 'i', ...]
# Index 0 = padding, Index 1 = unknown/OOV

# Vectorize text
vectorized = vectorizer(texts)
print(vectorized.numpy())
# [[4 10 3 2  0  0 ... 0]
#  [3  2  7 11  0  0 ... 0]
#  ...]
```

### Different Output Modes

```python
# Integer indices (for Embedding layer input)
vec_int = layers.TextVectorization(output_mode='int', output_sequence_length=50)

# Multi-hot encoding (for bag-of-words models)
vec_multi_hot = layers.TextVectorization(output_mode='multi_hot', max_tokens=5000)

# TF-IDF (weighted bag-of-words)
vec_tfidf = layers.TextVectorization(output_mode='tf_idf', max_tokens=5000)

# Count (raw word counts)
vec_count = layers.TextVectorization(output_mode='count', max_tokens=5000)
```

### Manual Tokenization (When You Need More Control)

```python
# Using tf.keras.preprocessing (legacy but still common)
from tensorflow.keras.preprocessing.text import Tokenizer
from tensorflow.keras.preprocessing.sequence import pad_sequences

# Fit tokenizer
tokenizer = Tokenizer(num_words=10000, oov_token='<OOV>')
tokenizer.fit_on_texts(texts)

# Convert to sequences
sequences = tokenizer.texts_to_sequences(texts)
print(sequences)
# [[2, 3, 4, 5], [4, 5, 6, 7], ...]

# Pad sequences to uniform length
padded = pad_sequences(sequences, maxlen=50, padding='post', truncating='post')
# padding='post': pad zeros at the end
# padding='pre': pad zeros at the beginning (default, often better for RNNs)

# Word index
print(tokenizer.word_index)
# {'<OOV>': 1, 'movie': 2, 'i': 3, ...}
```

### Pre-padding vs Post-padding

```
Pre-padding (default — better for RNNs):
[0, 0, 0, 4, 15, 237]
↑ padding     ↑ actual words
RNN processes padding first, then the meaningful words are 
closest to the output — less information loss

Post-padding:
[4, 15, 237, 0, 0, 0]
↑ actual words    ↑ padding
RNN must "remember" through padding — more information loss

For bidirectional models: doesn't matter much
For attention-based models: use masking instead
```

---

## Embedding Layer

### What is an Embedding?

```
One-Hot Encoding (sparse, high-dimensional, no relationships):
"cat" → [0, 0, 1, 0, 0, 0, 0, 0, 0, 0]  (10,000-dim!)
"dog" → [0, 0, 0, 0, 1, 0, 0, 0, 0, 0]
"kitten" → [0, 0, 0, 0, 0, 0, 1, 0, 0, 0]
# "cat" and "kitten" look equally different from each other as "cat" and "airplane"

Word Embedding (dense, low-dimensional, captures meaning):
"cat"    → [0.2, -0.4, 0.7, 0.1]  (128-dim)
"dog"    → [0.3, -0.3, 0.6, 0.2]  ← similar to cat!
"kitten" → [0.2, -0.3, 0.8, 0.1]  ← very similar to cat!
"airplane" → [-0.8, 0.5, -0.2, 0.9]  ← very different
```

### Embedding Layer in Keras

```python
# The Embedding layer is a trainable lookup table
embedding = layers.Embedding(
    input_dim=10000,      # Vocabulary size
    output_dim=128,       # Embedding dimension
    input_length=50,      # Sequence length (optional but helpful)
    mask_zero=True,       # Treat 0 as padding (important!)
)

# Input: (batch_size, sequence_length) — integer indices
# Output: (batch_size, sequence_length, embedding_dim)

# Example
input_seq = tf.constant([[1, 5, 23, 0, 0]])  # (1, 5)
embedded = embedding(input_seq)
print(embedded.shape)  # (1, 5, 128)
```

### How Embedding Learns

```
Epoch 1: Random vectors
"good" → [0.42, -0.13, 0.87, ...]   (random)
"great" → [-0.55, 0.71, 0.23, ...]  (random)

After training on sentiment data:
"good" → [0.85, 0.72, -0.15, ...]
"great" → [0.83, 0.75, -0.12, ...]  ← similar direction!
"bad" → [-0.82, -0.69, 0.20, ...]   ← opposite direction!
```

### Using Pretrained Embeddings

```python
import numpy as np

# Load GloVe embeddings (download from nlp.stanford.edu)
def load_glove_embeddings(filepath, word_index, embedding_dim=100):
    """Load pretrained GloVe vectors"""
    embeddings_index = {}
    
    with open(filepath, encoding='utf-8') as f:
        for line in f:
            values = line.split()
            word = values[0]
            coefs = np.asarray(values[1:], dtype='float32')
            embeddings_index[word] = coefs
    
    print(f"Loaded {len(embeddings_index)} word vectors")
    
    # Create embedding matrix
    vocab_size = len(word_index) + 1
    embedding_matrix = np.zeros((vocab_size, embedding_dim))
    
    found = 0
    for word, idx in word_index.items():
        vector = embeddings_index.get(word)
        if vector is not None:
            embedding_matrix[idx] = vector
            found += 1
    
    print(f"Found vectors for {found}/{len(word_index)} words")
    return embedding_matrix

# Use in model
# embedding_matrix = load_glove_embeddings('glove.6B.100d.txt', tokenizer.word_index)

# Create embedding layer with pretrained weights
# embedding_layer = layers.Embedding(
#     input_dim=vocab_size,
#     output_dim=100,
#     weights=[embedding_matrix],
#     trainable=False,  # Freeze embeddings (or True to fine-tune)
#     mask_zero=True
# )
```

### Choosing Embedding Dimension

| Embedding Dim | Vocabulary Size | Use Case |
|---------------|----------------|----------|
| 50-100 | < 10K words | Small datasets, simple tasks |
| 128-256 | 10K-50K words | Most tasks |
| 300 | Any | GloVe/Word2Vec pretrained |
| 512-768 | Large | Complex tasks, when data is abundant |

> **Rule of thumb**: $\text{embedding\_dim} \approx \text{vocab\_size}^{0.25}$ is a reasonable starting point. For 10,000 words → 10 dimensions minimum, but 128 is safer.

---

## SimpleRNN in Keras

### Basic SimpleRNN

```python
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers

# Simple text classification model
model = keras.Sequential([
    # Input: integer sequences
    layers.Embedding(input_dim=10000, output_dim=64, mask_zero=True),
    # Shape: (batch, seq_len) → (batch, seq_len, 64)
    
    # RNN layer
    layers.SimpleRNN(
        units=64,               # Hidden state size
        activation='tanh',      # Default activation
        return_sequences=False,  # Only return last hidden state
        # return_sequences=True → return hidden state at EVERY time step
    ),
    # Shape: (batch, seq_len, 64) → (batch, 64)
    
    # Classification
    layers.Dense(1, activation='sigmoid')
    # Shape: (batch, 64) → (batch, 1)
])

model.summary()
```

### return_sequences Explained

```python
# return_sequences=False (default): output only the LAST hidden state
rnn = layers.SimpleRNN(64, return_sequences=False)
# Input: (batch, 10, 32)  → Output: (batch, 64)
# Use for: classification (one output per sequence)

# return_sequences=True: output hidden state at EVERY time step
rnn = layers.SimpleRNN(64, return_sequences=True)
# Input: (batch, 10, 32)  → Output: (batch, 10, 64)
# Use for: stacking RNN layers, sequence-to-sequence, NER
```

```
return_sequences visualization:

Input:     x₁    x₂    x₃    x₄    x₅
            │     │     │     │     │
RNN:       h₁ → h₂ → h₃ → h₄ → h₅

return_sequences=False:  → h₅           (last state only)
return_sequences=True:   → [h₁, h₂, h₃, h₄, h₅]  (all states)
```

### Stacking RNN Layers

```python
model = keras.Sequential([
    layers.Embedding(10000, 64, mask_zero=True),
    
    # First RNN: MUST return sequences for next RNN to process
    layers.SimpleRNN(64, return_sequences=True),
    
    # Second RNN: can return sequences or not
    layers.SimpleRNN(32, return_sequences=False),
    
    layers.Dense(1, activation='sigmoid')
])
```

> **Why SimpleRNN is rarely used in practice**: It suffers severely from the vanishing gradient problem. For sequences longer than ~10-20 steps, it can't learn long-range dependencies. **Always use LSTM or GRU instead.**

---

## LSTM — Long Short-Term Memory

### What is LSTM?

LSTM is a special type of RNN that can learn **long-range dependencies** by using a sophisticated gating mechanism to control what information to remember, forget, and output.

### The LSTM Architecture

```
                    ┌─────────────────────────────────────────┐
                    │              LSTM Cell                    │
                    │                                           │
C_{t-1} ──────────▶│──── × ────── + ──────────────────────────▶│── C_t
                    │    ↑         ↑                            │
                    │  f_gate    i_gate × C̃_t                  │
                    │    ↑         ↑     ↑                      │
                    │    │    ┌────┴─────┘                      │
                    │    │    │                                  │
h_{t-1} ──────────▶│    ├────┤                                 │
                    │    │    │    tanh(C_t) × o_gate ──────────▶│── h_t
x_t ──────────────▶│────┘    │         ↑                        │
                    │         │       o_gate                     │
                    │         │         ↑                        │
                    └─────────┴─────────┴───────────────────────┘

f_gate = Forget Gate:  "What to forget from old memory?"
i_gate = Input Gate:   "What new info to add?"
C̃_t   = Candidate:    "What new info is available?"
o_gate = Output Gate:  "What to output from memory?"
```

### LSTM Equations

**Forget gate** — decides what to throw away from cell state:
$$f_t = \sigma(W_f \cdot [h_{t-1}, x_t] + b_f)$$

**Input gate** — decides what new information to store:
$$i_t = \sigma(W_i \cdot [h_{t-1}, x_t] + b_i)$$

**Candidate values** — creates new candidate values:
$$\tilde{C}_t = \tanh(W_C \cdot [h_{t-1}, x_t] + b_C)$$

**Cell state update** — update the cell state:
$$C_t = f_t \odot C_{t-1} + i_t \odot \tilde{C}_t$$

**Output gate** — decides what to output:
$$o_t = \sigma(W_o \cdot [h_{t-1}, x_t] + b_o)$$

**Hidden state** — final output:
$$h_t = o_t \odot \tanh(C_t)$$

Where $\sigma$ is the sigmoid function and $\odot$ is element-wise multiplication.

### LSTM in Keras

```python
model = keras.Sequential([
    layers.Embedding(10000, 128, mask_zero=True),
    
    # LSTM layer
    layers.LSTM(
        units=128,              # Hidden state dimensionality
        activation='tanh',       # Cell state activation (default)
        recurrent_activation='sigmoid',  # Gate activation (default)
        return_sequences=False,  # Same as SimpleRNN
        dropout=0.2,            # Input dropout
        recurrent_dropout=0.2,  # Recurrent dropout (on hidden state)
    ),
    
    layers.Dense(64, activation='relu'),
    layers.Dropout(0.5),
    layers.Dense(1, activation='sigmoid')
])

model.compile(
    optimizer='adam',
    loss='binary_crossentropy',
    metrics=['accuracy']
)

model.summary()
```

### LSTM Parameter Count

```python
# LSTM(units=128), input_dim=64
# 
# Each gate has:
#   W (input weights):     input_dim × units = 64 × 128
#   U (recurrent weights): units × units = 128 × 128
#   b (bias):              units = 128
#
# 4 gates (forget, input, candidate, output):
#   params = 4 × (64 × 128 + 128 × 128 + 128)
#          = 4 × (8,192 + 16,384 + 128)
#          = 4 × 24,704
#          = 98,816
#
# Formula: 4 × [(input_dim + units) × units + units]
#        = 4 × (input_dim + units + 1) × units
```

### Stacked LSTM

```python
model = keras.Sequential([
    layers.Embedding(10000, 128, mask_zero=True),
    
    # Stack 1: return sequences for next LSTM
    layers.LSTM(128, return_sequences=True, dropout=0.2),
    
    # Stack 2: return sequences for next LSTM
    layers.LSTM(64, return_sequences=True, dropout=0.2),
    
    # Stack 3: final LSTM, return only last state
    layers.LSTM(32, return_sequences=False, dropout=0.2),
    
    layers.Dense(1, activation='sigmoid')
])
```

### Accessing LSTM States

```python
# return_state=True gives you both hidden state AND cell state
lstm = layers.LSTM(64, return_sequences=True, return_state=True)

# Returns: (output_sequences, last_hidden_state, last_cell_state)
inputs = tf.random.normal([32, 10, 128])  # (batch, timesteps, features)
sequences, last_h, last_c = lstm(inputs)

print(f"All sequences: {sequences.shape}")  # (32, 10, 64)
print(f"Last hidden:   {last_h.shape}")      # (32, 64)
print(f"Last cell:     {last_c.shape}")       # (32, 64)
```

### LSTM Analogy: The Conveyor Belt

```
Think of the cell state as a conveyor belt running through the factory:

┌────────────────────────────────────────────────┐
│  CONVEYOR BELT (Cell State C_t)                 │
│  ═══════════════════════════════════════════════│
│      ↑ remove      ↑ add new                   │
│    old info       information                   │
│  (forget gate)   (input gate)                   │
│                                                 │
│  The belt runs straight through with minimal    │
│  interaction → gradients flow easily!           │
│                                                 │
│  Gates control what gets ON and OFF the belt    │
└────────────────────────────────────────────────┘

This is why LSTM solves the vanishing gradient problem:
the gradient highway through the cell state allows
information to flow across many time steps.
```

---

## GRU — Gated Recurrent Unit

### What is GRU?

GRU is a simplified version of LSTM with only **2 gates** instead of 3. It merges the forget and input gates and combines the cell state and hidden state.

```
LSTM: 3 gates (forget, input, output) + cell state + hidden state
GRU:  2 gates (reset, update) + hidden state only
```

### GRU Architecture

```
            ┌────────────────────────────────┐
            │           GRU Cell              │
            │                                 │
h_{t-1} ──▶│  z_t = update gate              │
            │  r_t = reset gate               │
x_t ──────▶│                                 │──▶ h_t
            │  h̃_t = candidate (using r_t)   │
            │  h_t = (1-z_t)×h_{t-1} + z_t×h̃_t│
            └────────────────────────────────┘
```

### GRU Equations

$$z_t = \sigma(W_z \cdot [h_{t-1}, x_t] + b_z) \quad \text{(Update gate)}$$

$$r_t = \sigma(W_r \cdot [h_{t-1}, x_t] + b_r) \quad \text{(Reset gate)}$$

$$\tilde{h}_t = \tanh(W_h \cdot [r_t \odot h_{t-1}, x_t] + b_h) \quad \text{(Candidate)}$$

$$h_t = (1 - z_t) \odot h_{t-1} + z_t \odot \tilde{h}_t \quad \text{(Final state)}$$

### GRU in Keras

```python
model = keras.Sequential([
    layers.Embedding(10000, 128, mask_zero=True),
    
    layers.GRU(
        units=128,
        dropout=0.2,
        recurrent_dropout=0.2,
        return_sequences=False
    ),
    
    layers.Dense(64, activation='relu'),
    layers.Dropout(0.5),
    layers.Dense(1, activation='sigmoid')
])

model.compile(optimizer='adam', loss='binary_crossentropy', metrics=['accuracy'])
```

### LSTM vs GRU Comparison

| Feature | LSTM | GRU |
|---------|------|-----|
| Gates | 3 (forget, input, output) | 2 (reset, update) |
| States | Hidden state + Cell state | Hidden state only |
| Parameters | More (~33% more) | Fewer |
| Training speed | Slower | Faster |
| Long sequences | Slightly better | Comparable |
| Small datasets | May overfit more | Better |
| Recommendation | Default choice | When speed/simplicity matters |

> **In practice**: Both perform similarly. Start with GRU (faster), switch to LSTM if you need every bit of performance on very long sequences.

---

## Bidirectional RNNs

### Why Bidirectional?

```
Forward only:
"The movie was not good" → processes left to right
When processing "not", it doesn't know "good" is coming next

Bidirectional:
Forward:  "The" → "movie" → "was" → "not" → "good"
Backward: "good" → "not" → "was" → "movie" → "The"
→ At every position, the model knows BOTH past and future context!
```

### Bidirectional in Keras

```python
model = keras.Sequential([
    layers.Embedding(10000, 128, mask_zero=True),
    
    # Wrap any RNN layer with Bidirectional
    layers.Bidirectional(layers.LSTM(64, return_sequences=True, dropout=0.2)),
    # Output: (batch, seq_len, 128)  ← 64 forward + 64 backward = 128
    
    layers.Bidirectional(layers.LSTM(32, dropout=0.2)),
    # Output: (batch, 64)  ← 32 forward + 32 backward = 64
    
    layers.Dense(64, activation='relu'),
    layers.Dropout(0.5),
    layers.Dense(1, activation='sigmoid')
])

model.summary()
# Note: Bidirectional doubles the output dimension!
```

### Merge Modes

```python
# How to combine forward and backward outputs
layers.Bidirectional(layers.LSTM(64), merge_mode='concat')   # Default: [h_f; h_b] → 128
layers.Bidirectional(layers.LSTM(64), merge_mode='sum')      # h_f + h_b → 64
layers.Bidirectional(layers.LSTM(64), merge_mode='mul')      # h_f * h_b → 64
layers.Bidirectional(layers.LSTM(64), merge_mode='ave')      # (h_f + h_b)/2 → 64
```

> **When NOT to use Bidirectional**: Real-time streaming (can't see the future), autoregressive generation (predicting next word), any task where causality matters.

---

## Text Classification — Complete Walkthrough

### IMDB Sentiment Analysis

```python
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers
import numpy as np
import matplotlib.pyplot as plt

# ============================================================
# 1. LOAD DATA
# ============================================================
# IMDB dataset: 50,000 movie reviews (25K train, 25K test)
VOCAB_SIZE = 20000
MAX_LENGTH = 200

(X_train, y_train), (X_test, y_test) = keras.datasets.imdb.load_data(
    num_words=VOCAB_SIZE  # Keep only top N most frequent words
)

print(f"Training samples: {len(X_train)}")    # 25,000
print(f"Test samples: {len(X_test)}")         # 25,000
print(f"Sequence lengths: {[len(x) for x in X_train[:5]]}")
# e.g., [218, 189, 141, 550, 147] — variable length!


# ============================================================
# 2. PAD SEQUENCES
# ============================================================
from tensorflow.keras.preprocessing.sequence import pad_sequences

X_train = pad_sequences(X_train, maxlen=MAX_LENGTH, padding='pre', truncating='pre')
X_test = pad_sequences(X_test, maxlen=MAX_LENGTH, padding='pre', truncating='pre')

print(f"Padded shape: {X_train.shape}")  # (25000, 200)


# ============================================================
# 3. BUILD DATA PIPELINE
# ============================================================
BATCH_SIZE = 64
AUTOTUNE = tf.data.AUTOTUNE

train_dataset = (
    tf.data.Dataset.from_tensor_slices((X_train, y_train))
    .shuffle(25000)
    .batch(BATCH_SIZE)
    .prefetch(AUTOTUNE)
)

test_dataset = (
    tf.data.Dataset.from_tensor_slices((X_test, y_test))
    .batch(BATCH_SIZE)
    .cache()
    .prefetch(AUTOTUNE)
)


# ============================================================
# 4. BUILD MODEL
# ============================================================
def build_lstm_model():
    model = keras.Sequential([
        # Embedding layer
        layers.Embedding(
            input_dim=VOCAB_SIZE,
            output_dim=128,
            mask_zero=False,  # IMDB uses 0 for padding already
        ),
        
        # Bidirectional LSTM
        layers.Bidirectional(layers.LSTM(64, return_sequences=True, dropout=0.3)),
        layers.Bidirectional(layers.LSTM(32, dropout=0.3)),
        
        # Classification head
        layers.Dense(64, activation='relu'),
        layers.Dropout(0.5),
        layers.Dense(1, activation='sigmoid')
    ])
    return model

model = build_lstm_model()
model.summary()


# ============================================================
# 5. COMPILE
# ============================================================
model.compile(
    optimizer=keras.optimizers.Adam(learning_rate=1e-3),
    loss='binary_crossentropy',
    metrics=['accuracy']
)


# ============================================================
# 6. TRAIN
# ============================================================
callbacks = [
    keras.callbacks.EarlyStopping(
        monitor='val_accuracy',
        patience=3,
        restore_best_weights=True
    ),
    keras.callbacks.ReduceLROnPlateau(
        monitor='val_loss',
        factor=0.5,
        patience=2,
        min_lr=1e-6
    ),
]

history = model.fit(
    train_dataset,
    validation_data=test_dataset,
    epochs=20,
    callbacks=callbacks
)


# ============================================================
# 7. EVALUATE
# ============================================================
test_loss, test_acc = model.evaluate(test_dataset, verbose=0)
print(f"\nTest Accuracy: {test_acc:.4f}")
# Expected: ~0.85-0.88


# ============================================================
# 8. PLOT TRAINING HISTORY
# ============================================================
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 5))

ax1.plot(history.history['accuracy'], label='Train')
ax1.plot(history.history['val_accuracy'], label='Validation')
ax1.set_title('Accuracy')
ax1.set_xlabel('Epoch')
ax1.legend()

ax2.plot(history.history['loss'], label='Train')
ax2.plot(history.history['val_loss'], label='Validation')
ax2.set_title('Loss')
ax2.set_xlabel('Epoch')
ax2.legend()

plt.tight_layout()
plt.show()


# ============================================================
# 9. INFERENCE
# ============================================================
# To predict on new text, you need the word index
word_index = keras.datasets.imdb.get_word_index()
# Offset indices (IMDB reserves 0=padding, 1=start, 2=OOV)
word_index = {k: v + 3 for k, v in word_index.items()}
word_index["<pad>"] = 0
word_index["<start>"] = 1
word_index["<unk>"] = 2

def predict_sentiment(text, model, word_index, max_length=200):
    """Predict sentiment of raw text"""
    # Tokenize
    words = text.lower().split()
    sequence = [word_index.get(w, 2) for w in words]  # 2 = OOV
    
    # Pad
    padded = pad_sequences([sequence], maxlen=max_length, padding='pre')
    
    # Predict
    prediction = model.predict(padded, verbose=0)[0][0]
    
    sentiment = "Positive" if prediction > 0.5 else "Negative"
    confidence = prediction if prediction > 0.5 else 1 - prediction
    
    return sentiment, confidence

# Test
text = "This movie was absolutely wonderful and I loved every minute of it"
sentiment, confidence = predict_sentiment(text, model, word_index)
print(f"'{text}'")
print(f"Sentiment: {sentiment} (confidence: {confidence:.2%})")
```

### Alternative: Using TextVectorization (End-to-End)

```python
# This approach handles raw text strings directly

# Assume raw text data
import tensorflow_datasets as tfds

# Load raw IMDB
raw_train = tfds.load('imdb_reviews', split='train', as_supervised=True)
raw_test = tfds.load('imdb_reviews', split='test', as_supervised=True)

# Create TextVectorization layer
vectorizer = layers.TextVectorization(
    max_tokens=20000,
    output_mode='int',
    output_sequence_length=200
)

# Adapt on training text
train_text = raw_train.map(lambda text, label: text)
vectorizer.adapt(train_text.batch(1000))

# Build model with vectorizer INSIDE
model = keras.Sequential([
    vectorizer,                                    # Takes raw strings!
    layers.Embedding(20000, 128, mask_zero=True),
    layers.Bidirectional(layers.LSTM(64, dropout=0.3)),
    layers.Dense(64, activation='relu'),
    layers.Dropout(0.5),
    layers.Dense(1, activation='sigmoid')
])

# Now model accepts raw text
model.predict(["This movie was great!"])
```

---

## Sequence-to-Sequence Models

### What is Seq2Seq?

```
Input:  "How are you?"  (English)
Output: "Comment allez-vous?"  (French)

Encoder:
"How" → "are" → "you" → "?" → [context vector]

Decoder:
[context vector] → "Comment" → "allez" → "vous" → "?"
```

### Seq2Seq with Keras (Encoder-Decoder)

```python
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers

# Hyperparameters
ENCODER_VOCAB = 10000
DECODER_VOCAB = 10000
EMBEDDING_DIM = 256
UNITS = 512
MAX_ENCODER_LEN = 50
MAX_DECODER_LEN = 50

# ============================================================
# ENCODER
# ============================================================
encoder_inputs = keras.Input(shape=(MAX_ENCODER_LEN,), name='encoder_input')
encoder_embedding = layers.Embedding(ENCODER_VOCAB, EMBEDDING_DIM, mask_zero=True)
encoder_lstm = layers.LSTM(UNITS, return_state=True, name='encoder_lstm')

# Encode the input sequence
encoder_embedded = encoder_embedding(encoder_inputs)
encoder_outputs, state_h, state_c = encoder_lstm(encoder_embedded)
# We discard encoder_outputs and only keep the states
encoder_states = [state_h, state_c]

# ============================================================
# DECODER
# ============================================================
decoder_inputs = keras.Input(shape=(MAX_DECODER_LEN,), name='decoder_input')
decoder_embedding = layers.Embedding(DECODER_VOCAB, EMBEDDING_DIM, mask_zero=True)
decoder_lstm = layers.LSTM(UNITS, return_sequences=True, return_state=True, name='decoder_lstm')
decoder_dense = layers.Dense(DECODER_VOCAB, activation='softmax', name='decoder_output')

# Decode using encoder's final state as initial state
decoder_embedded = decoder_embedding(decoder_inputs)
decoder_outputs, _, _ = decoder_lstm(decoder_embedded, initial_state=encoder_states)
decoder_predictions = decoder_dense(decoder_outputs)

# ============================================================
# FULL MODEL
# ============================================================
model = keras.Model(
    inputs=[encoder_inputs, decoder_inputs],
    outputs=decoder_predictions
)

model.compile(
    optimizer='adam',
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

model.summary()
```

### Teacher Forcing

```
Training with Teacher Forcing:
- Input to decoder: actual target sequence (shifted right)
- The decoder sees the CORRECT previous word, not its own prediction

Target:    <start> Comment allez vous ? <end>
Input:     <start> Comment allez vous ?
Output:    Comment  allez   vous  ?  <end>

Without teacher forcing (at inference):
- Input to decoder at each step: decoder's OWN previous prediction
- Errors can compound!
```

---

## Text Generation

### Character-Level Text Generation

```python
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers
import numpy as np

# ============================================================
# 1. PREPARE DATA
# ============================================================
# Sample text (in practice, use a large corpus)
text = """To be or not to be that is the question
Whether tis nobler in the mind to suffer
The slings and arrows of outrageous fortune
Or to take arms against a sea of troubles"""

# Create character-level vocabulary
chars = sorted(set(text))
char_to_idx = {c: i for i, c in enumerate(chars)}
idx_to_char = {i: c for c, i in char_to_idx.items()}
vocab_size = len(chars)

print(f"Vocabulary size: {vocab_size}")
print(f"Characters: {chars}")

# Convert text to integers
text_as_int = np.array([char_to_idx[c] for c in text])

# Create training sequences
SEQ_LENGTH = 40
BATCH_SIZE = 32

dataset = tf.data.Dataset.from_tensor_slices(text_as_int)

# Create input-target pairs: input=[0:39], target=[1:40]
sequences = dataset.batch(SEQ_LENGTH + 1, drop_remainder=True)

def split_input_target(chunk):
    input_text = chunk[:-1]
    target_text = chunk[1:]
    return input_text, target_text

dataset = sequences.map(split_input_target)
dataset = dataset.shuffle(1000).batch(BATCH_SIZE, drop_remainder=True).prefetch(tf.data.AUTOTUNE)


# ============================================================
# 2. BUILD MODEL
# ============================================================
model = keras.Sequential([
    layers.Embedding(vocab_size, 256, batch_input_shape=[BATCH_SIZE, None]),
    layers.LSTM(512, return_sequences=True, stateful=False, dropout=0.2),
    layers.LSTM(512, return_sequences=True, stateful=False, dropout=0.2),
    layers.Dense(vocab_size)  # No softmax — using from_logits=True in loss
])

model.compile(
    optimizer='adam',
    loss=keras.losses.SparseCategoricalCrossentropy(from_logits=True),
)


# ============================================================
# 3. TRAIN
# ============================================================
history = model.fit(dataset, epochs=50)


# ============================================================
# 4. GENERATE TEXT
# ============================================================
def generate_text(model, start_string, num_generate=200, temperature=1.0):
    """Generate text character by character"""
    # Convert start string to numbers
    input_indices = [char_to_idx[c] for c in start_string]
    input_indices = tf.expand_dims(input_indices, 0)  # Add batch dimension
    
    generated = []
    
    for _ in range(num_generate):
        # Get predictions
        predictions = model(input_indices)
        # Remove batch dimension
        predictions = tf.squeeze(predictions, 0)
        # Use last prediction only
        predictions = predictions[-1, :] / temperature
        
        # Sample from the distribution
        predicted_id = tf.random.categorical(
            tf.expand_dims(predictions, 0), num_samples=1
        )[-1, 0].numpy()
        
        # Append to input for next iteration
        input_indices = tf.expand_dims(
            tf.concat([input_indices[0], [predicted_id]], axis=0), 0
        )
        
        generated.append(idx_to_char[predicted_id])
    
    return start_string + ''.join(generated)

# Generate with different temperatures
print("Temperature 0.2 (conservative):")
print(generate_text(model, "To be", temperature=0.2))

print("\nTemperature 1.0 (balanced):")
print(generate_text(model, "To be", temperature=1.0))

print("\nTemperature 1.5 (creative):")
print(generate_text(model, "To be", temperature=1.5))
```

### Temperature Explained

```
Temperature controls randomness in generation:

Logits: [2.0, 1.0, 0.5]

Temperature = 0.1 (very conservative):
  logits/0.1 = [20.0, 10.0, 5.0]
  softmax → [0.9999, 0.00005, 0.0000]  ← almost always picks the top word

Temperature = 1.0 (neutral):
  logits/1.0 = [2.0, 1.0, 0.5]
  softmax → [0.59, 0.22, 0.13]  ← balanced sampling

Temperature = 2.0 (creative):
  logits/2.0 = [1.0, 0.5, 0.25]
  softmax → [0.42, 0.27, 0.21]  ← more uniform → more diverse

Low temp → repetitive but coherent
High temp → diverse but may be nonsensical
```

---

## Named Entity Recognition (NER)

### Sequence Labeling with RNNs

```python
# NER: classify each word in a sequence
# Input:  "John lives in New York"
# Output: [PER,  O,    O,  LOC, LOC]

import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers

# Tag scheme
tag_names = ['O', 'B-PER', 'I-PER', 'B-LOC', 'I-LOC', 'B-ORG', 'I-ORG']
num_tags = len(tag_names)

VOCAB_SIZE = 20000
EMBEDDING_DIM = 128
MAX_LENGTH = 100

model = keras.Sequential([
    layers.Embedding(VOCAB_SIZE, EMBEDDING_DIM, mask_zero=True),
    
    # Bidirectional — context from both directions is crucial for NER
    layers.Bidirectional(layers.LSTM(64, return_sequences=True, dropout=0.3)),
    layers.Bidirectional(layers.LSTM(32, return_sequences=True, dropout=0.3)),
    
    # TimeDistributed applies Dense to each time step independently
    layers.TimeDistributed(layers.Dense(num_tags, activation='softmax'))
])

model.compile(
    optimizer='adam',
    loss='sparse_categorical_crossentropy',
    metrics=['accuracy']
)

# Input shape:  (batch, seq_len) → integers
# Output shape: (batch, seq_len, num_tags) → tag probabilities per word
model.summary()
```

### TimeDistributed Explained

```
Without TimeDistributed:
Dense(7) applied to (batch, 64) → (batch, 7)
One prediction per sequence

With TimeDistributed(Dense(7)):
Dense(7) applied to (batch, seq_len, 64) → (batch, seq_len, 7)
One prediction per TIME STEP

┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
│Dense│ │Dense│ │Dense│ │Dense│ │Dense│
│(7)  │ │(7)  │ │(7)  │ │(7)  │ │(7)  │  (same weights!)
└──▲──┘ └──▲──┘ └──▲──┘ └──▲──┘ └──▲──┘
   │       │       │       │       │
  h₁      h₂      h₃      h₄      h₅
  ↓        ↓       ↓       ↓       ↓
 PER       O       O      LOC     LOC
"John"  "lives"  "in"   "New"   "York"
```

---

## Advanced Techniques

### 1. Attention Mechanism for RNNs

```python
# Simple attention mechanism (Bahdanau attention)
class BahdanauAttention(layers.Layer):
    def __init__(self, units):
        super().__init__()
        self.W1 = layers.Dense(units)  # Encoder projection
        self.W2 = layers.Dense(units)  # Decoder projection
        self.V = layers.Dense(1)       # Score

    def call(self, query, values):
        """
        query: decoder hidden state (batch, units)
        values: encoder outputs (batch, seq_len, units)
        """
        # Expand query dimensions for broadcasting
        query_with_time = tf.expand_dims(query, 1)  # (batch, 1, units)
        
        # Score each encoder step
        score = self.V(
            tf.nn.tanh(self.W1(values) + self.W2(query_with_time))
        )  # (batch, seq_len, 1)
        
        # Attention weights (softmax over time steps)
        attention_weights = tf.nn.softmax(score, axis=1)  # (batch, seq_len, 1)
        
        # Weighted sum of encoder outputs
        context = attention_weights * values  # (batch, seq_len, units)
        context = tf.reduce_sum(context, axis=1)  # (batch, units)
        
        return context, attention_weights
```

### 2. 1D Convolution + RNN Hybrid

```python
# CNN extracts local features, RNN captures long-range dependencies
model = keras.Sequential([
    layers.Embedding(20000, 128, mask_zero=False),
    
    # 1D Conv for local pattern extraction
    layers.Conv1D(128, 5, padding='same', activation='relu'),
    layers.MaxPooling1D(2),
    
    layers.Conv1D(64, 5, padding='same', activation='relu'),
    layers.MaxPooling1D(2),
    
    # RNN for sequential dependencies
    layers.Bidirectional(layers.LSTM(64, dropout=0.3)),
    
    layers.Dense(64, activation='relu'),
    layers.Dropout(0.5),
    layers.Dense(1, activation='sigmoid')
])
```

### 3. Multi-Input Model (Functional API)

```python
# Combine text with metadata (e.g., user rating + review text)

# Text input
text_input = keras.Input(shape=(200,), dtype='int32', name='text')
x = layers.Embedding(20000, 128)(text_input)
x = layers.Bidirectional(layers.LSTM(64, dropout=0.3))(x)
x = layers.Dense(64, activation='relu')(x)

# Metadata input (e.g., movie genre, year, etc.)
meta_input = keras.Input(shape=(5,), dtype='float32', name='metadata')
y = layers.Dense(32, activation='relu')(meta_input)

# Combine
combined = layers.Concatenate()([x, y])
combined = layers.Dense(64, activation='relu')(combined)
combined = layers.Dropout(0.5)(combined)
output = layers.Dense(1, activation='sigmoid')(combined)

model = keras.Model(inputs=[text_input, meta_input], outputs=output)

model.compile(optimizer='adam', loss='binary_crossentropy', metrics=['accuracy'])

# Train with dictionary input
# model.fit({'text': X_text, 'metadata': X_meta}, y_train, ...)
```

### 4. Gradient Clipping

```python
# Prevent exploding gradients in RNNs
optimizer = keras.optimizers.Adam(
    learning_rate=1e-3,
    clipnorm=1.0,     # Clip gradient norm to max 1.0
    # OR
    # clipvalue=0.5,  # Clip gradient values to [-0.5, 0.5]
)

model.compile(optimizer=optimizer, loss='binary_crossentropy')
```

### 5. Masking for Variable-Length Sequences

```python
# Option 1: mask_zero=True in Embedding
layers.Embedding(10000, 128, mask_zero=True)
# Tells subsequent layers to ignore positions where input = 0 (padding)

# Option 2: Explicit Masking layer
model = keras.Sequential([
    layers.Embedding(10000, 128),
    layers.Masking(mask_value=0.0),   # Mask where value = 0
    layers.LSTM(64),
    layers.Dense(1, activation='sigmoid')
])

# Why masking matters:
# Without masking: LSTM processes padding tokens → wastes computation, adds noise
# With masking: LSTM skips padding tokens → faster, cleaner
```

---

## Common Mistakes

### 1. Forgetting return_sequences Between Stacked RNNs

```python
# ❌ WRONG — second LSTM gets 2D input but expects 3D
model = keras.Sequential([
    layers.Embedding(10000, 128),
    layers.LSTM(64),                  # Output: (batch, 64) — 2D!
    layers.LSTM(32),                  # ERROR: expects 3D input!
    layers.Dense(1, activation='sigmoid')
])

# ✅ CORRECT — first LSTM returns sequences
model = keras.Sequential([
    layers.Embedding(10000, 128),
    layers.LSTM(64, return_sequences=True),  # Output: (batch, seq, 64) — 3D
    layers.LSTM(32),                          # Works!
    layers.Dense(1, activation='sigmoid')
])
```

### 2. Not Using Masking with Padded Sequences

```python
# ❌ WRONG — LSTM treats padding as real data
layers.Embedding(10000, 128)  # mask_zero defaults to False

# ✅ CORRECT — skip padding positions
layers.Embedding(10000, 128, mask_zero=True)
```

### 3. Wrong Padding Direction

```python
# ❌ WRONG for unidirectional RNNs — post-padding
# RNN must "remember through" padding to reach actual content
padded = pad_sequences(seqs, padding='post')

# ✅ CORRECT — pre-padding (default)
# Actual content is closest to the output
padded = pad_sequences(seqs, padding='pre')
```

### 4. Using recurrent_dropout with CuDNN

```python
# ❌ WRONG — recurrent_dropout disables CuDNN acceleration (10× slower!)
layers.LSTM(128, recurrent_dropout=0.2)  # Falls back to slow implementation

# ✅ CORRECT — use regular dropout (CuDNN compatible)
layers.LSTM(128, dropout=0.2)

# Or use Dropout layer between RNNs
layers.LSTM(128, return_sequences=True),
layers.Dropout(0.3),
layers.LSTM(64),
```

> **CuDNN-compatible LSTM requirements**: `activation='tanh'`, `recurrent_activation='sigmoid'`, `recurrent_dropout=0`, `unroll=False`, `use_bias=True`. Any deviation → 10× slower.

### 5. Too Many RNN Layers

```python
# ❌ OVERKILL — 5 stacked LSTMs
model = keras.Sequential([
    layers.Embedding(10000, 128),
    layers.LSTM(256, return_sequences=True),
    layers.LSTM(256, return_sequences=True),
    layers.LSTM(128, return_sequences=True),
    layers.LSTM(128, return_sequences=True),
    layers.LSTM(64),
    layers.Dense(1, activation='sigmoid')
])
# Slow training, likely overfitting, diminishing returns

# ✅ BETTER — 1-2 layers with Bidirectional
model = keras.Sequential([
    layers.Embedding(10000, 128),
    layers.Bidirectional(layers.LSTM(64, return_sequences=True, dropout=0.3)),
    layers.Bidirectional(layers.LSTM(32, dropout=0.3)),
    layers.Dense(1, activation='sigmoid')
])
```

### 6. Not Clipping Gradients

```python
# ❌ WRONG — RNNs are prone to exploding gradients
model.compile(optimizer='adam', loss='binary_crossentropy')

# ✅ CORRECT — clip gradients
model.compile(
    optimizer=keras.optimizers.Adam(clipnorm=1.0),
    loss='binary_crossentropy'
)
```

---

## Interview Questions

### Q1: Explain the vanishing gradient problem in RNNs and how LSTM solves it.

**Answer**: In vanilla RNNs, during backpropagation through time (BPTT), gradients are multiplied by the recurrent weight matrix at each time step. If this matrix has eigenvalues < 1, gradients shrink exponentially → **vanishing gradients** → can't learn long-range dependencies.

LSTM solves this with the **cell state** — a highway that runs through all time steps with minimal modification. The gradient can flow along this highway without multiplication by weight matrices. The **forget gate** uses addition (not multiplication) to update the cell state:

$$C_t = f_t \odot C_{t-1} + i_t \odot \tilde{C}_t$$

This additive update preserves gradients much better than the multiplicative update in vanilla RNNs.

### Q2: What is the difference between LSTM and GRU?

**Answer**:
- **LSTM**: 3 gates (forget, input, output), separate cell state and hidden state, more parameters
- **GRU**: 2 gates (reset, update), single hidden state, fewer parameters

GRU merges the forget and input gates into an "update gate" and combines the cell/hidden states. GRU trains faster and performs comparably on most tasks. LSTM may be slightly better on very long sequences. In practice, try GRU first (faster), switch to LSTM if needed.

### Q3: What does `return_sequences=True` do and when should you use it?

**Answer**: By default (`return_sequences=False`), an RNN returns only the **last** hidden state — shape `(batch, units)`. With `return_sequences=True`, it returns hidden states at **every time step** — shape `(batch, timesteps, units)`.

Use `return_sequences=True` when:
1. **Stacking RNN layers** — the next RNN needs 3D input
2. **Sequence-to-sequence** tasks — need output at every position
3. **NER/POS tagging** — classify each token
4. **Attention mechanisms** — need all encoder states

### Q4: Explain word embeddings and why they're better than one-hot encoding.

**Answer**:
- **One-hot**: Each word is a sparse vector of size V (vocabulary). "cat" = [0,0,1,0,...,0]. No semantic information. "cat" and "kitten" are equally distant from each other as "cat" and "airplane".
- **Embedding**: Each word is a dense vector of size d (typically 128-300). Trained to capture semantic relationships. Similar words have similar vectors.

Benefits:
1. **Dimensionality**: 300 vs 10,000+ dimensions
2. **Semantics**: Captures meaning and relationships (king - man + woman ≈ queen)
3. **Generalization**: Unseen combinations can be handled through similar embeddings
4. **Trainable**: Embeddings learn task-specific representations

### Q5: What is teacher forcing and what problem does it cause?

**Answer**: **Teacher forcing** means feeding the **ground truth** previous token as input to the decoder during training, rather than the decoder's own prediction.

**Problem — Exposure bias**: During training, the decoder always sees correct tokens. During inference, it sees its own (potentially wrong) predictions. If it makes one mistake, subsequent predictions may be worse because it never learned to recover from errors.

**Solutions**:
1. **Scheduled sampling** — gradually replace teacher forcing with model predictions during training
2. **Beam search** — explore multiple hypotheses at inference time
3. **Reinforcement learning** — train with sequence-level rewards

### Q6: When would you use RNNs vs Transformers?

**Answer**:
| Criterion | RNN/LSTM | Transformer |
|-----------|----------|-------------|
| Data size | Small-medium | Large |
| Sequence length | Short-medium | Any |
| Real-time/streaming | ✅ Better | ❌ Needs full sequence |
| Memory efficiency | ✅ Lower | ❌ $O(n^2)$ attention |
| Training speed | ❌ Sequential | ✅ Parallelizable |
| State of the art | No | Yes |
| Simplicity | ✅ Simpler | ❌ Complex |
| Time series | ✅ Still competitive | ✅ Good |

Use RNNs for: simple NLP tasks, time series, streaming data, low-resource settings.
Use Transformers for: machine translation, text generation, anything with enough data.

### Q7: What is Bidirectional RNN and when should you NOT use it?

**Answer**: A Bidirectional RNN processes the sequence in both directions (forward and backward) and concatenates the outputs. This gives each position context from both the past and the future.

**Don't use it when**:
1. **Autoregressive generation** — predicting the next word (can't see the future)
2. **Real-time streaming** — data arrives sequentially
3. **Causal tasks** — where future information would be "cheating" (e.g., stock prediction)
4. **Online/real-time inference** — needs the full sequence before processing

---

## Quick Reference

### Model Templates

```python
# Binary Text Classification (simplest)
model = keras.Sequential([
    layers.Embedding(vocab_size, 128, mask_zero=True),
    layers.Bidirectional(layers.LSTM(64, dropout=0.3)),
    layers.Dense(1, activation='sigmoid')
])

# Multi-class Text Classification
model = keras.Sequential([
    layers.Embedding(vocab_size, 128, mask_zero=True),
    layers.Bidirectional(layers.LSTM(64, dropout=0.3)),
    layers.Dense(num_classes, activation='softmax')
])

# Sequence Labeling (NER, POS tagging)
model = keras.Sequential([
    layers.Embedding(vocab_size, 128, mask_zero=True),
    layers.Bidirectional(layers.LSTM(64, return_sequences=True, dropout=0.3)),
    layers.TimeDistributed(layers.Dense(num_tags, activation='softmax'))
])

# Time Series Prediction
model = keras.Sequential([
    layers.LSTM(64, return_sequences=True, input_shape=(timesteps, features)),
    layers.LSTM(32),
    layers.Dense(1)  # Linear activation for regression
])
```

### Layer Cheat Sheet

| Layer | Input Shape | Output Shape | Use Case |
|-------|------------|-------------|----------|
| `Embedding(V, D)` | `(batch, seq)` int | `(batch, seq, D)` | Word → vector |
| `LSTM(U)` | `(batch, seq, F)` | `(batch, U)` | Sequence encoding |
| `LSTM(U, return_seq=True)` | `(batch, seq, F)` | `(batch, seq, U)` | Stack RNNs, seq labeling |
| `Bidirectional(LSTM(U))` | `(batch, seq, F)` | `(batch, 2*U)` | Both directions |
| `TimeDistributed(Dense(N))` | `(batch, seq, F)` | `(batch, seq, N)` | Per-timestep classification |
| `GRU(U)` | `(batch, seq, F)` | `(batch, U)` | Faster LSTM alternative |

### Hyperparameter Guide

| Parameter | Typical Values | Notes |
|-----------|---------------|-------|
| Vocabulary size | 10K-50K | Larger for diverse text |
| Embedding dim | 64-300 | 128 is safe default |
| LSTM units | 32-512 | 64-128 for most tasks |
| Num layers | 1-3 | 1-2 usually sufficient |
| Dropout | 0.2-0.5 | Start with 0.3 |
| Sequence length | 50-500 | Task dependent |
| Batch size | 32-128 | 64 is safe default |
| Learning rate | 1e-3 (Adam) | Reduce if unstable |
| Gradient clipping | clipnorm=1.0 | Always for RNNs |

### CuDNN Compatibility (10× Speed)

For GPU-accelerated LSTM/GRU, ensure:
- `activation='tanh'` (LSTM) / `'tanh'` (GRU)
- `recurrent_activation='sigmoid'`
- `recurrent_dropout=0` (no recurrent dropout!)
- `unroll=False`
- `use_bias=True`
- `reset_after=True` (GRU only)

### Decision Tree: Which RNN?

```
Task type?
├── Classification (one label per sequence)
│   ├── Short text (< 50 words) → Bidirectional GRU
│   └── Long text (50+ words) → Bidirectional LSTM
├── Sequence labeling (one label per token)
│   └── Bidirectional LSTM + return_sequences=True + TimeDistributed
├── Generation (predict next token)
│   └── Unidirectional LSTM (NO bidirectional!)
├── Seq2Seq (input → different output sequence)
│   └── Encoder-Decoder LSTM (or use Transformer)
└── Time series
    ├── Short-term → GRU
    └── Long-term → LSTM
```

---

*Previous: [05-CNN-with-TensorFlow.md](./05-CNN-with-TensorFlow.md)*
*Next: [07-Transfer-Learning-TF.md](./07-Transfer-Learning-TF.md)*
