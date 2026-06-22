# Chapter 03: Text Classification — Sentiment Analysis, Spam Detection, Multi-label & Beyond

---

## 1. What is Text Classification?

### What it is
Text classification is the task of assigning **predefined categories/labels** to a piece of text. It's the most common and commercially valuable NLP task — every time Gmail moves a message to spam, Netflix tags a show by genre, or a company routes a customer complaint to the right department, text classification is at work.

**Analogy:** Imagine a mail room worker sorting incoming letters into bins: "Bills," "Personal," "Junk." They glance at the envelope, maybe read a few lines, and toss it into the right bin. Text classification automates this at scale — millions of "letters" per second.

### Why it matters
- **Most deployed NLP task in industry** — every company with text data uses it
- Powers spam filters, content moderation, sentiment tracking, intent detection
- Revenue-critical: Amazon reviews, app store ratings, customer support routing
- Gateway to NLP — if you master this, you can tackle most NLP problems

### Types of Text Classification

```
                    Text Classification
                          |
        ┌─────────────────┼─────────────────┐
        ↓                 ↓                 ↓
   Binary            Multi-class       Multi-label
   (2 classes)       (N classes,       (N classes,
                      pick ONE)         pick MANY)
        |                 |                 |
   Spam / Not Spam   Pos/Neg/Neutral   Genre tags
   Toxic / Safe      Topic category    Skill tags
   Click / No-click  Language detect   Symptom tags
```

| Type | Example | Key Difference |
|------|---------|---------------|
| **Binary** | Spam vs. Not Spam | Exactly 2 classes |
| **Multi-class** | News → Sports/Politics/Tech/Health | Exactly 1 label per document |
| **Multi-label** | Movie → [Action, Comedy, Romance] | Multiple labels per document |

---

## 2. The Text Classification Pipeline

### End-to-End Architecture

```
Raw Text
   ↓
┌──────────────────────┐
│  1. Text Cleaning     │  Remove HTML, URLs, special chars
│  2. Tokenization      │  Split into words/subwords
│  3. Feature Extraction│  BoW, TF-IDF, or Embeddings
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│  4. Model Training    │  Naive Bayes, SVM, LR, or Deep Learning
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│  5. Evaluation        │  Accuracy, Precision, Recall, F1
└──────────┬───────────┘
           ↓
┌──────────────────────┐
│  6. Deployment        │  API, batch processing, monitoring
└──────────────────────┘
```

---

## 3. Sentiment Analysis (The Classic Use Case)

### What it is
Sentiment analysis determines the **emotional tone** behind text — is the writer happy, angry, neutral? It's the most popular form of text classification.

**Analogy:** Reading someone's facial expression, but for text. A smile = positive sentiment, a frown = negative, poker face = neutral.

### Why it matters
- Brands track sentiment across millions of social media posts
- Stock traders use news sentiment as trading signals
- Product teams prioritize features based on review sentiment
- Political campaigns monitor public opinion in real time

### Levels of Sentiment Analysis

```
Document Level:  "This movie was great!" → Positive
                 (Whole document gets one label)

Sentence Level:  "The acting was great but the plot was terrible."
                 Sentence 1: "The acting was great" → Positive
                 Sentence 2: "the plot was terrible" → Negative

Aspect Level:    "The food was delicious but the service was slow."
                 Food → Positive
                 Service → Negative
                 (Most granular and most useful commercially)
```

### Complete Sentiment Analysis Project

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.naive_bayes import MultinomialNB
from sklearn.svm import LinearSVC
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import (classification_report, confusion_matrix, 
                             accuracy_score, f1_score)
import re

# ──────────────────────────────────────────────
# Step 1: Load and Explore Data
# ──────────────────────────────────────────────

# Using a sample dataset (in production, use IMDB, Yelp, or Amazon reviews)
# Creating a sample dataset for demonstration
data = {
    'text': [
        "This movie was absolutely wonderful and I loved every moment",
        "Terrible film, complete waste of money and time",
        "The acting was superb, the plot was gripping and emotional",
        "Worst movie I have ever seen, boring and predictable",
        "A masterpiece of cinema, beautiful storytelling",
        "Awful movie with terrible acting and no plot",
        "Brilliant performance by the lead actor, must watch",
        "I fell asleep halfway through, incredibly dull",
        "One of the best films of the year, highly recommend",
        "Disappointing and overrated, don't waste your time",
        "Loved the soundtrack and the visuals were stunning",
        "The movie was okay, nothing special but not bad either",
        "Absolutely fantastic, I was on the edge of my seat",
        "Horrible script and poor direction throughout",
        "Amazing cinematography and a powerful message",
        "Not great not terrible, just an average film",
        "This is the worst thing I have ever watched",
        "Incredible storyline with unexpected twists",
        "Mediocre at best, forgettable and bland",
        "A beautiful and moving cinematic experience",
    ],
    'sentiment': [1,0,1,0,1,0,1,0,1,0,1,1,1,0,1,0,0,1,0,1]
    # 1 = positive, 0 = negative
}

df = pd.DataFrame(data)
print(f"Dataset shape: {df.shape}")
print(f"\nClass distribution:\n{df['sentiment'].value_counts()}")
print(f"\nSample reviews:")
print(df.head())
```

```python
# ──────────────────────────────────────────────
# Step 2: Text Preprocessing
# ──────────────────────────────────────────────

import nltk
from nltk.corpus import stopwords
from nltk.stem import WordNetLemmatizer
from nltk.tokenize import word_tokenize

nltk.download('punkt_tab', quiet=True)
nltk.download('stopwords', quiet=True)
nltk.download('wordnet', quiet=True)

# Custom stop words — KEEP negation words for sentiment!
stop_words = set(stopwords.words('english'))
negation_words = {'not', 'no', 'never', 'neither', 'nor', 'hardly',
                  'barely', 'scarcely', "n't", 'nobody', 'nothing',
                  'nowhere', 'neither', 'nor'}
sentiment_stop_words = stop_words - negation_words  # Keep negation!

lemmatizer = WordNetLemmatizer()

