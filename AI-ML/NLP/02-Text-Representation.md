# Chapter 02: Text Representation — BoW, TF-IDF, Word2Vec, GloVe, FastText & Embeddings

---

## 1. The Core Problem: How Do Computers "Read" Text?

### What it is
Text representation (also called **feature extraction** or **text vectorization**) is the process of converting text into numerical vectors that machine learning models can understand. Computers only understand numbers — we need to translate words into math.

**Analogy:** Imagine explaining a painting to someone over the phone using only numbers. You'd need a system: "color at position (0,0) is RGB(255,0,0)." Text representation does the same — translates the richness of language into a numeric format without losing too much meaning.

### Why it matters
- ML models need numbers, not strings — this is the bridge
- The quality of your text representation determines your model's ceiling
- Bad features → no amount of model tuning will help
- This is where the jump from "toy project" to "production system" happens

### The Evolution of Text Representation

```
1950s-2000s          2003-2013              2013-2018            2018-Present
Rule-based     →   Bag of Words/TF-IDF  →  Word Embeddings  →  Contextual Embeddings
                   (sparse, no meaning)    (dense, semantic)    (BERT, GPT)
                   
"one hot"          Count-based            Prediction-based      Attention-based
dim=vocab_size     dim=vocab_size         dim=100-300           dim=768-4096
```

---

## 2. One-Hot Encoding (The Simplest Approach)

### What it is
Each word gets a unique binary vector where only one position is "1" and everything else is "0". The vector length equals the vocabulary size.

### How it works

```
Vocabulary: [cat, dog, bird, fish]  (size = 4)

cat  → [1, 0, 0, 0]
dog  → [0, 1, 0, 0]
bird → [0, 0, 1, 0]
fish → [0, 0, 0, 1]
```

### Implementation

```python
from sklearn.preprocessing import LabelEncoder, OneHotEncoder
import numpy as np

# Method 1: Manual one-hot encoding
vocabulary = ['cat', 'dog', 'bird', 'fish', 'cat', 'dog']
unique_words = sorted(set(vocabulary))
word_to_idx = {word: idx for idx, word in enumerate(unique_words)}

def one_hot_encode(word, vocab_map):
    """Create one-hot vector for a word."""
    vector = np.zeros(len(vocab_map))
    if word in vocab_map:
        vector[vocab_map[word]] = 1
    return vector

# Encode a sentence
sentence = "the cat sat on the dog"
tokens = sentence.split()
# Only encode words in our vocabulary
for token in tokens:
    if token in word_to_idx:
        vec = one_hot_encode(token, word_to_idx)
        print(f"{token:5} → {vec}")

# Output:
# cat   → [0. 1. 0. 0.]
# dog   → [0. 0. 1. 0.]
```

### Problems with One-Hot Encoding

| Problem | Explanation |
|---------|------------|
| **Huge dimensionality** | 100K vocab → 100K-dimensional vectors |
| **No semantic meaning** | distance(cat, dog) = distance(cat, democracy) |
| **No similarity** | dot product of any two words = 0 |
| **Sparse** | 99.99% zeros — waste of memory |
| **OOV** | Can't handle unseen words |

> 💡 **Key Insight:** One-hot encoding treats every word as equally different from every other word. "king" is no more similar to "queen" than it is to "banana." We need something better.

---

## 3. Bag of Words (BoW)

### What it is
Bag of Words represents a document as a vector of word counts. It counts how many times each word appears in the document, ignoring order completely.

**Analogy:** Imagine dumping all words from a document into a bag, shaking it up, and then counting what's inside. You lose the order ("dog bites man" = "man bites dog") but you know what words are present and how often.

### Why it matters
- Simple yet surprisingly effective for many tasks
- Foundation for TF-IDF (the next step)
- Still used in production for fast text classification
- Great baseline — if BoW doesn't work, your data might be the problem

### How it works

```
Document 1: "The cat sat on the mat"
Document 2: "The dog sat on the log"
Document 3: "The cat and the dog"

Vocabulary: [and, cat, dog, log, mat, on, sat, the]

         and  cat  dog  log  mat  on  sat  the
Doc 1: [  0,   1,   0,   0,   1,  1,   1,   2]
Doc 2: [  0,   0,   1,   1,   0,  1,   1,   2]
Doc 3: [  1,   1,   1,   0,   0,  0,   0,   2]
```

### Implementation

```python
from sklearn.feature_extraction.text import CountVectorizer
import pandas as pd

# Sample corpus
corpus = [
    "The cat sat on the mat",
    "The dog sat on the log",
    "The cat and the dog played",
    "Cats and dogs are friends"
]

# Create BoW representation
vectorizer = CountVectorizer()
bow_matrix = vectorizer.fit_transform(corpus)

# View as DataFrame for clarity
feature_names = vectorizer.get_feature_names_out()
bow_df = pd.DataFrame(bow_matrix.toarray(), 
                      columns=feature_names,
                      index=[f"Doc {i+1}" for i in range(len(corpus))])
print(bow_df)
print(f"\nVocabulary size: {len(feature_names)}")
print(f"Matrix shape: {bow_matrix.shape}")
print(f"Sparsity: {1 - bow_matrix.nnz / (bow_matrix.shape[0] * bow_matrix.shape[1]):.2%}")
```

