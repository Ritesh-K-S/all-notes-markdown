# Chapter 01: NLP Fundamentals — Text Processing, Tokenization, Stemming, Lemmatization & Stop Words

---

## 1. What is NLP (Natural Language Processing)?

### What it is
NLP is the field of AI that gives computers the ability to understand, interpret, and generate human language. Think of it like teaching a computer to read, write, and "understand" text the way humans do.

**Analogy:** Imagine you're translating a letter written in a language you don't speak. You'd break it into words, look up meanings, understand grammar rules, and figure out the intent. NLP does exactly this — but programmatically, at massive scale.

### Why it matters
- **$5+ trillion industry** — Powers Google Search, Alexa, ChatGPT, spam filters, autocomplete
- Every company with text data needs NLP: customer reviews, emails, documents, chat logs
- Foundation for all modern AI assistants, translation services, and content moderation

### The NLP Pipeline (Big Picture)

```
Raw Text → Preprocessing → Feature Extraction → Model → Output
   ↓            ↓                ↓                ↓        ↓
"I love     Tokenize,        BoW/TF-IDF/       Classifier  "Positive
 this!"     Clean, Stem      Embeddings         /Generator  Sentiment"
```

---

## 2. Text Preprocessing — The Foundation

### What it is
Text preprocessing is cleaning and standardizing raw text so that a computer can work with it effectively. Raw text is messy — it has punctuation, different cases, extra spaces, HTML tags, emojis, and inconsistencies.

### Why it matters
- **Garbage in = Garbage out** — A model trained on dirty text will produce garbage results
- Reduces vocabulary size (fewer unique tokens = faster training)
- Helps the model focus on meaning rather than surface-level differences
- "Hello", "hello", "HELLO" should all mean the same thing to your model

### Basic Text Cleaning

```python
import re
import string

# Sample messy text
raw_text = """
<p>Hello World!!! I'm learning NLP... 
It's AMAZING!!! 🎉🎉 Visit https://example.com for more.
Price is $99.99  (limited   time).</p>
"""

# Step 1: Remove HTML tags
text = re.sub(r'<[^>]+>', '', raw_text)
print(f"After HTML removal: {text}")

# Step 2: Remove URLs
text = re.sub(r'https?://\S+|www\.\S+', '', text)
print(f"After URL removal: {text}")

# Step 3: Remove emojis (covers most emoji unicode ranges)
text = re.sub(r'[\U0001F600-\U0001F64F\U0001F300-\U0001F5FF\U0001F680-\U0001F6FF]', '', text)
print(f"After emoji removal: {text}")

# Step 4: Lowercase
text = text.lower()
print(f"After lowercasing: {text}")

# Step 5: Remove extra whitespace
text = re.sub(r'\s+', ' ', text).strip()
print(f"After whitespace cleanup: {text}")

# Step 6: Remove punctuation (be careful — sometimes punctuation matters!)
text_no_punct = text.translate(str.maketrans('', '', string.punctuation))
print(f"After punctuation removal: {text_no_punct}")
```

**Output:**
```
After HTML removal: Hello World!!! I'm learning NLP... It's AMAZING!!! 🎉🎉 Visit https://example.com for more. Price is $99.99  (limited   time).
After URL removal: Hello World!!! I'm learning NLP... It's AMAZING!!! 🎉🎉 Visit  for more. Price is $99.99  (limited   time).
After emoji removal: Hello World!!! I'm learning NLP... It's AMAZING!!!  Visit  for more. Price is $99.99  (limited   time).
After lowercasing: hello world!!! i'm learning nlp... it's amazing!!!  visit  for more. price is $99.99  (limited   time).
After whitespace cleanup: hello world!!! i'm learning nlp... it's amazing!!! visit for more. price is $99.99 (limited time).
After punctuation removal: hello world im learning nlp its amazing visit for more price is 9999 limited time
```

> ⚠️ **Warning:** Don't blindly remove all punctuation! In sentiment analysis, "!!!" might indicate strong emotion. In medical text, "3.5mg" needs the period. Always think about what information you're losing.

### Pro Tips for Text Cleaning

```python
# Pro Tip 1: Handle contractions BEFORE removing punctuation
contractions = {
    "i'm": "i am", "it's": "it is", "don't": "do not",
    "won't": "will not", "can't": "cannot", "i've": "i have",
    "they're": "they are", "we're": "we are", "you're": "you are",
    "isn't": "is not", "aren't": "are not", "wasn't": "was not",
    "weren't": "were not", "hasn't": "has not", "haven't": "have not",
    "didn't": "did not", "wouldn't": "would not", "shouldn't": "should not",
    "couldn't": "could not", "let's": "let us", "that's": "that is",
    "who's": "who is", "what's": "what is", "here's": "here is",
    "there's": "there is", "where's": "where is", "how's": "how is",
}

def expand_contractions(text, contraction_map=contractions):
    """Expand contractions in text."""
    for contraction, expansion in contraction_map.items():
        text = text.replace(contraction, expansion)
    return text

sample = "i'm learning nlp and it's amazing. don't stop!"
print(expand_contractions(sample))
# Output: i am learning nlp and it is amazing. do not stop!

# Pro Tip 2: Use a reusable cleaning pipeline
def clean_text(text, 
               remove_html=True, 
               remove_urls=True,
               lowercase=True, 
               expand_contractions_flag=True,
               remove_punctuation=False,
               remove_numbers=False):
    """Comprehensive text cleaning pipeline."""
    if remove_html:
        text = re.sub(r'<[^>]+>', '', text)
    if remove_urls:
        text = re.sub(r'https?://\S+|www\.\S+', '', text)
    if lowercase:
        text = text.lower()
    if expand_contractions_flag:
        text = expand_contractions(text)
    if remove_numbers:
        text = re.sub(r'\d+', '', text)
    if remove_punctuation:
        text = text.translate(str.maketrans('', '', string.punctuation))
    # Always clean whitespace
    text = re.sub(r'\s+', ' ', text).strip()
    return text

# Usage
dirty = "<b>I can't believe it's $99!! Visit http://sale.com NOW</b>"
clean = clean_text(dirty, remove_punctuation=True)
print(clean)  # "i can not believe it is  now"
```