def preprocess_for_sentiment(text):
    """Preprocess text specifically for sentiment analysis."""
    # Lowercase
    text = text.lower()
    
    # Remove HTML tags
    text = re.sub(r'<[^>]+>', '', text)
    
    # Remove URLs
    text = re.sub(r'https?://\S+|www\.\S+', '', text)
    
    # Handle negation — attach "not_" to next word
    # "not good" → "not_good" (treats it as a single negative feature)
    text = re.sub(r'\b(not|no|never|n\'t)\s+(\w+)', r'\1_\2', text)
    
    # Remove punctuation but keep underscores (for not_good)
    text = re.sub(r'[^\w\s]', '', text)
    
    # Tokenize
    tokens = word_tokenize(text)
    
    # Remove stop words (but keep negation) and lemmatize
    tokens = [lemmatizer.lemmatize(t) for t in tokens 
              if t not in sentiment_stop_words and len(t) > 1]
    
    return ' '.join(tokens)

# Apply preprocessing
df['clean_text'] = df['text'].apply(preprocess_for_sentiment)

print("Original vs Cleaned:")
for i in range(3):
    print(f"\n  Original: {df['text'].iloc[i]}")
    print(f"  Cleaned:  {df['clean_text'].iloc[i]}")
```

```python
# ──────────────────────────────────────────────
# Step 3: Feature Extraction (TF-IDF)
# ──────────────────────────────────────────────

# Split data
X_train, X_test, y_train, y_test = train_test_split(
    df['clean_text'], df['sentiment'], 
    test_size=0.25, random_state=42, stratify=df['sentiment']
)

# TF-IDF Vectorization
tfidf = TfidfVectorizer(
    max_features=5000,         # Limit vocab size
    ngram_range=(1, 2),        # Unigrams + bigrams
    min_df=1,                  # Min document frequency
    max_df=0.95,               # Max document frequency
    sublinear_tf=True          # Log-scaled term frequency
)

X_train_tfidf = tfidf.fit_transform(X_train)
X_test_tfidf = tfidf.transform(X_test)  # Only transform, don't fit!

print(f"Training features shape: {X_train_tfidf.shape}")
print(f"Test features shape: {X_test_tfidf.shape}")
```

```python
# ──────────────────────────────────────────────
# Step 4: Train Multiple Models and Compare
# ──────────────────────────────────────────────

models = {
    'Logistic Regression': LogisticRegression(max_iter=1000, C=1.0),
    'Naive Bayes': MultinomialNB(alpha=1.0),
    'Linear SVM': LinearSVC(max_iter=10000, C=1.0),
    'Random Forest': RandomForestClassifier(n_estimators=100, random_state=42),
}

results = {}
for name, model in models.items():
    # Train
    model.fit(X_train_tfidf, y_train)
    
    # Predict
    y_pred = model.predict(X_test_tfidf)
    
    # Evaluate
    acc = accuracy_score(y_test, y_pred)
    f1 = f1_score(y_test, y_pred, average='weighted')
    results[name] = {'accuracy': acc, 'f1': f1}
    
    print(f"\n{'='*50}")
    print(f"Model: {name}")
    print(f"Accuracy: {acc:.4f} | F1: {f1:.4f}")
    print(classification_report(y_test, y_pred, target_names=['Negative', 'Positive']))

# Compare results
print("\n" + "="*60)
print(f"{'Model':<25} {'Accuracy':<12} {'F1-Score':<12}")
print("-"*49)
for name, metrics in sorted(results.items(), key=lambda x: x[1]['f1'], reverse=True):
    print(f"{name:<25} {metrics['accuracy']:<12.4f} {metrics['f1']:<12.4f}")
```

> 💡 **Pro Tip:** For sentiment analysis, **Logistic Regression + TF-IDF** is an extremely strong baseline. It often matches or beats deep learning on small-to-medium datasets. Always try this first before going to complex models.

---

## 4. Traditional ML Models for Text Classification

### 4.1 Naive Bayes — The Text Classification Workhorse

#### What it is
Naive Bayes uses Bayes' theorem with the "naive" assumption that features (words) are independent of each other. Despite this unrealistic assumption, it works surprisingly well for text.

**Analogy:** A doctor diagnosing illness by checking symptoms independently — "fever? yes. cough? yes. headache? no." They don't consider that fever+cough together might mean something different. Naive? Yes. Effective? Surprisingly, yes.

#### The Math

$$P(\text{class} | \text{document}) = \frac{P(\text{document} | \text{class}) \times P(\text{class})}{P(\text{document})}$$

With the naive independence assumption:

$$P(\text{document} | \text{class}) = \prod_{i=1}^{n} P(w_i | \text{class})$$

Where $P(w_i | \text{class})$ = probability of word $w_i$ appearing in documents of this class.

**Multinomial Naive Bayes** (for word counts/TF-IDF):

$$P(w_i | c) = \frac{\text{count}(w_i, c) + \alpha}{\text{total words in class } c + \alpha \times |V|}$$

Where $\alpha$ = smoothing parameter (prevents zero probabilities for unseen words).

#### Worked Example

```
Training data:
  "cheap buy now discount"  → Spam
  "cheap offer limited"     → Spam  
  "meeting tomorrow office" → Not Spam
  "lunch meeting today"     → Not Spam

P(Spam) = 2/4 = 0.5
P(Not Spam) = 2/4 = 0.5

Word probabilities (with Laplace smoothing α=1):
P("cheap"|Spam) = (2+1)/(7+10) = 3/17 = 0.176
P("cheap"|Not Spam) = (0+1)/(7+10) = 1/17 = 0.059

New email: "cheap meeting"
P(Spam|"cheap meeting") ∝ P(Spam) × P("cheap"|Spam) × P("meeting"|Spam)
                        ∝ 0.5 × 0.176 × 0.059 = 0.0052
P(Not Spam|"cheap meeting") ∝ 0.5 × 0.059 × 0.176 = 0.0052

→ Equal in this toy example, but with more data, clear differences emerge
```

```python
from sklearn.naive_bayes import MultinomialNB, BernoulliNB, ComplementNB

# MultinomialNB — best for word counts/TF-IDF
mnb = MultinomialNB(alpha=1.0)  # alpha = Laplace smoothing

# BernoulliNB — best for binary features (word present/absent)
bnb = BernoulliNB(alpha=1.0)

# ComplementNB — better for imbalanced datasets
cnb = ComplementNB(alpha=1.0)

# Pro Tip: Tune alpha with cross-validation
from sklearn.model_selection import GridSearchCV