**Output:**
```
       and  are  cat  cats  dog  dogs  friends  log  mat  on  played  sat  the
Doc 1    0    0    1     0    0     0        0    0    1   1       0    1    2
Doc 2    0    0    0     0    1     0        0    1    0   1       0    1    2
Doc 3    1    0    1     0    1     0        0    0    0   0       1    0    2
Doc 4    1    1    0     1    0     1        1    0    0   0       0    0    0

Vocabulary size: 13
Matrix shape: (4, 13)
Sparsity: 69.23%
```

### Advanced BoW Options

```python
# N-gram BoW (captures some word order)
vectorizer_ngram = CountVectorizer(ngram_range=(1, 2))  # unigrams + bigrams
bow_ngram = vectorizer_ngram.fit_transform(corpus)
print(f"Unigram+Bigram features: {len(vectorizer_ngram.get_feature_names_out())}")
# Much larger feature space but captures "not good" as a single feature

# With preprocessing options
vectorizer_advanced = CountVectorizer(
    lowercase=True,           # Convert to lowercase
    stop_words='english',     # Remove English stop words
    max_features=1000,        # Keep top 1000 features only
    min_df=2,                 # Word must appear in at least 2 documents
    max_df=0.95,              # Ignore words appearing in >95% of documents
    ngram_range=(1, 2),       # Unigrams and bigrams
    token_pattern=r'\b\w{2,}\b'  # Only words with 2+ characters
)

bow_advanced = vectorizer_advanced.fit_transform(corpus)
print(f"Advanced BoW features: {bow_advanced.shape[1]}")
```

### Limitations of BoW

| Limitation | Example |
|-----------|---------|
| **No word order** | "dog bites man" = "man bites dog" |
| **No semantics** | "good" and "great" are unrelated |
| **High dimensionality** | Real corpora → 100K+ features |
| **Sparse matrices** | >99% zeros |
| **No context** | "bank" (river) = "bank" (financial) |

---

## 4. TF-IDF (Term Frequency — Inverse Document Frequency)

### What it is
TF-IDF measures how important a word is to a document relative to the entire corpus. It's BoW on steroids — instead of raw counts, it weights words by their **uniqueness** across documents.

**Analogy:** In a class of students writing essays about animals, if everyone uses the word "the," it's not useful for distinguishing essays. But if only one student uses "platypus," that word is highly distinctive. TF-IDF captures this — common words get low scores, rare/distinctive words get high scores.

### Why it matters
- **Industry standard** for traditional text classification, search engines
- Google used TF-IDF variants in its early ranking algorithm
- Excellent baseline — often beats complex models on small datasets
- Fast to compute, easy to interpret
- Still used in production at companies like Elasticsearch, Solr

### How it works — The Math

$$\text{TF-IDF}(t, d, D) = \text{TF}(t, d) \times \text{IDF}(t, D)$$

**Term Frequency (TF):** How often a word appears in a document

$$\text{TF}(t, d) = \frac{\text{count of term } t \text{ in document } d}{\text{total terms in document } d}$$

**Inverse Document Frequency (IDF):** How rare/unique a word is across all documents

$$\text{IDF}(t, D) = \log\frac{N}{|\{d \in D : t \in d\}|}$$

Where $N$ = total number of documents, denominator = number of documents containing term $t$.

**Intuition:**
- Word appears in ALL documents → IDF ≈ 0 (useless word, like "the")
- Word appears in ONE document → IDF is high (very distinctive)

### Worked Example

```
Corpus:
Doc 1: "cat cat dog"
Doc 2: "cat bird"
Doc 3: "dog bird fish"

N = 3 documents

For "cat" in Doc 1:
  TF = 2/3 = 0.667
  IDF = log(3/2) = 0.405   (appears in 2 of 3 docs)
  TF-IDF = 0.667 × 0.405 = 0.270

For "dog" in Doc 1:
  TF = 1/3 = 0.333
  IDF = log(3/2) = 0.405   (appears in 2 of 3 docs)
  TF-IDF = 0.333 × 0.405 = 0.135

For "fish" in Doc 3:
  TF = 1/3 = 0.333
  IDF = log(3/1) = 1.099   (appears in only 1 doc — rare!)
  TF-IDF = 0.333 × 1.099 = 0.366  ← Higher score! More distinctive.
```

### Implementation

```python
from sklearn.feature_extraction.text import TfidfVectorizer
import pandas as pd
import numpy as np

corpus = [
    "The cat sat on the mat",
    "The dog sat on the log", 
    "The cat chased the dog in the park",
    "The bird flew over the lazy dog",
    "Machine learning is a subset of artificial intelligence"
]

# Basic TF-IDF
tfidf_vectorizer = TfidfVectorizer()
tfidf_matrix = tfidf_vectorizer.fit_transform(corpus)

# Display as DataFrame
feature_names = tfidf_vectorizer.get_feature_names_out()
tfidf_df = pd.DataFrame(
    tfidf_matrix.toarray().round(3),
    columns=feature_names,
    index=[f"Doc {i+1}" for i in range(len(corpus))]
)
print(tfidf_df.to_string())
print(f"\nShape: {tfidf_matrix.shape}")
```

