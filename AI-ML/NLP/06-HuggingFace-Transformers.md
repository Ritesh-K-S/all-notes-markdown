# Chapter 06: HuggingFace Transformers

## Table of Contents
- [Introduction to HuggingFace](#introduction-to-huggingface)
- [The Pipeline API — Instant NLP](#the-pipeline-api--instant-nlp)
- [Tokenizers — Deep Dive](#tokenizers--deep-dive)
- [AutoModel and AutoTokenizer](#automodel-and-autotokenizer)
- [Fine-Tuning with Trainer API](#fine-tuning-with-trainer-api)
- [Fine-Tuning with Native PyTorch](#fine-tuning-with-native-pytorch)
- [Working with the Model Hub](#working-with-the-model-hub)
- [Datasets Library](#datasets-library)
- [Evaluate Library](#evaluate-library)
- [Advanced Topics](#advanced-topics)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## Introduction to HuggingFace

### What It Is
HuggingFace is the **open-source ecosystem** that makes state-of-the-art NLP models accessible to everyone. Think of it as the "npm/pip of AI models" — instead of implementing BERT or GPT from scratch, you load a pre-trained model in 3 lines of code.

### Why It Matters
- **400,000+ models** on the Hub — covering every language and task
- **Standard interface**: same API for BERT, GPT, T5, LLaMA, and thousands more
- **Production-ready**: used by Google, Microsoft, Amazon, Meta in production
- **Ecosystem**: `transformers` (models) + `datasets` (data) + `evaluate` (metrics) + `accelerate` (distributed training)
- Without HuggingFace, using Transformers requires understanding each model's unique codebase — HF unifies everything

### Installation

```python
# Core libraries — install all at once
# pip install transformers datasets evaluate accelerate

# With PyTorch backend (recommended)
# pip install transformers[torch]

# With TensorFlow backend  
# pip install transformers[tf-cpu]  (or tf-gpu)

# Verify installation
import transformers
print(f"Transformers version: {transformers.__version__}")
```

### The HuggingFace Ecosystem

```
┌─────────────────────────────────────────────────────────────┐
│                    HuggingFace Hub                           │
│         (400K+ models, 100K+ datasets, Spaces)              │
└──────────┬──────────┬──────────────┬───────────┬────────────┘
           │          │              │           │
    ┌──────▼──────┐  ┌▼────────┐  ┌─▼────────┐ ┌▼──────────┐
    │transformers │  │datasets │  │evaluate  │ │accelerate │
    │             │  │         │  │          │ │           │
    │ Models      │  │ Load &  │  │ Metrics  │ │Distributed│
    │ Tokenizers  │  │ Process │  │ Eval     │ │ Training  │
    │ Pipeline    │  │ Data    │  │          │ │ Multi-GPU │
    │ Trainer     │  │         │  │          │ │           │
    └─────────────┘  └─────────┘  └──────────┘ └───────────┘
```

---

## The Pipeline API — Instant NLP

### What It Is
The `pipeline` function is the **simplest way to use any NLP model**. One line of code gives you a production-ready NLP system. It handles tokenization, model inference, and post-processing automatically.

### Why It Matters
- **3 lines to state-of-the-art NLP** — no model/tokenizer setup needed
- **Great for prototyping** — test an idea in seconds
- **Production-usable** — supports batching, GPU, and custom models

### All Available Pipelines

```python
from transformers import pipeline

# ═══════════════════════════════════════
# 1. SENTIMENT ANALYSIS
# ═══════════════════════════════════════
classifier = pipeline("sentiment-analysis")
result = classifier("I absolutely loved this movie! Best I've ever seen.")
print(result)
# [{'label': 'POSITIVE', 'score': 0.9998}]

# Batch processing — much faster than one-at-a-time
results = classifier([
    "This product is amazing!",
    "Terrible service, never coming back.",
    "It was okay, nothing special."
])
print(results)
# [{'label': 'POSITIVE', 'score': 0.9998},
#  {'label': 'NEGATIVE', 'score': 0.9995},
#  {'label': 'NEGATIVE', 'score': 0.7432}]


# ═══════════════════════════════════════
# 2. NAMED ENTITY RECOGNITION (NER)
# ═══════════════════════════════════════
ner = pipeline("ner", grouped_entities=True)
result = ner("Elon Musk founded SpaceX in Hawthorne, California in 2002.")
for entity in result:
    print(f"  {entity['word']:20s} → {entity['entity_group']:6s} "
          f"(confidence: {entity['score']:.3f})")
# Output:
#   Elon Musk            → PER    (confidence: 0.998)
#   SpaceX               → ORG    (confidence: 0.997)
#   Hawthorne            → LOC    (confidence: 0.995)
#   California           → LOC    (confidence: 0.999)


# ═══════════════════════════════════════
# 3. QUESTION ANSWERING (Extractive)
# ═══════════════════════════════════════
qa = pipeline("question-answering")
result = qa(
    question="What is the capital of France?",
    context="France is a country in Western Europe. Its capital is Paris, "
            "which is also the largest city in France."
)
print(f"Answer: {result['answer']} (confidence: {result['score']:.3f})")
# Answer: Paris (confidence: 0.992)


# ═══════════════════════════════════════
# 4. TEXT SUMMARIZATION
# ═══════════════════════════════════════
summarizer = pipeline("summarization")
article = """
The Amazon rainforest, also known as Amazonia, is a moist broadleaf tropical 
rainforest in the Amazon biome that covers most of the Amazon basin of South 
America. This basin encompasses 7,000,000 km2, of which 5,500,000 km2 are 
covered by the rainforest. This region includes territory belonging to nine 
nations and 3,344 formally acknowledged indigenous territories. The majority 
of the forest is contained within Brazil, with 60% of the rainforest, 
followed by Peru with 13%, Colombia with 10%, and with minor amounts in 
Bolivia, Ecuador, French Guiana, Guyana, Suriname, and Venezuela.
"""
result = summarizer(article, max_length=60, min_length=20)
print(f"Summary: {result[0]['summary_text']}")


# ═══════════════════════════════════════
# 5. TEXT GENERATION
# ═══════════════════════════════════════
generator = pipeline("text-generation", model="gpt2")
result = generator(
    "Artificial intelligence will",
    max_new_tokens=50,
    num_return_sequences=2,     # generate 2 different completions
    temperature=0.7,            # moderate creativity
    do_sample=True              # enable sampling (not greedy)
)
for i, seq in enumerate(result):
    print(f"Completion {i+1}: {seq['generated_text']}")


# ═══════════════════════════════════════
# 6. TRANSLATION
# ═══════════════════════════════════════
translator = pipeline("translation_en_to_fr", model="Helsinki-NLP/opus-mt-en-fr")
result = translator("Machine learning is transforming every industry.")
print(f"French: {result[0]['translation_text']}")
# French: L'apprentissage automatique transforme toutes les industries.


# ═══════════════════════════════════════
# 7. ZERO-SHOT CLASSIFICATION
# ═══════════════════════════════════════
# Classify text into categories WITHOUT any training!
zsc = pipeline("zero-shot-classification")
result = zsc(
    "The stock market crashed by 500 points today.",
    candidate_labels=["politics", "finance", "sports", "technology"]
)
print(f"Label: {result['labels'][0]} ({result['scores'][0]:.3f})")
# Label: finance (0.967)

# This works for ANY set of labels — no training required!
result = zsc(
    "My head hurts and I feel dizzy.",
    candidate_labels=["medical", "legal", "cooking", "travel"]
)
print(f"Label: {result['labels'][0]} ({result['scores'][0]:.3f})")
# Label: medical (0.923)


# ═══════════════════════════════════════
# 8. FILL-MASK (BERT's original task)
# ═══════════════════════════════════════
unmasker = pipeline("fill-mask")
results = unmasker("The capital of France is [MASK].")
for r in results[:3]:
    print(f"  {r['token_str']:10s} (score: {r['score']:.3f})")
# Output:
#   Paris      (score: 0.952)
#   Lyon       (score: 0.012)
#   Marseille  (score: 0.008)


# ═══════════════════════════════════════
# 9. TEXT SIMILARITY / EMBEDDINGS
# ═══════════════════════════════════════
from transformers import pipeline
embedder = pipeline("feature-extraction", model="bert-base-uncased")
import numpy as np

def get_sentence_embedding(text):
    """Get sentence embedding by mean-pooling token embeddings."""
    features = embedder(text, return_tensors=True)
    # features shape: (1, seq_len, 768)
    return np.array(features[0]).mean(axis=0)  # mean pooling → (768,)

emb1 = get_sentence_embedding("I love machine learning")
emb2 = get_sentence_embedding("AI and deep learning are fascinating")
emb3 = get_sentence_embedding("The weather is nice today")

from numpy.linalg import norm
cos_sim = lambda a, b: np.dot(a, b) / (norm(a) * norm(b))

print(f"ML vs AI:      {cos_sim(emb1, emb2):.4f}")  # ~0.85 (similar)
print(f"ML vs Weather: {cos_sim(emb1, emb3):.4f}")  # ~0.55 (different)
```

### Pipeline with Custom Models

```python
from transformers import pipeline

# Use a specific model from the Hub
classifier = pipeline(
    "sentiment-analysis",
    model="cardiffnlp/twitter-roberta-base-sentiment-latest",  # Twitter-optimized
    tokenizer="cardiffnlp/twitter-roberta-base-sentiment-latest"
)

# Use on GPU
classifier = pipeline(
    "sentiment-analysis", 
    device=0  # GPU 0. Use -1 for CPU, "cuda:0" also works
)

# Use your own fine-tuned model
classifier = pipeline(
    "sentiment-analysis",
    model="./my_fine_tuned_model",      # local directory
    tokenizer="./my_fine_tuned_model"
)
```

> **Pro Tip**: Pipeline is great for prototyping, but for production with high throughput, use the model directly (shown later) to avoid overhead and enable batching optimizations.

---

## Tokenizers — Deep Dive

### What It Is
HuggingFace `tokenizers` is a Rust-backed library (Python bindings) that handles converting text to model-ready inputs. It's extremely fast — 10-100x faster than pure Python tokenization.

### The Tokenization Pipeline

```
Raw Text → Normalize → Pre-tokenize → Tokenize → Post-process → Model Input

"Hello, World!"
     ↓ (normalize)
"hello, world!"              ← lowercase, unicode normalization
     ↓ (pre-tokenize)
["hello", ",", "world", "!"]  ← split on whitespace/punctuation
     ↓ (tokenize/subword)
["hello", ",", "world", "!"]  ← apply BPE/WordPiece/Unigram
     ↓ (post-process)
[101, 7592, 1010, 2088, 999, 102]  ← add [CLS], [SEP], convert to IDs
```

### Essential Tokenizer Operations

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")

# ═══════════════════════════════════════
# Basic Usage
# ═══════════════════════════════════════
text = "Hello, how are you doing today?"

# Method 1: Simple encoding
ids = tokenizer.encode(text)
print(f"IDs: {ids}")
# [101, 7592, 1010, 2129, 2024, 2017, 2725, 2651, 1029, 102]

# Method 2: Full encoding (returns dict with all needed tensors)
encoded = tokenizer(text, return_tensors="pt")
print(encoded.keys())
# dict_keys(['input_ids', 'token_type_ids', 'attention_mask'])
print(f"input_ids shape:     {encoded['input_ids'].shape}")      # (1, 10)
print(f"attention_mask shape: {encoded['attention_mask'].shape}")  # (1, 10)

# Decode back to text
decoded = tokenizer.decode(ids)
print(f"Decoded: {decoded}")
# "[CLS] hello, how are you doing today? [SEP]"

decoded_clean = tokenizer.decode(ids, skip_special_tokens=True)
print(f"Clean: {decoded_clean}")
# "hello, how are you doing today?"


# ═══════════════════════════════════════
# Batch Encoding (for training)
# ═══════════════════════════════════════
texts = [
    "Short text.",
    "This is a medium length sentence for testing.",
    "A longer sentence that contains more words to demonstrate padding behavior."
]

# Pad to same length + truncate if needed
batch = tokenizer(
    texts,
    padding=True,          # pad shorter sequences to max length in batch
    truncation=True,       # truncate sequences longer than max_length
    max_length=20,         # maximum sequence length
    return_tensors="pt"    # return PyTorch tensors
)

print(f"Batch input_ids shape: {batch['input_ids'].shape}")      # (3, 20)
print(f"Batch attention_mask shape: {batch['attention_mask'].shape}")  # (3, 20)

# Inspect padding
for i, text in enumerate(texts):
    tokens = tokenizer.convert_ids_to_tokens(batch['input_ids'][i])
    mask = batch['attention_mask'][i].tolist()
    real_tokens = sum(mask)
    print(f"  Text {i}: {real_tokens} real tokens, "
          f"{len(mask) - real_tokens} padding tokens")


# ═══════════════════════════════════════
# Sentence Pairs (for NLI, QA, etc.)
# ═══════════════════════════════════════
question = "What is the capital of France?"
context = "France is a country in Europe. Its capital is Paris."

encoded_pair = tokenizer(
    question, context,      # pass two texts → BERT formats as [CLS] Q [SEP] C [SEP]
    padding="max_length",
    max_length=64,
    truncation=True,
    return_tensors="pt"
)

# token_type_ids: 0 for question tokens, 1 for context tokens
print(f"Token type IDs: {encoded_pair['token_type_ids'][0][:20].tolist()}")
# [0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]
#  ↑ question (type 0) ↑       ↑ context (type 1)              ↑


# ═══════════════════════════════════════
# Token ↔ Character Offset Mapping
# ═══════════════════════════════════════
# Critical for NER and QA — map tokens back to original character positions

text = "HuggingFace is based in New York City"
encoded = tokenizer(
    text, 
    return_offsets_mapping=True,  # ← this is the key!
    return_tensors="pt"
)

tokens = tokenizer.convert_ids_to_tokens(encoded['input_ids'][0])
offsets = encoded['offset_mapping'][0].tolist()

print(f"{'Token':<15} {'Start':>5} {'End':>5} {'Original':>15}")
print("-" * 45)
for token, (start, end) in zip(tokens, offsets):
    original = text[start:end] if start != end else "[special]"
    print(f"{token:<15} {start:>5} {end:>5} {original:>15}")
```

### Fast vs Slow Tokenizers

```python
# HuggingFace has two tokenizer implementations:
# 1. "Slow" — Pure Python (PreTrainedTokenizer)
# 2. "Fast" — Rust-backed (PreTrainedTokenizerFast) ← ALWAYS prefer this

from transformers import BertTokenizer, BertTokenizerFast

# Slow tokenizer
slow_tokenizer = BertTokenizer.from_pretrained("bert-base-uncased")

# Fast tokenizer (default when using AutoTokenizer)
fast_tokenizer = BertTokenizerFast.from_pretrained("bert-base-uncased")

# Speed comparison
import time

text = "This is a test sentence for benchmarking tokenizer speed." * 100

start = time.time()
for _ in range(1000):
    slow_tokenizer(text)
slow_time = time.time() - start

start = time.time()
for _ in range(1000):
    fast_tokenizer(text)
fast_time = time.time() - start

print(f"Slow tokenizer: {slow_time:.2f}s")
print(f"Fast tokenizer: {fast_time:.2f}s")
print(f"Speedup: {slow_time/fast_time:.1f}x")
# Typical speedup: 5-20x

# Fast tokenizer also provides offset mapping (slow doesn't)
encoded = fast_tokenizer(text, return_offsets_mapping=True)
# This is essential for NER and QA span extraction
```

---

## AutoModel and AutoTokenizer

### What It Is
The `Auto` classes automatically detect and load the correct model architecture and tokenizer from a model name or path. You don't need to know if a model is BERT, RoBERTa, or GPT — `Auto` figures it out.

### Why It Matters
- **One API for all models** — `AutoModel.from_pretrained("any-model-name")`
- **Future-proof** — new architectures work automatically
- **Reduces errors** — no mismatch between model and tokenizer types

### AutoModel Variants

```python
from transformers import (
    AutoModel,                          # Base model (just embeddings)
    AutoModelForSequenceClassification, # Text classification
    AutoModelForTokenClassification,    # NER, POS tagging
    AutoModelForQuestionAnswering,      # Extractive QA
    AutoModelForCausalLM,              # Text generation (GPT-style)
    AutoModelForMaskedLM,              # Fill-mask (BERT-style)
    AutoModelForSeq2SeqLM,            # Translation, summarization (T5-style)
    AutoTokenizer                       # Matching tokenizer
)

# ═══════════════════════════════════════
# Base Model — Returns raw embeddings
# ═══════════════════════════════════════
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
model = AutoModel.from_pretrained("bert-base-uncased")

inputs = tokenizer("Hello world", return_tensors="pt")
outputs = model(**inputs)

print(f"Last hidden state: {outputs.last_hidden_state.shape}")
# (1, 4, 768) — 4 tokens, 768-dim embedding each
# Use this when you need embeddings for downstream tasks


# ═══════════════════════════════════════
# Classification Model — Returns logits
# ═══════════════════════════════════════
model = AutoModelForSequenceClassification.from_pretrained(
    "bert-base-uncased",
    num_labels=3  # 3-class classification
)

outputs = model(**inputs)
print(f"Logits: {outputs.logits.shape}")  # (1, 3)
# This adds a classification head on top of BERT


# ═══════════════════════════════════════
# Causal LM — For text generation
# ═══════════════════════════════════════
tokenizer = AutoTokenizer.from_pretrained("gpt2")
model = AutoModelForCausalLM.from_pretrained("gpt2")

inputs = tokenizer("The future of AI is", return_tensors="pt")
outputs = model.generate(
    **inputs, 
    max_new_tokens=30,
    temperature=0.7,
    do_sample=True,
    top_p=0.9
)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))


# ═══════════════════════════════════════
# Token Classification — For NER
# ═══════════════════════════════════════
model = AutoModelForTokenClassification.from_pretrained(
    "dbmdz/bert-large-cased-finetuned-conll03-english"
)

inputs = tokenizer("Elon Musk founded SpaceX", return_tensors="pt")
outputs = model(**inputs)
print(f"Tag logits per token: {outputs.logits.shape}")
# (1, 6, 9) — 6 tokens, 9 possible tags
```

### Model Configuration

```python
from transformers import AutoConfig, AutoModel

# Inspect model configuration before loading
config = AutoConfig.from_pretrained("bert-base-uncased")
print(f"Model type:     {config.model_type}")
print(f"Hidden size:    {config.hidden_size}")
print(f"Num layers:     {config.num_hidden_layers}")
print(f"Num heads:      {config.num_attention_heads}")
print(f"Vocab size:     {config.vocab_size}")
print(f"Max position:   {config.max_position_embeddings}")
# Output:
# Model type:     bert
# Hidden size:    768
# Num layers:     12
# Num heads:      12
# Vocab size:     30522
# Max position:   512

# Modify config for custom model
config.num_labels = 5  # 5-class classification
model = AutoModelForSequenceClassification.from_config(config)
# Creates a randomly initialized model with BERT architecture but 5 output classes
```

---

## Fine-Tuning with Trainer API

### What It Is
The `Trainer` class is HuggingFace's **high-level training loop** that handles everything: training, evaluation, checkpointing, logging, mixed precision, distributed training, and more. It's like Keras's `model.fit()` but for Transformers.

### Why It Matters
- **10x less boilerplate** than writing your own training loop
- **Built-in best practices**: gradient accumulation, mixed precision, learning rate scheduling
- **Production-ready**: supports multi-GPU, TPU, and distributed training out of the box
- **Integrations**: Weights & Biases, TensorBoard, MLflow logging

### Complete Fine-Tuning Example: Sentiment Analysis

```python
from transformers import (
    AutoTokenizer, 
    AutoModelForSequenceClassification,
    TrainingArguments, 
    Trainer
)
from datasets import load_dataset
import evaluate
import numpy as np

# ═══════════════════════════════════════
# Step 1: Load Dataset
# ═══════════════════════════════════════
dataset = load_dataset("imdb")
print(dataset)
# DatasetDict({
#     train: Dataset({features: ['text', 'label'], num_rows: 25000})
#     test:  Dataset({features: ['text', 'label'], num_rows: 25000})
# })

# Use a small subset for demo (remove these lines for full training)
small_train = dataset["train"].shuffle(seed=42).select(range(2000))
small_test = dataset["test"].shuffle(seed=42).select(range(500))


# ═══════════════════════════════════════
# Step 2: Tokenize Data
# ═══════════════════════════════════════
tokenizer = AutoTokenizer.from_pretrained("distilbert-base-uncased")

def tokenize_function(examples):
    """
    Tokenize a batch of examples.
    Called by dataset.map() — processes batches efficiently.
    """
    return tokenizer(
        examples["text"],
        padding="max_length",
        truncation=True,
        max_length=256      # IMDb reviews can be long; 256 is a good trade-off
    )

# Apply tokenization to entire dataset (cached automatically!)
tokenized_train = small_train.map(tokenize_function, batched=True)
tokenized_test = small_test.map(tokenize_function, batched=True)

# Remove raw text column (model doesn't need it)
tokenized_train = tokenized_train.remove_columns(["text"])
tokenized_test = tokenized_test.remove_columns(["text"])

# Set format to PyTorch tensors
tokenized_train.set_format("torch")
tokenized_test.set_format("torch")


# ═══════════════════════════════════════
# Step 3: Load Model
# ═══════════════════════════════════════
model = AutoModelForSequenceClassification.from_pretrained(
    "distilbert-base-uncased",
    num_labels=2,       # binary: positive/negative
    id2label={0: "NEGATIVE", 1: "POSITIVE"},  # human-readable labels
    label2id={"NEGATIVE": 0, "POSITIVE": 1}
)


# ═══════════════════════════════════════
# Step 4: Define Metrics
# ═══════════════════════════════════════
accuracy_metric = evaluate.load("accuracy")
f1_metric = evaluate.load("f1")

def compute_metrics(eval_pred):
    """
    Compute metrics during evaluation.
    Called automatically by Trainer at each evaluation step.
    """
    logits, labels = eval_pred
    predictions = np.argmax(logits, axis=-1)
    
    accuracy = accuracy_metric.compute(
        predictions=predictions, references=labels
    )
    f1 = f1_metric.compute(
        predictions=predictions, references=labels, average="binary"
    )
    
    return {**accuracy, **f1}


# ═══════════════════════════════════════
# Step 5: Configure Training
# ═══════════════════════════════════════
training_args = TrainingArguments(
    # Output
    output_dir="./results",                # save checkpoints here
    overwrite_output_dir=True,
    
    # Training hyperparameters
    num_train_epochs=3,                    # 3 epochs (usually enough for fine-tuning)
    per_device_train_batch_size=16,        # batch size per GPU
    per_device_eval_batch_size=32,         # eval can be larger (no gradients)
    learning_rate=2e-5,                    # BERT sweet spot: 2e-5 to 5e-5
    weight_decay=0.01,                     # L2 regularization
    warmup_ratio=0.1,                      # 10% of steps for LR warmup
    
    # Evaluation
    eval_strategy="epoch",                 # evaluate after each epoch
    save_strategy="epoch",                 # save checkpoint each epoch
    load_best_model_at_end=True,           # load best model when training ends
    metric_for_best_model="f1",            # use F1 to determine "best"
    greater_is_better=True,
    
    # Optimization
    fp16=True,                             # mixed precision (2x speedup on GPU)
    gradient_accumulation_steps=2,         # effective batch = 16 × 2 = 32
    
    # Logging
    logging_dir="./logs",
    logging_steps=50,                      # log every 50 steps
    report_to="none",                      # "wandb", "tensorboard", etc.
    
    # Reproducibility
    seed=42,
)


# ═══════════════════════════════════════
# Step 6: Create Trainer and Train!
# ═══════════════════════════════════════
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_train,
    eval_dataset=tokenized_test,
    compute_metrics=compute_metrics,
    tokenizer=tokenizer,                   # for proper padding in DataLoader
)

# Train!
train_result = trainer.train()
print(f"Training time: {train_result.metrics['train_runtime']:.1f}s")
print(f"Samples/sec: {train_result.metrics['train_samples_per_second']:.1f}")

# Evaluate
eval_results = trainer.evaluate()
print(f"\nEval Accuracy: {eval_results['eval_accuracy']:.4f}")
print(f"Eval F1:       {eval_results['eval_f1']:.4f}")


# ═══════════════════════════════════════
# Step 7: Save and Load
# ═══════════════════════════════════════
# Save
trainer.save_model("./my_sentiment_model")
tokenizer.save_pretrained("./my_sentiment_model")

# Load for inference
from transformers import pipeline

classifier = pipeline(
    "sentiment-analysis",
    model="./my_sentiment_model",
    tokenizer="./my_sentiment_model"
)

print(classifier("This course is incredibly well-taught!"))
# [{'label': 'POSITIVE', 'score': 0.9987}]
```

### Key TrainingArguments Explained

| Parameter | Purpose | Recommended Value |
|-----------|---------|-------------------|
| `learning_rate` | How fast to update weights | 2e-5 to 5e-5 |
| `num_train_epochs` | Number of passes through data | 2-4 for fine-tuning |
| `per_device_train_batch_size` | Samples per GPU per step | 8-32 (depends on GPU RAM) |
| `gradient_accumulation_steps` | Simulate larger batches | 2-8 (effective batch = this × batch_size) |
| `warmup_ratio` | Fraction of steps for LR warmup | 0.05-0.1 |
| `weight_decay` | L2 regularization | 0.01-0.1 |
| `fp16` | Mixed precision training | True (if GPU supports it) |
| `eval_strategy` | When to evaluate | "epoch" or "steps" |
| `load_best_model_at_end` | Keep best checkpoint | True (always!) |
| `metric_for_best_model` | Which metric decides "best" | Task-dependent |

---

## Fine-Tuning with Native PyTorch

### When to Use Native PyTorch Instead of Trainer
- You need **custom training logic** (custom losses, multi-task, adversarial training)
- You want **full control** over every step
- You're doing **research** and need to experiment with non-standard setups
- You need to **understand what's happening** under the hood

```python
import torch
from torch.utils.data import DataLoader
from transformers import AutoTokenizer, AutoModelForSequenceClassification
from transformers import get_linear_schedule_with_warmup
from datasets import load_dataset
from tqdm import tqdm

# ═══════════════════════════════════════
# Setup
# ═══════════════════════════════════════
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(f"Using device: {device}")

tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
model = AutoModelForSequenceClassification.from_pretrained(
    "bert-base-uncased", num_labels=2
).to(device)

# Load and tokenize data
dataset = load_dataset("imdb")
small_train = dataset["train"].shuffle(seed=42).select(range(2000))
small_test = dataset["test"].shuffle(seed=42).select(range(500))

def tokenize(batch):
    return tokenizer(batch["text"], padding="max_length", 
                    truncation=True, max_length=256)

train_data = small_train.map(tokenize, batched=True)
test_data = small_test.map(tokenize, batched=True)
train_data.set_format("torch", columns=["input_ids", "attention_mask", "label"])
test_data.set_format("torch", columns=["input_ids", "attention_mask", "label"])

train_loader = DataLoader(train_data, batch_size=16, shuffle=True)
test_loader = DataLoader(test_data, batch_size=32)

# ═══════════════════════════════════════
# Optimizer with Discriminative Learning Rates
# ═══════════════════════════════════════
# Key insight: use lower LR for pre-trained layers, higher for new head
no_decay = ["bias", "LayerNorm.weight"]
optimizer_grouped_parameters = [
    {
        "params": [p for n, p in model.named_parameters() 
                   if not any(nd in n for nd in no_decay) and "classifier" not in n],
        "weight_decay": 0.01,
        "lr": 2e-5           # pre-trained layers: small LR
    },
    {
        "params": [p for n, p in model.named_parameters() 
                   if any(nd in n for nd in no_decay) and "classifier" not in n],
        "weight_decay": 0.0,
        "lr": 2e-5
    },
    {
        "params": model.classifier.parameters(),
        "weight_decay": 0.01,
        "lr": 1e-3           # new classification head: larger LR
    }
]
optimizer = torch.optim.AdamW(optimizer_grouped_parameters)

# Learning rate scheduler with warmup
num_epochs = 3
total_steps = len(train_loader) * num_epochs
warmup_steps = int(0.1 * total_steps)

scheduler = get_linear_schedule_with_warmup(
    optimizer, 
    num_warmup_steps=warmup_steps,
    num_training_steps=total_steps
)

# ═══════════════════════════════════════
# Training Loop
# ═══════════════════════════════════════
best_f1 = 0
scaler = torch.amp.GradScaler("cuda")  # for mixed precision

for epoch in range(num_epochs):
    model.train()
    total_loss = 0
    
    progress_bar = tqdm(train_loader, desc=f"Epoch {epoch+1}/{num_epochs}")
    
    for batch in progress_bar:
        # Move batch to device
        input_ids = batch["input_ids"].to(device)
        attention_mask = batch["attention_mask"].to(device)
        labels = batch["label"].to(device)
        
        optimizer.zero_grad()
        
        # Mixed precision forward pass
        with torch.amp.autocast("cuda"):
            outputs = model(
                input_ids=input_ids,
                attention_mask=attention_mask,
                labels=labels
            )
            loss = outputs.loss
        
        # Mixed precision backward pass
        scaler.scale(loss).backward()
        
        # Gradient clipping (unscale first for proper clipping)
        scaler.unscale_(optimizer)
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        
        scaler.step(optimizer)
        scaler.update()
        scheduler.step()
        
        total_loss += loss.item()
        progress_bar.set_postfix({"loss": loss.item():.4f})
    
    avg_loss = total_loss / len(train_loader)
    
    # ═══════════════════════════════════════
    # Evaluation
    # ═══════════════════════════════════════
    model.eval()
    all_preds = []
    all_labels = []
    
    with torch.no_grad():
        for batch in test_loader:
            input_ids = batch["input_ids"].to(device)
            attention_mask = batch["attention_mask"].to(device)
            labels = batch["label"].to(device)
            
            outputs = model(input_ids=input_ids, attention_mask=attention_mask)
            preds = outputs.logits.argmax(dim=-1)
            
            all_preds.extend(preds.cpu().tolist())
            all_labels.extend(labels.cpu().tolist())
    
    # Calculate metrics
    from sklearn.metrics import accuracy_score, f1_score, classification_report
    
    accuracy = accuracy_score(all_labels, all_preds)
    f1 = f1_score(all_labels, all_preds, average="binary")
    
    print(f"\nEpoch {epoch+1}: Loss={avg_loss:.4f}, "
          f"Accuracy={accuracy:.4f}, F1={f1:.4f}")
    
    # Save best model
    if f1 > best_f1:
        best_f1 = f1
        model.save_pretrained("./best_model")
        tokenizer.save_pretrained("./best_model")
        print(f"  → Saved best model (F1: {f1:.4f})")

print(f"\nBest F1: {best_f1:.4f}")
print(classification_report(all_labels, all_preds, 
                           target_names=["Negative", "Positive"]))
```

---

## Working with the Model Hub

### What It Is
The HuggingFace Hub is a platform hosting **400K+ pre-trained models** and **100K+ datasets**. Think of it as GitHub for AI models — you can browse, download, and share models.

### Finding the Right Model

```python
from huggingface_hub import HfApi, list_models

api = HfApi()

# Search for sentiment analysis models
models = api.list_models(
    filter="text-classification",
    sort="downloads",
    direction=-1,           # descending (most downloaded first)
    limit=5
)

print("Top 5 text classification models:")
for m in models:
    print(f"  {m.id:50s} Downloads: {m.downloads:>10,}")

# Search for specific model
models = api.list_models(
    search="sentiment",
    filter="text-classification",
    sort="likes",
    limit=5
)
```

### Pushing Your Model to the Hub

```python
from transformers import AutoModelForSequenceClassification, AutoTokenizer
from huggingface_hub import login

# Login (one time) — get token from https://huggingface.co/settings/tokens
# login(token="hf_your_token_here")  # or use CLI: huggingface-cli login

# Load your fine-tuned model
model = AutoModelForSequenceClassification.from_pretrained("./best_model")
tokenizer = AutoTokenizer.from_pretrained("./best_model")

# Push to Hub
model.push_to_hub("your-username/my-sentiment-model")
tokenizer.push_to_hub("your-username/my-sentiment-model")

# Now anyone can use it:
# pipeline("sentiment-analysis", model="your-username/my-sentiment-model")

# Create a model card (README.md) for your model
card = """
---
language: en
tags:
  - sentiment-analysis
  - bert
datasets:
  - imdb
metrics:
  - accuracy
  - f1
model-index:
  - name: my-sentiment-model
    results:
      - task:
          type: text-classification
        dataset:
          name: imdb
          type: imdb
        metrics:
          - name: Accuracy
            type: accuracy
            value: 0.92
          - name: F1
            type: f1
            value: 0.91
---

# My Sentiment Analysis Model

Fine-tuned DistilBERT on IMDB dataset for binary sentiment classification.

## Usage
```python
from transformers import pipeline
classifier = pipeline("sentiment-analysis", model="your-username/my-sentiment-model")
classifier("This movie was great!")
```
"""
```

---

## Datasets Library

### What It Is
The `datasets` library provides a unified interface to **100K+ datasets** with efficient caching, memory-mapped loading, and Apache Arrow backend for fast processing.

### Why It Matters
- **Memory efficient**: processes datasets larger than RAM using memory-mapping
- **Fast**: Apache Arrow columnar format — 10-100x faster than pandas for ML workloads
- **Cached**: tokenization and processing are cached — re-running is instant
- **Standardized**: same API for every dataset

### Essential Operations

```python
from datasets import load_dataset, Dataset, DatasetDict

# ═══════════════════════════════════════
# Loading Datasets
# ═══════════════════════════════════════

# From the Hub
imdb = load_dataset("imdb")                    # popular datasets
squad = load_dataset("squad")                   # question answering
conll = load_dataset("conll2003")               # NER

# Specific split
train = load_dataset("imdb", split="train")
test = load_dataset("imdb", split="test")

# Subset of data
small = load_dataset("imdb", split="train[:1000]")  # first 1000 examples
percentage = load_dataset("imdb", split="train[:10%]")  # first 10%

# From CSV/JSON/text files
csv_dataset = load_dataset("csv", data_files="my_data.csv")
json_dataset = load_dataset("json", data_files="my_data.jsonl")

# From pandas DataFrame
import pandas as pd
df = pd.DataFrame({
    "text": ["Great movie!", "Terrible film.", "Average."],
    "label": [1, 0, 1]
})
hf_dataset = Dataset.from_pandas(df)

# From dict
dataset = Dataset.from_dict({
    "text": ["example 1", "example 2"],
    "label": [0, 1]
})


# ═══════════════════════════════════════
# Exploring Datasets
# ═══════════════════════════════════════
dataset = load_dataset("imdb", split="train")

print(f"Num rows:    {len(dataset)}")
print(f"Features:    {dataset.features}")
print(f"Columns:     {dataset.column_names}")
print(f"First item:  {dataset[0]}")
print(f"Text field:  {dataset[0]['text'][:100]}...")
print(f"Label dist:  {dataset.to_pandas()['label'].value_counts().to_dict()}")


# ═══════════════════════════════════════
# Processing Data (map, filter, select)
# ═══════════════════════════════════════

# MAP — Apply function to each example (or batch)
def add_length(example):
    """Add a 'length' column with word count."""
    example['length'] = len(example['text'].split())
    return example

dataset = dataset.map(add_length)
print(f"Average length: {sum(dataset['length'])/len(dataset):.0f} words")

# Batched map (much faster for tokenization)
def tokenize_batch(batch):
    return tokenizer(batch['text'], padding=True, truncation=True, max_length=256)

tokenized = dataset.map(tokenize_batch, batched=True, batch_size=1000)

# FILTER — Remove examples that don't match condition
long_reviews = dataset.filter(lambda x: x['length'] > 100)
print(f"Long reviews: {len(long_reviews)} / {len(dataset)}")

# SELECT — Choose specific indices
subset = dataset.select(range(100))  # first 100 examples

# SORT
sorted_dataset = dataset.sort("length", reverse=True)

# SHUFFLE
shuffled = dataset.shuffle(seed=42)

# TRAIN/TEST SPLIT
split = dataset.train_test_split(test_size=0.2, seed=42, stratify_by_column="label")
print(f"Train: {len(split['train'])}, Test: {len(split['test'])}")


# ═══════════════════════════════════════
# Saving and Loading Processed Datasets
# ═══════════════════════════════════════
# Save to disk (Arrow format — fast to reload)
tokenized.save_to_disk("./processed_imdb")

# Load back instantly (no re-processing!)
from datasets import load_from_disk
reloaded = load_from_disk("./processed_imdb")

# Save as CSV/JSON
dataset.to_csv("output.csv")
dataset.to_json("output.jsonl")
```

---

## Evaluate Library

### What It Is
The `evaluate` library provides standardized implementations of common ML metrics. It ensures you're computing metrics exactly the same way as published papers.

```python
import evaluate
import numpy as np

# ═══════════════════════════════════════
# Available Metrics
# ═══════════════════════════════════════
# List all available metrics
all_metrics = evaluate.list_evaluation_modules(module_type="metric")
print(f"Total metrics available: {len(all_metrics)}")

# Common NLP metrics
accuracy = evaluate.load("accuracy")
f1 = evaluate.load("f1")
precision = evaluate.load("precision")
recall = evaluate.load("recall")
bleu = evaluate.load("bleu")      # for translation
rouge = evaluate.load("rouge")    # for summarization
bertscore = evaluate.load("bertscore")  # semantic similarity


# ═══════════════════════════════════════
# Classification Metrics
# ═══════════════════════════════════════
predictions = [1, 0, 1, 1, 0, 1, 0, 0, 1, 1]
references  = [1, 0, 1, 0, 0, 1, 1, 0, 1, 0]

print("Accuracy:", accuracy.compute(
    predictions=predictions, references=references
))
# {'accuracy': 0.7}

print("F1:", f1.compute(
    predictions=predictions, references=references, average="binary"
))
# {'f1': 0.727}

# Multi-class: use average="macro" or "weighted"
print("F1 (macro):", f1.compute(
    predictions=predictions, references=references, average="macro"
))


# ═══════════════════════════════════════
# Text Generation Metrics
# ═══════════════════════════════════════
# BLEU — for translation quality
bleu = evaluate.load("bleu")
predictions = ["The cat sat on the mat"]
references = [["The cat is sitting on the mat"]]
print("BLEU:", bleu.compute(predictions=predictions, references=references))

# ROUGE — for summarization quality
rouge = evaluate.load("rouge")
predictions = ["The cat sat on the mat"]
references = ["A cat was sitting on a mat in the room"]
print("ROUGE:", rouge.compute(predictions=predictions, references=references))
# Returns rouge1, rouge2, rougeL, rougeLsum


# ═══════════════════════════════════════
# Combining Multiple Metrics
# ═══════════════════════════════════════
clf_metrics = evaluate.combine(["accuracy", "f1", "precision", "recall"])

results = clf_metrics.compute(
    predictions=predictions,
    references=references
)
print("Combined:", results)
```

---

## Advanced Topics

### Efficient Inference

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch
import time

tokenizer = AutoTokenizer.from_pretrained("distilbert-base-uncased")
model = AutoModelForSequenceClassification.from_pretrained(
    "distilbert-base-uncased-finetuned-sst-2-english"
)

texts = ["This is a great product!"] * 100  # 100 predictions

# ═══════════════════════════════════════
# Technique 1: Batched Inference
# ═══════════════════════════════════════
model.eval()

# ❌ SLOW — one at a time
start = time.time()
for text in texts:
    inputs = tokenizer(text, return_tensors="pt", truncation=True)
    with torch.no_grad():
        model(**inputs)
slow_time = time.time() - start

# ✅ FAST — batched
start = time.time()
inputs = tokenizer(texts, return_tensors="pt", padding=True, truncation=True)
with torch.no_grad():
    outputs = model(**inputs)
fast_time = time.time() - start

print(f"One-at-a-time: {slow_time:.2f}s")
print(f"Batched:       {fast_time:.2f}s")
print(f"Speedup:       {slow_time/fast_time:.1f}x")


# ═══════════════════════════════════════
# Technique 2: ONNX Export (2-5x faster inference)
# ═══════════════════════════════════════
# pip install optimum[onnxruntime]
from optimum.onnxruntime import ORTModelForSequenceClassification

# Convert and save
ort_model = ORTModelForSequenceClassification.from_pretrained(
    "distilbert-base-uncased-finetuned-sst-2-english",
    export=True  # converts to ONNX automatically
)
ort_model.save_pretrained("./onnx_model")

# Use for inference (same API!)
from transformers import pipeline
onnx_classifier = pipeline(
    "sentiment-analysis", 
    model=ort_model, 
    tokenizer=tokenizer
)
print(onnx_classifier("This is amazing!"))


# ═══════════════════════════════════════
# Technique 3: Quantization (smaller model, faster)
# ═══════════════════════════════════════
# Dynamic quantization (simplest, CPU only)
quantized_model = torch.quantization.quantize_dynamic(
    model, 
    {torch.nn.Linear},  # quantize linear layers
    dtype=torch.qint8   # 8-bit integers
)

# Compare sizes
import os

torch.save(model.state_dict(), "full_model.pt")
torch.save(quantized_model.state_dict(), "quantized_model.pt")

full_size = os.path.getsize("full_model.pt") / 1e6
quant_size = os.path.getsize("quantized_model.pt") / 1e6
print(f"Full model:      {full_size:.1f} MB")
print(f"Quantized model: {quant_size:.1f} MB")
print(f"Reduction:       {(1 - quant_size/full_size)*100:.0f}%")

# Cleanup
os.remove("full_model.pt")
os.remove("quantized_model.pt")
```

### Multi-Task Fine-Tuning with Custom Head

```python
import torch
import torch.nn as nn
from transformers import AutoModel, AutoTokenizer

class MultiTaskModel(nn.Module):
    """
    One BERT encoder shared across multiple NLP tasks.
    
    Tasks:
    - Sentiment classification (2 classes)
    - Topic classification (5 classes)
    - Toxicity detection (binary)
    """
    def __init__(self, model_name="bert-base-uncased"):
        super().__init__()
        self.encoder = AutoModel.from_pretrained(model_name)
        hidden_size = self.encoder.config.hidden_size  # 768 for bert-base
        
        # Task-specific heads
        self.sentiment_head = nn.Linear(hidden_size, 2)
        self.topic_head = nn.Linear(hidden_size, 5)
        self.toxicity_head = nn.Linear(hidden_size, 1)
        
        self.dropout = nn.Dropout(0.3)
    
    def forward(self, input_ids, attention_mask, task="sentiment"):
        """
        Forward pass — shared encoder, task-specific output.
        """
        outputs = self.encoder(input_ids=input_ids, attention_mask=attention_mask)
        pooled = self.dropout(outputs.last_hidden_state[:, 0, :])  # [CLS] token
        
        if task == "sentiment":
            return self.sentiment_head(pooled)
        elif task == "topic":
            return self.topic_head(pooled)
        elif task == "toxicity":
            return self.toxicity_head(pooled)
        else:
            raise ValueError(f"Unknown task: {task}")

# Usage
model = MultiTaskModel()
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")

text = "This movie was absolutely wonderful!"
inputs = tokenizer(text, return_tensors="pt", truncation=True, padding=True)

sentiment_logits = model(inputs["input_ids"], inputs["attention_mask"], task="sentiment")
topic_logits = model(inputs["input_ids"], inputs["attention_mask"], task="topic")
toxicity_logits = model(inputs["input_ids"], inputs["attention_mask"], task="toxicity")

print(f"Sentiment: {sentiment_logits.shape}")  # (1, 2)
print(f"Topic:     {topic_logits.shape}")      # (1, 5)
print(f"Toxicity:  {toxicity_logits.shape}")   # (1, 1)
```

### Custom Callbacks for Training

```python
from transformers import TrainerCallback

class CustomLoggingCallback(TrainerCallback):
    """
    Custom callback for Trainer — runs at specific training events.
    """
    def on_train_begin(self, args, state, control, **kwargs):
        print("Training started!")
        print(f"Total steps: {state.max_steps}")
    
    def on_epoch_end(self, args, state, control, **kwargs):
        print(f"\nEpoch {state.epoch:.0f} completed")
    
    def on_evaluate(self, args, state, control, metrics=None, **kwargs):
        if metrics:
            print(f"Eval metrics: {metrics}")
    
    def on_save(self, args, state, control, **kwargs):
        print(f"Checkpoint saved at step {state.global_step}")

class EarlyStoppingOnPlateau(TrainerCallback):
    """
    Stop training if metric doesn't improve for N evaluations.
    More flexible than Trainer's built-in early stopping.
    """
    def __init__(self, patience=3, metric="eval_loss", greater_is_better=False):
        self.patience = patience
        self.metric = metric
        self.greater_is_better = greater_is_better
        self.best_value = None
        self.no_improvement_count = 0
    
    def on_evaluate(self, args, state, control, metrics=None, **kwargs):
        if metrics is None:
            return
        
        current = metrics.get(self.metric)
        if current is None:
            return
        
        if self.best_value is None:
            self.best_value = current
            return
        
        improved = (current > self.best_value if self.greater_is_better 
                   else current < self.best_value)
        
        if improved:
            self.best_value = current
            self.no_improvement_count = 0
        else:
            self.no_improvement_count += 1
            if self.no_improvement_count >= self.patience:
                print(f"\nEarly stopping! No improvement for {self.patience} evaluations.")
                control.should_training_stop = True

# Usage with Trainer
# trainer = Trainer(
#     ...,
#     callbacks=[
#         CustomLoggingCallback(),
#         EarlyStoppingOnPlateau(patience=3, metric="eval_f1", greater_is_better=True)
#     ]
# )
```

---

## Common Mistakes

### 1. Not Setting `padding` and `truncation` Correctly
```python
# ❌ WRONG — variable length tensors can't be batched
encoded = tokenizer(["short", "this is a much longer sentence"])

# ✅ CORRECT — always specify both for batched processing
encoded = tokenizer(
    ["short", "this is a much longer sentence"],
    padding=True,       # pad shorter sequences
    truncation=True,    # truncate longer ones
    max_length=128,     # explicit max length
    return_tensors="pt"
)
```

### 2. Forgetting `model.eval()` and `torch.no_grad()` During Inference
```python
# ❌ WRONG — dropout is active, gradients computed (slow + wrong results)
outputs = model(**inputs)

# ✅ CORRECT — disable dropout, skip gradient computation
model.eval()
with torch.no_grad():
    outputs = model(**inputs)
```

### 3. Using Wrong AutoModel Class
```python
# ❌ WRONG — AutoModel returns embeddings, not predictions
model = AutoModel.from_pretrained("bert-base-uncased")
# model(**inputs) → hidden states, NOT class logits

# ✅ CORRECT — use task-specific class
model = AutoModelForSequenceClassification.from_pretrained(
    "bert-base-uncased", num_labels=2
)
# model(**inputs) → logits for 2 classes
```

### 4. Tokenizer Mismatch
```python
# ❌ WRONG — using BERT tokenizer with GPT model (different vocab!)
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")
model = AutoModelForCausalLM.from_pretrained("gpt2")
# This will produce garbage results

# ✅ CORRECT — always match tokenizer and model
tokenizer = AutoTokenizer.from_pretrained("gpt2")
model = AutoModelForCausalLM.from_pretrained("gpt2")
```

### 5. Not Using `load_best_model_at_end`
```python
# ❌ WRONG — final model might be worse than earlier checkpoints
training_args = TrainingArguments(
    output_dir="./results",
    num_train_epochs=10,
    save_strategy="epoch",
    # Missing: load_best_model_at_end=True
)

# ✅ CORRECT — automatically load the best checkpoint
training_args = TrainingArguments(
    output_dir="./results",
    num_train_epochs=10,
    eval_strategy="epoch",
    save_strategy="epoch",
    load_best_model_at_end=True,    # ← essential!
    metric_for_best_model="f1",     # which metric to optimize
    greater_is_better=True
)
```

### 6. GPU Memory Issues
```python
# ❌ WRONG — OOM (Out of Memory) on large models
model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-2-7b-hf")

# ✅ CORRECT — load in lower precision
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    torch_dtype=torch.float16,          # half precision (halves memory)
    device_map="auto",                  # automatically distribute across GPUs
    load_in_8bit=True                   # 8-bit quantization (quarters memory)
)

# ✅ Also: reduce batch size, use gradient accumulation, use gradient checkpointing
training_args = TrainingArguments(
    per_device_train_batch_size=4,          # smaller batch
    gradient_accumulation_steps=8,          # effective batch = 4 × 8 = 32
    gradient_checkpointing=True,            # trade compute for memory
    fp16=True,                              # mixed precision
)
```

### 7. Forgetting to Set pad_token for GPT-2
```python
# ❌ WRONG — GPT-2 has no pad token by default, crashes on batched input
tokenizer = AutoTokenizer.from_pretrained("gpt2")
tokenizer(["short", "longer sentence"], padding=True)  # Error!

# ✅ CORRECT — set pad token
tokenizer = AutoTokenizer.from_pretrained("gpt2")
tokenizer.pad_token = tokenizer.eos_token  # use end-of-sequence as padding
model.config.pad_token_id = tokenizer.pad_token_id
```

---

## Interview Questions

### Conceptual Questions

**Q1: What is HuggingFace's `pipeline` and when would you use it vs. direct model usage?**

> **Answer**: `pipeline` is a high-level API that wraps tokenization, model inference, and post-processing in one call. Use it for rapid prototyping, demos, or simple production use. Use direct model access when you need: custom batch processing, access to raw logits/embeddings, custom pre/post-processing, or multi-task inference. Pipeline adds overhead (~10-20%) vs direct usage.

**Q2: Explain the difference between `AutoModel`, `AutoModelForSequenceClassification`, and `AutoModelForCausalLM`.**

> **Answer**: `AutoModel` returns raw hidden states (embeddings) — use for custom architectures. `AutoModelForSequenceClassification` adds a linear classification head on top — returns logits for classification. `AutoModelForCausalLM` adds a language modeling head — returns logits over vocabulary for next-token prediction. All auto-detect architecture from model name, but the task-specific head differs.

**Q3: What does `attention_mask` do and why is it important?**

> **Answer**: `attention_mask` is a binary tensor (0s and 1s) that tells the model which tokens are real (1) and which are padding (0). Without it, the model attends to padding tokens, corrupting representations. It's essential for batched inference where sequences have different lengths and are padded to the same size.

**Q4: How does the `Trainer` API handle learning rate scheduling?**

> **Answer**: By default, `Trainer` uses a linear warmup + linear decay schedule. During warmup (controlled by `warmup_steps` or `warmup_ratio`), LR increases from 0 to the specified LR. Then it linearly decays to 0. You can customize with `lr_scheduler_type` parameter: "cosine", "polynomial", "constant_with_warmup", etc. You can also pass a custom scheduler.

**Q5: How would you deploy a HuggingFace model for production inference?**

> **Answer**: (1) **ONNX export** with `optimum` for 2-5x CPU speedup; (2) **Quantization** (dynamic int8 or QAT) to reduce model size 2-4x; (3) **TorchScript** compilation for graph optimization; (4) **Model distillation** (use DistilBERT instead of BERT); (5) **Batched inference** to maximize throughput; (6) **GPU inference** with `fp16` for latency-critical apps; (7) **Inference endpoints** on HuggingFace Hub for serverless deployment.

**Q6: What is the `datasets` library's `.map()` function and why is it efficient?**

> **Answer**: `.map()` applies a function to each example (or batch) in the dataset. It's efficient because: (1) Results are cached on disk — rerunning skips processing; (2) Uses Apache Arrow format — memory-mapped, processes datasets larger than RAM; (3) Supports multi-processing via `num_proc` parameter; (4) Batched mode (`batched=True`) is much faster, especially for tokenization.

### Practical Coding Questions

**Q7: Write code to fine-tune BERT on a custom CSV dataset with HuggingFace.**

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification, Trainer, TrainingArguments
from datasets import load_dataset

dataset = load_dataset("csv", data_files={"train": "train.csv", "test": "test.csv"})
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")

tokenized = dataset.map(
    lambda x: tokenizer(x["text"], padding="max_length", truncation=True, max_length=128),
    batched=True
)

model = AutoModelForSequenceClassification.from_pretrained("bert-base-uncased", num_labels=2)
trainer = Trainer(
    model=model,
    args=TrainingArguments(output_dir="./out", num_train_epochs=3, learning_rate=2e-5),
    train_dataset=tokenized["train"],
    eval_dataset=tokenized["test"],
    tokenizer=tokenizer,
)
trainer.train()
```

**Q8: How do you handle a model that's too large for your GPU?**

> Strategies: (1) `fp16=True` or `bf16=True`; (2) `load_in_8bit=True` or `load_in_4bit=True` with bitsandbytes; (3) `device_map="auto"` for multi-GPU; (4) Gradient checkpointing; (5) Gradient accumulation (smaller batch, same effective batch); (6) Use a smaller model variant (DistilBERT, TinyBERT); (7) LoRA/QLoRA for parameter-efficient fine-tuning.

---

## Quick Reference

### Pipeline Tasks Cheat Sheet

| Task | Pipeline String | Default Model | Output |
|------|----------------|---------------|--------|
| Sentiment | `"sentiment-analysis"` | distilbert-base-uncased-finetuned-sst-2-english | label + score |
| NER | `"ner"` | dbmdz/bert-large-cased-finetuned-conll03-english | entities + positions |
| QA | `"question-answering"` | distilbert-base-cased-distilled-squad | answer + score |
| Summarization | `"summarization"` | sshleifer/distilbart-cnn-12-6 | summary text |
| Translation | `"translation_XX_to_YY"` | Helsinki-NLP models | translated text |
| Generation | `"text-generation"` | gpt2 | generated text |
| Fill-Mask | `"fill-mask"` | distilroberta-base | top tokens + scores |
| Zero-Shot | `"zero-shot-classification"` | facebook/bart-large-mnli | labels + scores |
| Embeddings | `"feature-extraction"` | varies | hidden state tensors |

### Model Loading Cheat Sheet

```python
from transformers import (
    AutoTokenizer,                          # Always matches tokenizer to model
    AutoModel,                              # Base: returns embeddings
    AutoModelForSequenceClassification,     # Text → class label
    AutoModelForTokenClassification,        # Token → tag (NER)
    AutoModelForQuestionAnswering,          # Context + Q → answer span
    AutoModelForCausalLM,                   # Text generation (GPT-like)
    AutoModelForMaskedLM,                   # Fill-mask (BERT-like)
    AutoModelForSeq2SeqLM,                 # Seq2Seq (T5/BART-like)
)

# Load model + tokenizer (always use same name!)
model_name = "bert-base-uncased"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSequenceClassification.from_pretrained(model_name, num_labels=2)

# Save locally
model.save_pretrained("./my_model")
tokenizer.save_pretrained("./my_model")

# Load from local
model = AutoModelForSequenceClassification.from_pretrained("./my_model")

# Push to Hub
model.push_to_hub("username/model-name")
```

### Tokenizer Operations Cheat Sheet

| Operation | Code | Returns |
|-----------|------|---------|
| Encode text | `tokenizer("hello")` | dict with input_ids, attention_mask |
| Encode batch | `tokenizer(["a", "b"], padding=True)` | padded batch dict |
| Decode IDs | `tokenizer.decode([101, 7592, 102])` | string |
| Get tokens | `tokenizer.tokenize("hello world")` | list of token strings |
| Token→ID | `tokenizer.convert_tokens_to_ids(["hello"])` | list of ints |
| ID→Token | `tokenizer.convert_ids_to_tokens([7592])` | list of strings |
| Vocab size | `tokenizer.vocab_size` | int |
| Special tokens | `tokenizer.special_tokens_map` | dict |
| Max length | `tokenizer.model_max_length` | int (512 for BERT) |

### Training Arguments Cheat Sheet

```python
TrainingArguments(
    output_dir="./results",              # checkpoint directory
    
    # Core training
    num_train_epochs=3,                  # or max_steps=1000
    learning_rate=2e-5,                  # 2e-5 to 5e-5 for fine-tuning
    per_device_train_batch_size=16,      # per GPU
    gradient_accumulation_steps=2,       # effective batch = 16 × 2 = 32
    
    # Regularization
    weight_decay=0.01,                   # L2 regularization
    warmup_ratio=0.1,                    # LR warmup
    
    # Evaluation & Saving
    eval_strategy="epoch",               # "epoch", "steps", "no"
    save_strategy="epoch",               # match eval_strategy
    load_best_model_at_end=True,
    metric_for_best_model="f1",
    
    # Performance
    fp16=True,                           # mixed precision
    dataloader_num_workers=4,            # parallel data loading
    
    # Logging
    logging_steps=50,
    report_to="wandb",                   # "tensorboard", "none"
)
```

### Common Model Sizes and Requirements

| Model | Parameters | GPU RAM (fp32) | GPU RAM (fp16) | GPU RAM (int8) |
|-------|-----------|---------------|---------------|---------------|
| DistilBERT | 66M | ~0.3 GB | ~0.15 GB | ~0.08 GB |
| BERT-base | 110M | ~0.5 GB | ~0.25 GB | ~0.12 GB |
| BERT-large | 340M | ~1.4 GB | ~0.7 GB | ~0.35 GB |
| GPT-2 | 124M | ~0.5 GB | ~0.25 GB | ~0.12 GB |
| GPT-2 Large | 774M | ~3.1 GB | ~1.6 GB | ~0.8 GB |
| T5-base | 220M | ~0.9 GB | ~0.45 GB | ~0.22 GB |
| LLaMA-7B | 7B | ~28 GB | ~14 GB | ~7 GB |
| LLaMA-13B | 13B | ~52 GB | ~26 GB | ~13 GB |

> **Rule of thumb**: Model RAM ≈ Parameters × bytes per param. fp32 = 4 bytes, fp16 = 2 bytes, int8 = 1 byte. Add ~20% overhead for optimizer states during training.

---

*Previous: [05-Attention-and-Transformers-NLP](05-Attention-and-Transformers-NLP.md)*  
*Next: [07-Named-Entity-Recognition](07-Named-Entity-Recognition.md) — Extracting structured information from text*