param_grid = {'alpha': [0.001, 0.01, 0.1, 0.5, 1.0, 2.0, 5.0, 10.0]}
grid_search = GridSearchCV(MultinomialNB(), param_grid, cv=5, scoring='f1')
grid_search.fit(X_train_tfidf, y_train)
print(f"Best alpha: {grid_search.best_params_['alpha']}")
print(f"Best F1: {grid_search.best_score_:.4f}")
```

### Naive Bayes Variants Comparison

| Variant | Best For | Feature Type | Key Parameter |
|---------|----------|-------------|---------------|
| **MultinomialNB** | Word counts, TF-IDF | Continuous (≥0) | `alpha` (smoothing) |
| **BernoulliNB** | Binary features | Binary (0/1) | `alpha`, `binarize` |
| **ComplementNB** | Imbalanced classes | Continuous (≥0) | `alpha` |
| **GaussianNB** | Dense features | Any real number | `var_smoothing` |

### 4.2 Logistic Regression — The Strong Baseline

```python
from sklearn.linear_model import LogisticRegression, SGDClassifier

# Standard Logistic Regression
lr = LogisticRegression(
    C=1.0,               # Inverse of regularization strength
    penalty='l2',        # Regularization type (l1 for feature selection)
    solver='lbfgs',      # Optimization algorithm
    max_iter=1000,       # Maximum iterations
    class_weight='balanced',  # Handle imbalanced classes
    random_state=42
)

lr.fit(X_train_tfidf, y_train)
y_pred = lr.predict(X_test_tfidf)

# Get prediction probabilities (useful for thresholding)
y_proba = lr.predict_proba(X_test_tfidf)
print(f"Prediction probabilities shape: {y_proba.shape}")
print(f"Sample: text has P(negative)={y_proba[0][0]:.3f}, P(positive)={y_proba[0][1]:.3f}")

# Feature importance — which words are most predictive?
feature_names = tfidf.get_feature_names_out()
coefs = lr.coef_[0]

# Top positive sentiment words
top_positive_idx = coefs.argsort()[-15:][::-1]
print("\nTop 15 POSITIVE indicators:")
for idx in top_positive_idx:
    print(f"  {feature_names[idx]:20} weight: {coefs[idx]:.4f}")

# Top negative sentiment words
top_negative_idx = coefs.argsort()[:15]
print("\nTop 15 NEGATIVE indicators:")
for idx in top_negative_idx:
    print(f"  {feature_names[idx]:20} weight: {coefs[idx]:.4f}")
```

```python
# For very large datasets — use SGDClassifier (Stochastic Gradient Descent)
# Same model, but scales to millions of documents
sgd = SGDClassifier(
    loss='log_loss',         # Logistic regression loss
    penalty='l2',
    alpha=1e-4,              # Regularization
    max_iter=1000,
    random_state=42,
    class_weight='balanced'
)

sgd.fit(X_train_tfidf, y_train)
print(f"SGD Accuracy: {accuracy_score(y_test, sgd.predict(X_test_tfidf)):.4f}")

# Pro Tip: SGDClassifier with loss='hinge' = Linear SVM
# SGDClassifier with loss='log_loss' = Logistic Regression
# Both scale to millions of samples!
```

### 4.3 Support Vector Machines (SVM)

```python
from sklearn.svm import LinearSVC, SVC

# LinearSVC — the go-to for text classification
svm = LinearSVC(
    C=1.0,               # Regularization parameter
    max_iter=10000,
    class_weight='balanced',
    random_state=42
)

svm.fit(X_train_tfidf, y_train)
y_pred_svm = svm.predict(X_test_tfidf)
print(f"Linear SVM Accuracy: {accuracy_score(y_test, y_pred_svm):.4f}")

# For probabilities with SVM (LinearSVC doesn't support predict_proba)
from sklearn.calibration import CalibratedClassifierCV

calibrated_svm = CalibratedClassifierCV(svm, cv=3)
calibrated_svm.fit(X_train_tfidf, y_train)
y_proba_svm = calibrated_svm.predict_proba(X_test_tfidf)
print(f"SVM with probabilities: {y_proba_svm[0]}")
```

> 💡 **Pro Tip:** For text classification, **LinearSVC is almost always better than SVC with RBF kernel**. Text data is already high-dimensional and often linearly separable. Non-linear kernels add computational cost without benefit.

### Model Comparison Summary

| Model | Strengths | Weaknesses | Best For |
|-------|-----------|------------|----------|
| **Naive Bayes** | Fast, works with small data, probabilistic | Naive independence assumption | Spam, quick baseline |
| **Logistic Regression** | Interpretable, probabilistic, strong baseline | Linear decision boundary | General text classification |
| **Linear SVM** | Best accuracy for high-dim sparse data | No probabilities (without calibration) | When accuracy matters most |
| **Random Forest** | Handles non-linear, feature importance | Slow on sparse data, less effective | Structured + text features |
| **Gradient Boosting** | State-of-the-art for tabular | Very slow on high-dim text | Combined feature types |

---

## 5. Deep Learning for Text Classification

### 5.1 Why Deep Learning?

Traditional ML works great with TF-IDF, but deep learning offers:
- **Automatic feature learning** — no need for manual feature engineering
- **Contextual understanding** — "bank" means different things in different contexts
- **Transfer learning** — pre-trained models (BERT) bring knowledge from massive corpora
- **Better on large datasets** — DL shines with 100K+ examples

### 5.2 Simple Neural Network Classifier

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, TensorDataset

# Convert TF-IDF to PyTorch tensors
X_train_tensor = torch.FloatTensor(X_train_tfidf.toarray())
X_test_tensor = torch.FloatTensor(X_test_tfidf.toarray())
y_train_tensor = torch.LongTensor(y_train.values)
y_test_tensor = torch.LongTensor(y_test.values)

# Create DataLoaders
train_dataset = TensorDataset(X_train_tensor, y_train_tensor)
train_loader = DataLoader(train_dataset, batch_size=8, shuffle=True)

# Define model
class TextClassifier(nn.Module):
    def __init__(self, input_dim, hidden_dim=128, num_classes=2, dropout=0.3):
        super(TextClassifier, self).__init__()
        self.network = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(hidden_dim, hidden_dim // 2),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(hidden_dim // 2, num_classes)
        )
    
    def forward(self, x):
        return self.network(x)

# Initialize
input_dim = X_train_tfidf.shape[1]
model = TextClassifier(input_dim)
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=0.001)

# Train
model.train()
for epoch in range(20):
    total_loss = 0
    for batch_X, batch_y in train_loader:
        optimizer.zero_grad()
        outputs = model(batch_X)
        loss = criterion(outputs, batch_y)
        loss.backward()
        optimizer.step()
        total_loss += loss.item()
    
    if (epoch + 1) % 5 == 0:
        print(f"Epoch {epoch+1}/20, Loss: {total_loss/len(train_loader):.4f}")

# Evaluate
model.eval()
with torch.no_grad():
    outputs = model(X_test_tensor)
    _, predicted = torch.max(outputs, 1)
    acc = (predicted == y_test_tensor).sum().item() / len(y_test_tensor)
    print(f"\nNeural Network Accuracy: {acc:.4f}")
```