```python
# Production-grade TF-IDF configuration
tfidf_production = TfidfVectorizer(
    # Preprocessing
    lowercase=True,
    strip_accents='unicode',
    
    # Tokenization
    analyzer='word',            # 'word', 'char', or 'char_wb'
    ngram_range=(1, 2),         # Unigrams + bigrams
    
    # Vocabulary control
    max_features=10000,         # Limit vocabulary size
    min_df=5,                   # Ignore terms in <5 documents
    max_df=0.85,                # Ignore terms in >85% of documents
    stop_words='english',
    
    # TF-IDF specific
    use_idf=True,               # Use IDF weighting
    smooth_idf=True,            # Add 1 to denominators (prevents division by zero)
    sublinear_tf=True,          # Use log(1 + tf) instead of raw tf
    norm='l2'                   # L2 normalize vectors (unit length)
)

# Fit and transform
tfidf_features = tfidf_production.fit_transform(corpus)
print(f"Features shape: {tfidf_features.shape}")
```

### TF-IDF for Document Similarity

```python
from sklearn.metrics.pairwise import cosine_similarity

corpus = [
    "Python is a great programming language",
    "Python programming is fun and powerful",
    "Java is also a popular programming language",
    "I love cooking Italian food",
    "Italian cuisine is delicious and healthy"
]

tfidf = TfidfVectorizer(stop_words='english')
tfidf_matrix = tfidf.fit_transform(corpus)

# Compute pairwise cosine similarity
similarity_matrix = cosine_similarity(tfidf_matrix)

# Display similarity
sim_df = pd.DataFrame(
    similarity_matrix.round(3),
    columns=[f"Doc{i+1}" for i in range(len(corpus))],
    index=[f"Doc{i+1}" for i in range(len(corpus))]
)
print(sim_df)
```

**Output:**
```
       Doc1   Doc2   Doc3   Doc4   Doc5
Doc1  1.000  0.574  0.449  0.000  0.000
Doc2  0.574  1.000  0.243  0.000  0.000
Doc3  0.449  0.243  1.000  0.000  0.000
Doc4  0.000  0.000  0.000  1.000  0.355
Doc5  0.000  0.000  0.000  0.355  1.000
```

> 📝 **Key Insight:** Programming docs (1-3) cluster together, food docs (4-5) cluster together. TF-IDF + cosine similarity is a simple yet effective search engine!

### TF-IDF as a Simple Search Engine

```python
def simple_search_engine(query, documents, top_k=3):
    """Build a simple search engine using TF-IDF."""
    # Fit TF-IDF on documents
    vectorizer = TfidfVectorizer(stop_words='english')
    doc_vectors = vectorizer.fit_transform(documents)
    
    # Transform query using the same vectorizer
    query_vector = vectorizer.transform([query])
    
    # Compute similarity between query and all documents
    similarities = cosine_similarity(query_vector, doc_vectors).flatten()
    
    # Get top-k results
    top_indices = similarities.argsort()[-top_k:][::-1]
    
    results = []
    for idx in top_indices:
        if similarities[idx] > 0:
            results.append((documents[idx], similarities[idx]))
    
    return results

# Test search engine
documents = [
    "Introduction to machine learning algorithms",
    "Deep learning with neural networks and backpropagation",
    "Natural language processing with transformers",
    "Computer vision and image classification",
    "Reinforcement learning for game playing",
    "Statistical methods for data analysis",
    "Python programming best practices",
]

query = "neural network deep learning"
results = simple_search_engine(query, documents)
for doc, score in results:
    print(f"[{score:.3f}] {doc}")
# [0.756] Deep learning with neural networks and backpropagation
# [0.150] Introduction to machine learning algorithms
```

---

## 5. Word2Vec — Words as Vectors with Meaning

### What it is
Word2Vec learns dense vector representations (embeddings) of words where **semantically similar words have similar vectors**. It's the breakthrough that transformed NLP — words finally become meaningful numbers.

**Analogy:** Imagine a map where every word is a city. In one-hot encoding, all cities are equally far apart. In Word2Vec, synonyms are neighboring cities, antonyms are in opposite directions, and you can even navigate between concepts: "Paris - France + Italy ≈ Rome."

### Why it matters
- **Revolutionary paper** (Mikolov et al., 2013) — started the embedding era
- Captures semantic relationships: king - man + woman ≈ queen
- Dense vectors (100-300 dims) vs sparse (100K+ dims for BoW)
- Pre-trained embeddings transfer knowledge to new tasks
- Foundation for all modern NLP (BERT, GPT are built on this idea)

### How it works — Two Architectures

