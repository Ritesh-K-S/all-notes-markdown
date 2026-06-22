# Chapter 05: Attention and Transformers for NLP

## Table of Contents
- [Attention Mechanism for NLP](#attention-mechanism-for-nlp)
- [From Seq2Seq Attention to Self-Attention](#from-seq2seq-attention-to-self-attention)
- [The Transformer Architecture — NLP Perspective](#the-transformer-architecture--nlp-perspective)
- [BERT — Bidirectional Encoder Representations from Transformers](#bert--bidirectional-encoder-representations-from-transformers)
- [GPT — Generative Pre-trained Transformer](#gpt--generative-pre-trained-transformer)
- [BERT vs GPT — The Great Divide](#bert-vs-gpt--the-great-divide)
- [Other Key Transformer Models for NLP](#other-key-transformer-models-for-nlp)
- [Tokenization in Transformer Models](#tokenization-in-transformer-models)
- [Transfer Learning in NLP](#transfer-learning-in-nlp)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Attention Mechanism for NLP

### What It Is
Attention is a mechanism that lets a model **"look back"** at all parts of the input when producing each part of the output, instead of compressing everything into a single vector. Think of it as a student answering a test question by flipping back to the relevant page in the textbook — instead of relying on memory alone.

> For the full mathematical derivation of attention, self-attention, and multi-head attention, see [Deep Learning Chapter 06](../Deep-Learning/06-Attention-and-Transformers.md). This chapter focuses on **NLP-specific applications and models**.

### Why It Matters for NLP
- **Solved the Seq2Seq bottleneck**: Long sentences no longer lose information
- **Enabled BERT and GPT** — the foundation of all modern NLP
- **Word relationships**: Attention naturally captures which words relate to which (e.g., "it" refers to "cat")
- **Parallelizable**: Unlike RNNs, attention processes all positions simultaneously

### Attention in Machine Translation — The Origin Story

The original Bahdanau attention (2014) was designed specifically for NLP translation:

```
Source:  "The  cat  sat  on  the  mat"
          ↑    ↑↑↑   ↑    ↑    ↑    ↑
          │    ███   │    │    │    │     ← attention weights
          │    ███   │    │    │    │        (darker = higher)
Target:       "chat"

When generating "chat", the model attends most strongly to "cat"
```

#### How Attention Weights Work

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class BahdanauAttention(nn.Module):
    """
    Additive attention (Bahdanau et al., 2014).
    Used in Seq2Seq models to let the decoder focus on 
    relevant parts of the encoded input.
    """
    def __init__(self, encoder_dim, decoder_dim, attention_dim):
        super().__init__()
        # Project encoder and decoder states to same dimension
        self.W_encoder = nn.Linear(encoder_dim, attention_dim, bias=False)
        self.W_decoder = nn.Linear(decoder_dim, attention_dim, bias=False)
        # Score vector
        self.v = nn.Linear(attention_dim, 1, bias=False)
    
    def forward(self, encoder_outputs, decoder_hidden):
        """
        Args:
            encoder_outputs: (batch, src_len, encoder_dim) — all encoder hidden states
            decoder_hidden: (batch, decoder_dim) — current decoder state
        Returns:
            context: (batch, encoder_dim) — weighted sum of encoder outputs
            weights: (batch, src_len) — attention distribution
        """
        # Project both to attention space
        encoder_proj = self.W_encoder(encoder_outputs)  # (batch, src_len, attn_dim)
        decoder_proj = self.W_decoder(decoder_hidden).unsqueeze(1)  # (batch, 1, attn_dim)
        
        # Compute alignment scores
        energy = torch.tanh(encoder_proj + decoder_proj)  # (batch, src_len, attn_dim)
        scores = self.v(energy).squeeze(-1)               # (batch, src_len)
        
        # Normalize to probabilities
        weights = F.softmax(scores, dim=1)  # (batch, src_len)
        
        # Weighted sum of encoder outputs
        context = torch.bmm(weights.unsqueeze(1), encoder_outputs).squeeze(1)
        
        return context, weights


class LuongAttention(nn.Module):
    """
    Multiplicative attention (Luong et al., 2015).
    Simpler and often faster than Bahdanau.
    
    Three variants:
    - dot: score = decoder · encoder  (simplest, no extra params)
    - general: score = decoder · W · encoder
    - concat: same as Bahdanau
    """
    def __init__(self, encoder_dim, decoder_dim, method='general'):
        super().__init__()
        self.method = method
        if method == 'general':
            self.W = nn.Linear(encoder_dim, decoder_dim, bias=False)
    
    def forward(self, encoder_outputs, decoder_hidden):
        if self.method == 'dot':
            # Requires encoder_dim == decoder_dim
            scores = torch.bmm(
                encoder_outputs,
                decoder_hidden.unsqueeze(2)
            ).squeeze(2)
        
        elif self.method == 'general':
            scores = torch.bmm(
                self.W(encoder_outputs),
                decoder_hidden.unsqueeze(2)
            ).squeeze(2)
        
        weights = F.softmax(scores, dim=1)
        context = torch.bmm(weights.unsqueeze(1), encoder_outputs).squeeze(1)
        
        return context, weights

# Comparison
print("Bahdanau: additive, slower, works when dims differ")
print("Luong-dot: fastest, requires same dims")
print("Luong-general: good balance, most commonly used")
```

### Visualizing Attention in Translation

```python
import matplotlib.pyplot as plt
import numpy as np

def plot_attention(source_words, target_words, attention_matrix):
    """
    Visualize attention weights as a heatmap.
    Rows = target (generated), Columns = source (input).
    """
    fig, ax = plt.subplots(figsize=(8, 6))
    
    im = ax.imshow(attention_matrix, cmap='Blues', aspect='auto')
    
    ax.set_xticks(range(len(source_words)))
    ax.set_yticks(range(len(target_words)))
    ax.set_xticklabels(source_words, rotation=45, ha='right')
    ax.set_yticklabels(target_words)
    
    # Add weight values in cells
    for i in range(len(target_words)):
        for j in range(len(source_words)):
            ax.text(j, i, f'{attention_matrix[i][j]:.2f}',
                   ha='center', va='center', fontsize=8)
    
    ax.set_xlabel('Source (English)')
    ax.set_ylabel('Target (French)')
    ax.set_title('Attention Weights')
    plt.colorbar(im)
    plt.tight_layout()
    plt.savefig('attention_heatmap.png', dpi=150)
    plt.show()

# Simulated attention weights for English→French
source = ["The", "cat", "sat", "on", "the", "mat"]
target = ["Le", "chat", "assis", "sur", "le", "tapis"]

# Ideally near-diagonal for similar word order languages
attention = np.array([
    [0.7, 0.1, 0.05, 0.05, 0.05, 0.05],  # "Le" → "The"
    [0.05, 0.8, 0.05, 0.02, 0.03, 0.05],  # "chat" → "cat"
    [0.02, 0.05, 0.8, 0.05, 0.03, 0.05],  # "assis" → "sat"
    [0.02, 0.03, 0.05, 0.8, 0.05, 0.05],  # "sur" → "on"
    [0.05, 0.05, 0.02, 0.05, 0.75, 0.08],  # "le" → "the"
    [0.02, 0.03, 0.05, 0.05, 0.05, 0.8],  # "tapis" → "mat"
])

plot_attention(source, target, attention)
```

---

## From Seq2Seq Attention to Self-Attention

### The Key Insight
Seq2Seq attention lets the **decoder** attend to the **encoder**. But what if we let each word in a sentence attend to **every other word in the same sentence**? This is **self-attention** — and it's the core of the Transformer.

### Why Self-Attention Changes Everything

```
Sentence: "The animal didn't cross the street because it was too tired"

What does "it" refer to?
  → A human instantly knows "it" = "animal" (not "street")

Self-attention can learn this:
  "it" attends strongly to "animal" (high weight)
  "it" attends weakly to "street" (low weight)
  → The model learns coreference resolution automatically!
```

### Self-Attention vs RNN — Speed Comparison

| Operation | RNN | Self-Attention |
|-----------|-----|---------------|
| Sequential steps | O(n) — must process one by one | O(1) — all positions in parallel |
| Max path length | O(n) — info from step 1 takes n steps to reach step n | O(1) — direct connection between any two positions |
| Computation per layer | O(n · d²) | O(n² · d) |
| Best for | Short sequences, streaming | Long sequences, when parallelism matters |

> **Key trade-off**: Self-attention is O(n²) in sequence length (every token attends to every other token). This is why Transformers struggle with very long sequences (>4K tokens without special techniques).

---

## The Transformer Architecture — NLP Perspective

> For the full architecture breakdown (Q/K/V, multi-head attention, positional encoding math), see [Deep Learning Chapter 06](../Deep-Learning/06-Attention-and-Transformers.md). Here we focus on how Transformers are used specifically for NLP.

### The Two Paradigms: Encoder vs Decoder

The original Transformer ("Attention Is All You Need", 2017) had both encoder and decoder. But NLP research split into two camps:

```
┌─────────────────────────────────────────────────────────┐
│                  Original Transformer                    │
│              (Encoder-Decoder, for translation)          │
└───────────┬───────────────────────────┬─────────────────┘
            │                           │
            ▼                           ▼
    ┌───────────────┐           ┌───────────────┐
    │ ENCODER-ONLY  │           │ DECODER-ONLY  │
    │               │           │               │
    │  BERT (2018)  │           │  GPT (2018)   │
    │  RoBERTa      │           │  GPT-2, GPT-3 │
    │  ALBERT       │           │  GPT-4        │
    │  DeBERTa      │           │  LLaMA        │
    │  ELECTRA      │           │  Mistral      │
    │               │           │               │
    │ Bidirectional │           │ Autoregressive│
    │ Understanding │           │ Generation    │
    └───────────────┘           └───────────────┘
    
    ┌───────────────┐
    │  ENC-DECODER  │
    │               │
    │  T5, BART     │
    │  mBART, FLAN  │
    │               │
    │ Seq2Seq tasks │
    └───────────────┘
```

---

## BERT — Bidirectional Encoder Representations from Transformers

### What It Is
BERT is a **pre-trained Transformer encoder** that reads text **bidirectionally** (both left-to-right AND right-to-left simultaneously). It creates deep contextualized word representations that can be fine-tuned for virtually any NLP task.

Think of BERT as a person who has read millions of books and can understand what any word means in any context. You then teach them a specific task (sentiment analysis, Q&A) with just a few examples.

### Why It Matters
- **Revolutionized NLP in 2018** — state-of-the-art on 11 NLP benchmarks simultaneously
- **Pre-train once, fine-tune anywhere** — one model for dozens of tasks
- **Contextual embeddings**: "bank" in "river bank" ≠ "bank" in "bank account" (unlike Word2Vec)
- **Still the backbone** of many production NLP systems

### How BERT Works

#### Pre-training Objectives

BERT uses TWO pre-training tasks on massive unlabeled text:

**1. Masked Language Modeling (MLM)**

```
Input:  "The [MASK] sat on the [MASK]"
Target: "The  cat   sat on the  mat"

- Randomly mask 15% of tokens
- Model predicts the masked tokens
- Forces bidirectional understanding (uses both left AND right context)
```

The 15% masking strategy (important detail):
- 80% of the time: replace with [MASK] token
- 10% of the time: replace with random word
- 10% of the time: keep original word

> **Why not 100% [MASK]?** The model would learn to only predict when it sees [MASK], which never appears during fine-tuning. The random replacements force the model to maintain a good representation for ALL tokens.

**2. Next Sentence Prediction (NSP)**

```
Input:  [CLS] "The cat sat on the mat" [SEP] "It was a fluffy cat" [SEP]
Label:  IsNext ✓

Input:  [CLS] "The cat sat on the mat" [SEP] "The stock market crashed" [SEP]
Label:  NotNext ✗

- 50% actual consecutive sentences, 50% random pairs
- Teaches sentence-level relationships
```

> **Note**: Later research (RoBERTa) showed NSP doesn't help much. RoBERTa removed it and performed better.

#### BERT Architecture

```
Input:  [CLS] The cat sat on the mat [SEP]

Token Embeddings:    E_[CLS]  E_The  E_cat  E_sat  E_on  E_the  E_mat  E_[SEP]
    +
Segment Embeddings:  E_A      E_A    E_A    E_A    E_A   E_A    E_A    E_A
    +                                                                    
Position Embeddings: E_0      E_1    E_2    E_3    E_4   E_5    E_6    E_7
    =
Input Embeddings → [Transformer Encoder × 12/24 layers] → Contextualized Embeddings

Special Tokens:
  [CLS] → Classification token (aggregate representation of entire input)
  [SEP] → Separator (between sentence A and sentence B)
  [MASK] → Masked token (for MLM pre-training)
  [PAD] → Padding token (for batching variable-length sequences)
```

#### BERT Variants

| Model | Layers | Hidden Size | Attention Heads | Parameters |
|-------|--------|-------------|-----------------|------------|
| BERT-Base | 12 | 768 | 12 | 110M |
| BERT-Large | 24 | 1024 | 16 | 340M |
| DistilBERT | 6 | 768 | 12 | 66M (40% smaller, 97% performance) |
| ALBERT | 12 | 128→768 | 12 | 12M (parameter sharing) |
| RoBERTa | 12/24 | 768/1024 | 12/16 | 125M/355M |

### Code Example: BERT for Text Classification

```python
import torch
import torch.nn as nn

class BERTClassifier(nn.Module):
    """
    BERT fine-tuning for text classification.
    
    Architecture:
    Input → BERT Encoder → [CLS] token representation → Linear → Prediction
    
    The [CLS] token acts as a "summary" of the entire input.
    """
    def __init__(self, bert_model, num_classes, dropout=0.3):
        super().__init__()
        self.bert = bert_model
        self.dropout = nn.Dropout(dropout)
        # BERT-base hidden size is 768
        self.classifier = nn.Linear(768, num_classes)
    
    def forward(self, input_ids, attention_mask):
        """
        Args:
            input_ids: (batch, seq_len) — tokenized text
            attention_mask: (batch, seq_len) — 1 for real tokens, 0 for padding
        """
        # Get BERT outputs
        outputs = self.bert(input_ids=input_ids, attention_mask=attention_mask)
        
        # Extract [CLS] token representation (first token)
        cls_output = outputs.last_hidden_state[:, 0, :]  # (batch, 768)
        
        # Classify
        cls_output = self.dropout(cls_output)
        logits = self.classifier(cls_output)  # (batch, num_classes)
        
        return logits
```

### Code Example: BERT Embeddings — Understanding Contextualization

```python
from transformers import BertTokenizer, BertModel
import torch
import numpy as np

tokenizer = BertTokenizer.from_pretrained('bert-base-uncased')
model = BertModel.from_pretrained('bert-base-uncased')
model.eval()

def get_word_embedding(sentence, target_word):
    """
    Get BERT's contextual embedding for a specific word in a sentence.
    """
    inputs = tokenizer(sentence, return_tensors='pt', padding=True)
    
    with torch.no_grad():
        outputs = model(**inputs)
    
    # Find the target word's token position(s)
    tokens = tokenizer.tokenize(sentence)
    token_ids = tokenizer.convert_tokens_to_ids(tokens)
    
    # Get embeddings for all tokens (layer -1 = last layer)
    embeddings = outputs.last_hidden_state[0]  # (seq_len, 768)
    
    # Find target word position (simple case: single token)
    for i, token in enumerate(tokens):
        if target_word.lower() in token:
            return embeddings[i + 1].numpy()  # +1 for [CLS] token
    
    return None

# Demonstrate contextual embeddings
# Same word "bank" in different contexts gets DIFFERENT embeddings
emb1 = get_word_embedding("I went to the bank to deposit money", "bank")
emb2 = get_word_embedding("The river bank was covered with flowers", "bank")
emb3 = get_word_embedding("I went to the bank to withdraw cash", "bank")

# Cosine similarity
from numpy.linalg import norm
def cosine_sim(a, b):
    return np.dot(a, b) / (norm(a) * norm(b))

print(f"bank(financial) vs bank(river):     {cosine_sim(emb1, emb2):.4f}")
print(f"bank(deposit) vs bank(withdraw):    {cosine_sim(emb1, emb3):.4f}")

# Expected output:
# bank(financial) vs bank(river):     ~0.65  (different meanings → lower similarity)
# bank(deposit) vs bank(withdraw):    ~0.95  (same meaning → high similarity)
# This is impossible with Word2Vec/GloVe where "bank" always has ONE embedding!
```

### Fine-tuning BERT: Complete Pipeline

```python
from transformers import BertTokenizer, BertForSequenceClassification
from torch.utils.data import DataLoader, Dataset
import torch
import torch.optim as optim

class TextDataset(Dataset):
    """Custom dataset for BERT fine-tuning."""
    def __init__(self, texts, labels, tokenizer, max_length=128):
        self.texts = texts
        self.labels = labels
        self.tokenizer = tokenizer
        self.max_length = max_length
    
    def __len__(self):
        return len(self.texts)
    
    def __getitem__(self, idx):
        encoding = self.tokenizer(
            self.texts[idx],
            max_length=self.max_length,
            padding='max_length',       # pad to max_length
            truncation=True,            # truncate if longer
            return_tensors='pt'
        )
        return {
            'input_ids': encoding['input_ids'].squeeze(),
            'attention_mask': encoding['attention_mask'].squeeze(),
            'label': torch.tensor(self.labels[idx], dtype=torch.long)
        }

# Setup
tokenizer = BertTokenizer.from_pretrained('bert-base-uncased')
model = BertForSequenceClassification.from_pretrained(
    'bert-base-uncased', 
    num_labels=2  # binary classification
)

# Sample data
texts = [
    "This movie was absolutely fantastic!", 
    "Terrible film, waste of time.",
    "Great acting and wonderful story.",
    "I fell asleep, so boring."
] * 50  # repeat for demo

labels = [1, 0, 1, 0] * 50  # 1=positive, 0=negative

dataset = TextDataset(texts, labels, tokenizer)
dataloader = DataLoader(dataset, batch_size=16, shuffle=True)

# Optimizer with different learning rates (important!)
# BERT layers: small LR (don't destroy pre-trained knowledge)
# Classifier head: larger LR (learn from scratch)
optimizer = optim.AdamW([
    {'params': model.bert.parameters(), 'lr': 2e-5},           # pre-trained: tiny LR
    {'params': model.classifier.parameters(), 'lr': 1e-3}      # new layer: normal LR
], weight_decay=0.01)

# Training loop
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model.to(device)
model.train()

for epoch in range(3):
    total_loss = 0
    correct = 0
    total = 0
    
    for batch in dataloader:
        input_ids = batch['input_ids'].to(device)
        attention_mask = batch['attention_mask'].to(device)
        labels_batch = batch['label'].to(device)
        
        outputs = model(
            input_ids=input_ids,
            attention_mask=attention_mask,
            labels=labels_batch  # pass labels → returns loss automatically
        )
        
        loss = outputs.loss
        logits = outputs.logits
        
        optimizer.zero_grad()
        loss.backward()
        
        # Gradient clipping (still important for fine-tuning!)
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        
        optimizer.step()
        
        total_loss += loss.item()
        predictions = logits.argmax(dim=1)
        correct += (predictions == labels_batch).sum().item()
        total += labels_batch.size(0)
    
    print(f"Epoch {epoch+1}: Loss={total_loss/len(dataloader):.4f}, "
          f"Accuracy={correct/total:.4f}")

# Save fine-tuned model
model.save_pretrained('./my_sentiment_model')
tokenizer.save_pretrained('./my_sentiment_model')
```

> **Pro Tips for BERT Fine-tuning:**
> - Learning rate: 2e-5 to 5e-5 for BERT layers (too high destroys pre-trained weights)
> - Epochs: 2-4 is usually enough (BERT already knows language)
> - Batch size: 16 or 32 (larger if GPU allows)
> - Max sequence length: 128 for most tasks, 512 for long documents
> - Always use `AdamW` (not `Adam`) — weight decay regularization matters

---

## GPT — Generative Pre-trained Transformer

### What It Is
GPT is a **decoder-only Transformer** that reads text left-to-right and predicts the next token. Unlike BERT which understands text bidirectionally, GPT generates text autoregressively — one token at a time, each conditioned on all previous tokens.

Think of BERT as someone who reads the whole page to understand each word. GPT is like someone writing a story — each word follows from everything written so far.

### Why It Matters
- **GPT-3/4 demonstrated that scale = emergent abilities** (few-shot learning, reasoning)
- **Foundation of ChatGPT**, GitHub Copilot, and modern AI assistants
- **In-context learning**: Can perform tasks from instructions/examples without fine-tuning
- **Unified framework**: One model for translation, summarization, Q&A, code, math

### How GPT Differs from BERT

```
BERT (Masked LM — Bidirectional):
  Input:  "The [MASK] sat on the mat"
  Task:   Predict [MASK] = "cat" using BOTH left and right context
  Sees:   ← "The" ... "sat on the mat" →
  
GPT (Causal LM — Unidirectional):
  Input:  "The cat sat on the"
  Task:   Predict next word = "mat" using ONLY left context
  Sees:   ← "The cat sat on the" (CANNOT see future tokens)
```

#### Causal Masking Visualized

```
GPT uses a causal attention mask to prevent looking ahead:

        The  cat  sat  on   mat
The   [  1    0    0    0    0  ]   ← "The" can only attend to itself
cat   [  1    1    0    0    0  ]   ← "cat" can attend to "The" and itself
sat   [  1    1    1    0    0  ]   ← "sat" can attend to "The", "cat", itself
on    [  1    1    1    1    0  ]   ← etc.
mat   [  1    1    1    1    1  ]   ← "mat" attends to everything before it

1 = can attend, 0 = masked (cannot see)
```

### GPT Evolution

| Model | Year | Parameters | Training Data | Key Innovation |
|-------|------|-----------|---------------|----------------|
| GPT-1 | 2018 | 117M | BookCorpus (5GB) | Transformer decoder + pre-training |
| GPT-2 | 2019 | 1.5B | WebText (40GB) | Scale up, zero-shot ability |
| GPT-3 | 2020 | 175B | 570GB text | Few-shot learning, in-context learning |
| GPT-4 | 2023 | ~1.7T (est.) | Unknown | Multimodal, advanced reasoning |

### Code Example: GPT-style Causal Language Model

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import math

class CausalSelfAttention(nn.Module):
    """
    Masked self-attention for GPT — prevents attending to future tokens.
    """
    def __init__(self, d_model, n_heads, max_len=512, dropout=0.1):
        super().__init__()
        assert d_model % n_heads == 0
        
        self.n_heads = n_heads
        self.head_dim = d_model // n_heads
        
        self.qkv = nn.Linear(d_model, 3 * d_model)  # combined Q, K, V projection
        self.proj = nn.Linear(d_model, d_model)
        self.dropout = nn.Dropout(dropout)
        
        # Causal mask: lower triangular matrix
        # Register as buffer (not a parameter, but saved with model)
        mask = torch.tril(torch.ones(max_len, max_len))
        self.register_buffer('mask', mask.view(1, 1, max_len, max_len))
    
    def forward(self, x):
        B, T, C = x.shape  # batch, sequence length, embedding dim
        
        # Compute Q, K, V in one go (efficient!)
        qkv = self.qkv(x).reshape(B, T, 3, self.n_heads, self.head_dim)
        qkv = qkv.permute(2, 0, 3, 1, 4)  # (3, B, n_heads, T, head_dim)
        q, k, v = qkv[0], qkv[1], qkv[2]
        
        # Scaled dot-product attention with causal mask
        scale = math.sqrt(self.head_dim)
        attn = (q @ k.transpose(-2, -1)) / scale  # (B, n_heads, T, T)
        
        # Apply causal mask: set future positions to -inf
        attn = attn.masked_fill(self.mask[:, :, :T, :T] == 0, float('-inf'))
        attn = F.softmax(attn, dim=-1)
        attn = self.dropout(attn)
        
        # Weighted sum of values
        out = (attn @ v).transpose(1, 2).reshape(B, T, C)
        return self.proj(out)


class GPTBlock(nn.Module):
    """Single Transformer decoder block."""
    def __init__(self, d_model, n_heads, dropout=0.1):
        super().__init__()
        self.ln1 = nn.LayerNorm(d_model)
        self.attn = CausalSelfAttention(d_model, n_heads, dropout=dropout)
        self.ln2 = nn.LayerNorm(d_model)
        self.mlp = nn.Sequential(
            nn.Linear(d_model, 4 * d_model),   # expand
            nn.GELU(),                          # activation (GPT uses GELU, not ReLU)
            nn.Linear(4 * d_model, d_model),   # contract
            nn.Dropout(dropout)
        )
    
    def forward(self, x):
        # Pre-norm architecture (GPT-2+): LayerNorm BEFORE attention/MLP
        x = x + self.attn(self.ln1(x))   # residual + attention
        x = x + self.mlp(self.ln2(x))    # residual + feedforward
        return x


class MiniGPT(nn.Module):
    """
    Minimal GPT implementation for educational purposes.
    """
    def __init__(self, vocab_size, d_model=256, n_heads=8, 
                 n_layers=6, max_len=512, dropout=0.1):
        super().__init__()
        
        self.token_emb = nn.Embedding(vocab_size, d_model)
        self.pos_emb = nn.Embedding(max_len, d_model)  # learned positional embeddings
        self.dropout = nn.Dropout(dropout)
        
        # Stack of Transformer blocks
        self.blocks = nn.Sequential(
            *[GPTBlock(d_model, n_heads, dropout) for _ in range(n_layers)]
        )
        
        self.ln_f = nn.LayerNorm(d_model)  # final layer norm
        self.head = nn.Linear(d_model, vocab_size, bias=False)  # language model head
        
        # Weight tying: share weights between embedding and output
        # This is a key trick — reduces parameters and improves performance
        self.head.weight = self.token_emb.weight
    
    def forward(self, idx):
        """
        Args:
            idx: (batch, seq_len) token indices
        Returns:
            logits: (batch, seq_len, vocab_size) next-token predictions
        """
        B, T = idx.shape
        
        # Create position indices
        positions = torch.arange(T, device=idx.device).unsqueeze(0)  # (1, T)
        
        # Embed tokens + positions
        x = self.token_emb(idx) + self.pos_emb(positions)
        x = self.dropout(x)
        
        # Pass through Transformer blocks
        x = self.blocks(x)
        x = self.ln_f(x)
        
        # Project to vocabulary
        logits = self.head(x)  # (B, T, vocab_size)
        
        return logits
    
    @torch.no_grad()
    def generate(self, idx, max_new_tokens, temperature=1.0, top_k=None):
        """
        Autoregressive generation.
        """
        self.eval()
        for _ in range(max_new_tokens):
            # Crop to max length if needed
            idx_cond = idx[:, -512:]  # last 512 tokens
            
            logits = self(idx_cond)
            logits = logits[:, -1, :] / temperature  # last position only
            
            if top_k is not None:
                v, _ = torch.topk(logits, min(top_k, logits.size(-1)))
                logits[logits < v[:, [-1]]] = float('-inf')
            
            probs = F.softmax(logits, dim=-1)
            next_token = torch.multinomial(probs, num_samples=1)
            idx = torch.cat([idx, next_token], dim=1)
        
        return idx

# Create and test
model = MiniGPT(vocab_size=10000, d_model=256, n_heads=8, n_layers=6)
total_params = sum(p.numel() for p in model.parameters())
print(f"MiniGPT parameters: {total_params:,}")  # ~10M

# Forward pass
x = torch.randint(0, 10000, (4, 100))  # batch of 4, seq len 100
logits = model(x)
print(f"Output shape: {logits.shape}")  # (4, 100, 10000)

# Generate
prompt = torch.randint(0, 10000, (1, 5))  # 5-token prompt
generated = model.generate(prompt, max_new_tokens=20, temperature=0.8, top_k=50)
print(f"Generated shape: {generated.shape}")  # (1, 25)
```

---

## BERT vs GPT — The Great Divide

### Fundamental Comparison

| Aspect | BERT | GPT |
|--------|------|-----|
| Architecture | Encoder only | Decoder only |
| Attention | Bidirectional (sees all tokens) | Causal (sees only past tokens) |
| Pre-training | Masked LM + NSP | Next token prediction |
| Strength | Understanding text | Generating text |
| Input/Output | Text in → embedding out | Text in → text out |
| Fine-tuning | Task-specific heads needed | Prompt-based (few/zero-shot) |
| Best for | Classification, NER, QA extraction | Generation, summarization, chatbots |

### When to Use What

```
Use BERT when:
├── You need to CLASSIFY text (sentiment, spam, topic)
├── You need to EXTRACT information (NER, QA span extraction)  
├── You have labeled data for fine-tuning
├── You need deterministic outputs
└── Efficiency matters (BERT-base is small: 110M params)

Use GPT when:
├── You need to GENERATE text (summaries, translations, creative text)
├── You want few-shot or zero-shot capability
├── You want a single model for many tasks (via prompting)
├── You need conversational AI
└── You can afford larger models (175B+ params)

Use T5/BART (encoder-decoder) when:
├── You need sequence-to-sequence transformation
├── Translation, summarization, or text-to-text tasks
└── You want the benefits of both encoding and decoding
```

### Code: Same Task, BERT vs GPT Approach

```python
# BERT approach: Fine-tune for sentiment classification
# - Needs labeled training data
# - Train a classification head
# - Very accurate for specific task

from transformers import pipeline

# BERT-style: dedicated sentiment model
bert_classifier = pipeline(
    "sentiment-analysis", 
    model="nlptown/bert-base-multilingual-uncased-sentiment"
)
result = bert_classifier("This movie was absolutely amazing!")
print(f"BERT: {result}")
# Output: [{'label': '5 stars', 'score': 0.73}]


# GPT approach: Zero-shot via prompting
# - No training data needed
# - Just describe the task in natural language
# - Less accurate but infinitely flexible

from transformers import pipeline

gpt_classifier = pipeline("text-generation", model="gpt2")
prompt = """Classify the sentiment of the following review as positive or negative.

Review: "This movie was absolutely amazing!"
Sentiment:"""

result = gpt_classifier(prompt, max_new_tokens=5, do_sample=False)
print(f"GPT: {result[0]['generated_text']}")
# Output: "...Sentiment: positive" (GPT-2 may struggle; GPT-3+ excels)
```

---

## Other Key Transformer Models for NLP

### The Model Zoo — When to Use What

| Model | Type | Key Innovation | Best For |
|-------|------|---------------|----------|
| **RoBERTa** | Encoder | Better training (no NSP, more data, dynamic masking) | Classification, when BERT isn't enough |
| **DistilBERT** | Encoder | Knowledge distillation (40% smaller, 97% perf) | Production/mobile deployment |
| **ALBERT** | Encoder | Parameter sharing across layers | Low-resource environments |
| **ELECTRA** | Encoder | Replaced token detection (more efficient than MLM) | Best accuracy per compute |
| **DeBERTa** | Encoder | Disentangled attention (separate content + position) | State-of-the-art understanding |
| **T5** | Enc-Dec | "Text-to-Text" framing for all tasks | Multi-task, translation, summarization |
| **BART** | Enc-Dec | Denoising autoencoder | Summarization, text generation |
| **XLNet** | Encoder | Permutation language modeling | Long documents |
| **LLaMA** | Decoder | Efficient open-source GPT alternative | Research, open-source applications |
| **Mistral** | Decoder | Sliding window attention, efficient | Cost-effective generation |

### T5: Text-to-Text Transfer Transformer

```python
# T5 frames EVERY task as text-to-text
# This is its genius — one format for everything

# Translation:
# Input:  "translate English to French: The cat sat on the mat"
# Output: "Le chat est assis sur le tapis"

# Summarization:
# Input:  "summarize: <long article>"
# Output: "<short summary>"

# Sentiment:
# Input:  "sentiment: This movie was great!"
# Output: "positive"

# Question Answering:
# Input:  "question: What color is the sky? context: The sky is blue."
# Output: "blue"

from transformers import T5ForConditionalGeneration, T5Tokenizer

tokenizer = T5Tokenizer.from_pretrained('t5-small')
model = T5ForConditionalGeneration.from_pretrained('t5-small')

# Summarization
input_text = "summarize: The quick brown fox jumps over the lazy dog. " \
             "The dog was sleeping peacefully. The fox was in a hurry."

input_ids = tokenizer(input_text, return_tensors='pt').input_ids
outputs = model.generate(input_ids, max_length=50)
summary = tokenizer.decode(outputs[0], skip_special_tokens=True)
print(f"Summary: {summary}")
```

---

## Tokenization in Transformer Models

### What It Is
Tokenization for Transformers is NOT the same as simple word splitting. Modern models use **subword tokenization** — breaking words into smaller meaningful pieces. This handles unknown words, different languages, and keeps vocabulary size manageable.

### Why It Matters
- **Unknown words**: "transformerify" → ["transform", "##eri", "##fy"] (BERT can handle words it's never seen)
- **Vocabulary size**: Instead of millions of words, use 30K-50K subword tokens
- **Multilingual**: Subwords are shared across languages (efficient)

### Tokenization Algorithms Compared

| Algorithm | Used By | How It Works |
|-----------|---------|-------------|
| **WordPiece** | BERT, DistilBERT | Bottom-up: merge most frequent character pairs |
| **BPE** (Byte Pair Encoding) | GPT-2, RoBERTa | Similar to WordPiece, slightly different merge criteria |
| **SentencePiece** | T5, ALBERT, XLNet | Language-agnostic, works on raw text (no pre-tokenization) |
| **Unigram** | XLNet, ALBERT | Probabilistic: keep subwords that maximize likelihood |

### Code Example: Understanding Tokenization

```python
from transformers import AutoTokenizer

# Compare tokenizers across models
models = {
    'BERT': 'bert-base-uncased',
    'GPT-2': 'gpt2',
    'T5': 't5-small'
}

test_sentences = [
    "Hello world!",
    "Transformers are amazing!",
    "supercalifragilisticexpialidocious",  # rare word
    "I love NLP and machine learning",
    "COVID-19 pandemic",                    # domain-specific
]

for model_name, model_path in models.items():
    tokenizer = AutoTokenizer.from_pretrained(model_path)
    print(f"\n{'='*50}")
    print(f"Model: {model_name} (Vocab size: {tokenizer.vocab_size})")
    print(f"{'='*50}")
    
    for sentence in test_sentences:
        tokens = tokenizer.tokenize(sentence)
        ids = tokenizer.encode(sentence)
        print(f"  '{sentence}'")
        print(f"    Tokens: {tokens}")
        print(f"    IDs:    {ids}")
        print(f"    Count:  {len(tokens)} tokens")

# Example output for BERT:
# 'supercalifragilisticexpialidocious'
#   Tokens: ['super', '##cal', '##if', '##rag', '##ilis', '##tic', '##exp', '##iali', '##do', '##cious']
#   The ## prefix means "continuation of previous token"

# Example output for GPT-2:
# 'supercalifragilisticexpialidocious'
#   Tokens: ['super', 'cal', 'ifrag', 'ilis', 'tice', 'xp', 'ial', 'ido', 'cious']
#   GPT-2 uses Ġ prefix for tokens that start a new word
```

### Tokenization Gotchas

```python
from transformers import BertTokenizer

tokenizer = BertTokenizer.from_pretrained('bert-base-uncased')

# Gotcha 1: BERT lowercases everything (uncased version)
print(tokenizer.tokenize("Apple"))  # ['apple'] — loses case info!
# Solution: Use bert-base-CASED for NER or case-sensitive tasks

# Gotcha 2: Special tokens affect sequence length
text = "Hello world"
encoded = tokenizer.encode(text)
print(encoded)  # [101, 7592, 2088, 102] ← 101=[CLS], 102=[SEP] added!
# Max length 512 means only 510 actual tokens

# Gotcha 3: Subword tokenization affects NER/token-level tasks
text = "I live in Washington"
tokens = tokenizer.tokenize(text)
# If "Washington" splits into ["wash", "##ington"], 
# you need to align NER labels carefully!

# Gotcha 4: Different models have different special tokens
# BERT: [CLS], [SEP], [MASK], [PAD]
# GPT-2: <|endoftext|> (no [CLS] or [SEP])
# T5: </s>, <pad>
```

---

## Transfer Learning in NLP

### What It Is
Transfer learning is taking a model trained on one task (usually language modeling on massive text) and adapting it for a specific task (sentiment analysis, NER, etc.). It's like a doctor who went to medical school (pre-training) and then specializes in cardiology (fine-tuning).

### The Three Approaches

```
1. FEATURE EXTRACTION (Freeze BERT, train only classifier)
   ┌──────────┐     ┌──────────┐
   │  BERT    │ ──→ │ Linear   │ → Prediction
   │ (frozen) │     │ (trained)│
   └──────────┘     └──────────┘
   Pro: Fast, works with small data
   Con: Limited task adaptation

2. FINE-TUNING (Train entire model end-to-end)
   ┌──────────┐     ┌──────────┐
   │  BERT    │ ──→ │ Linear   │ → Prediction
   │ (trained │     │ (trained)│
   │ small LR)│     └──────────┘
   └──────────┘
   Pro: Best accuracy, full adaptation
   Con: Needs more data, risk of catastrophic forgetting

3. PROMPT-BASED (No training, just clever prompting)
   ┌──────────┐     
   │  GPT     │ → "Is this positive or negative: 'Great movie!'" → "positive"
   │ (frozen) │     
   └──────────┘     
   Pro: Zero-shot, infinitely flexible
   Con: Less accurate, expensive inference
```

### Code Example: Feature Extraction vs Fine-Tuning

```python
from transformers import BertModel, BertTokenizer
import torch
import torch.nn as nn

tokenizer = BertTokenizer.from_pretrained('bert-base-uncased')
bert = BertModel.from_pretrained('bert-base-uncased')

# ═══════════════════════════════════════
# Approach 1: Feature Extraction (Frozen BERT)
# ═══════════════════════════════════════
class FeatureExtractor(nn.Module):
    def __init__(self, bert_model, num_classes):
        super().__init__()
        self.bert = bert_model
        
        # FREEZE all BERT parameters
        for param in self.bert.parameters():
            param.requires_grad = False
        
        self.classifier = nn.Sequential(
            nn.Linear(768, 256),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(256, num_classes)
        )
    
    def forward(self, input_ids, attention_mask):
        with torch.no_grad():  # no gradients through BERT
            outputs = self.bert(input_ids=input_ids, attention_mask=attention_mask)
        cls_output = outputs.last_hidden_state[:, 0, :]
        return self.classifier(cls_output)

# Only classifier parameters are trained
fe_model = FeatureExtractor(bert, num_classes=3)
trainable = sum(p.numel() for p in fe_model.parameters() if p.requires_grad)
total = sum(p.numel() for p in fe_model.parameters())
print(f"Feature Extraction: {trainable:,} / {total:,} trainable "
      f"({trainable/total*100:.1f}%)")
# Output: ~198K / ~110M trainable (0.2%)


# ═══════════════════════════════════════
# Approach 2: Full Fine-Tuning
# ═══════════════════════════════════════
class FineTuner(nn.Module):
    def __init__(self, bert_model, num_classes):
        super().__init__()
        self.bert = bert_model
        # All parameters remain trainable (requires_grad=True by default)
        self.classifier = nn.Linear(768, num_classes)
    
    def forward(self, input_ids, attention_mask):
        outputs = self.bert(input_ids=input_ids, attention_mask=attention_mask)
        cls_output = outputs.last_hidden_state[:, 0, :]
        return self.classifier(cls_output)

# All parameters are trained
ft_model = FineTuner(BertModel.from_pretrained('bert-base-uncased'), num_classes=3)
trainable = sum(p.numel() for p in ft_model.parameters() if p.requires_grad)
print(f"Fine-Tuning: {trainable:,} trainable (100%)")
# Output: ~110M trainable (100%)


# ═══════════════════════════════════════
# Approach 3: Gradual Unfreezing (Best Practice)
# ═══════════════════════════════════════
def gradual_unfreeze(model, epoch):
    """
    Unfreeze BERT layers gradually — start from top (closest to output),
    work down to bottom. This prevents catastrophic forgetting.
    
    Epoch 0: Only classifier trained
    Epoch 1: Unfreeze last 2 BERT layers
    Epoch 2: Unfreeze last 4 BERT layers
    Epoch 3+: Unfreeze all
    """
    # Freeze everything first
    for param in model.bert.parameters():
        param.requires_grad = False
    
    # Unfreeze layers based on epoch
    n_layers = 12  # BERT-base has 12 layers
    layers_to_unfreeze = min(epoch * 2, n_layers)
    
    if layers_to_unfreeze > 0:
        for layer in model.bert.encoder.layer[-layers_to_unfreeze:]:
            for param in layer.parameters():
                param.requires_grad = True
    
    trainable = sum(p.numel() for p in model.parameters() if p.requires_grad)
    print(f"Epoch {epoch}: {trainable:,} trainable parameters "
          f"({layers_to_unfreeze}/{n_layers} layers unfrozen)")
```

---

## Common Mistakes

### 1. Using BERT for Text Generation
```python
# ❌ WRONG — BERT is an encoder, not designed for generation
from transformers import BertForMaskedLM
model = BertForMaskedLM.from_pretrained('bert-base-uncased')
# This can only fill in [MASK] tokens, not generate free text

# ✅ CORRECT — Use GPT or T5 for generation
from transformers import GPT2LMHeadModel
model = GPT2LMHeadModel.from_pretrained('gpt2')
```

### 2. Wrong Learning Rate for Fine-Tuning
```python
# ❌ WRONG — too high, destroys pre-trained weights
optimizer = torch.optim.Adam(model.parameters(), lr=1e-2)

# ❌ WRONG — too low, won't learn anything
optimizer = torch.optim.Adam(model.parameters(), lr=1e-8)

# ✅ CORRECT — sweet spot for BERT fine-tuning
optimizer = torch.optim.AdamW(model.parameters(), lr=2e-5, weight_decay=0.01)
```

### 3. Exceeding Max Sequence Length
```python
# ❌ WRONG — silently truncates or crashes
long_text = "word " * 600  # 600 tokens > BERT's 512 limit
encoded = tokenizer.encode(long_text)  # truncated without warning!

# ✅ CORRECT — handle long documents explicitly
def encode_long_document(text, tokenizer, max_length=510, stride=256):
    """
    Split long document into overlapping chunks.
    Each chunk stays within BERT's 512 limit.
    """
    tokens = tokenizer.tokenize(text)
    chunks = []
    
    for i in range(0, len(tokens), max_length - stride):
        chunk = tokens[i:i + max_length]
        chunk_ids = tokenizer.convert_tokens_to_ids(chunk)
        # Add [CLS] and [SEP]
        chunk_ids = [tokenizer.cls_token_id] + chunk_ids + [tokenizer.sep_token_id]
        chunks.append(chunk_ids)
    
    return chunks
```

### 4. Not Using Attention Mask with Padding
```python
# ❌ WRONG — model attends to padding tokens
outputs = model(input_ids=padded_batch)

# ✅ CORRECT — tell model to ignore padding
outputs = model(input_ids=padded_batch, attention_mask=attention_mask)
# attention_mask: 1 for real tokens, 0 for [PAD] tokens
```

### 5. Forgetting Weight Decay Exclusion
```python
# ❌ WRONG — applying weight decay to bias and LayerNorm (hurts performance)
optimizer = torch.optim.AdamW(model.parameters(), weight_decay=0.01)

# ✅ CORRECT — exclude bias and LayerNorm from weight decay
no_decay = ['bias', 'LayerNorm.weight', 'LayerNorm.bias']
optimizer_params = [
    {
        'params': [p for n, p in model.named_parameters() 
                   if not any(nd in n for nd in no_decay)],
        'weight_decay': 0.01
    },
    {
        'params': [p for n, p in model.named_parameters() 
                   if any(nd in n for nd in no_decay)],
        'weight_decay': 0.0
    }
]
optimizer = torch.optim.AdamW(optimizer_params, lr=2e-5)
```

---

## Interview Questions

### Conceptual Questions

**Q1: Explain the difference between BERT's MLM and GPT's causal LM pre-training. Why does each approach exist?**

> **Answer**: BERT's MLM masks 15% of tokens and predicts them using bidirectional context — the model sees both left and right. This gives superior text understanding but can't generate text (it needs [MASK] placeholders). GPT's causal LM predicts the next token using only left context — this is autoregressive, enabling text generation but limiting understanding since it can't "peek" ahead. BERT excels at classification/extraction tasks; GPT excels at generation tasks. The trade-off is bidirectional understanding vs. generative capability.

**Q2: Why does BERT use WordPiece tokenization instead of word-level tokenization?**

> **Answer**: Word-level tokenization creates huge vocabularies (millions of words) and can't handle out-of-vocabulary (OOV) words. WordPiece breaks words into subword units (e.g., "unhappiness" → "un", "##happi", "##ness"), keeping vocabulary ~30K while handling any word. It also captures morphological patterns (prefixes, suffixes) and works across languages. The trade-off is longer sequences (more tokens per sentence).

**Q3: What is catastrophic forgetting and how do you prevent it during fine-tuning?**

> **Answer**: Catastrophic forgetting is when fine-tuning on a new task destroys the pre-trained knowledge. Prevention strategies: (1) Use very small learning rates (2e-5 to 5e-5); (2) Train for few epochs (2-4); (3) Gradual unfreezing — unfreeze layers from top to bottom over epochs; (4) Discriminative learning rates — lower LR for bottom layers, higher for top; (5) Use AdamW with weight decay.

**Q4: Explain the attention mechanism intuitively. What are Q, K, V?**

> **Answer**: Analogy: searching a library. Query (Q) = "What am I looking for?", Key (K) = "What does each book's label say?", Value (V) = "What's inside each book." Attention computes how relevant each Key is to the Query (dot product → softmax → weights), then returns a weighted sum of Values. Multi-head attention is like having multiple librarians, each searching for different aspects (syntax, semantics, coreference).

**Q5: Why is positional encoding needed in Transformers but not RNNs?**

> **Answer**: RNNs process tokens sequentially, so position is implicit (step 1, step 2, ...). Transformers process all tokens in parallel — without positional encoding, "dog bites man" and "man bites dog" would have identical representations. Positional encoding injects position information. BERT uses learned embeddings; the original Transformer used sinusoidal functions. Both work similarly in practice.

**Q6: What are the advantages of DistilBERT over BERT? When would you use it?**

> **Answer**: DistilBERT is 40% smaller (66M vs 110M params), 60% faster, and retains 97% of BERT's performance. It uses knowledge distillation — a smaller "student" model learns to mimic the larger "teacher" model's outputs. Use it for: production deployment (lower latency, less memory), mobile/edge devices, when you need to serve many requests, and when that 3% accuracy loss is acceptable.

**Q7: How does T5 differ from BERT and GPT?**

> **Answer**: T5 uses the full encoder-decoder architecture and frames every NLP task as text-to-text. Input: "translate/summarize/classify: <text>" → Output: "<result as text>". This unifies classification, generation, extraction, and translation under one framework. BERT is encoder-only (output: embeddings/logits), GPT is decoder-only (output: generated text). T5 can do both understanding and generation.

### Coding Questions

**Q8: Implement scaled dot-product attention from scratch.**

```python
import torch
import math

def scaled_dot_product_attention(Q, K, V, mask=None):
    d_k = Q.size(-1)
    scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(d_k)
    if mask is not None:
        scores = scores.masked_fill(mask == 0, float('-inf'))
    weights = torch.softmax(scores, dim=-1)
    return torch.matmul(weights, V), weights
```

**Q9: What's the computational complexity of self-attention? How can it be reduced?**

> **Answer**: O(n² · d) where n = sequence length, d = dimension. For long sequences, this is prohibitive. Solutions: (1) Sparse attention (Longformer, BigBird) — attend to local windows + global tokens; (2) Linear attention (Performer) — approximate softmax with kernel functions; (3) Flash Attention — hardware-optimized exact attention (no approximation); (4) Sliding window attention (Mistral) — fixed-size attention window.

---

## Quick Reference

### Model Selection Guide

```
Task: Text Classification (sentiment, spam, topic)
  → BERT-base / RoBERTa (fine-tune) or DistilBERT (production)

Task: Named Entity Recognition
  → BERT-base-cased (case matters for NER!)

Task: Question Answering (extractive)
  → BERT / RoBERTa / DeBERTa (fine-tune on SQuAD)

Task: Text Generation
  → GPT-2 / GPT-3 / LLaMA / Mistral

Task: Summarization
  → BART / T5 / Pegasus

Task: Translation
  → T5 / mBART / MarianMT (Helsinki-NLP models)

Task: Multi-task / Flexible
  → T5 (text-to-text) or GPT-3+ (prompting)

Task: Production / Low latency
  → DistilBERT / TinyBERT / ALBERT
```

### Key Hyperparameters for Fine-Tuning

| Parameter | BERT Fine-Tuning | GPT Fine-Tuning |
|-----------|-----------------|-----------------|
| Learning Rate | 2e-5 to 5e-5 | 1e-5 to 5e-5 |
| Batch Size | 16 or 32 | 4-16 (larger models need smaller batch) |
| Epochs | 2-4 | 1-3 |
| Weight Decay | 0.01 | 0.01-0.1 |
| Warmup Steps | 10% of total steps | 5-10% of total steps |
| Max Seq Length | 128-512 | 512-2048 |
| Optimizer | AdamW | AdamW |
| Gradient Clipping | 1.0 | 1.0 |

### Architecture Comparison Table

| Feature | BERT | GPT | T5 | RoBERTa | DistilBERT |
|---------|------|-----|----|---------|-----------| 
| Type | Encoder | Decoder | Enc-Dec | Encoder | Encoder |
| Params | 110M/340M | 117M-175B | 60M-11B | 125M/355M | 66M |
| Pre-training | MLM+NSP | Causal LM | Span corruption | MLM only | Distillation |
| Bidirectional | ✅ | ❌ | Encoder: ✅ | ✅ | ✅ |
| Generation | ❌ | ✅ | ✅ | ❌ | ❌ |
| Tokenizer | WordPiece | BPE | SentencePiece | BPE | WordPiece |
| Vocab Size | 30,522 | 50,257 | 32,000 | 50,265 | 30,522 |

### Special Tokens Quick Reference

| Model | CLS/Start | SEP/End | PAD | MASK | Vocab Size |
|-------|-----------|---------|-----|------|-----------|
| BERT | [CLS] | [SEP] | [PAD] | [MASK] | 30,522 |
| GPT-2 | — | <\|endoftext\|> | — | — | 50,257 |
| T5 | — | </s> | <pad> | <extra_id_N> | 32,000 |
| RoBERTa | <s> | </s> | <pad> | <mask> | 50,265 |

---

*Previous: [04-Sequence-Models-for-NLP](04-Sequence-Models-for-NLP.md)*  
*Next: [06-HuggingFace-Transformers](06-HuggingFace-Transformers.md) — Practical toolkit for working with all these models*