### 5.3 LSTM Text Classifier

```python
import torch
import torch.nn as nn
from collections import Counter

class LSTMTextClassifier(nn.Module):
    """LSTM-based text classifier with embedding layer."""
    
    def __init__(self, vocab_size, embed_dim=128, hidden_dim=256, 
                 num_classes=2, num_layers=2, dropout=0.3, bidirectional=True):
        super().__init__()
        
        self.embedding = nn.Embedding(vocab_size, embed_dim, padding_idx=0)
        self.lstm = nn.LSTM(
            embed_dim, hidden_dim,
            num_layers=num_layers,
            batch_first=True,
            dropout=dropout if num_layers > 1 else 0,
            bidirectional=bidirectional
        )
        
        direction_factor = 2 if bidirectional else 1
        self.fc = nn.Sequential(
            nn.Dropout(dropout),
            nn.Linear(hidden_dim * direction_factor, hidden_dim),
            nn.ReLU(),
            nn.Dropout(dropout),
            nn.Linear(hidden_dim, num_classes)
        )
    
    def forward(self, x):
        # x shape: (batch_size, seq_len)
        embedded = self.embedding(x)          # (batch, seq_len, embed_dim)
        lstm_out, (hidden, cell) = self.lstm(embedded)  # hidden: (num_layers*dirs, batch, hidden)
        
        # Concatenate last hidden states from both directions
        if self.lstm.bidirectional:
            hidden = torch.cat((hidden[-2], hidden[-1]), dim=1)
        else:
            hidden = hidden[-1]
        
        output = self.fc(hidden)              # (batch, num_classes)
        return output

# Build vocabulary
def build_vocab(texts, max_vocab=10000):
    """Build word-to-index mapping."""
    counter = Counter()
    for text in texts:
        counter.update(text.lower().split())
    
    # Reserve 0 for padding, 1 for unknown
    word2idx = {'<PAD>': 0, '<UNK>': 1}
    for word, count in counter.most_common(max_vocab - 2):
        word2idx[word] = len(word2idx)
    
    return word2idx

def text_to_indices(text, word2idx, max_len=100):
    """Convert text to padded sequence of indices."""
    tokens = text.lower().split()
    indices = [word2idx.get(t, 1) for t in tokens[:max_len]]  # 1 = <UNK>
    # Pad to max_len
    indices += [0] * (max_len - len(indices))
    return indices

# Usage
# vocab = build_vocab(df['clean_text'])
# model = LSTMTextClassifier(vocab_size=len(vocab))
# ... training loop similar to above
print("LSTM classifier defined — ready for training with proper data")
```

### 5.4 BERT Fine-tuning (State-of-the-Art)

```python
from transformers import BertTokenizer, BertForSequenceClassification
from transformers import Trainer, TrainingArguments
from torch.utils.data import Dataset

class TextDataset(Dataset):
    """Custom dataset for text classification with BERT."""
    
    def __init__(self, texts, labels, tokenizer, max_length=128):
        self.texts = texts
        self.labels = labels
        self.tokenizer = tokenizer
        self.max_length = max_length
    
    def __len__(self):
        return len(self.texts)
    
    def __getitem__(self, idx):
        text = str(self.texts[idx])
        label = self.labels[idx]
        
        encoding = self.tokenizer(
            text,
            add_special_tokens=True,
            max_length=self.max_length,
            padding='max_length',
            truncation=True,
            return_tensors='pt'
        )
        
        return {
            'input_ids': encoding['input_ids'].flatten(),
            'attention_mask': encoding['attention_mask'].flatten(),
            'labels': torch.tensor(label, dtype=torch.long)
        }

# Initialize BERT
tokenizer = BertTokenizer.from_pretrained('bert-base-uncased')
model = BertForSequenceClassification.from_pretrained(
    'bert-base-uncased',
    num_labels=2  # Binary classification
)

# Create datasets
train_dataset = TextDataset(
    X_train.tolist(), y_train.tolist(), tokenizer
)
test_dataset = TextDataset(
    X_test.tolist(), y_test.tolist(), tokenizer
)

# Training arguments
training_args = TrainingArguments(
    output_dir='./results',
    num_train_epochs=3,
    per_device_train_batch_size=16,
    per_device_eval_batch_size=32,
    warmup_steps=100,
    weight_decay=0.01,
    logging_dir='./logs',
    logging_steps=10,
    eval_strategy='epoch',
    save_strategy='epoch',
    load_best_model_at_end=True,
    metric_for_best_model='f1',
)

# Compute metrics function
from sklearn.metrics import accuracy_score, f1_score

def compute_metrics(pred):
    labels = pred.label_ids
    preds = pred.predictions.argmax(-1)
    acc = accuracy_score(labels, preds)
    f1 = f1_score(labels, preds, average='weighted')
    return {'accuracy': acc, 'f1': f1}

# Train
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=test_dataset,
    compute_metrics=compute_metrics,
)

# trainer.train()  # Uncomment to train (needs GPU for speed)
# results = trainer.evaluate()
print("BERT fine-tuning pipeline ready!")
```

> ⚠️ **Important:** BERT fine-tuning requires minimal text preprocessing — just provide raw text. BERT's tokenizer handles everything. **Don't** remove stop words, stem, or lemmatize for BERT — it needs the full context!

### Quick Pipeline with HuggingFace