---

## 3. Tokenization

### What it is
Tokenization is splitting text into smaller pieces called **tokens**. A token can be a word, a subword, a character, or even a sentence — depending on the method you choose.

**Analogy:** Imagine you're a chef preparing ingredients. Before you can cook (analyze text), you need to chop your vegetables (break text into pieces). Tokenization is that chopping step — the size of your pieces depends on what you're making.

### Why it matters
- Computers can't process raw text — they need discrete units
- The choice of tokenization method dramatically affects model performance
- Modern LLMs (GPT, BERT) use subword tokenization — understanding this is crucial
- Wrong tokenization = wrong understanding (e.g., "New York" split as "New" + "York" loses meaning)

### Types of Tokenization

```
                    Tokenization Methods
                           |
        ┌──────────────────┼──────────────────┐
        ↓                  ↓                  ↓
   Word-Level         Subword-Level      Character-Level
   "I love NLP"      "un" + "happi"     "N" + "L" + "P"
   → [I, love, NLP]   + "ness"          → [N, L, P]
                      → [un, happi, ness]
        |                  |                  |
   Simple but         Best balance       Handles any word
   large vocab        (used in GPT,      but loses meaning
   + OOV issues       BERT, etc.)        + very long sequences
```

### Word Tokenization

```python
# Method 1: Simple split (NEVER use in production)
text = "I love NLP! It's amazing."
tokens_naive = text.split()
print(tokens_naive)  # ['I', 'love', 'NLP!', "It's", 'amazing.']
# Problem: punctuation sticks to words!

# Method 2: NLTK word_tokenize (good for English)
import nltk
nltk.download('punkt_tab', quiet=True)
from nltk.tokenize import word_tokenize

tokens_nltk = word_tokenize(text)
print(tokens_nltk)  # ['I', 'love', 'NLP', '!', 'It', "'s", 'amazing', '.']
# Notice: handles punctuation, contractions split

# Method 3: spaCy tokenizer (production-grade)
import spacy
nlp = spacy.load("en_core_web_sm")

doc = nlp(text)
tokens_spacy = [token.text for token in doc]
print(tokens_spacy)  # ['I', 'love', 'NLP', '!', 'It', "'s", 'amazing', '.']

# spaCy gives you much more than just text:
for token in doc:
    print(f"{token.text:10} | POS: {token.pos_:6} | Lemma: {token.lemma_:10} | Stop: {token.is_stop}")
```

**Output:**
```
I          | POS: PRON   | Lemma: I          | Stop: True
love       | POS: VERB   | Lemma: love       | Stop: False
NLP        | POS: PROPN  | Lemma: NLP        | Stop: False
!          | POS: PUNCT  | Lemma: !          | Stop: False
It         | POS: PRON   | Lemma: it         | Stop: True
's         | POS: AUX    | Lemma: be         | Stop: True
amazing    | POS: ADJ    | Lemma: amazing    | Stop: False
.          | POS: PUNCT  | Lemma: .          | Stop: False
```

### Sentence Tokenization

```python
from nltk.tokenize import sent_tokenize

paragraph = """Dr. Smith went to Washington D.C. He arrived at 3 p.m. 
The meeting was productive. "Let's do this again!" he said."""

sentences = sent_tokenize(paragraph)
for i, sent in enumerate(sentences):
    print(f"Sentence {i+1}: {sent}")
```

**Output:**
```
Sentence 1: Dr. Smith went to Washington D.C.
Sentence 2: He arrived at 3 p.m.
Sentence 3: The meeting was productive.
Sentence 4: "Let's do this again!" he said.
```

> 📝 **Note:** Sentence tokenization is tricky! "Dr." and "U.S." contain periods but aren't sentence endings. NLTK handles these with pre-trained models.

### Subword Tokenization (How Modern LLMs Work)