```
┌─────────────────────────────────────────────────────────┐
│                    Word2Vec                               │
│                                                          │
│  ┌─────────────────┐       ┌─────────────────────────┐  │
│  │    CBOW          │       │    Skip-gram             │  │
│  │                  │       │                          │  │
│  │  Context → Word  │       │  Word → Context          │  │
│  │                  │       │                          │  │
│  │  "the __ sat"    │       │  "cat" → [the, ?, sat]   │  │
│  │   → predict      │       │   → predict surrounding  │  │
│  │   "cat"          │       │   words                  │  │
│  │                  │       │                          │  │
│  │  Faster          │       │  Better for rare words   │  │
│  │  Better for      │       │  Better quality on       │  │
│  │  frequent words  │       │  small datasets          │  │
│  └─────────────────┘       └─────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Skip-gram Architecture (More Popular)

```
Input          Hidden Layer       Output
(one-hot)      (embedding!)       (softmax)

  cat    →    [0.2, -0.5, ...]  →   the (prob=0.15)
  V×1         dim=300                sat (prob=0.12)
                                     on  (prob=0.10)
                                     ...

Training objective: maximize P(context_words | center_word)
The hidden layer weights BECOME the word embeddings!
```

### Mathematical Objective

For Skip-gram, maximize:

$$J(\theta) = \frac{1}{T} \sum_{t=1}^{T} \sum_{-c \leq j \leq c, j \neq 0} \log P(w_{t+j} | w_t)$$

Where:
- $T$ = total words in corpus
- $c$ = context window size
- $P(w_O | w_I) = \frac{\exp(v'_{w_O} \cdot v_{w_I})}{\sum_{w=1}^{V} \exp(v'_w \cdot v_{w_I})}$

> The denominator (softmax over entire vocabulary) is expensive → use **Negative Sampling** or **Hierarchical Softmax** to approximate.

### Training Word2Vec from Scratch

```python
from gensim.models import Word2Vec
from nltk.tokenize import word_tokenize, sent_tokenize
import nltk

# Sample corpus (in production, use millions of sentences)
corpus_text = """
Natural language processing is a field of artificial intelligence.
Machine learning algorithms can process natural language.
Deep learning has revolutionized natural language processing.
Neural networks are used in machine learning and deep learning.
Artificial intelligence includes machine learning and NLP.
Word embeddings capture semantic meaning of words.
Word2Vec and GloVe are popular word embedding methods.
Transformers have replaced RNNs for most NLP tasks.
BERT and GPT are transformer-based language models.
"""

# Tokenize into sentences, then words
sentences = sent_tokenize(corpus_text.strip())
tokenized_sentences = [word_tokenize(sent.lower()) for sent in sentences]

# Train Word2Vec model
model = Word2Vec(
    sentences=tokenized_sentences,
    vector_size=100,       # Embedding dimension (100-300 typical)
    window=5,              # Context window size
    min_count=1,           # Minimum word frequency (use 5+ for large corpora)
    workers=4,             # Parallel threads
    sg=1,                  # 1=Skip-gram, 0=CBOW
    epochs=100,            # Training epochs (more for small corpus)
    negative=5,            # Negative samples (5-20 typical)
    seed=42
)

# Get word vector
vector = model.wv['learning']
print(f"Vector for 'learning': shape={vector.shape}")
print(f"First 10 dims: {vector[:10].round(3)}")

# Find similar words
similar = model.wv.most_similar('learning', topn=5)
print(f"\nMost similar to 'learning':")
for word, score in similar:
    print(f"  {word:20} {score:.4f}")

# Word analogies (king - man + woman = ?)
# Note: needs a large corpus to work well
try:
    result = model.wv.most_similar(positive=['deep', 'language'], negative=['machine'], topn=3)
    print(f"\ndeep - machine + language ≈")
    for word, score in result:
        print(f"  {word}: {score:.4f}")
except KeyError as e:
    print(f"Word not in vocabulary: {e}")
```

### Using Pre-trained Word2Vec (Google News Vectors)

```python
import gensim.downloader as api

# Download pre-trained vectors (1.7GB, trained on Google News, 3M words, 300 dims)
# Uncomment to download (takes time):
# model = api.load('word2vec-google-news-300')

# Or use a smaller pre-trained model
model = api.load('glove-wiki-gigaword-50')  # 50-dim GloVe vectors

# Famous analogy: king - man + woman = queen
result = model.most_similar(positive=['king', 'woman'], negative=['man'], topn=5)
print("king - man + woman =")
for word, score in result:
    print(f"  {word}: {score:.4f}")

# Semantic similarity
print(f"\nSimilarity scores:")
print(f"  cat - dog: {model.similarity('cat', 'dog'):.4f}")
print(f"  cat - car: {model.similarity('cat', 'car'):.4f}")
print(f"  good - great: {model.similarity('good', 'great'):.4f}")
print(f"  good - bad: {model.similarity('good', 'bad'):.4f}")