```python
from transformers import pipeline

# Zero-shot classification — no training needed!
classifier = pipeline("zero-shot-classification", model="facebook/bart-large-mnli")

text = "The new iPhone has an incredible camera and amazing battery life"
labels = ["technology", "sports", "politics", "food", "entertainment"]

result = classifier(text, candidate_labels=labels)
print(f"Text: {text}")
for label, score in zip(result['labels'], result['scores']):
    print(f"  {label:15} → {score:.4f}")

# Sentiment analysis — pre-trained
sentiment = pipeline("sentiment-analysis")
texts = [
    "I absolutely love this product!",
    "This is the worst experience ever",
    "The weather is okay today"
]
results = sentiment(texts)
for text, result in zip(texts, results):
    print(f"  {text:45} → {result['label']} ({result['score']:.4f})")
```

---

## 6. Spam Detection — A Complete Case Study

### Real-world Spam Classification Pipeline

```python
import pandas as pd
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
from sklearn.metrics import classification_report, confusion_matrix
import re

# In production, load SMS Spam Collection or Enron dataset
# df = pd.read_csv('spam.csv', encoding='latin-1')

# Sample data for demonstration
spam_data = {
    'text': [
        "Congratulations! You've won a free iPhone! Click here now!!!",
        "Hey, are we still meeting for lunch tomorrow?",
        "URGENT: Your account will be suspended. Verify now!",
        "Can you pick up groceries on your way home?",
        "FREE entry to win $10000 cash! Text WIN to 12345",
        "The meeting has been rescheduled to 3pm",
        "Limited time offer! Buy 1 get 3 free! Act NOW!!!",
        "Thanks for the birthday wishes, it meant a lot",
        "You have been selected for a special discount offer",
        "Don't forget to submit the report by Friday",
        "WINNER!! You have been chosen for a prize draw",
        "Let me know when you arrive at the airport",
        "Make money fast working from home! No experience needed!",
        "Movie tonight? I'll book the tickets",
        "Claim your free vacation now! Limited spots available!",
        "Happy anniversary! Hope you have a wonderful day",
        "Earn $5000 per week from home! Click link below",
        "The kids have soccer practice at 4pm",
        "Your loan has been approved! Low interest rates!",
        "See you at the conference next week",
    ],
    'label': ['spam','ham','spam','ham','spam','ham','spam','ham','spam','ham',
              'spam','ham','spam','ham','spam','ham','spam','ham','spam','ham']
}

df = pd.DataFrame(spam_data)
df['is_spam'] = (df['label'] == 'spam').astype(int)

# ──── Feature Engineering for Spam Detection ────

def extract_spam_features(text):
    """Extract hand-crafted features useful for spam detection."""
    features = {}
    
    # Text statistics
    features['char_count'] = len(text)
    features['word_count'] = len(text.split())
    features['avg_word_length'] = np.mean([len(w) for w in text.split()]) if text.split() else 0
    
    # Spam indicators
    features['exclamation_count'] = text.count('!')
    features['question_count'] = text.count('?')
    features['uppercase_ratio'] = sum(1 for c in text if c.isupper()) / max(len(text), 1)
    features['has_url'] = int(bool(re.search(r'https?://|www\.', text)))
    features['has_phone'] = int(bool(re.search(r'\d{5,}', text)))
    features['has_currency'] = int(bool(re.search(r'[$£€]', text)))
    
    # Spam keywords
    spam_keywords = ['free', 'win', 'winner', 'cash', 'prize', 'urgent',
                     'click', 'offer', 'limited', 'congratulations', 'selected']
    features['spam_keyword_count'] = sum(1 for kw in spam_keywords if kw in text.lower())
    
    return features

# Extract features
feature_df = pd.DataFrame([extract_spam_features(t) for t in df['text']])
print("Spam feature statistics:")
print(feature_df.groupby(df['is_spam']).mean().round(2).T)
```

```python
# ──── Full Spam Classification Pipeline ────

# Method 1: TF-IDF only
pipe_tfidf = Pipeline([
    ('tfidf', TfidfVectorizer(ngram_range=(1, 2), max_features=5000)),
    ('clf', LogisticRegression(max_iter=1000, class_weight='balanced'))
])

# Cross-validation
scores = cross_val_score(pipe_tfidf, df['text'], df['is_spam'], cv=5, scoring='f1')
print(f"TF-IDF + LR: F1 = {scores.mean():.4f} (+/- {scores.std():.4f})")

# Method 2: TF-IDF + Naive Bayes (classic spam filter)
pipe_nb = Pipeline([
    ('tfidf', TfidfVectorizer(ngram_range=(1, 2), max_features=5000)),
    ('clf', MultinomialNB(alpha=0.1))
])

scores_nb = cross_val_score(pipe_nb, df['text'], df['is_spam'], cv=5, scoring='f1')
print(f"TF-IDF + NB:  F1 = {scores_nb.mean():.4f} (+/- {scores_nb.std():.4f})")

# Train final model
pipe_tfidf.fit(df['text'], df['is_spam'])

# Predict new emails
new_emails = [
    "You just won a lottery! Claim your $1,000,000 prize NOW!!!",
    "Hey, the team lunch is at noon in the cafeteria",
    "EXCLUSIVE offer for you! 90% discount on all products!",
    "Can you review the PR I submitted this morning?"
]

for email in new_emails:
    pred = pipe_tfidf.predict([email])[0]
    proba = pipe_tfidf.predict_proba([email])[0]
    label = "SPAM" if pred == 1 else "HAM"
    print(f"[{label}] (conf: {max(proba):.2f}) {email[:60]}...")
```

---

## 7. Multi-label Classification

### What it is
In multi-label classification, each document can belong to **multiple categories simultaneously**. A movie can be both "Action" AND "Comedy." An article can be tagged "Technology" AND "Business" AND "AI."

### How it differs from multi-class

```
Multi-class:  "This is sports OR politics OR tech"    → ONE label
Multi-label:  "This is sports AND politics AND tech"  → MULTIPLE labels
```

### Implementation