```python
# BPE (Byte-Pair Encoding) — used by GPT-2, GPT-3, GPT-4
from tokenizers import Tokenizer
from tokenizers.models import BPE
from tokenizers.trainers import BpeTrainer
from tokenizers.pre_tokenizers import Whitespace

# Create a BPE tokenizer from scratch
tokenizer = Tokenizer(BPE(unk_token="[UNK]"))
tokenizer.pre_tokenizer = Whitespace()

# Train on sample corpus
trainer = BpeTrainer(special_tokens=["[UNK]", "[PAD]", "[CLS]", "[SEP]"], vocab_size=1000)
# In practice, you'd train on a large corpus:
# tokenizer.train(["corpus.txt"], trainer)

# Using HuggingFace's pre-trained tokenizer (GPT-2)
from transformers import GPT2Tokenizer

gpt2_tokenizer = GPT2Tokenizer.from_pretrained("gpt2")

text = "unhappiness is overwhelming"
tokens = gpt2_tokenizer.tokenize(text)
print(f"Text: {text}")
print(f"Tokens: {tokens}")
# Output: ['un', 'happiness', ' is', ' over', 'wh', 'elming']

# See token IDs (what the model actually sees)
token_ids = gpt2_tokenizer.encode(text)
print(f"Token IDs: {token_ids}")

# BERT's WordPiece tokenizer
from transformers import BertTokenizer

bert_tokenizer = BertTokenizer.from_pretrained("bert-base-uncased")
tokens_bert = bert_tokenizer.tokenize(text)
print(f"BERT tokens: {tokens_bert}")
# Output: ['un', '##happiness', 'is', 'over', '##wh', '##el', '##ming']
# Note: ## prefix means "continuation of previous word"
```

### Tokenization Comparison Table

| Method | Vocab Size | OOV Handling | Speed | Used By |
|--------|-----------|--------------|-------|---------|
| Word-level | Very Large (100K+) | Poor (unknown words) | Fast | Traditional ML |
| Character-level | Very Small (100-200) | Perfect (no OOV) | Slow (long sequences) | Some specialized models |
| BPE (Byte-Pair Encoding) | Medium (30K-50K) | Good (breaks into subwords) | Medium | GPT-2, GPT-3, GPT-4 |
| WordPiece | Medium (30K) | Good | Medium | BERT, DistilBERT |
| Unigram/SentencePiece | Medium (32K) | Good | Medium | T5, ALBERT, XLNet |

### How BPE Works (Algorithm Intuition)

```
Step 1: Start with character-level vocabulary
   "low" → ['l', 'o', 'w']
   "lower" → ['l', 'o', 'w', 'e', 'r']
   "newest" → ['n', 'e', 'w', 'e', 's', 't']

Step 2: Count all adjacent pairs
   ('l','o'): 2, ('o','w'): 2, ('w','e'): 2, ('e','r'): 1, ...

Step 3: Merge most frequent pair → ('l','o') becomes 'lo'
   "low" → ['lo', 'w']
   "lower" → ['lo', 'w', 'e', 'r']

Step 4: Repeat until desired vocab size
   Next merge: ('lo','w') → 'low'
   "low" → ['low']
   "lower" → ['low', 'e', 'r']

Result: Common words stay whole, rare words get split into known subwords
```

---

## 4. Stop Words

### What it is
Stop words are extremely common words that carry little meaning on their own — words like "the", "is", "at", "which", "on", "a", "an", "in". They're the filler words that help grammar but don't contribute to the core meaning.

**Analogy:** Imagine reading a telegram where you pay per word. You'd write "Meeting Tuesday 3PM office" instead of "The meeting is on Tuesday at 3 PM in the office." The removed words are stop words — they add grammar but not meaning.

### Why it matters
- Reduces feature space significantly (stop words can be 20-30% of text)
- Speeds up training and inference
- Improves model focus on meaningful words
- **BUT** — removing them isn't always correct! (More on this below)

### When to Remove vs. Keep Stop Words

| Remove Stop Words | Keep Stop Words |
|-------------------|-----------------|
| Keyword extraction | Sentiment analysis ("not good" ≠ "good") |
| Topic modeling | Machine translation |
| Document search/retrieval | Question answering ("what", "who" matter) |
| TF-IDF based models | Language modeling (GPT, BERT) |
| Text classification (traditional ML) | Text generation |

### Implementation

```python
import nltk
from nltk.corpus import stopwords
from nltk.tokenize import word_tokenize

nltk.download('stopwords', quiet=True)
nltk.download('punkt_tab', quiet=True)

# Get English stop words
stop_words = set(stopwords.words('english'))
print(f"Number of NLTK stop words: {len(stop_words)}")
print(f"Sample: {list(stop_words)[:20]}")

# Remove stop words from text
text = "The quick brown fox jumps over the lazy dog in the park"
tokens = word_tokenize(text.lower())
filtered = [word for word in tokens if word not in stop_words]

print(f"Original: {tokens}")
print(f"Filtered: {filtered}")
# Original: ['the', 'quick', 'brown', 'fox', 'jumps', 'over', 'the', 'lazy', 'dog', 'in', 'the', 'park']
# Filtered: ['quick', 'brown', 'fox', 'jumps', 'lazy', 'dog', 'park']
```

```python
# Using spaCy (more comprehensive)
import spacy
nlp = spacy.load("en_core_web_sm")

text = "This is not a good movie at all"
doc = nlp(text)

# spaCy marks tokens as stop words
for token in doc:
    print(f"{token.text:8} → is_stop: {token.is_stop}")

# Filter
filtered = [token.text for token in doc if not token.is_stop and not token.is_punct]
print(f"Filtered: {filtered}")
# Filtered: ['good', 'movie']
# ⚠️ PROBLEM: "not" was removed! "not good" became "good" — meaning reversed!
```