# Odd one out
odd = model.doesnt_match(['breakfast', 'lunch', 'dinner', 'python'])
print(f"\nOdd one out: {odd}")  # python
```

### Word2Vec Hyperparameters Guide

| Parameter | Default | Effect | Recommendation |
|-----------|---------|--------|----------------|
| `vector_size` | 100 | Embedding dimension | 100-300 (more data → bigger) |
| `window` | 5 | Context window | 5-10 (larger = more topical) |
| `min_count` | 5 | Min word frequency | 5-10 (filters noise) |
| `sg` | 0 | Architecture | 1 (skip-gram) for small data |
| `negative` | 5 | Negative samples | 5-20 |
| `epochs` | 5 | Training passes | 5-50 (more for small corpus) |
| `alpha` | 0.025 | Learning rate | 0.01-0.05 |

---

## 6. GloVe (Global Vectors for Word Representation)

### What it is
GloVe combines the best of count-based methods (like TF-IDF) and prediction-based methods (like Word2Vec). It learns embeddings from a **global word co-occurrence matrix** — capturing how often words appear together across the entire corpus.

**Analogy:** Word2Vec looks at local neighborhoods (sliding window). GloVe looks at the entire city map — counting every time two streets intersect across the whole city. This global view captures relationships that local windows might miss.

### Why it matters
- Often performs as well or better than Word2Vec
- More principled training objective
- Stanford's contribution to the embedding revolution
- Pre-trained on massive corpora (42B tokens from Common Crawl)

### How GloVe Works

```
Step 1: Build co-occurrence matrix X
        where X[i][j] = how often word i appears near word j

Step 2: Train to satisfy:
        w_i · w_j + b_i + b_j = log(X[i][j])
        
        "The dot product of two word vectors should approximate 
         the log of their co-occurrence count"

Step 3: Weighted loss function:
        J = Σ f(X_ij) * (w_i · w_j + b_i + b_j - log(X_ij))²
        
        f(x) = (x/x_max)^α  if x < x_max
              = 1             otherwise
        
        (Downweight very frequent co-occurrences like "the the")
```

### Using Pre-trained GloVe Vectors

```python
import numpy as np

# Method 1: Load GloVe text file manually
def load_glove_vectors(filepath, dim=100):
    """Load GloVe vectors from text file."""
    embeddings = {}
    with open(filepath, 'r', encoding='utf-8') as f:
        for line in f:
            values = line.split()
            word = values[0]
            vector = np.asarray(values[1:], dtype='float32')
            embeddings[word] = vector
    print(f"Loaded {len(embeddings)} word vectors of dimension {dim}")
    return embeddings

# glove_vectors = load_glove_vectors('glove.6B.100d.txt', dim=100)

# Method 2: Using Gensim (easier)
import gensim.downloader as api

glove = api.load('glove-wiki-gigaword-100')  # 100-dim, trained on Wikipedia + Gigaword

# Word relationships
print("king - man + woman =", glove.most_similar(positive=['king', 'woman'], negative=['man'], topn=3))
print("paris - france + japan =", glove.most_similar(positive=['paris', 'japan'], negative=['france'], topn=3))

# Visualize word embeddings with t-SNE
from sklearn.manifold import TSNE
import matplotlib.pyplot as plt

words = ['king', 'queen', 'man', 'woman', 'prince', 'princess',
         'cat', 'dog', 'fish', 'bird',
         'python', 'java', 'code', 'computer']

vectors = np.array([glove[w] for w in words])

# Reduce to 2D for visualization
tsne = TSNE(n_components=2, random_state=42, perplexity=5)
vectors_2d = tsne.fit_transform(vectors)

plt.figure(figsize=(12, 8))
for i, word in enumerate(words):
    plt.scatter(vectors_2d[i, 0], vectors_2d[i, 1])
    plt.annotate(word, (vectors_2d[i, 0]+0.5, vectors_2d[i, 1]+0.5))
plt.title("GloVe Word Embeddings (t-SNE Visualization)")
plt.show()
```

### GloVe Available Pre-trained Models

| Model | Dimensions | Trained On | Vocab Size |
|-------|-----------|------------|-----------|
| glove.6B | 50, 100, 200, 300 | Wikipedia + Gigaword (6B tokens) | 400K |
| glove.42B | 300 | Common Crawl (42B tokens) | 1.9M |
| glove.840B | 300 | Common Crawl (840B tokens) | 2.2M |
| glove.twitter.27B | 25, 50, 100, 200 | Twitter (27B tokens) | 1.2M |

---

## 7. FastText — Subword Embeddings

### What it is
FastText (by Facebook/Meta) extends Word2Vec by representing each word as a **bag of character n-grams**. This means it can generate embeddings for words it's never seen before by combining the embeddings of its character parts.

**Analogy:** Word2Vec memorizes every person's face individually. FastText learns facial features (eyes, nose shape, jaw) and can recognize relatives it's never seen — because family members share features.

### Why it matters
- **Handles OOV (Out-of-Vocabulary) words** — the killer feature
- Works great for morphologically rich languages (Turkish, Finnish, German)
- Handles misspellings gracefully ("amazng" → close to "amazing")
- Better for rare words (they still share subwords with common ones)

### How FastText Works

```
Word: "where"
Character n-grams (n=3): <wh, whe, her, ere, re>
(< and > mark word boundaries)

Vector for "where" = sum of vectors for all its n-grams

New word "wherever" → shares "whe", "her", "ere" with "where"
→ Gets a meaningful vector even if never seen during training!
```

### Implementation

```python
from gensim.models import FastText
from nltk.tokenize import word_tokenize, sent_tokenize

# Training corpus
corpus_text = """
Natural language processing enables computers to understand human language.
Machine learning models can be trained on text data for various NLP tasks.
Deep learning has significantly improved natural language understanding.
Word embeddings provide dense vector representations of words.
FastText handles out-of-vocabulary words using subword information.
Transformers use attention mechanisms for sequence processing.
"""