```python
from sklearn.multiclass import OneVsRestClassifier
from sklearn.preprocessing import MultiLabelBinarizer
from sklearn.linear_model import LogisticRegression
from sklearn.feature_extraction.text import TfidfVectorizer

# Sample multi-label data
texts = [
    "Explosive action movie with great comedy moments",
    "Romantic drama with beautiful cinematography",
    "Sci-fi thriller with horror elements",
    "Comedy drama about family relationships",
    "Action adventure with romance subplot",
    "Horror comedy with sci-fi twists",
    "Romantic comedy set in New York",
    "Dark thriller with dramatic performances",
]

# Each document has multiple labels
labels = [
    ['action', 'comedy'],
    ['romance', 'drama'],
    ['sci-fi', 'thriller', 'horror'],
    ['comedy', 'drama'],
    ['action', 'adventure', 'romance'],
    ['horror', 'comedy', 'sci-fi'],
    ['romance', 'comedy'],
    ['thriller', 'drama'],
]

# Binarize labels
mlb = MultiLabelBinarizer()
y = mlb.fit_transform(labels)
print(f"Label classes: {mlb.classes_}")
print(f"Binarized labels shape: {y.shape}")
print(f"Sample binarization: {labels[0]} → {y[0]}")

# TF-IDF features
tfidf = TfidfVectorizer(max_features=1000)
X = tfidf.fit_transform(texts)

# Train multi-label classifier
# OneVsRestClassifier trains one binary classifier per label
clf = OneVsRestClassifier(
    LogisticRegression(max_iter=1000, class_weight='balanced')
)
clf.fit(X, y)

# Predict
new_text = ["A funny action movie with romantic scenes"]
X_new = tfidf.transform(new_text)
predictions = clf.predict(X_new)
predicted_labels = mlb.inverse_transform(predictions)
print(f"\nText: {new_text[0]}")
print(f"Predicted labels: {predicted_labels[0]}")

# Get probabilities for each label
probas = clf.predict_proba(X_new)
print("\nLabel probabilities:")
for label, prob in zip(mlb.classes_, probas[0]):
    print(f"  {label:12} → {prob:.4f}")
```

### Multi-label Evaluation Metrics

```python
from sklearn.metrics import (
    hamming_loss, 
    jaccard_score,
    f1_score
)

# Hamming Loss — fraction of labels that are incorrectly predicted
# Lower is better (0 = perfect)
y_true = np.array([[1,0,1,0], [0,1,0,1], [1,1,0,0]])
y_pred = np.array([[1,0,0,0], [0,1,0,1], [1,1,1,0]])

print(f"Hamming Loss: {hamming_loss(y_true, y_pred):.4f}")

# Subset Accuracy — exact match (all labels must match)
# Very strict metric
exact_match = np.all(y_true == y_pred, axis=1).mean()
print(f"Subset Accuracy: {exact_match:.4f}")

# F1 Score — can compute micro, macro, or sample-averaged
print(f"F1 (micro):   {f1_score(y_true, y_pred, average='micro'):.4f}")
print(f"F1 (macro):   {f1_score(y_true, y_pred, average='macro'):.4f}")
print(f"F1 (samples): {f1_score(y_true, y_pred, average='samples'):.4f}")
```

> 📝 **Note on F1 averages:**
> - **Micro**: Global TP/FP/FN across all labels → favors performance on common labels
> - **Macro**: Average F1 per label → treats all labels equally
> - **Samples**: Average F1 per sample → useful for multi-label

---

## 8. Evaluation Metrics Deep Dive

### The Confusion Matrix

```
                    Predicted
                 Positive  Negative
Actual Positive [   TP    |   FN   ]     ← Recall = TP / (TP + FN)
Actual Negative [   FP    |   TN   ]

                   ↑
           Precision = TP / (TP + FP)

TP = True Positive   (predicted positive, actually positive)
FP = False Positive  (predicted positive, actually negative) — Type I Error
FN = False Negative  (predicted negative, actually positive) — Type II Error
TN = True Negative   (predicted negative, actually negative)
```

### Key Metrics

$$\text{Accuracy} = \frac{TP + TN}{TP + TN + FP + FN}$$

$$\text{Precision} = \frac{TP}{TP + FP} \quad \text{("Of all predicted positive, how many are actually positive?")}$$

$$\text{Recall} = \frac{TP}{TP + FN} \quad \text{("Of all actual positives, how many did we catch?")}$$

$$\text{F1 Score} = 2 \times \frac{\text{Precision} \times \text{Recall}}{\text{Precision} + \text{Recall}} \quad \text{(harmonic mean)}$$

### When to Prioritize Which Metric

| Metric | Prioritize When | Example |
|--------|----------------|---------|
| **Precision** | False positives are costly | Email marked as spam but isn't (user misses important email) |
| **Recall** | False negatives are costly | Cancer detection (missing a real case is dangerous) |
| **F1** | Need balance between precision and recall | General text classification |
| **Accuracy** | Classes are balanced | Balanced sentiment (50/50 pos/neg) |
| **AUC-ROC** | Need threshold-independent evaluation | Ranking/scoring systems |

### Comprehensive Evaluation Code

```python
from sklearn.metrics import (
    classification_report, confusion_matrix, 
    accuracy_score, precision_score, recall_score, f1_score,
    roc_auc_score, roc_curve, precision_recall_curve,
    average_precision_score
)
import matplotlib.pyplot as plt
import seaborn as sns

def evaluate_classifier(y_true, y_pred, y_proba=None, class_names=None):
    """Comprehensive evaluation of a text classifier."""
    
    if class_names is None:
        class_names = ['Negative', 'Positive']
    
    # 1. Basic metrics
    print("=" * 60)
    print("CLASSIFICATION REPORT")
    print("=" * 60)
    print(classification_report(y_true, y_pred, target_names=class_names))
    
    # 2. Confusion matrix
    cm = confusion_matrix(y_true, y_pred)
    plt.figure(figsize=(8, 6))
    sns.heatmap(cm, annot=True, fmt='d', cmap='Blues',
                xticklabels=class_names, yticklabels=class_names)
    plt.title('Confusion Matrix')
    plt.ylabel('Actual')
    plt.xlabel('Predicted')
    plt.tight_layout()
    plt.show()
    
    # 3. ROC Curve (if probabilities available)
    if y_proba is not None:
        # For binary classification
        if len(y_proba.shape) == 2:
            y_score = y_proba[:, 1]
        else:
            y_score = y_proba
        
        fpr, tpr, thresholds = roc_curve(y_true, y_score)
        auc = roc_auc_score(y_true, y_score)
        
        plt.figure(figsize=(8, 6))
        plt.plot(fpr, tpr, 'b-', linewidth=2, label=f'ROC (AUC = {auc:.4f})')
        plt.plot([0, 1], [0, 1], 'r--', label='Random')
        plt.xlabel('False Positive Rate')
        plt.ylabel('True Positive Rate')
        plt.title('ROC Curve')
        plt.legend()
        plt.grid(True, alpha=0.3)
        plt.tight_layout()
        plt.show()
        
        print(f"\nAUC-ROC: {auc:.4f}")
    
    # 4. Summary
    print(f"\nSummary:")
    print(f"  Accuracy:  {accuracy_score(y_true, y_pred):.4f}")
    print(f"  Precision: {precision_score(y_true, y_pred, average='weighted'):.4f}")
    print(f"  Recall:    {recall_score(y_true, y_pred, average='weighted'):.4f}")
    print(f"  F1:        {f1_score(y_true, y_pred, average='weighted'):.4f}")

# Usage:
# evaluate_classifier(y_test, y_pred, y_proba)
```