> ⚠️ **Critical Warning:** Removing "not", "no", "never" can flip sentiment entirely! For sentiment analysis, either keep negation words or use n-grams.

### Custom Stop Words

```python
# Sometimes you need domain-specific stop words
from sklearn.feature_extraction.text import CountVectorizer

# Add custom stop words for a medical domain
custom_stop_words = list(stop_words) + ['patient', 'doctor', 'hospital', 'medical']

# Or use sklearn's built-in stop words + custom ones
vectorizer = CountVectorizer(stop_words='english')  # sklearn's stop word list
# Or pass custom list:
vectorizer = CountVectorizer(stop_words=custom_stop_words)

# Pro Tip: Build stop words from your corpus (words appearing in >90% of documents)
from collections import Counter

corpus = [
    "the cat sat on the mat",
    "the dog sat on the log",
    "the cat chased the dog",
    "the bird flew over the mat"
]

# Find words that appear in all documents (likely stop words for this corpus)
all_tokens = [word_tokenize(doc.lower()) for doc in corpus]
doc_frequency = Counter()
for tokens in all_tokens:
    unique_tokens = set(tokens)
    for token in unique_tokens:
        doc_frequency[token] += 1

# Words appearing in more than 75% of documents
threshold = 0.75 * len(corpus)
auto_stop_words = [word for word, count in doc_frequency.items() if count > threshold]
print(f"Auto-detected stop words: {auto_stop_words}")
# Output: Auto-detected stop words: ['the', 'on', 'sat', 'the']
```

---

## 5. Stemming

### What it is
Stemming is the process of reducing words to their **root/base form** by chopping off suffixes (and sometimes prefixes). It's a crude, rule-based approach that doesn't always produce valid English words.

**Analogy:** Imagine cutting branches off a tree to find the trunk. Sometimes you cut perfectly; sometimes you cut too much or too little. Stemming is aggressive pruning — fast but imprecise.

```
running  → run    ✓ (correct)
studies  → studi  ✗ (not a real word, but consistent)
universe → univers ✗ (not a real word)
better   → better ✗ (should be "good" but stemmer doesn't know that)
```