sentences = [word_tokenize(sent.lower()) for sent in sent_tokenize(corpus_text.strip())]

# Train FastText model
ft_model = FastText(
    sentences=sentences,
    vector_size=100,        # Embedding dimension
    window=5,               # Context window
    min_count=1,            # Min word frequency
    workers=4,
    sg=1,                   # Skip-gram
    epochs=100,
    min_n=3,                # Min character n-gram length
    max_n=6,                # Max character n-gram length
)

# Get vector for a known word
vec = ft_model.wv['language']
print(f"'language' vector shape: {vec.shape}")

# THE MAGIC: Get vector for an UNSEEN word!
vec_unseen = ft_model.wv['languages']  # Never in training data!
print(f"'languages' (OOV) vector shape: {vec_unseen.shape}")

# Even misspellings work
vec_typo = ft_model.wv['languag']  # Typo!
print(f"'languag' (typo) vector shape: {vec_typo.shape}")

# Similarity between original and typo
from numpy import dot
from numpy.linalg import norm

similarity = dot(ft_model.wv['language'], ft_model.wv['languag']) / (
    norm(ft_model.wv['language']) * norm(ft_model.wv['languag'])
)
print(f"Similarity 'language' vs 'languag': {similarity:.4f}")
# High similarity! FastText handles typos gracefully.

# Using pre-trained FastText (Facebook's models)
# import fasttext
# ft_pretrained = fasttext.load_model('cc.en.300.bin')  # 300-dim, trained on Common Crawl
```

### Word2Vec vs GloVe vs FastText Comparison

| Feature | Word2Vec | GloVe | FastText |
|---------|----------|-------|----------|
| **Method** | Predictive (neural net) | Count-based (matrix factorization) | Predictive + subword |
| **OOV Handling** | ❌ None | ❌ None | ✅ Uses character n-grams |
| **Training Speed** | Medium | Fast (matrix ops) | Slower (more params) |
| **Rare Words** | Poor | Medium | Excellent |
| **Morphology** | No | No | Yes |
| **Typo Handling** | ❌ | ❌ | ✅ Graceful degradation |
| **Best For** | General purpose | Large corpora | Morphologically rich languages |
| **Model Size** | Small | Small | Larger (subword vectors) |

---

## 8. Using Embeddings in ML Models

### Creating Document Vectors from Word Embeddings

```python
import numpy as np
from gensim.models import KeyedVectors
import gensim.downloader as api

# Load pre-trained embeddings
wv = api.load('glove-wiki-gigaword-100')

# Method 1: Simple Average (most common)
def document_vector_avg(doc, model, dim=100):
    """Average word vectors to get document vector."""
    words = doc.lower().split()
    vectors = [model[w] for w in words if w in model]
    if vectors:
        return np.mean(vectors, axis=0)
    return np.zeros(dim)

# Method 2: TF-IDF Weighted Average (better!)
from sklearn.feature_extraction.text import TfidfVectorizer

def document_vector_tfidf(doc, corpus, model, dim=100):
    """TF-IDF weighted average of word vectors."""
    # Get TF-IDF weights
    tfidf = TfidfVectorizer()
    tfidf_matrix = tfidf.fit_transform(corpus)
    feature_names = tfidf.get_feature_names_out()
    
    # Get document's TF-IDF values
    doc_idx = corpus.index(doc)
    doc_tfidf = tfidf_matrix[doc_idx].toarray().flatten()
    
    # Weighted average
    weighted_vectors = []
    total_weight = 0
    for i, word in enumerate(feature_names):
        if word in model and doc_tfidf[i] > 0:
            weighted_vectors.append(model[word] * doc_tfidf[i])
            total_weight += doc_tfidf[i]
    
    if weighted_vectors:
        return np.sum(weighted_vectors, axis=0) / total_weight
    return np.zeros(dim)

# Example: Text Classification with Word Embeddings
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score

# Sample dataset
texts = [
    "I love this movie it was great",
    "This film is wonderful and amazing",
    "Terrible movie waste of time",
    "Awful film horrible acting",
    "Best movie I have ever seen",
    "Worst experience terrible direction",
    "Fantastic story beautiful cinematography",
    "Boring plot bad screenplay"
]
labels = [1, 1, 0, 0, 1, 0, 1, 0]  # 1=positive, 0=negative

# Create feature vectors
X = np.array([document_vector_avg(text, wv) for text in texts])
y = np.array(labels)

# Train classifier
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.25, random_state=42)
clf = LogisticRegression(max_iter=1000)
clf.fit(X_train, y_train)
y_pred = clf.predict(X_test)
print(f"Accuracy: {accuracy_score(y_test, y_pred):.2f}")

# Predict new text
new_text = "This movie is absolutely fantastic"
new_vector = document_vector_avg(new_text, wv).reshape(1, -1)
prediction = clf.predict(new_vector)
print(f"'{new_text}' → {'Positive' if prediction[0] == 1 else 'Negative'}")
```

### Embedding Layer in Deep Learning (PyTorch)

```python
import torch
import torch.nn as nn