---

## 9. Handling Imbalanced Datasets

### Why it Matters
Real-world text datasets are almost always imbalanced. Spam might be 5% of emails. Toxic comments might be 2% of all comments. A model that always predicts "not spam" gets 95% accuracy but is useless.

### Strategies

```python
# ──── Strategy 1: Class Weights ────
from sklearn.linear_model import LogisticRegression
from sklearn.utils.class_weight import compute_class_weight

# Automatic class weighting
lr_balanced = LogisticRegression(class_weight='balanced', max_iter=1000)
# 'balanced' automatically adjusts weights inversely proportional to class frequencies

# Manual class weights
class_weights = compute_class_weight('balanced', classes=np.unique(y_train), y=y_train)
weight_dict = dict(zip(np.unique(y_train), class_weights))
print(f"Computed class weights: {weight_dict}")
# e.g., {0: 0.56, 1: 2.33} — minority class gets higher weight

# ──── Strategy 2: Resampling ────
from imblearn.over_sampling import SMOTE, RandomOverSampler
from imblearn.under_sampling import RandomUnderSampler
from imblearn.pipeline import Pipeline as ImbPipeline

# SMOTE (Synthetic Minority Over-sampling Technique)
# Creates synthetic examples of the minority class
smote = SMOTE(random_state=42)
X_resampled, y_resampled = smote.fit_resample(X_train_tfidf, y_train)
print(f"Before SMOTE: {Counter(y_train)}")
print(f"After SMOTE:  {Counter(y_resampled)}")

# Combined pipeline with resampling
imb_pipeline = ImbPipeline([
    ('tfidf', TfidfVectorizer(max_features=5000)),
    ('smote', SMOTE(random_state=42)),
    ('clf', LogisticRegression(max_iter=1000))
])

# ──── Strategy 3: Threshold Tuning ────
# Instead of default 0.5 threshold, find optimal threshold
from sklearn.metrics import precision_recall_curve

def find_optimal_threshold(y_true, y_proba, metric='f1'):
    """Find the threshold that maximizes F1 score."""
    precisions, recalls, thresholds = precision_recall_curve(y_true, y_proba)
    
    # Compute F1 for each threshold
    f1_scores = 2 * (precisions * recalls) / (precisions + recalls + 1e-8)
    
    optimal_idx = np.argmax(f1_scores)
    optimal_threshold = thresholds[optimal_idx] if optimal_idx < len(thresholds) else 0.5
    
    print(f"Optimal threshold: {optimal_threshold:.4f}")
    print(f"F1 at optimal threshold: {f1_scores[optimal_idx]:.4f}")
    
    return optimal_threshold

# Usage:
# optimal_threshold = find_optimal_threshold(y_test, y_proba[:, 1])
# y_pred_tuned = (y_proba[:, 1] >= optimal_threshold).astype(int)
```

---

## 10. Production Tips & Best Practices

### Model Persistence and Deployment

```python
import joblib

# ──── Save complete pipeline ────
pipeline = Pipeline([
    ('tfidf', TfidfVectorizer(max_features=5000, ngram_range=(1, 2))),
    ('clf', LogisticRegression(max_iter=1000))
])
pipeline.fit(df['text'], df['is_spam'])

# Save
joblib.dump(pipeline, 'spam_classifier.joblib')

# Load and predict
loaded_pipeline = joblib.load('spam_classifier.joblib')
prediction = loaded_pipeline.predict(["Free money click now!!!"])
print(f"Prediction: {'spam' if prediction[0] == 1 else 'ham'}")
```

### Prediction Monitoring

```python
def predict_with_confidence(pipeline, text, confidence_threshold=0.7):
    """Predict with confidence check — flag uncertain predictions for review."""
    proba = pipeline.predict_proba([text])[0]
    predicted_class = pipeline.predict([text])[0]
    confidence = max(proba)
    
    result = {
        'text': text,
        'prediction': predicted_class,
        'confidence': confidence,
        'needs_review': confidence < confidence_threshold,
        'probabilities': dict(zip(pipeline.classes_, proba))
    }
    
    if result['needs_review']:
        print(f"⚠️  LOW CONFIDENCE ({confidence:.2f}): '{text[:50]}...'")
    
    return result

# Usage:
# result = predict_with_confidence(loaded_pipeline, "Some ambiguous text")
```

---

## 11. Common Mistakes & Pitfalls

### Mistake 1: Data leakage in text processing
```python
# BAD: Fitting TF-IDF on entire dataset (including test)
tfidf = TfidfVectorizer()
X_all = tfidf.fit_transform(all_texts)  # LEAKAGE! Test data influences vocab
X_train, X_test = X_all[:n_train], X_all[n_train:]

# GOOD: Fit only on training data
tfidf = TfidfVectorizer()
X_train = tfidf.fit_transform(train_texts)    # Fit + Transform
X_test = tfidf.transform(test_texts)           # Only Transform!
```

### Mistake 2: Using accuracy for imbalanced datasets
```python
# BAD: "My model has 98% accuracy!" (on a 98% majority class dataset)
# A model that always predicts the majority class would also get 98%!

# GOOD: Use F1, Precision, Recall, and AUC-ROC
# Also look at the confusion matrix to see per-class performance
```

### Mistake 3: Preprocessing for BERT like traditional ML
```python
# BAD: Heavy preprocessing for BERT
text = remove_stopwords(stem(lowercase(text)))  # DON'T do this for BERT!

# GOOD: Minimal preprocessing for BERT
# BERT was trained on full sentences with punctuation, case, and stop words
# Just clean obvious noise (HTML, URLs) and let BERT handle the rest
```