### Why it matters
- Reduces vocabulary size drastically
- Groups related words: "running", "runs", "ran" → same stem
- Fast — purely rule-based, no dictionary lookup needed
- Good enough for search/information retrieval
- Used in search engines (Google's early days used stemming)

### How it works — Porter Stemmer Algorithm

The Porter Stemmer applies a series of rules in phases:

```
Phase 1: Remove plurals and -ed/-ing
   caresses → caress
   ponies   → poni
   cats     → cat
   agreed   → agree
   plastered → plaster
   
Phase 2: Map double suffixes to single
   -ational → -ate  (relational → relate)
   -tional  → -tion (conditional → condition)
   -enci    → -ence (valenci → valence)
   
Phase 3: Deal with -ic, -full, -ness
   triplicate → triplic
   hopeful    → hope
   goodness   → good
   
Phase 4: Remove -ant, -ence, -ment, etc.
   allowance → allow
   adjustment → adjust
   
Phase 5: Remove final -e
   probate → probat
   rate    → rate (if stem is short)
```

### Implementation

```python
from nltk.stem import PorterStemmer, SnowballStemmer, LancasterStemmer

# Initialize stemmers
porter = PorterStemmer()
snowball = SnowballStemmer('english')
lancaster = LancasterStemmer()

# Test words
words = ['running', 'runs', 'ran', 'runner', 'easily', 'fairly',
         'studies', 'studying', 'generous', 'generosity', 
         'university', 'universe', 'flying', 'flies', 'flyer']

print(f"{'Word':<15} {'Porter':<12} {'Snowball':<12} {'Lancaster':<12}")
print("-" * 51)
for word in words:
    print(f"{word:<15} {porter.stem(word):<12} {snowball.stem(word):<12} {lancaster.stem(word):<12}")
```

**Output:**
```
Word            Porter       Snowball     Lancaster   
---------------------------------------------------
running         run          run          run         
runs            run          run          run         
ran             ran          ran          ran         
runner          runner       runner       run         
easily          easili       easili       easy        
fairly          fairli       fairli       fair        
studies         studi        studi        study       
studying        studi        studi        study       
generous        gener        gener        gen         
generosity      gener        gener        gen         
university      univers      univers      univers     
universe        univers      univers      univers     
flying          fli          fli          fly         
flies           fli          fli          fly         
flyer           flyer        flyer        fly         
```

### Stemmer Comparison

| Stemmer | Aggressiveness | Speed | Accuracy | Use When |
|---------|---------------|-------|----------|----------|
| **Porter** | Medium | Fast | Good | Default choice, well-balanced |
| **Snowball** | Medium | Fast | Better than Porter | Need multi-language support |
| **Lancaster** | Very Aggressive | Fastest | Lower | Need maximum compression |
| **Regex** | Custom | Fast | Varies | Simple suffix removal |

```python
# When stemming helps in practice: Search/IR
def search_with_stemming(query, documents):
    """Simple search engine using stemming."""
    stemmer = PorterStemmer()
    
    # Stem the query
    query_stems = [stemmer.stem(w.lower()) for w in query.split()]
    
    # Score documents by stem overlap
    results = []
    for doc in documents:
        doc_stems = [stemmer.stem(w.lower()) for w in doc.split()]
        # Count matching stems
        score = sum(1 for qs in query_stems if qs in doc_stems)
        if score > 0:
            results.append((doc, score))
    
    return sorted(results, key=lambda x: x[1], reverse=True)

documents = [
    "The runner was running in the marathon",
    "She studies computer science at university",
    "He studied the studying habits of students",
    "The dog ran quickly across the field"
]

# Search for "running" also finds "ran" and "runner"!
results = search_with_stemming("running", documents)
for doc, score in results:
    print(f"Score {score}: {doc}")
```

---

## 6. Lemmatization

### What it is
Lemmatization reduces words to their **dictionary form (lemma)** using vocabulary and morphological analysis. Unlike stemming, it always produces valid English words.

**Analogy:** If stemming is "chopping branches with an axe," lemmatization is "carefully tracing branches back to where they grow from the trunk." It's slower but always gives you a real word.

```
Stemming:       better → better (fails)
Lemmatization:  better → good   (correct! knows "better" is comparative of "good")

Stemming:       studies → studi (not a word)
Lemmatization:  studies → study (valid word)

Stemming:       ran → ran (fails to connect to "run")
Lemmatization:  ran → run (correct! knows "ran" is past tense of "run")
```

### Why it matters
- Produces valid dictionary words (more interpretable)
- Better for tasks where meaning matters (NER, text generation)
- Handles irregular forms (went→go, better→good, mice→mouse)
- More accurate grouping of related words
- Slower than stemming but more precise

### How it works

Lemmatization uses:
1. **WordNet dictionary** — Maps words to their base forms
2. **POS (Part of Speech)** — "saw" as noun → "saw", "saw" as verb → "see"
3. **Morphological rules** — Understanding word formation patterns

```python
import nltk
from nltk.stem import WordNetLemmatizer
from nltk.corpus import wordnet

nltk.download('wordnet', quiet=True)
nltk.download('averaged_perceptron_tagger_eng', quiet=True)

lemmatizer = WordNetLemmatizer()

# Basic lemmatization (defaults to noun)
words = ['studies', 'studying', 'better', 'ran', 'geese', 'mice', 'feet', 'children']
print("Without POS (defaults to noun):")
for word in words:
    print(f"  {word:12} → {lemmatizer.lemmatize(word)}")

# Output:
# studies      → study
# studying     → studying  ← WRONG! Treated as noun, not verb
# better       → better    ← WRONG! Needs adjective POS
# ran          → ran       ← WRONG! Needs verb POS
# geese        → goose     ✓ (irregular plural)
# mice         → mouse     ✓
# feet         → foot      ✓
# children     → child     ✓
```

```python
# With POS tagging (CORRECT approach)
print("\nWith correct POS:")
# pos parameter: 'n'=noun, 'v'=verb, 'a'=adjective, 'r'=adverb
print(f"  studying (verb)  → {lemmatizer.lemmatize('studying', pos='v')}")   # study
print(f"  better (adj)     → {lemmatizer.lemmatize('better', pos='a')}")     # good
print(f"  ran (verb)       → {lemmatizer.lemmatize('ran', pos='v')}")        # run
print(f"  running (verb)   → {lemmatizer.lemmatize('running', pos='v')}")    # run
print(f"  worst (adj)      → {lemmatizer.lemmatize('worst', pos='a')}")      # bad
```

```python
# Automatic POS detection for proper lemmatization
def get_wordnet_pos(treebank_tag):
    """Convert NLTK POS tag to WordNet POS tag."""
    if treebank_tag.startswith('J'):
        return wordnet.ADJ
    elif treebank_tag.startswith('V'):
        return wordnet.VERB
    elif treebank_tag.startswith('N'):
        return wordnet.NOUN
    elif treebank_tag.startswith('R'):
        return wordnet.ADV
    else:
        return wordnet.NOUN  # Default to noun

def lemmatize_sentence(sentence):
    """Properly lemmatize a sentence using POS tags."""
    tokens = nltk.word_tokenize(sentence)
    pos_tags = nltk.pos_tag(tokens)  # Get POS tags
    
    lemmas = []
    for word, tag in pos_tags:
        wn_tag = get_wordnet_pos(tag)
        lemma = lemmatizer.lemmatize(word.lower(), pos=wn_tag)
        lemmas.append(lemma)
    
    return lemmas

# Test
sentence = "The striped bats were hanging on their feet and eating best fishes"
result = lemmatize_sentence(sentence)
print(f"Original: {sentence}")
print(f"Lemmatized: {' '.join(result)}")
# Output: the striped bat be hang on their foot and eat best fish
```

### spaCy Lemmatization (Production-Grade)

```python
import spacy
nlp = spacy.load("en_core_web_sm")

text = "The children were running faster than the geese flying overhead"
doc = nlp(text)

print(f"{'Token':<12} {'Lemma':<10} {'POS':<6}")
print("-" * 30)
for token in doc:
    print(f"{token.text:<12} {token.lemma_:<10} {token.pos_:<6}")
```

**Output:**
```
Token        Lemma      POS   
------------------------------
The          the        DET   
children     child      NOUN  
were         be         AUX   
running      run        VERB  
faster       fast       ADV   
than         than       SCONJ 
the          the        DET   
geese        goose      NOUN  
flying       fly        VERB  
overhead     overhead   ADV   
```

### Stemming vs. Lemmatization — Complete Comparison

| Feature | Stemming | Lemmatization |
|---------|----------|---------------|
| **Output** | May not be a real word | Always a valid word |
| **Speed** | Very fast (rule-based) | Slower (dictionary lookup) |
| **Accuracy** | Lower | Higher |
| **Context-aware** | No | Yes (uses POS) |
| **"better"** | better | good |
| **"ran"** | ran | run |
| **"studies"** | studi | study |
| **Use case** | Search engines, IR | Text analysis, NLU |
| **Library** | NLTK, custom rules | NLTK+WordNet, spaCy |

### When to Use Which?

```python
# Decision helper
def choose_normalization(task):
    """Choose between stemming and lemmatization based on task."""
    use_stemming = [
        "search_engine",
        "information_retrieval", 
        "document_clustering",
        "speed_critical",
        "bag_of_words_model"
    ]
    
    use_lemmatization = [
        "text_generation",
        "chatbot",
        "question_answering",
        "sentiment_analysis",
        "named_entity_recognition",
        "readability_matters"
    ]
    
    if task in use_stemming:
        return "Use Stemming (faster, good enough)"
    elif task in use_lemmatization:
        return "Use Lemmatization (more accurate)"
    else:
        return "Try both, measure performance on your specific task"
```

---

## 7. Putting It All Together — Complete NLP Preprocessing Pipeline

```python
import re
import nltk
import spacy
from nltk.tokenize import word_tokenize
from nltk.corpus import stopwords
from nltk.stem import PorterStemmer, WordNetLemmatizer

# Download required resources
nltk.download('punkt_tab', quiet=True)
nltk.download('stopwords', quiet=True)
nltk.download('wordnet', quiet=True)
nltk.download('averaged_perceptron_tagger_eng', quiet=True)

class TextPreprocessor:
    """Production-ready text preprocessing pipeline."""
    
    def __init__(self, 
                 lowercase=True,
                 remove_html=True,
                 remove_urls=True,
                 remove_punctuation=True,
                 remove_numbers=False,
                 remove_stopwords=True,
                 normalization='lemma',  # 'lemma', 'stem', or None
                 min_word_length=2):
        
        self.lowercase = lowercase
        self.remove_html = remove_html
        self.remove_urls = remove_urls
        self.remove_punctuation = remove_punctuation
        self.remove_numbers = remove_numbers
        self.remove_stopwords = remove_stopwords
        self.normalization = normalization
        self.min_word_length = min_word_length
        
        self.stop_words = set(stopwords.words('english'))
        self.stemmer = PorterStemmer()
        self.lemmatizer = WordNetLemmatizer()
    
    def _clean(self, text):
        """Apply text cleaning steps."""
        if self.remove_html:
            text = re.sub(r'<[^>]+>', '', text)
        if self.remove_urls:
            text = re.sub(r'https?://\S+|www\.\S+', '', text)
        if self.lowercase:
            text = text.lower()
        if self.remove_numbers:
            text = re.sub(r'\d+', '', text)
        if self.remove_punctuation:
            text = re.sub(r'[^\w\s]', '', text)
        text = re.sub(r'\s+', ' ', text).strip()
        return text
    
    def _normalize(self, tokens):
        """Apply stemming or lemmatization."""
        if self.normalization == 'stem':
            return [self.stemmer.stem(t) for t in tokens]
        elif self.normalization == 'lemma':
            # With POS tagging for accuracy
            pos_tags = nltk.pos_tag(tokens)
            lemmas = []
            for word, tag in pos_tags:
                if tag.startswith('V'):
                    lemmas.append(self.lemmatizer.lemmatize(word, pos='v'))
                elif tag.startswith('J'):
                    lemmas.append(self.lemmatizer.lemmatize(word, pos='a'))
                elif tag.startswith('R'):
                    lemmas.append(self.lemmatizer.lemmatize(word, pos='r'))
                else:
                    lemmas.append(self.lemmatizer.lemmatize(word, pos='n'))
            return lemmas
        return tokens
    
    def process(self, text):
        """Full preprocessing pipeline."""
        # Step 1: Clean
        text = self._clean(text)
        
        # Step 2: Tokenize
        tokens = word_tokenize(text)
        
        # Step 3: Remove stop words
        if self.remove_stopwords:
            tokens = [t for t in tokens if t not in self.stop_words]
        
        # Step 4: Filter by length
        tokens = [t for t in tokens if len(t) >= self.min_word_length]
        
        # Step 5: Normalize
        tokens = self._normalize(tokens)
        
        return tokens
    
    def process_batch(self, texts):
        """Process multiple texts efficiently."""
        return [self.process(text) for text in texts]


# Usage
preprocessor = TextPreprocessor(normalization='lemma')

texts = [
    "The cats were running quickly across the beautiful garden!",
    "<p>I can't believe how AMAZING this product is!!! 🎉</p>",
    "Dr. Smith's patients are studying the effects of the new medicine.",
]

for text in texts:
    result = preprocessor.process(text)
    print(f"Input:  {text[:60]}...")
    print(f"Output: {result}\n")
```

**Output:**
```
Input:  The cats were running quickly across the beautiful garden!...
Output: ['cat', 'run', 'quickly', 'across', 'beautiful', 'garden']

Input:  <p>I can't believe how AMAZING this product is!!! 🎉</p>...
Output: ['believe', 'amazing', 'product']

Input:  Dr. Smith's patients are studying the effects of the new me...
Output: ['dr', 'smiths', 'patient', 'study', 'effect', 'new', 'medicine']
```

---

## 8. Regular Expressions for NLP

### Why Regex Matters in NLP
Regular expressions are the Swiss Army knife of text processing. Every NLP engineer must be comfortable with regex.

```python
import re

text = "Contact us at support@company.com or sales@company.com. Call 123-456-7890 or (098) 765-4321."

# Extract emails
emails = re.findall(r'[\w.+-]+@[\w-]+\.[\w.-]+', text)
print(f"Emails: {emails}")
# ['support@company.com', 'sales@company.com']

# Extract phone numbers
phones = re.findall(r'[\(]?\d{3}[\)]?[-.\s]?\d{3}[-.\s]?\d{4}', text)
print(f"Phones: {phones}")
# ['123-456-7890', '(098) 765-4321']

# Extract hashtags from social media
tweet = "Learning #NLP and #MachineLearning is so much fun! #AI #DataScience"
hashtags = re.findall(r'#\w+', tweet)
print(f"Hashtags: {hashtags}")
# ['#NLP', '#MachineLearning', '#AI', '#DataScience']

# Extract monetary values
financial = "The price dropped from $1,234.56 to $999.99"
amounts = re.findall(r'\$[\d,]+\.?\d*', financial)
print(f"Amounts: {amounts}")
# ['$1,234.56', '$999.99']

# Common NLP regex patterns
patterns = {
    'email': r'[\w.+-]+@[\w-]+\.[\w.-]+',
    'url': r'https?://(?:www\.)?[\w./\-?=&]+',
    'phone': r'[\(]?\d{3}[\)]?[-.\s]?\d{3}[-.\s]?\d{4}',
    'date': r'\d{1,2}[/-]\d{1,2}[/-]\d{2,4}',
    'time': r'\d{1,2}:\d{2}(?::\d{2})?(?:\s?[AaPp][Mm])?',
    'hashtag': r'#\w+',
    'mention': r'@\w+',
    'money': r'\$[\d,]+\.?\d*',
    'percentage': r'\d+\.?\d*%',
}
```

---

## 9. Common Mistakes & Pitfalls

### Mistake 1: Removing stop words for all tasks
```python
# BAD: Removing stop words for sentiment analysis
text = "This movie is not good at all"
# After stop word removal: ["movie", "good"] → Positive? WRONG!

# GOOD: Keep negation words
important_words = {'not', 'no', 'never', 'neither', 'nor', 'hardly', 
                   'barely', 'scarcely', "n't", 'nobody', 'nothing'}
custom_stop = stop_words - important_words  # Keep negation
```

### Mistake 2: Applying stemming/lemmatization blindly
```python
# BAD: Lemmatizing named entities
text = "Apple launched the iPhone in California"
# "Apple" → "apple" (fruit??) — Lost the company name!

# GOOD: Use NER first, then selectively lemmatize
doc = nlp(text)
processed = []
for token in doc:
    if token.ent_type_:  # It's a named entity
        processed.append(token.text)  # Keep as-is
    else:
        processed.append(token.lemma_.lower())
print(processed)
# ['Apple', 'launch', 'the', 'iPhone', 'in', 'California']
```

### Mistake 3: Not handling encoding issues
```python
# BAD: Assuming text is always clean ASCII
# text = open("file.txt").read()  # May crash on special characters

# GOOD: Always specify encoding
text = open("file.txt", encoding='utf-8', errors='replace').read()
# Or use 'ignore' to skip problematic characters
```

### Mistake 4: Over-preprocessing
```python
# BAD: Applying every possible preprocessing step
# Removing numbers from: "iPhone 15 Pro costs $999"
# Result: "iPhone Pro costs" — Lost critical information!

# GOOD: Choose preprocessing based on your specific task
# For product reviews: keep numbers (ratings, prices matter)
# For topic modeling: might remove numbers
```

### Mistake 5: Not considering language
```python
# BAD: Using English stop words for French text
# BAD: Using Porter Stemmer for German text

# GOOD: Use language-appropriate tools
from nltk.corpus import stopwords
french_stops = set(stopwords.words('french'))
german_stops = set(stopwords.words('german'))

# For stemming non-English:
from nltk.stem import SnowballStemmer
german_stemmer = SnowballStemmer('german')
french_stemmer = SnowballStemmer('french')
```

---

## 10. Interview Questions

### Conceptual Questions

**Q1: What is the difference between stemming and lemmatization? When would you use each?**
> **A:** Stemming chops suffixes using rules (fast, may produce non-words). Lemmatization uses dictionary + POS tags to find the true base form (slower, always valid words). Use stemming for search/IR where speed matters. Use lemmatization for NLU tasks where meaning matters.

**Q2: Why might removing stop words hurt your model's performance?**
> **A:** Stop words carry grammatical and contextual meaning. "Not good" loses negation. "What is NLP" loses the question word. For deep learning models (BERT, GPT), stop words provide valuable context. Only remove them for traditional ML (BoW, TF-IDF) models.

**Q3: Explain how BPE (Byte-Pair Encoding) tokenization works and why it's used in modern LLMs.**
> **A:** BPE starts with characters, iteratively merges the most frequent adjacent pairs until reaching desired vocab size. It's used because: (1) handles any word (no OOV), (2) balanced vocab size (~50K), (3) captures morphology naturally ("un" + "happi" + "ness"), (4) works across languages.

**Q4: How would you preprocess text differently for a sentiment analysis task vs. a named entity recognition task?**
> **A:** Sentiment: keep negation words, might keep punctuation ("!!!" = emphasis), lowercase (usually), might keep emojis. NER: keep capitalization (helps identify proper nouns), don't lemmatize entities, keep numbers (part of addresses, dates).

**Q5: What are the challenges of tokenizing languages like Chinese, Japanese, or Thai?**
> **A:** These languages don't have spaces between words. You need specialized word segmentation tools (jieba for Chinese, MeCab for Japanese). Subword tokenization (BPE) helps but still needs appropriate training data.

### Coding Questions

**Q6: Write a function to find the most common n-grams in a text.**
```python
from collections import Counter
from nltk import ngrams, word_tokenize

def top_ngrams(text, n=2, top_k=10):
    """Find top-k most common n-grams."""
    tokens = word_tokenize(text.lower())
    n_grams = list(ngrams(tokens, n))
    return Counter(n_grams).most_common(top_k)

text = "the cat sat on the mat. the cat is on the mat again."
print(top_ngrams(text, n=2, top_k=5))
# [('the', 'cat'): 2, ('the', 'mat'): 2, ('on', 'the'): 2, ...]
```

**Q7: Implement a simple spell-checker using edit distance.**
```python
def edit_distance(word1, word2):
    """Compute minimum edit distance (Levenshtein distance)."""
    m, n = len(word1), len(word2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    
    for i in range(m + 1):
        dp[i][0] = i
    for j in range(n + 1):
        dp[0][j] = j
    
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i-1] == word2[j-1]:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])
    
    return dp[m][n]

# Simple spell checker
vocabulary = ['hello', 'help', 'world', 'word', 'work', 'natural', 'language']

def suggest_correction(word, vocab, max_distance=2):
    """Suggest corrections for a misspelled word."""
    suggestions = []
    for v_word in vocab:
        dist = edit_distance(word.lower(), v_word)
        if dist <= max_distance:
            suggestions.append((v_word, dist))
    return sorted(suggestions, key=lambda x: x[1])

print(suggest_correction("helo", vocabulary))
# [('hello', 1), ('help', 2)]
```

---

## 11. Quick Reference

| Concept | What | Key Tool | When to Use |
|---------|------|----------|-------------|
| **Text Cleaning** | Remove noise (HTML, URLs, etc.) | `re` module | Always — first step |
| **Tokenization** | Split text into units | `nltk.word_tokenize`, `spacy` | Always — after cleaning |
| **Stop Words** | Remove common filler words | `nltk.corpus.stopwords` | BoW/TF-IDF, NOT deep learning |
| **Stemming** | Chop suffixes (rule-based) | `PorterStemmer` | Search, IR, speed-critical |
| **Lemmatization** | Dictionary base form | `WordNetLemmatizer`, `spaCy` | NLU, when accuracy matters |
| **Subword Tokenization** | Split into subword units | `transformers` tokenizers | Modern LLMs (BERT, GPT) |

### Preprocessing Decision Flowchart

```
Is it a deep learning model (BERT/GPT)?
├── YES → Minimal preprocessing (the model handles it)
│         Just: lowercase (maybe), remove HTML/URLs
│         Let the model's tokenizer handle the rest
│
└── NO → Traditional ML (BoW, TF-IDF, etc.)
          Apply full pipeline:
          Clean → Tokenize → Stop words → Stem/Lemma
          
          Is the task sentiment analysis?
          ├── YES → Keep negation words, consider n-grams
          └── NO → Standard stop word removal is fine
          
          Is speed critical?
          ├── YES → Use stemming
          └── NO → Use lemmatization
```

### Essential Libraries

| Library | Strength | Install |
|---------|----------|---------|
| **NLTK** | Education, breadth of tools | `pip install nltk` |
| **spaCy** | Production, speed, pipelines | `pip install spacy` |
| **HuggingFace Tokenizers** | Modern tokenization (BPE, etc.) | `pip install tokenizers` |
| **regex** | Advanced regex (Unicode support) | `pip install regex` |
| **ftfy** | Fix broken Unicode text | `pip install ftfy` |

---

*Next Chapter: [02-Text-Representation.md](02-Text-Representation.md) — BoW, TF-IDF, Word2Vec, GloVe, FastText*