# Create embedding layer
vocab_size = 10000
embedding_dim = 300

# Random initialization
embedding = nn.Embedding(vocab_size, embedding_dim)

# Input: batch of token IDs
input_ids = torch.tensor([[1, 5, 3, 8], [2, 7, 4, 0]])  # batch_size=2, seq_len=4
embedded = embedding(input_ids)
print(f"Input shape: {input_ids.shape}")
print(f"Output shape: {embedded.shape}")  # (2, 4, 300)

# Initialize with pre-trained vectors
def load_pretrained_embeddings(model, word2idx, pretrained_vectors):
    """Initialize embedding layer with pre-trained vectors."""
    embedding_matrix = np.zeros((len(word2idx), pretrained_vectors.vector_size))
    found = 0
    for word, idx in word2idx.items():
        if word in pretrained_vectors:
            embedding_matrix[idx] = pretrained_vectors[word]
            found += 1
        # else: stays as zeros (or could use random init)
    
    model.weight = nn.Parameter(torch.FloatTensor(embedding_matrix))
    model.weight.requires_grad = True  # Fine-tune embeddings
    print(f"Loaded {found}/{len(word2idx)} pre-trained vectors")
    return model
```

---

## 9. Sentence & Document Embeddings (Beyond Word Level)

### Modern Approaches

```python
# Method 1: Sentence-BERT (production-grade sentence embeddings)
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')  # Fast and good

sentences = [
    "The cat sits on the mat",
    "A feline is resting on a rug",
    "Python is a programming language",
    "The stock market crashed today"
]

# Generate embeddings
embeddings = model.encode(sentences)
print(f"Embedding shape: {embeddings.shape}")  # (4, 384)

# Compute similarity
from sklearn.metrics.pairwise import cosine_similarity
sim_matrix = cosine_similarity(embeddings)

for i in range(len(sentences)):
    for j in range(i+1, len(sentences)):
        print(f"Sim({i+1},{j+1}): {sim_matrix[i][j]:.3f} | '{sentences[i][:30]}' vs '{sentences[j][:30]}'")

# Output:
# Sim(1,2): 0.632 | 'The cat sits on the mat' vs 'A feline is resting on a rug'    (semantically similar!)
# Sim(1,3): 0.041 | 'The cat sits on the mat' vs 'Python is a programming languag'  (unrelated)
# Sim(1,4): 0.052 | 'The cat sits on the mat' vs 'The stock market crashed today'   (unrelated)
```

```python
# Method 2: Using Hugging Face transformers directly
from transformers import AutoTokenizer, AutoModel
import torch

tokenizer = AutoTokenizer.from_pretrained('bert-base-uncased')
model = AutoModel.from_pretrained('bert-base-uncased')

def get_bert_embedding(text):
    """Get BERT [CLS] token embedding for a sentence."""
    inputs = tokenizer(text, return_tensors='pt', padding=True, truncation=True, max_length=512)
    with torch.no_grad():
        outputs = model(**inputs)
    # Use [CLS] token embedding as sentence representation
    cls_embedding = outputs.last_hidden_state[:, 0, :]
    return cls_embedding.numpy().flatten()

emb = get_bert_embedding("Natural language processing is fascinating")
print(f"BERT embedding shape: {emb.shape}")  # (768,)
```

---

## 10. Common Mistakes & Pitfalls

### Mistake 1: Using raw BoW/TF-IDF without proper preprocessing
```python
# BAD: No preprocessing
vectorizer = TfidfVectorizer()
X = vectorizer.fit_transform(["Running runs runner", "running RUNNING"])
# "Running" and "running" are different features!

# GOOD: Preprocess first or use built-in options
vectorizer = TfidfVectorizer(lowercase=True, stop_words='english')
```

### Mistake 2: Training Word2Vec on too small a corpus
```python
# BAD: Training on 100 sentences
# Word2Vec needs millions of words to learn good representations

# GOOD: Use pre-trained embeddings for small datasets
import gensim.downloader as api
pretrained = api.load('word2vec-google-news-300')  # Trained on 100B words
```

### Mistake 3: Ignoring OOV words
```python
# BAD: Silently dropping unknown words
def get_doc_vector_bad(doc, model):
    words = doc.split()
    vectors = [model[w] for w in words if w in model]
    return np.mean(vectors, axis=0) if vectors else np.zeros(100)
    # If most words are OOV, you get a garbage vector!

# GOOD: Track OOV rate and use FastText if high
def get_doc_vector_good(doc, model, dim=100):
    words = doc.split()
    vectors = [model[w] for w in words if w in model]
    oov_count = sum(1 for w in words if w not in model)
    oov_rate = oov_count / len(words) if words else 0
    
    if oov_rate > 0.5:
        print(f"WARNING: {oov_rate:.0%} OOV rate for: '{doc[:50]}...'")
        # Consider using FastText instead!
    
    return np.mean(vectors, axis=0) if vectors else np.zeros(dim)
```

### Mistake 4: Using wrong similarity metric
```python
# BAD: Using Euclidean distance for high-dim embeddings
from scipy.spatial.distance import euclidean
# Euclidean distance doesn't work well in high dimensions (curse of dimensionality)

