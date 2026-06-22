# Chapter 07: Named Entity Recognition (NER)

## Table of Contents
- [What is NER](#what-is-ner)
- [Why NER Matters](#why-ner-matters)
- [How NER Works](#how-ner-works)
- [Tagging Schemes](#tagging-schemes)
- [Traditional Approaches](#traditional-approaches)
- [Deep Learning Approaches](#deep-learning-approaches)
- [spaCy for NER](#spacy-for-ner)
- [HuggingFace for NER](#huggingface-for-ner)
- [Custom NER Training](#custom-ner-training)
- [Advanced NER Topics](#advanced-ner-topics)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is NER

Named Entity Recognition (NER) is the task of identifying and classifying **named entities** (specific things with proper names or categories) in text into predefined categories like:

- **Person** — names of people (e.g., "Elon Musk")
- **Organization** — companies, institutions (e.g., "Google")
- **Location** — places, countries (e.g., "New York")
- **Date/Time** — temporal expressions (e.g., "January 2024")
- **Money** — monetary values (e.g., "$500 million")
- **Miscellaneous** — events, products, etc.

### Simple Analogy

Imagine you're a highlighter detective reading a newspaper. Your job is to highlight every person's name in yellow, every company in blue, every location in green, and every date in pink. That's exactly what NER does — but automatically, using algorithms.

### Example

```
Input:  "Apple CEO Tim Cook announced the new iPhone in San Francisco on September 12."

Output: [Apple]_ORG CEO [Tim Cook]_PERSON announced the new [iPhone]_PRODUCT 
        in [San Francisco]_LOCATION on [September 12]_DATE
```

---

## Why NER Matters

### Real-World Applications

| Application | How NER is Used |
|-------------|----------------|
| **Search Engines** | Understanding what entities a query is about |
| **News Aggregation** | Categorizing news by people, companies, locations |
| **Customer Support** | Extracting product names, order IDs, dates from tickets |
| **Healthcare** | Identifying drug names, diseases, symptoms in medical records |
| **Finance** | Extracting company names, stock tickers, monetary values |
| **Legal** | Identifying parties, dates, clauses in contracts |
| **Resume Parsing** | Extracting skills, companies, degrees, dates |
| **Knowledge Graphs** | Building structured data from unstructured text |

### When You'd Use NER

- Pre-processing step for information extraction pipelines
- Building chatbots that need to understand user mentions
- Any scenario where you need to extract structured information from free text
- Feeding downstream tasks like relation extraction or question answering

---

## How NER Works

### The Core Idea

NER is fundamentally a **sequence labeling** task. For every token (word) in a sentence, the model assigns a label:

```
Token:    Apple    CEO    Tim    Cook    announced   in    San      Francisco
Label:    B-ORG   O      B-PER  I-PER   O          O     B-LOC    I-LOC
```

### Processing Pipeline

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Raw Text    │───>│  Tokenize    │───>│  Feature     │───>│  Classify    │
│              │    │              │    │  Extraction  │    │  Each Token  │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                                                                    │
                                                                    ▼
                                                            ┌──────────────┐
                                                            │  Entity      │
                                                            │  Labels      │
                                                            └──────────────┘
```

### Why It's Hard

1. **Ambiguity**: "Apple" could be a fruit or a company
2. **Context-dependence**: "Washington" could be a person, city, or state
3. **Nested entities**: "New York University" contains "New York" (LOC) inside an ORG
4. **Unknown entities**: New companies/people appear constantly
5. **Multi-word entities**: "United States of America" is one entity spanning 4 tokens

---

## Tagging Schemes

### BIO (Beginning-Inside-Outside)

The most common scheme. Each token gets one of three prefixes:
- **B-** = Beginning of an entity
- **I-** = Inside (continuation) of an entity
- **O** = Outside (not an entity)

```
Tim     Cook    works   at    Apple   Inc
B-PER   I-PER   O      O     B-ORG   I-ORG
```

### BIOES (Begin-Inside-Outside-End-Single)

More expressive scheme that also marks:
- **E-** = End of a multi-token entity
- **S-** = Single-token entity

```
Tim     Cook    works   at    Apple   Inc     in    London
B-PER   E-PER   O      O     B-ORG   E-ORG   O     S-LOC
```

### IOB2 vs IOB1

| Scheme | Rule |
|--------|------|
| IOB1 | B- only used when two entities of same type are adjacent |
| IOB2 | B- always used at the start of an entity (most common today) |

### Comparison Table

| Scheme | Tags | Expressiveness | Usage |
|--------|------|---------------|-------|
| IO | 2 per entity + O | Low (can't separate adjacent same-type entities) | Rarely used |
| BIO/IOB2 | 2 per entity + O | Medium (standard) | Most common |
| BIOES | 4 per entity + O | High (explicit boundaries) | Often better for CRFs |

> **Pro Tip**: BIOES often gives slightly better results than BIO because it gives the model more information about entity boundaries, but BIO is the standard for most benchmarks.

---

## Traditional Approaches

### 1. Rule-Based / Dictionary-Based

```python
# Simple rule-based NER using regex and dictionaries
import re

# Define entity dictionaries
PERSONS = {"Tim Cook", "Elon Musk", "Jeff Bezos", "Satya Nadella"}
ORGS = {"Apple", "Google", "Microsoft", "Tesla", "Amazon"}
LOCATIONS = {"New York", "San Francisco", "London", "Tokyo"}

def rule_based_ner(text):
    """Simple dictionary-based NER"""
    entities = []
    
    # Check for known entities (longest match first)
    for name in sorted(PERSONS, key=len, reverse=True):
        if name in text:
            start = text.index(name)
            entities.append((name, "PERSON", start, start + len(name)))
    
    for org in sorted(ORGS, key=len, reverse=True):
        if org in text:
            start = text.index(org)
            entities.append((org, "ORG", start, start + len(org)))
    
    # Regex for dates
    date_pattern = r'\b\d{1,2}[/-]\d{1,2}[/-]\d{2,4}\b'
    for match in re.finditer(date_pattern, text):
        entities.append((match.group(), "DATE", match.start(), match.end()))
    
    # Regex for money
    money_pattern = r'\$[\d,]+\.?\d*\s*(million|billion|thousand)?'
    for match in re.finditer(money_pattern, text, re.IGNORECASE):
        entities.append((match.group(), "MONEY", match.start(), match.end()))
    
    return entities

# Test
text = "Tim Cook announced Apple's $3 billion deal in New York on 01/15/2024"
results = rule_based_ner(text)
for entity, label, start, end in results:
    print(f"  {entity:20s} -> {label}")
```

**Output:**
```
  Tim Cook             -> PERSON
  Apple                -> ORG
  New York             -> LOCATION
  $3 billion           -> MONEY
  01/15/2024           -> DATE
```

### 2. Feature-Based ML (CRF - Conditional Random Fields)

CRFs are the classic ML approach for sequence labeling:

```python
# Feature-based NER with CRF using sklearn-crfsuite
# pip install sklearn-crfsuite

import sklearn_crfsuite
from sklearn_crfsuite import metrics

def word_to_features(sentence, i):
    """Extract features for a single word in context"""
    word = sentence[i]
    
    features = {
        # Current word features
        'word.lower()': word.lower(),
        'word[-3:]': word[-3:],          # suffix
        'word[-2:]': word[-2:],          # shorter suffix
        'word.isupper()': word.isupper(),
        'word.istitle()': word.istitle(),  # Capitalized?
        'word.isdigit()': word.isdigit(),
        'word.len': len(word),
        
        # Word shape features
        'word.has_hyphen': '-' in word,
        'word.has_digit': any(c.isdigit() for c in word),
    }
    
    # Previous word features (context)
    if i > 0:
        prev_word = sentence[i-1]
        features.update({
            '-1:word.lower()': prev_word.lower(),
            '-1:word.istitle()': prev_word.istitle(),
            '-1:word.isupper()': prev_word.isupper(),
        })
    else:
        features['BOS'] = True  # Beginning of sentence
    
    # Next word features (context)
    if i < len(sentence) - 1:
        next_word = sentence[i+1]
        features.update({
            '+1:word.lower()': next_word.lower(),
            '+1:word.istitle()': next_word.istitle(),
            '+1:word.isupper()': next_word.isupper(),
        })
    else:
        features['EOS'] = True  # End of sentence
    
    return features

def sent_to_features(sentence):
    """Convert entire sentence to feature dicts"""
    return [word_to_features(sentence, i) for i in range(len(sentence))]

# Example training data (tokens and their labels)
train_sents = [
    (["Tim", "Cook", "works", "at", "Apple"], 
     ["B-PER", "I-PER", "O", "O", "B-ORG"]),
    (["Google", "is", "in", "Mountain", "View"],
     ["B-ORG", "O", "O", "B-LOC", "I-LOC"]),
]

# Prepare features
X_train = [sent_to_features(tokens) for tokens, labels in train_sents]
y_train = [labels for tokens, labels in train_sents]

# Train CRF model
crf = sklearn_crfsuite.CRF(
    algorithm='lbfgs',           # Optimization algorithm
    c1=0.1,                      # L1 regularization
    c2=0.1,                      # L2 regularization
    max_iterations=100,
    all_possible_transitions=True  # Allow all label transitions
)
crf.fit(X_train, y_train)

# Predict on new sentence
test_sent = ["Jeff", "Bezos", "founded", "Amazon"]
X_test = [sent_to_features(test_sent)]
predictions = crf.predict(X_test)
print(list(zip(test_sent, predictions[0])))
# [('Jeff', 'B-PER'), ('Bezos', 'I-PER'), ('founded', 'O'), ('Amazon', 'B-ORG')]
```

### Why CRFs Work Well for NER

CRFs model the **joint probability** of the entire label sequence, not just individual labels. This means they can learn:
- "I-PER" should follow "B-PER", not "B-LOC"
- "O" is very unlikely to appear between "B-PER" and "I-PER"

$$P(y|x) = \frac{1}{Z(x)} \exp\left(\sum_{i=1}^{n} \sum_{k} \lambda_k f_k(y_{i-1}, y_i, x, i)\right)$$

Where:
- $y$ = label sequence
- $x$ = input sequence
- $f_k$ = feature functions
- $\lambda_k$ = learned weights
- $Z(x)$ = normalization constant

---

## Deep Learning Approaches

### 1. BiLSTM-CRF (The Classic DL Approach)

```
┌─────────────────────────────────────────────────────────────┐
│                        CRF Layer                             │
│    (Models label transition probabilities)                   │
└─────────────────┬───────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│              BiLSTM Layer                                     │
│  ──→ LSTM ──→    (Forward captures left context)             │
│  ←── LSTM ←──    (Backward captures right context)           │
└─────────────────┬───────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│           Embedding Layer                                     │
│  (Word embeddings + Character embeddings)                    │
└─────────────────┬───────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│              Input Tokens                                     │
│    ["Tim", "Cook", "works", "at", "Apple"]                  │
└─────────────────────────────────────────────────────────────┘
```

```python
# BiLSTM-CRF for NER using PyTorch
import torch
import torch.nn as nn
from torchcrf import CRF  # pip install pytorch-crf

class BiLSTM_CRF(nn.Module):
    def __init__(self, vocab_size, embedding_dim, hidden_dim, num_tags, padding_idx=0):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embedding_dim, padding_idx=padding_idx)
        self.lstm = nn.LSTM(
            embedding_dim, 
            hidden_dim // 2,  # Bidirectional doubles the output
            num_layers=2,
            bidirectional=True,
            batch_first=True,
            dropout=0.3
        )
        self.hidden2tag = nn.Linear(hidden_dim, num_tags)
        self.crf = CRF(num_tags, batch_first=True)
        self.dropout = nn.Dropout(0.5)
    
    def forward(self, x, tags=None, mask=None):
        """
        x: input token IDs [batch_size, seq_len]
        tags: gold labels [batch_size, seq_len] (for training)
        mask: attention mask [batch_size, seq_len]
        """
        # Embed tokens
        embeds = self.dropout(self.embedding(x))
        
        # BiLSTM encoding
        lstm_out, _ = self.lstm(embeds)
        lstm_out = self.dropout(lstm_out)
        
        # Project to tag space
        emissions = self.hidden2tag(lstm_out)
        
        if tags is not None:
            # Training: return negative log-likelihood
            loss = -self.crf(emissions, tags, mask=mask, reduction='mean')
            return loss
        else:
            # Inference: Viterbi decoding
            return self.crf.decode(emissions, mask=mask)

# Model instantiation example
vocab_size = 10000
embedding_dim = 100
hidden_dim = 256
num_tags = 9  # B-PER, I-PER, B-ORG, I-ORG, B-LOC, I-LOC, B-MISC, I-MISC, O

model = BiLSTM_CRF(vocab_size, embedding_dim, hidden_dim, num_tags)
print(f"Model parameters: {sum(p.numel() for p in model.parameters()):,}")
```

### 2. Transformer-Based NER (State of the Art)

Modern NER uses pre-trained transformers with a token classification head:

```
┌─────────────────────────────────────────────────────────────┐
│              Classification Head                              │
│    Linear(hidden_dim → num_labels) per token                 │
└─────────────────┬───────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│              Transformer Encoder (BERT/RoBERTa)               │
│    Self-attention captures full bidirectional context         │
└─────────────────┬───────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────────┐
│              Subword Tokenization                             │
│    "Playing" → ["Play", "##ing"]                             │
│    (Need to handle subword → word alignment!)                │
└─────────────────────────────────────────────────────────────┘
```

> **Critical Point**: Transformers use subword tokenization. A single word like "Washington" might become ["Wash", "##ington"]. You must align predictions back to original words — typically by only using the prediction for the FIRST subword token of each word.

---

## spaCy for NER

### Using Pre-trained Models

```python
# pip install spacy
# python -m spacy download en_core_web_sm  (small)
# python -m spacy download en_core_web_trf  (transformer-based, most accurate)

import spacy

# Load model (sm=small, md=medium, lg=large, trf=transformer)
nlp = spacy.load("en_core_web_sm")

# Process text
text = """Apple Inc. CEO Tim Cook announced a $3 billion acquisition 
in San Francisco on January 15, 2024. The deal involves Google's 
cloud division and will create 5,000 jobs in New York."""

doc = nlp(text)

# Extract entities
print("=" * 60)
print(f"{'Entity':<25} {'Label':<10} {'Start':<6} {'End':<6}")
print("=" * 60)
for ent in doc.ents:
    print(f"{ent.text:<25} {ent.label_:<10} {ent.start_char:<6} {ent.end_char:<6}")

# Output:
# Apple Inc.                ORG        0      10    
# Tim Cook                  PERSON     16     24    
# $3 billion                MONEY      37     47    
# San Francisco             GPE        63     76    
# January 15, 2024          DATE       80     96    
# Google                    ORG        119    125   
# 5,000                     CARDINAL   161    166   
# New York                  GPE        175    183   
```

### spaCy Entity Labels

| Label | Description | Example |
|-------|-------------|---------|
| PERSON | People's names | "Barack Obama" |
| ORG | Organizations | "Apple Inc." |
| GPE | Countries, cities, states | "France" |
| LOC | Non-GPE locations | "Pacific Ocean" |
| DATE | Dates/periods | "January 2024" |
| TIME | Times | "3:00 PM" |
| MONEY | Monetary values | "$500" |
| CARDINAL | Numerals | "five thousand" |
| ORDINAL | "first", "second" | "3rd" |
| PRODUCT | Products | "iPhone 15" |
| EVENT | Named events | "World Cup" |
| WORK_OF_ART | Titles | "The Great Gatsby" |
| LAW | Laws/documents | "First Amendment" |
| LANGUAGE | Languages | "French" |
| NORP | Nationalities/groups | "American" |

### Visualizing NER with displaCy

```python
from spacy import displacy

# Render in Jupyter notebook
displacy.render(doc, style="ent", jupyter=True)

# Save as HTML file
html = displacy.render(doc, style="ent", page=True)
with open("ner_visualization.html", "w") as f:
    f.write(html)

# Customize colors
colors = {"ORG": "#ff6b6b", "PERSON": "#4ecdc4", "GPE": "#45b7d1"}
options = {"colors": colors}
displacy.render(doc, style="ent", options=options, jupyter=True)
```

### spaCy EntityRuler (Rule + ML Hybrid)

```python
import spacy
from spacy.pipeline import EntityRuler

nlp = spacy.load("en_core_web_sm")

# Add custom rules ON TOP of ML predictions
ruler = nlp.add_pipe("entity_ruler", before="ner")

# Define patterns
patterns = [
    {"label": "PRODUCT", "pattern": "iPhone 15"},
    {"label": "PRODUCT", "pattern": "MacBook Pro"},
    {"label": "ORG", "pattern": [{"LOWER": "open"}, {"LOWER": "ai"}]},  # Token pattern
    {"label": "TECH", "pattern": [{"LOWER": {"IN": ["python", "pytorch", "tensorflow"]}}]},
]

ruler.add_patterns(patterns)

# Test
doc = nlp("OpenAI uses Python to build models for the iPhone 15")
for ent in doc.ents:
    print(f"{ent.text:20} {ent.label_}")
# OpenAI               ORG
# Python               TECH
# iPhone 15            PRODUCT
```

---

## HuggingFace for NER

### Using Pre-trained NER Models

```python
from transformers import pipeline

# Load NER pipeline (uses a fine-tuned BERT model by default)
ner_pipeline = pipeline(
    "ner",
    model="dslim/bert-base-NER",  # Popular NER model
    aggregation_strategy="simple"  # Merge subword tokens
)

text = "Elon Musk founded SpaceX in Hawthorne, California in 2002."

# Run NER
results = ner_pipeline(text)

for entity in results:
    print(f"  {entity['word']:20s} | {entity['entity_group']:8s} | "
          f"Score: {entity['score']:.4f} | "
          f"Span: [{entity['start']}, {entity['end']}]")

# Output:
#   Elon Musk            | PER      | Score: 0.9987 | Span: [0, 9]
#   SpaceX               | ORG      | Score: 0.9992 | Span: [18, 24]
#   Hawthorne            | LOC      | Score: 0.9985 | Span: [28, 37]
#   California           | LOC      | Score: 0.9991 | Span: [39, 49]
```

### Aggregation Strategies Explained

```python
# Without aggregation - you see subword tokens
ner_raw = pipeline("ner", model="dslim/bert-base-NER", aggregation_strategy="none")
results_raw = ner_raw("SpaceX launched from Cape Canaveral")
# Shows: "Space", "##X", "Cape", "Can", "##av", "##eral" as separate predictions

# aggregation_strategy options:
# "none"   - No grouping, raw subword predictions
# "simple" - Group adjacent same-type entities (most common)
# "first"  - Use first token's label for the group
# "average"- Average scores across tokens in group
# "max"    - Use highest scoring token's label for group
```

### Fine-Tuning BERT for Custom NER

```python
from transformers import (
    AutoTokenizer, 
    AutoModelForTokenClassification,
    TrainingArguments, 
    Trainer,
    DataCollatorForTokenClassification
)
from datasets import load_dataset
import numpy as np
import evaluate

# Load CoNLL-2003 dataset (standard NER benchmark)
dataset = load_dataset("conll2003")
print(dataset)
# DatasetDict({
#     train: Dataset({features: ['tokens', 'ner_tags'], num_rows: 14041})
#     validation: Dataset({features: ['tokens', 'ner_tags'], num_rows: 3250})
#     test: Dataset({features: ['tokens', 'ner_tags'], num_rows: 3453})
# })

# Label mapping
label_names = dataset["train"].features["ner_tags"].feature.names
# ['O', 'B-PER', 'I-PER', 'B-ORG', 'I-ORG', 'B-LOC', 'I-LOC', 'B-MISC', 'I-MISC']

id2label = {i: label for i, label in enumerate(label_names)}
label2id = {label: i for i, label in enumerate(label_names)}

# Load tokenizer and model
model_name = "bert-base-cased"  # Cased is important for NER!
tokenizer = AutoTokenizer.from_pretrained(model_name)

model = AutoModelForTokenClassification.from_pretrained(
    model_name,
    num_labels=len(label_names),
    id2label=id2label,
    label2id=label2id
)

# Tokenize and align labels (CRITICAL for subword tokenization)
def tokenize_and_align_labels(examples):
    """
    Key challenge: BERT tokenizes words into subwords.
    'Washington' -> ['Wash', '##ington']
    We need to align NER labels to subword tokens.
    Strategy: Only label the first subword; set others to -100 (ignored in loss).
    """
    tokenized = tokenizer(
        examples["tokens"],
        truncation=True,
        is_split_into_words=True,  # Input is already tokenized
        max_length=128
    )
    
    labels = []
    for i, label_ids in enumerate(examples["ner_tags"]):
        word_ids = tokenized.word_ids(batch_index=i)
        previous_word_idx = None
        label_list = []
        
        for word_idx in word_ids:
            if word_idx is None:
                # Special tokens ([CLS], [SEP], [PAD])
                label_list.append(-100)
            elif word_idx != previous_word_idx:
                # First token of a word -> use the word's label
                label_list.append(label_ids[word_idx])
            else:
                # Subsequent subword tokens -> ignore in loss
                label_list.append(-100)
            previous_word_idx = word_idx
        
        labels.append(label_list)
    
    tokenized["labels"] = labels
    return tokenized

# Apply tokenization
tokenized_datasets = dataset.map(
    tokenize_and_align_labels,
    batched=True,
    remove_columns=dataset["train"].column_names
)

# Evaluation metric
seqeval = evaluate.load("seqeval")

def compute_metrics(eval_preds):
    """Compute entity-level F1, precision, recall"""
    logits, labels = eval_preds
    predictions = np.argmax(logits, axis=-1)
    
    # Remove ignored tokens (label == -100)
    true_labels = []
    true_predictions = []
    
    for prediction, label in zip(predictions, labels):
        true_label = []
        true_pred = []
        for p, l in zip(prediction, label):
            if l != -100:
                true_label.append(id2label[l])
                true_pred.append(id2label[p])
        true_labels.append(true_label)
        true_predictions.append(true_pred)
    
    results = seqeval.compute(
        predictions=true_predictions, 
        references=true_labels
    )
    return {
        "precision": results["overall_precision"],
        "recall": results["overall_recall"],
        "f1": results["overall_f1"],
        "accuracy": results["overall_accuracy"],
    }

# Training arguments
training_args = TrainingArguments(
    output_dir="./ner-bert-conll2003",
    evaluation_strategy="epoch",
    save_strategy="epoch",
    learning_rate=2e-5,
    per_device_train_batch_size=16,
    per_device_eval_batch_size=32,
    num_train_epochs=3,
    weight_decay=0.01,
    warmup_ratio=0.1,
    load_best_model_at_end=True,
    metric_for_best_model="f1",
    push_to_hub=False,
)

# Data collator (handles dynamic padding)
data_collator = DataCollatorForTokenClassification(tokenizer=tokenizer)

# Initialize trainer
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_datasets["train"],
    eval_dataset=tokenized_datasets["validation"],
    tokenizer=tokenizer,
    data_collator=data_collator,
    compute_metrics=compute_metrics,
)

# Train!
trainer.train()

# Evaluate on test set
results = trainer.evaluate(tokenized_datasets["test"])
print(f"Test F1: {results['eval_f1']:.4f}")
# Typical result: F1 ~0.91 on CoNLL-2003
```

> **Pro Tip**: Always use `bert-base-cased` (not uncased) for NER! Case information is critical — "Apple" (company) vs "apple" (fruit).

---

## Custom NER Training

### Training Custom NER with spaCy 3

```python
import spacy
from spacy.tokens import DocBin
from spacy.training import Example
import random

# Step 1: Prepare training data in spaCy format
# Format: (text, {"entities": [(start, end, label), ...]})
TRAINING_DATA = [
    ("Uber blew through $1 million in legal fees", 
     {"entities": [(0, 4, "ORG"), (18, 28, "MONEY")]}),
    ("Google rebranded its Bard chatbot to Gemini",
     {"entities": [(0, 6, "ORG"), (21, 25, "PRODUCT"), (37, 43, "PRODUCT")]}),
    ("Microsoft acquired Activision for $69 billion",
     {"entities": [(0, 9, "ORG"), (19, 29, "ORG"), (34, 45, "MONEY")]}),
    ("The iPhone 15 Pro was launched in September 2023",
     {"entities": [(4, 17, "PRODUCT"), (33, 47, "DATE")]}),
    ("Sam Altman is the CEO of OpenAI based in San Francisco",
     {"entities": [(0, 10, "PERSON"), (25, 31, "ORG"), (41, 54, "GPE")]}),
]

# Step 2: Convert to spaCy DocBin format
nlp = spacy.blank("en")  # Start with blank model
doc_bin = DocBin()

for text, annotations in TRAINING_DATA:
    doc = nlp.make_doc(text)
    ents = []
    for start, end, label in annotations["entities"]:
        span = doc.char_span(start, end, label=label)
        if span is not None:
            ents.append(span)
    doc.ents = ents
    doc_bin.add(doc)

# Save training data
doc_bin.to_disk("./train.spacy")

# Step 3: Create spaCy config (typically done via CLI)
# python -m spacy init config config.cfg --lang en --pipeline ner
# Then train:
# python -m spacy train config.cfg --output ./output --paths.train ./train.spacy --paths.dev ./dev.spacy
```

### Training with spaCy Python API

```python
import spacy
from spacy.training import Example
import random

def train_custom_ner(training_data, n_iter=30, model=None):
    """Train a custom NER model from scratch or update existing"""
    
    if model:
        nlp = spacy.load(model)  # Load existing model
    else:
        nlp = spacy.blank("en")  # Start fresh
    
    # Create NER component if not present
    if "ner" not in nlp.pipe_names:
        ner = nlp.add_pipe("ner", last=True)
    else:
        ner = nlp.get_pipe("ner")
    
    # Add entity labels
    for _, annotations in training_data:
        for ent in annotations.get("entities"):
            ner.add_label(ent[2])
    
    # Only train NER
    other_pipes = [pipe for pipe in nlp.pipe_names if pipe != "ner"]
    
    with nlp.disable_pipes(*other_pipes):
        optimizer = nlp.begin_training()
        
        for iteration in range(n_iter):
            random.shuffle(training_data)
            losses = {}
            
            for text, annotations in training_data:
                doc = nlp.make_doc(text)
                example = Example.from_dict(doc, annotations)
                nlp.update([example], drop=0.35, sgd=optimizer, losses=losses)
            
            if iteration % 10 == 0:
                print(f"Iteration {iteration}, Loss: {losses['ner']:.4f}")
    
    return nlp

# Train
nlp = train_custom_ner(TRAINING_DATA)

# Test the trained model
test_text = "Apple CEO Tim Cook revealed the iPhone 16 in New York"
doc = nlp(test_text)
for ent in doc.ents:
    print(f"  {ent.text:20} -> {ent.label_}")
```

### Data Annotation Tools

| Tool | Type | Best For |
|------|------|----------|
| **Label Studio** | Open source, self-hosted | General annotation, team collaboration |
| **Prodigy** | Commercial (by spaCy team) | Fast annotation, active learning |
| **Doccano** | Open source | Simple NER/classification tasks |
| **LightTag** | Commercial | Large teams, quality control |
| **Amazon SageMaker GT** | Cloud | AWS integration, workforce management |

---

## Advanced NER Topics

### 1. Nested NER

Regular NER assumes entities don't overlap. Nested NER handles cases like:
```
[New York]_LOC University  →  [[New York]_LOC University]_ORG
```

```python
# Nested NER approach: Span-based classification
# Instead of labeling each token, enumerate all possible spans
# and classify each span independently

def enumerate_spans(tokens, max_span_length=8):
    """Generate all possible spans up to max length"""
    spans = []
    for start in range(len(tokens)):
        for end in range(start + 1, min(start + max_span_length + 1, len(tokens) + 1)):
            spans.append((start, end))
    return spans

tokens = ["New", "York", "University", "is", "great"]
spans = enumerate_spans(tokens, max_span_length=4)
print(f"Tokens: {len(tokens)}, Possible spans: {len(spans)}")
# Each span gets classified: LOC, ORG, PER, ... or NONE
```

### 2. Few-Shot NER

When you have very little labeled data:

```python
from transformers import pipeline

# Use a zero-shot/few-shot approach with large language models
classifier = pipeline(
    "zero-shot-classification",
    model="facebook/bart-large-mnli"
)

# Or use SetFit/few-shot approaches
# Or use GPT-style models with in-context learning:

def few_shot_ner_prompt(text, examples=None):
    """Create a prompt for few-shot NER using LLMs"""
    prompt = """Extract named entities from the text. 
Classify each entity as: PERSON, ORG, LOCATION, DATE, MONEY, PRODUCT.

Examples:
Text: "Tim Cook announced Apple's new iPhone in San Francisco"
Entities: Tim Cook (PERSON), Apple (ORG), iPhone (PRODUCT), San Francisco (LOCATION)

Text: "Google invested $5 billion in AI research in 2023"  
Entities: Google (ORG), $5 billion (MONEY), 2023 (DATE)

Now extract entities from:
Text: "{text}"
Entities:"""
    return prompt.format(text=text)

prompt = few_shot_ner_prompt("Microsoft CEO Satya Nadella visited London last Tuesday")
print(prompt)
```

### 3. Cross-lingual NER

```python
from transformers import pipeline

# Multilingual NER model
ner_multi = pipeline(
    "ner",
    model="Davlan/xlm-roberta-large-ner-hrl",  # Supports 10+ languages
    aggregation_strategy="simple"
)

# Works across languages!
texts = [
    "Barack Obama visited Paris last summer.",        # English
    "Angela Merkel besuchte Berlin gestern.",         # German  
    "Emmanuel Macron a visité Tokyo en mars.",        # French
]

for text in texts:
    results = ner_multi(text)
    print(f"\n{text}")
    for ent in results:
        print(f"  {ent['word']:20} -> {ent['entity_group']}")
```

### 4. Entity Linking (NER + Disambiguation)

After NER, link entities to a knowledge base (e.g., Wikipedia/Wikidata):

```python
# Entity linking connects mentions to unique entities
# "Apple" in "Apple released iOS 17" -> Q312 (Apple Inc.) in Wikidata
# "Apple" in "I ate an apple" -> Q89 (apple fruit) in Wikidata

# Using spaCy's entity linker
import spacy

# pip install spacy-entity-linker
# Requires: python -m spacy_entity_linker "download_knowledge_base"
nlp = spacy.load("en_core_web_md")
nlp.add_pipe("entityLinker", last=True)

doc = nlp("Apple CEO Tim Cook spoke about Google's new AI features")
for ent in doc.ents:
    print(f"{ent.text:15} | Label: {ent.label_:8} | KB ID: {ent.kb_id_}")
```

### 5. Evaluation Metrics for NER

```python
from seqeval.metrics import (
    classification_report, 
    f1_score, 
    precision_score, 
    recall_score
)

# Ground truth and predictions (in BIO format)
y_true = [['O', 'O', 'B-PER', 'I-PER', 'O', 'B-ORG', 'I-ORG', 'O']]
y_pred = [['O', 'O', 'B-PER', 'I-PER', 'O', 'B-ORG', 'O',     'O']]

# Entity-level evaluation (strict matching)
print(classification_report(y_true, y_pred))
#               precision    recall  f1-score   support
#          ORG       0.00      0.00      0.00         1
#          PER       1.00      1.00      1.00         1
#    micro avg       0.50      0.50      0.50         2
#    macro avg       0.50      0.50      0.50         2

# Key: "Apple Inc" predicted as "Apple" = WRONG (partial match = miss)
```

> **Important**: NER uses **entity-level** (not token-level) evaluation. An entity is correct only if BOTH the boundary AND the type match exactly. Partial matches count as both a false positive and a false negative.

### Evaluation Types

| Type | Rule | Strictness |
|------|------|-----------|
| **Strict** | Exact boundary + correct type | Most strict (standard) |
| **Type** | Any overlap + correct type | Lenient on boundaries |
| **Partial** | Partial boundary match counted | Used for analysis |
| **Surface** | Exact string match regardless of position | Rare |

---

## Common Mistakes

### 1. Using Uncased Models for NER
```python
# WRONG - loses crucial capitalization signal
tokenizer = AutoTokenizer.from_pretrained("bert-base-uncased")

# RIGHT - preserves case information
tokenizer = AutoTokenizer.from_pretrained("bert-base-cased")
```

### 2. Not Handling Subword Alignment
```python
# WRONG - assigning labels to all subword tokens
# "Washington" -> ["Wash", "##ington"] both get B-PER? No!

# RIGHT - only label first subword, ignore rest with -100
# "Washington" -> ["Wash", "##ington"] -> [B-PER, -100]
```

### 3. Inconsistent Annotation
```python
# WRONG - inconsistent labeling in training data
# "Apple Inc." -> ORG in one place
# "Apple Inc." -> PRODUCT in another
# "Apple" -> ORG sometimes, not annotated other times

# RIGHT - clear annotation guidelines + inter-annotator agreement
```

### 4. Ignoring Entity Boundaries in Evaluation
```python
# If true entity is "New York City" but you predict "New York"
# This counts as: 1 False Negative (missed "New York City") 
#                + 1 False Positive (wrong entity "New York")
# NOT as a partial match!
```

### 5. Not Using BIO Constraints in Decoding
```python
# WRONG - allowing impossible sequences like I-PER after B-ORG
# Use CRF layer or constrained decoding to prevent this

# RIGHT - enforce valid transitions
valid_transitions = {
    "O": ["O", "B-PER", "B-ORG", "B-LOC"],
    "B-PER": ["I-PER", "O", "B-PER", "B-ORG", "B-LOC"],
    "I-PER": ["I-PER", "O", "B-PER", "B-ORG", "B-LOC"],
    # I-PER cannot follow B-ORG!
}
```

### 6. Too Little Training Data
- Rule of thumb: Need at least **200-500 examples per entity type**
- For rare entities, use data augmentation:
  - Entity replacement: swap entity mentions with others of same type
  - Context augmentation: use different sentence structures
  - Active learning: prioritize annotation of uncertain examples

### 7. Not Handling Long Documents
```python
# WRONG - truncating at 512 tokens loses entities
doc = tokenizer(long_text, truncation=True, max_length=512)

# RIGHT - sliding window approach
def process_long_document(text, tokenizer, model, stride=128, max_length=512):
    """Process documents longer than model's max length using sliding windows"""
    encodings = tokenizer(
        text,
        return_offsets_mapping=True,
        truncation=False,  # Don't truncate!
        return_tensors="pt"
    )
    
    input_ids = encodings["input_ids"][0]
    total_length = len(input_ids)
    
    all_predictions = []
    for start in range(0, total_length, max_length - stride):
        end = min(start + max_length, total_length)
        chunk_ids = input_ids[start:end].unsqueeze(0)
        # ... run model on chunk, merge predictions
    
    # Merge overlapping predictions (take highest confidence)
    return merge_predictions(all_predictions)
```

---

## Interview Questions

### Conceptual Questions

**Q1: What is NER and how does it differ from POS tagging?**
> Both are sequence labeling tasks, but NER identifies semantic entities (names, organizations) while POS tagging identifies grammatical roles (noun, verb, adjective). NER typically uses BIO tagging over entity categories, while POS uses a fixed grammatical tagset.

**Q2: Explain the BIO tagging scheme. Why not just use entity labels?**
> BIO (Begin-Inside-Outside) is needed to handle multi-word entities and distinguish adjacent entities of the same type. Without B/I distinction, "Tim Cook works at Apple" would have PER, PER which could mean one or two entities. With BIO: B-PER, I-PER clearly marks it as one entity.

**Q3: Why is a CRF layer helpful on top of BiLSTM/BERT?**
> A CRF models the transition probabilities between labels, enforcing valid sequences. Without CRF, the model might predict impossible transitions like I-PER following B-ORG. The CRF learns that I-PER should only follow B-PER or I-PER, improving overall consistency.

**Q4: How do you handle subword tokenization in transformer-based NER?**
> When a word is split into subwords (e.g., "Washington" → "Wash" + "##ington"), we typically assign the NER label only to the first subword token and use -100 (ignore index) for subsequent subwords. During inference, we take the prediction of the first subword as the prediction for the entire word.

**Q5: What's the difference between entity-level and token-level evaluation in NER?**
> Token-level counts each token's prediction separately. Entity-level requires the entire entity span (all tokens) AND the entity type to match exactly. Entity-level is the standard because predicting "New York" when the true entity is "New York City" should be wrong, even though 2/3 tokens are correct.

### Practical Questions

**Q6: You have 50 labeled examples for a new entity type. What approaches would you try?**
> 1. Few-shot learning with LLMs (GPT-4, Claude) using in-context examples
> 2. Transfer learning from a related NER model, fine-tuned on the 50 examples
> 3. Active learning to efficiently label more data
> 4. Data augmentation (entity replacement, paraphrasing)
> 5. Rule-based system bootstrapped by the examples + human refinement

**Q7: How would you build a NER system for a domain with no existing labeled data (e.g., aerospace)?**
> 1. Start with rule-based + dictionary approach using domain glossaries
> 2. Use pre-trained NER model → annotate errors → iterate
> 3. Leverage domain experts for annotation guidelines
> 4. Use active learning to maximize annotation efficiency
> 5. Consider weak supervision (Snorkel) with labeling functions
> 6. Zero-shot approach with LLMs for initial bootstrap

**Q8: How do you handle nested entities?**
> Options: (1) Span-based models that classify all possible spans independently, (2) Multi-layer sequence labeling with separate predictions per entity type, (3) Hypergraph-based methods, (4) Machine Reading Comprehension formulation where each entity type becomes a query.

---

## Quick Reference

### Model Selection Guide

| Scenario | Recommended Approach | F1 (CoNLL-2003) |
|----------|---------------------|-----------------|
| Quick prototype | spaCy `en_core_web_sm` | ~85% |
| Production (English) | Fine-tuned BERT-large | ~92-93% |
| Multilingual | XLM-RoBERTa fine-tuned | ~90% |
| Low resource | Few-shot + LLM | ~80-85% |
| Domain-specific | Fine-tune on domain data | Varies |
| Speed-critical | DistilBERT or BiLSTM-CRF | ~88-90% |

### Key Libraries

| Library | Strength | Use Case |
|---------|----------|----------|
| **spaCy** | Fast, production-ready | Industry NER pipelines |
| **HuggingFace** | Model variety, fine-tuning | Custom model training |
| **Flair** | Stacked embeddings, easy API | Research, multi-task NER |
| **Stanza** | Multi-language, Stanford NLP | Academic, multilingual |
| **sklearn-crfsuite** | Classical ML baseline | Quick baselines, interpretability |

### NER Datasets

| Dataset | Language | Entity Types | Size |
|---------|----------|-------------|------|
| CoNLL-2003 | English/German | PER, ORG, LOC, MISC | 20K sentences |
| OntoNotes 5.0 | English/Chinese/Arabic | 18 types | 1.7M tokens |
| WNUT-2017 | English (social media) | 6 types | 5K tweets |
| Few-NERD | English | 66 fine-grained types | 188K sentences |
| MultiNERD | 10 languages | 15 types | 164K sentences |

### Performance Tips

```python
# 1. Use GPU for transformer models
model.to("cuda")

# 2. Batch processing in spaCy
docs = list(nlp.pipe(texts, batch_size=64))

# 3. Disable unnecessary pipeline components
with nlp.select_pipes(enable=["ner"]):
    doc = nlp(text)

# 4. Use ONNX for inference speedup
from optimum.onnxruntime import ORTModelForTokenClassification
model = ORTModelForTokenClassification.from_pretrained("model_path", export=True)

# 5. Quantization for edge deployment
from transformers import AutoModelForTokenClassification
import torch
model = AutoModelForTokenClassification.from_pretrained("model_path")
quantized = torch.quantization.quantize_dynamic(model, {torch.nn.Linear}, dtype=torch.qint8)
```

### Common Entity Types Across Standards

| CoNLL | OntoNotes | Custom (typical) |
|-------|-----------|-----------------|
| PER | PERSON | PERSON |
| ORG | ORG | ORGANIZATION |
| LOC | GPE, LOC, FAC | LOCATION |
| MISC | NORP, EVENT, PRODUCT, ... | PRODUCT, EVENT, ... |

---

## Summary

NER is a foundational NLP task that bridges unstructured text and structured information. The field has evolved from:

1. **Rule-based** (dictionaries + regex) → Fast but brittle
2. **Feature-engineered ML** (CRF) → Good but requires manual features
3. **Deep Learning** (BiLSTM-CRF) → Strong but needs lots of data
4. **Transformers** (BERT + token classification) → State of the art
5. **LLM-based** (GPT/Claude + prompting) → Flexible, few-shot capable

The best approach depends on your specific needs: data availability, latency requirements, entity types, and languages.