### Mistake 4: Not stratifying train/test split
```python
# BAD: Random split on imbalanced data
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
# Might get 0% of minority class in test set!

# GOOD: Stratified split
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)
```

### Mistake 5: Ignoring class imbalance
```python
# BAD: Training without addressing imbalance
model = LogisticRegression()

# GOOD: Use class_weight='balanced' as minimum
model = LogisticRegression(class_weight='balanced')
# Or use SMOTE, focal loss, or threshold tuning
```

---

## 12. Interview Questions

### Conceptual

**Q1: You're building a spam filter. Should you optimize for precision or recall? Why?**
> **A:** **Precision** — because a false positive (legitimate email marked as spam) means the user misses an important email. A false negative (spam in inbox) is annoying but not as costly. However, you might tune the threshold based on the business context.

**Q2: Explain the bias-variance tradeoff in the context of text classification.**
> **A:** High bias (underfit): using a simple BoW + Naive Bayes on complex sarcasm detection — the model is too simple. High variance (overfit): using a deep neural network on 100 training samples — memorizes training data. TF-IDF + Logistic Regression is often the sweet spot (medium complexity).

**Q3: How would you handle a dataset where 95% of samples are one class?**
> **A:** (1) Use stratified splits, (2) class_weight='balanced', (3) oversample minority with SMOTE, (4) undersample majority, (5) use appropriate metrics (F1, AUC-ROC, not accuracy), (6) collect more minority samples if possible, (7) try threshold tuning on prediction probabilities.

**Q4: Why does Logistic Regression often outperform deep learning for text classification on small datasets?**
> **A:** With small data (<10K samples), deep learning overfits because it has millions of parameters to learn. LR has far fewer parameters, regularizes well (L1/L2), and TF-IDF features are already very informative. Deep learning excels when data is abundant (>100K) and patterns are complex.

**Q5: What's the difference between micro and macro F1 score?**
> **A:** Micro-F1 computes global TP/FP/FN across all classes — dominated by large classes. Macro-F1 computes F1 per class, then averages — treats all classes equally. Use macro when all classes matter equally (even rare ones). Use micro when you care about overall performance.

**Q6: How would you build a text classifier that can say "I don't know"?**
> **A:** Use prediction probabilities. Set a confidence threshold (e.g., 0.7). If max probability < threshold, output "uncertain" instead of a label. Tune the threshold using a validation set. This is critical in production — wrong confident predictions are worse than admitting uncertainty.

### Coding Question

**Q7: Implement a simple text classification pipeline from scratch (no sklearn Pipeline).**
```python
def text_classification_pipeline(train_texts, train_labels, test_texts):
    """Complete pipeline: preprocess → vectorize → train → predict."""
    import re
    from sklearn.feature_extraction.text import TfidfVectorizer
    from sklearn.linear_model import LogisticRegression
    
    # 1. Preprocess
    def preprocess(text):
        text = text.lower()
        text = re.sub(r'[^\w\s]', '', text)
        text = re.sub(r'\s+', ' ', text).strip()
        return text
    
    train_clean = [preprocess(t) for t in train_texts]
    test_clean = [preprocess(t) for t in test_texts]
    
    # 2. Vectorize
    tfidf = TfidfVectorizer(max_features=5000, ngram_range=(1, 2))
    X_train = tfidf.fit_transform(train_clean)
    X_test = tfidf.transform(test_clean)
    
    # 3. Train
    model = LogisticRegression(max_iter=1000, class_weight='balanced')
    model.fit(X_train, train_labels)
    
    # 4. Predict
    predictions = model.predict(X_test)
    probabilities = model.predict_proba(X_test)
    
    return predictions, probabilities
```

---

## 13. Quick Reference

### Model Selection Flowchart

```
How much labeled data do you have?
│
├── < 100 samples
│   └── Zero-shot classification (HuggingFace pipeline)
│       or few-shot with GPT/Claude API
│
├── 100 - 10K samples
│   └── TF-IDF + Logistic Regression / LinearSVC
│       (strong baseline, fast, interpretable)
│
├── 10K - 100K samples
│   └── Fine-tuned BERT / DistilBERT
│       (best quality, needs GPU)
│
└── 100K+ samples
    └── Fine-tuned large transformer
        or custom LSTM if resource-constrained
```

### Metrics Cheat Sheet

| Metric | Formula | Range | When to Use |
|--------|---------|-------|-------------|
| **Accuracy** | $(TP+TN)/Total$ | [0, 1] | Balanced classes only |
| **Precision** | $TP/(TP+FP)$ | [0, 1] | FP is costly |
| **Recall** | $TP/(TP+FN)$ | [0, 1] | FN is costly |
| **F1** | $2 \cdot P \cdot R / (P+R)$ | [0, 1] | Imbalanced, need balance |
| **AUC-ROC** | Area under ROC curve | [0, 1] | Threshold-independent |
| **Hamming Loss** | Fraction of wrong labels | [0, 1] | Multi-label |

### Sklearn Pipeline Quick Reference

```python
from sklearn.pipeline import Pipeline
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression

# Minimal production pipeline
pipe = Pipeline([
    ('tfidf', TfidfVectorizer(max_features=10000, ngram_range=(1, 2), 
                               sublinear_tf=True, stop_words='english')),
    ('clf', LogisticRegression(C=1.0, max_iter=1000, class_weight='balanced'))
])

pipe.fit(train_texts, train_labels)
predictions = pipe.predict(new_texts)
```

### Common Libraries

| Task | Library | Key Function |
|------|---------|-------------|
| Feature extraction | sklearn | `TfidfVectorizer`, `CountVectorizer` |
| Traditional ML | sklearn | `LogisticRegression`, `MultinomialNB`, `LinearSVC` |
| Deep Learning | PyTorch / TF | `nn.LSTM`, `nn.Linear` |
| Transformers | HuggingFace | `BertForSequenceClassification`, `pipeline` |
| Imbalanced data | imbalanced-learn | `SMOTE`, `RandomOverSampler` |
| Evaluation | sklearn | `classification_report`, `confusion_matrix` |

---

*Previous: [02-Text-Representation.md](02-Text-Representation.md) — BoW, TF-IDF, Word2Vec, GloVe, Embeddings*
*Next: [04-Sequence-Models-for-NLP.md](04-Sequence-Models-for-NLP.md) — RNN, LSTM, GRU, Seq2Seq*