# GOOD: Always use cosine similarity for word/document embeddings
from sklearn.metrics.pairwise import cosine_similarity
# Cosine similarity measures angle between vectors (direction = meaning)
```

### Mistake 5: Not normalizing TF-IDF vectors
```python
# BAD: Comparing documents of different lengths without normalization
vectorizer = TfidfVectorizer(norm=None)  # No normalization!
# Long documents get higher TF-IDF scores simply because they have more words

# GOOD: L2 normalize (default in sklearn)
vectorizer = TfidfVectorizer(norm='l2')  # Unit vectors, fair comparison
```

---

## 11. Interview Questions

### Conceptual

**Q1: Explain the difference between TF-IDF and word embeddings. When would you use each?**
> **A:** TF-IDF creates sparse vectors based on word frequency/uniqueness — no semantic understanding. Word embeddings create dense vectors that capture meaning (similar words → similar vectors). Use TF-IDF for: small datasets, interpretability needed, traditional ML. Use embeddings for: large datasets, deep learning, when semantic similarity matters.

**Q2: How does Word2Vec capture the relationship "king - man + woman ≈ queen"?**
> **A:** Word2Vec learns vectors where linear relationships correspond to semantic relationships. The vector from "man" to "king" (royalty offset) is similar to the vector from "woman" to "queen." The model learns this because "king" and "queen" appear in similar contexts (royalty), and "king/man" and "queen/woman" share gender context patterns.

**Q3: What is the curse of dimensionality and how do word embeddings help?**
> **A:** In high-dimensional sparse spaces (one-hot, BoW), distance metrics become meaningless — all points are equally far apart. Word embeddings compress into 100-300 dense dimensions where distances are meaningful. This also reduces overfitting and speeds up training.

**Q4: Explain negative sampling in Word2Vec. Why is it needed?**
> **A:** The original softmax requires computing over the entire vocabulary (expensive: O(V)). Negative sampling approximates this by only updating weights for the correct word and k random "negative" (incorrect) words. This makes training O(k) instead of O(V), where k=5-20 vs V=100K+.

**Q5: How does FastText handle misspellings and rare words better than Word2Vec?**
> **A:** FastText represents words as bags of character n-grams. "apple" → {<ap, app, ppl, ple, le>}. A misspelling "aple" shares n-grams with "apple." A rare word shares subwords with common words. Word2Vec assigns a random/zero vector to unseen words — FastText constructs a meaningful one from known parts.

**Q6: What is `sublinear_tf=True` in sklearn's TfidfVectorizer and why use it?**
> **A:** It replaces raw term frequency with $1 + \log(\text{tf})$. Why? A word appearing 100 times isn't 100x more important than appearing once. The log dampens this — diminishing returns on word frequency. Especially important for long documents with repetitive language.

---

## 12. Quick Reference

### Method Selection Guide

| Scenario | Recommended Method | Why |
|----------|-------------------|-----|
| Small dataset, traditional ML | TF-IDF | Fast, no training needed |
| Need semantic similarity | Word2Vec/GloVe | Captures meaning |
| Many OOV/rare words | FastText | Handles unseen words |
| Sentence-level similarity | Sentence-BERT | Purpose-built for sentences |
| State-of-the-art accuracy | BERT/GPT embeddings | Contextual understanding |
| Search engine / IR | TF-IDF + cosine similarity | Simple, fast, effective |
| Production with limited compute | TF-IDF or FastText | Low resource requirements |

### Key Formulas

| Concept | Formula |
|---------|---------|
| **Cosine Similarity** | $\cos(\theta) = \frac{A \cdot B}{\|A\| \|B\|}$ |
| **TF** | $\text{TF}(t,d) = \frac{f_{t,d}}{\sum_{t' \in d} f_{t',d}}$ |
| **IDF** | $\text{IDF}(t) = \log\frac{N}{n_t}$ |
| **TF-IDF** | $\text{TF-IDF} = \text{TF} \times \text{IDF}$ |
| **Word2Vec objective** | $\max \sum_t \sum_{j \in \text{window}} \log P(w_{t+j}|w_t)$ |
| **GloVe objective** | $\min \sum_{i,j} f(X_{ij})(w_i \cdot w_j + b_i + b_j - \log X_{ij})^2$ |

### Embedding Dimensions Cheat Sheet

| Model | Dimensions | Best For |
|-------|-----------|----------|
| GloVe-50d | 50 | Quick experiments, limited memory |
| Word2Vec/GloVe-100d | 100 | Good balance |
| Word2Vec/GloVe-300d | 300 | Production, high accuracy |
| FastText-300d | 300 | Multilingual, rare words |
| BERT-base | 768 | State-of-the-art NLU |
| BERT-large | 1024 | Maximum accuracy |
| GPT-2 | 768 | Text generation |
| Sentence-BERT | 384-768 | Sentence similarity |

---

*Previous: [01-NLP-Fundamentals.md](01-NLP-Fundamentals.md) — Text Processing, Tokenization, Stemming, Lemmatization*
*Next: [03-Text-Classification.md](03-Text-Classification.md) — Sentiment Analysis, Spam Detection, Multi-label Classification*
