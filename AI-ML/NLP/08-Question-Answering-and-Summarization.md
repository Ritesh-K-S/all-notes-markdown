# Chapter 08: Question Answering and Summarization

## Table of Contents
- [What is Question Answering](#what-is-question-answering)
- [Types of QA Systems](#types-of-qa-systems)
- [Extractive QA](#extractive-qa)
- [Abstractive QA](#abstractive-qa)
- [Text Summarization](#text-summarization)
- [Extractive Summarization](#extractive-summarization)
- [Abstractive Summarization](#abstractive-summarization)
- [Retrieval-Augmented Generation (RAG)](#retrieval-augmented-generation-rag)
- [Building a Complete QA Pipeline](#building-a-complete-qa-pipeline)
- [Evaluation Metrics](#evaluation-metrics)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is Question Answering

Question Answering (QA) is the task of automatically answering questions posed in natural language. Think of it like building a super-smart assistant that can read a document and answer your questions about it — like having a study buddy who never forgets anything they read.

### Simple Analogy

Imagine you have a 500-page textbook and your friend asks you a question. You'd:
1. Find the relevant page/paragraph (retrieval)
2. Read and understand it (comprehension)
3. Either point to the exact sentence with the answer (extractive) or explain in your own words (abstractive)

That's exactly what QA systems do!

### The QA Landscape

```
                        Question Answering
                              │
              ┌───────────────┼───────────────┐
              │               │               │
        Extractive QA    Abstractive QA    Generative QA
        (highlight        (rephrase         (create answer
         answer in         answer from       from knowledge)
         context)          context)
              │               │               │
        ┌─────┴─────┐   ┌────┴────┐    ┌────┴────┐
        │           │   │         │    │         │
    Reading     Open   Seq2Seq  LLM   RAG    Pure LLM
    Comprehension Domain          Based
    (SQuAD)     (TriviaQA)
```

---

## Types of QA Systems

### 1. Extractive QA (Reading Comprehension)

Given a context passage and a question, find the exact span of text that answers the question.

```
Context: "The Eiffel Tower was built in 1889 by Gustave Eiffel for the World's Fair."
Question: "When was the Eiffel Tower built?"
Answer: "1889"  (extracted directly from the context)
```

### 2. Abstractive QA

Generate a natural language answer that may not appear verbatim in the source.

```
Context: "The company reported revenue of $50B in Q1 and $55B in Q2."
Question: "How did revenue change?"
Answer: "Revenue increased by $5 billion (10%) from Q1 to Q2."  (generated, not copied)
```

### 3. Open-Domain QA

Answer questions using a large knowledge base (like all of Wikipedia) without a specific context given.

```
Question: "What is the capital of Mongolia?"
Answer: "Ulaanbaatar"  (retrieved from knowledge base)
```

### 4. Conversational QA

Multi-turn QA where context builds across questions:

```
Q1: "Who founded Tesla?"  → "Elon Musk (along with other co-founders)"
Q2: "When was it founded?"  → "2003" (understands "it" refers to Tesla)
Q3: "Where is it headquartered?"  → "Austin, Texas"
```

### Comparison Table

| Type | Input | Output | Example Model |
|------|-------|--------|---------------|
| Extractive | Context + Question | Span from context | BERT-QA |
| Abstractive | Context + Question | Generated text | T5, GPT |
| Open-Domain | Question only | Answer text | RAG, DPR+Reader |
| Conversational | History + Question | Answer text | CoQA models |
| Knowledge-Based | Question + KB | Structured answer | KGQA systems |

---

## Extractive QA

### How It Works

The model learns to predict two things:
1. **Start position** — where the answer begins in the context
2. **End position** — where the answer ends in the context

```
Context tokens: [CLS] The Eiffel Tower was built in 1889 by Gustave Eiffel [SEP] When was ... [SEP]
                  0    1    2      3     4    5    6   7   8    9      10    11   12   13
                                                       ↑                    
                                              Start=7, End=7 → "1889"
```

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Start Logits    [0.01, 0.02, ..., 0.95, ..., 0.01]     │
│  End Logits      [0.01, 0.02, ..., 0.01, ..., 0.93]     │
└──────────────────────────┬──────────────────────────────┘
                           │ Linear layers (2 x hidden_dim → 1)
┌──────────────────────────▼──────────────────────────────┐
│              Transformer Encoder (BERT)                    │
│  Contextual representations for each token                │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│  [CLS] context tokens [SEP] question tokens [SEP]        │
└─────────────────────────────────────────────────────────┘
```

### Mathematics

For each token $i$, compute:

$$P_{start}(i) = \frac{e^{S \cdot h_i}}{\sum_j e^{S \cdot h_j}}$$

$$P_{end}(i) = \frac{e^{E \cdot h_i}}{\sum_j e^{E \cdot h_j}}$$

Where:
- $h_i$ = hidden state of token $i$ from the transformer
- $S, E$ = learned start/end weight vectors
- The answer span is: $\arg\max_{i \leq j} P_{start}(i) \times P_{end}(j)$

### Implementation with HuggingFace

```python
from transformers import pipeline

# Load extractive QA pipeline
qa_pipeline = pipeline(
    "question-answering",
    model="deepset/roberta-base-squad2",  # Trained on SQuAD 2.0
    device=0  # Use GPU if available, -1 for CPU
)

# Simple usage
context = """
The Amazon rainforest, also known as Amazonia, is a moist broadleaf tropical 
rainforest in the Amazon biome that covers most of the Amazon basin of South 
America. This basin encompasses 7,000,000 km² of which 5,500,000 km² are 
covered by the rainforest. This region includes territory belonging to nine 
nations and 3,344 formally acknowledged indigenous territories.
"""

questions = [
    "What is the Amazon rainforest also known as?",
    "How large is the Amazon basin?",
    "How many nations have territory in this region?",
]

for question in questions:
    result = qa_pipeline(question=question, context=context)
    print(f"Q: {question}")
    print(f"A: {result['answer']} (confidence: {result['score']:.4f})")
    print(f"   Span: [{result['start']}, {result['end']}]")
    print()

# Output:
# Q: What is the Amazon rainforest also known as?
# A: Amazonia (confidence: 0.9876)
#    Span: [42, 50]
#
# Q: How large is the Amazon basin?
# A: 7,000,000 km² (confidence: 0.8234)
#    Span: [178, 193]
#
# Q: How many nations have territory in this region?
# A: nine (confidence: 0.9512)
#    Span: [273, 277]
```

### Handling Unanswerable Questions (SQuAD 2.0)

```python
# SQuAD 2.0 includes questions that CANNOT be answered from the context
# The model should return "no answer" with high confidence

context = "Python was created by Guido van Rossum and first released in 1991."
question = "What is Python's mascot?"  # Not in context!

result = qa_pipeline(question=question, context=context)
print(f"Answer: '{result['answer']}' | Score: {result['score']:.4f}")

# If score is low (< threshold), treat as unanswerable
CONFIDENCE_THRESHOLD = 0.5
if result['score'] < CONFIDENCE_THRESHOLD:
    print("→ Question likely unanswerable from this context")
```

### Fine-Tuning for Extractive QA

```python
from transformers import (
    AutoTokenizer,
    AutoModelForQuestionAnswering,
    TrainingArguments,
    Trainer,
    DefaultDataCollator
)
from datasets import load_dataset

# Load SQuAD dataset
dataset = load_dataset("squad")
print(dataset["train"][0])
# {'id': '...', 'title': '...', 'context': '...', 
#  'question': '...', 'answers': {'text': ['...'], 'answer_start': [515]}}

model_name = "bert-base-cased"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForQuestionAnswering.from_pretrained(model_name)

def preprocess_qa(examples):
    """Tokenize and find answer positions in tokenized input"""
    questions = [q.strip() for q in examples["question"]]
    
    # Tokenize question + context pairs
    inputs = tokenizer(
        questions,
        examples["context"],
        max_length=384,
        truncation="only_second",  # Only truncate context, not question
        stride=128,               # Overlap for long contexts
        return_overflowing_mapping=True,
        return_offsets_mapping=True,
        padding="max_length"
    )
    
    # Find start/end token positions of answers
    offset_mapping = inputs.pop("offset_mapping")
    sample_map = inputs.pop("overflow_to_sample_mapping")
    answers = examples["answers"]
    
    start_positions = []
    end_positions = []
    
    for i, offset in enumerate(offset_mapping):
        sample_idx = sample_map[i]
        answer = answers[sample_idx]
        start_char = answer["answer_start"][0]
        end_char = start_char + len(answer["text"][0])
        
        # Find the token positions that contain the answer
        sequence_ids = inputs.sequence_ids(i)
        
        # Find context start and end in token space
        context_start = 0
        while sequence_ids[context_start] != 1:
            context_start += 1
        context_end = len(sequence_ids) - 1
        while sequence_ids[context_end] != 1:
            context_end -= 1
        
        # Check if answer is within this chunk
        if offset[context_start][0] > end_char or offset[context_end][1] < start_char:
            # Answer not in this chunk
            start_positions.append(0)  # CLS token
            end_positions.append(0)
        else:
            # Find start token
            idx = context_start
            while idx <= context_end and offset[idx][0] <= start_char:
                idx += 1
            start_positions.append(idx - 1)
            
            # Find end token
            idx = context_end
            while idx >= context_start and offset[idx][1] >= end_char:
                idx -= 1
            end_positions.append(idx + 1)
    
    inputs["start_positions"] = start_positions
    inputs["end_positions"] = end_positions
    return inputs

# Process dataset
tokenized = dataset.map(
    preprocess_qa,
    batched=True,
    remove_columns=dataset["train"].column_names
)

# Training
training_args = TrainingArguments(
    output_dir="./qa-model",
    evaluation_strategy="epoch",
    learning_rate=3e-5,
    per_device_train_batch_size=16,
    num_train_epochs=2,
    weight_decay=0.01,
    warmup_ratio=0.1,
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized["train"],
    eval_dataset=tokenized["validation"],
    tokenizer=tokenizer,
    data_collator=DefaultDataCollator(),
)

trainer.train()
```

### Handling Long Documents

```python
def answer_from_long_document(question, document, qa_pipeline, chunk_size=400, stride=100):
    """
    Process documents longer than model's max context length.
    Split into overlapping chunks and find the best answer across all chunks.
    """
    # Split document into overlapping chunks
    words = document.split()
    chunks = []
    
    for i in range(0, len(words), chunk_size - stride):
        chunk = " ".join(words[i:i + chunk_size])
        chunks.append(chunk)
    
    # Get answers from each chunk
    best_answer = {"score": 0, "answer": "", "chunk_idx": -1}
    
    for idx, chunk in enumerate(chunks):
        try:
            result = qa_pipeline(question=question, context=chunk)
            if result["score"] > best_answer["score"]:
                best_answer = {
                    "score": result["score"],
                    "answer": result["answer"],
                    "chunk_idx": idx
                }
        except Exception:
            continue
    
    return best_answer

# Usage
long_document = "..." * 5000  # Very long document
answer = answer_from_long_document(
    "What was the quarterly revenue?", 
    long_document, 
    qa_pipeline
)
print(f"Answer: {answer['answer']} (from chunk {answer['chunk_idx']})")
```

---

## Abstractive QA

### Using Generative Models for QA

```python
from transformers import pipeline

# T5-based abstractive QA
qa_generative = pipeline(
    "text2text-generation",
    model="google/flan-t5-base"  # Instruction-tuned T5
)

context = """
Tesla reported Q3 2024 revenue of $25.18 billion, up 8% year-over-year. 
Automotive revenue was $20.02 billion while energy generation and storage 
revenue grew 52% to $2.38 billion. The company delivered 462,890 vehicles 
in the quarter, slightly below analyst expectations of 463,000.
"""

question = "Summarize Tesla's Q3 performance in one sentence."

prompt = f"Answer the question based on the context.\n\nContext: {context}\n\nQuestion: {question}\n\nAnswer:"
result = qa_generative(prompt, max_length=100, do_sample=False)
print(result[0]["generated_text"])
# "Tesla's Q3 2024 revenue grew 8% to $25.18B with strong energy growth 
#  but vehicle deliveries slightly missed analyst expectations."
```

### Closed-Book QA (No Context Provided)

```python
from transformers import pipeline

# Large language models can answer from parametric knowledge
generator = pipeline("text2text-generation", model="google/flan-t5-large")

questions = [
    "What is the capital of Australia?",
    "Who wrote Romeo and Juliet?",
    "What year did the Berlin Wall fall?",
]

for q in questions:
    answer = generator(f"Answer: {q}", max_length=50)
    print(f"Q: {q}")
    print(f"A: {answer[0]['generated_text']}")
    print()
```

---

## Text Summarization

### What is Summarization

Summarization condenses a long document into a shorter version while preserving the key information. Like writing a book report — you capture the essential points without all the details.

### Two Types

| Type | How it works | Analogy |
|------|-------------|---------|
| **Extractive** | Select important sentences from the original | Highlighting key sentences in a textbook |
| **Abstractive** | Generate new sentences capturing the meaning | Writing a summary in your own words |

---

## Extractive Summarization

### How It Works

1. Split document into sentences
2. Score each sentence for importance
3. Select top-K sentences as the summary

### Methods for Scoring Sentences

```
┌──────────────────────────────────────────────────────┐
│              Sentence Scoring Methods                  │
├──────────────────────────────────────────────────────┤
│ 1. Frequency-based (TF-IDF of sentence words)        │
│ 2. Position-based (first/last sentences matter more) │
│ 3. Graph-based (TextRank — like PageRank for text)   │
│ 4. Embedding-based (similarity to document centroid) │
│ 5. ML-based (trained classifier: important or not)   │
└──────────────────────────────────────────────────────┘
```

### TextRank Implementation

```python
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity
import nltk
nltk.download('punkt')

def textrank_summarize(text, num_sentences=3):
    """
    TextRank: Graph-based extractive summarization.
    
    Idea: Build a graph where sentences are nodes and edges represent 
    similarity. Important sentences are connected to many other important 
    sentences (like PageRank for web pages).
    """
    # Step 1: Split into sentences
    sentences = nltk.sent_tokenize(text)
    
    if len(sentences) <= num_sentences:
        return text
    
    # Step 2: Create sentence vectors using TF-IDF
    vectorizer = TfidfVectorizer(stop_words='english')
    tfidf_matrix = vectorizer.fit_transform(sentences)
    
    # Step 3: Build similarity graph
    similarity_matrix = cosine_similarity(tfidf_matrix)
    
    # Step 4: Apply PageRank algorithm
    # Normalize similarity matrix (make it a proper transition matrix)
    np.fill_diagonal(similarity_matrix, 0)  # No self-loops
    
    # Power iteration for PageRank
    n = len(sentences)
    damping = 0.85
    scores = np.ones(n) / n  # Initial uniform scores
    
    for _ in range(50):  # Iterate until convergence
        new_scores = np.zeros(n)
        for i in range(n):
            for j in range(n):
                if similarity_matrix[j].sum() > 0:
                    new_scores[i] += similarity_matrix[j][i] / similarity_matrix[j].sum() * scores[j]
        new_scores = (1 - damping) / n + damping * new_scores
        
        if np.abs(new_scores - scores).sum() < 1e-6:
            break
        scores = new_scores
    
    # Step 5: Select top sentences (preserve original order)
    ranked_indices = np.argsort(scores)[::-1][:num_sentences]
    ranked_indices = sorted(ranked_indices)  # Maintain document order
    
    summary = " ".join([sentences[i] for i in ranked_indices])
    return summary

# Test
article = """
Artificial intelligence has transformed numerous industries in recent years. 
Healthcare has seen particular benefits, with AI systems now capable of 
detecting diseases from medical images with accuracy rivaling human experts. 
In finance, AI algorithms process millions of transactions per second to 
detect fraud. The technology sector itself has been revolutionized, with 
AI-powered coding assistants now helping developers write code faster. 
However, concerns about AI safety and job displacement continue to grow. 
Researchers are working on aligning AI systems with human values to ensure 
beneficial outcomes. The next decade will be critical in determining how 
society adapts to these rapid technological changes.
"""

summary = textrank_summarize(article, num_sentences=3)
print("Summary:")
print(summary)
```

### Using sumy Library

```python
# pip install sumy
from sumy.parsers.plaintext import PlaintextParser
from sumy.nlp.tokenizers import Tokenizer
from sumy.summarizers.text_rank import TextRankSummarizer
from sumy.summarizers.lsa import LsaSummarizer
from sumy.summarizers.luhn import LuhnSummarizer

text = """..."""  # Your document

parser = PlaintextParser.from_string(text, Tokenizer("english"))

# TextRank summarizer
summarizer = TextRankSummarizer()
summary = summarizer(parser.document, sentences_count=3)
print("TextRank Summary:")
for sentence in summary:
    print(f"  • {sentence}")

# LSA (Latent Semantic Analysis) summarizer
summarizer_lsa = LsaSummarizer()
summary_lsa = summarizer_lsa(parser.document, sentences_count=3)
print("\nLSA Summary:")
for sentence in summary_lsa:
    print(f"  • {sentence}")
```

### BERT-based Extractive Summarization

```python
# pip install bert-extractive-summarizer
from summarizer import Summarizer

# Uses BERT embeddings to find representative sentences
model = Summarizer()

text = """..."""  # Long document

# Summarize to ~30% of original length
summary = model(text, ratio=0.3)
print(summary)

# Or specify number of sentences
summary = model(text, num_sentences=5)
print(summary)
```

---

## Abstractive Summarization

### Using Pre-trained Models

```python
from transformers import pipeline

# BART - trained on CNN/DailyMail summarization dataset
summarizer = pipeline(
    "summarization",
    model="facebook/bart-large-cnn",
    device=0  # GPU
)

article = """
The United Nations climate summit concluded with a landmark agreement among 
195 nations to limit global warming to 1.5 degrees Celsius above pre-industrial 
levels. The deal, reached after two weeks of intense negotiations in Dubai, 
includes provisions for a $100 billion annual fund to help developing nations 
transition to clean energy. Key commitments include phasing down fossil fuel 
usage by 2050, tripling renewable energy capacity by 2030, and establishing 
a loss and damage fund for vulnerable nations. However, critics argue the 
agreement lacks enforceable mechanisms and relies too heavily on voluntary 
compliance. Small island nations expressed disappointment that the language 
was weakened from "phase out" to "phase down" regarding fossil fuels.
"""

summary = summarizer(
    article,
    max_length=80,      # Maximum summary length
    min_length=30,      # Minimum summary length
    do_sample=False,    # Deterministic (greedy/beam search)
    num_beams=4,        # Beam search for better quality
    length_penalty=2.0, # Encourage longer summaries
    no_repeat_ngram_size=3  # Prevent repetition
)

print("Summary:")
print(summary[0]["summary_text"])
```

### Different Models for Summarization

```python
from transformers import pipeline

# Model comparison for summarization
models = {
    "BART": "facebook/bart-large-cnn",
    "T5": "t5-base",
    "Pegasus": "google/pegasus-xsum",    # Better for single-sentence summaries
    "LED": "allenai/led-base-16384",     # Long documents (up to 16K tokens)
    "FLAN-T5": "google/flan-t5-base",    # Instruction-following
}

# For T5, prefix the input with "summarize: "
t5_summarizer = pipeline("summarization", model="t5-base")
t5_summary = t5_summarizer("summarize: " + article, max_length=80, min_length=30)

# For very long documents, use LED (Longformer Encoder-Decoder)
led_summarizer = pipeline("summarization", model="allenai/led-base-16384")
# Can handle documents up to 16,384 tokens!
```

### Fine-Tuning for Custom Summarization

```python
from transformers import (
    AutoTokenizer,
    AutoModelForSeq2SeqLM,
    Seq2SeqTrainingArguments,
    Seq2SeqTrainer,
    DataCollatorForSeq2Seq
)
from datasets import load_dataset
import numpy as np
import evaluate

# Load dataset
dataset = load_dataset("cnn_dailymail", "3.0.0")
# Or custom: dataset = load_dataset("csv", data_files={"train": "train.csv", "val": "val.csv"})

model_name = "facebook/bart-base"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForSeq2SeqLM.from_pretrained(model_name)

# Preprocessing
def preprocess_function(examples):
    """Tokenize articles and summaries"""
    # Tokenize inputs (articles)
    model_inputs = tokenizer(
        examples["article"],
        max_length=1024,
        truncation=True,
        padding="max_length"
    )
    
    # Tokenize targets (summaries) — use as labels
    labels = tokenizer(
        text_target=examples["highlights"],
        max_length=128,
        truncation=True,
        padding="max_length"
    )
    
    # Replace padding token id with -100 (ignored by loss function)
    labels["input_ids"] = [
        [(l if l != tokenizer.pad_token_id else -100) for l in label]
        for label in labels["input_ids"]
    ]
    
    model_inputs["labels"] = labels["input_ids"]
    return model_inputs

tokenized = dataset.map(preprocess_function, batched=True)

# Evaluation metric
rouge = evaluate.load("rouge")

def compute_metrics(eval_pred):
    predictions, labels = eval_pred
    # Decode predictions
    decoded_preds = tokenizer.batch_decode(predictions, skip_special_tokens=True)
    
    # Replace -100 with pad token for decoding
    labels = np.where(labels != -100, labels, tokenizer.pad_token_id)
    decoded_labels = tokenizer.batch_decode(labels, skip_special_tokens=True)
    
    # Compute ROUGE scores
    result = rouge.compute(
        predictions=decoded_preds,
        references=decoded_labels,
        use_stemmer=True
    )
    return {k: round(v, 4) for k, v in result.items()}

# Training arguments
training_args = Seq2SeqTrainingArguments(
    output_dir="./summarization-model",
    evaluation_strategy="epoch",
    learning_rate=5e-5,
    per_device_train_batch_size=4,
    per_device_eval_batch_size=8,
    num_train_epochs=3,
    weight_decay=0.01,
    predict_with_generate=True,  # Use generation for evaluation
    generation_max_length=128,
    fp16=True,  # Mixed precision training
)

# Data collator
data_collator = DataCollatorForSeq2Seq(tokenizer=tokenizer, model=model)

# Trainer
trainer = Seq2SeqTrainer(
    model=model,
    args=training_args,
    train_dataset=tokenized["train"],
    eval_dataset=tokenized["validation"],
    tokenizer=tokenizer,
    data_collator=data_collator,
    compute_metrics=compute_metrics,
)

trainer.train()
```

### Controlling Summary Style

```python
from transformers import pipeline

summarizer = pipeline("summarization", model="facebook/bart-large-cnn")

article = "..."  # Your text

# Short, headline-style summary
short = summarizer(article, max_length=30, min_length=10)

# Detailed summary
detailed = summarizer(article, max_length=150, min_length=80)

# Using T5 with custom prompts for style control
from transformers import T5ForConditionalGeneration, T5Tokenizer

tokenizer = T5Tokenizer.from_pretrained("google/flan-t5-large")
model = T5ForConditionalGeneration.from_pretrained("google/flan-t5-large")

prompts = {
    "bullet_points": "Summarize the following in bullet points:\n",
    "eli5": "Explain this to a 5 year old:\n",
    "technical": "Give a technical summary:\n",
    "one_sentence": "Summarize in one sentence:\n",
}

for style, prompt in prompts.items():
    inputs = tokenizer(prompt + article, return_tensors="pt", max_length=512, truncation=True)
    outputs = model.generate(**inputs, max_length=150)
    result = tokenizer.decode(outputs[0], skip_special_tokens=True)
    print(f"\n[{style}]: {result}")
```

---

## Retrieval-Augmented Generation (RAG)

### What is RAG

RAG combines the best of retrieval and generation:
1. **Retrieve** relevant documents from a knowledge base
2. **Augment** the input with retrieved context
3. **Generate** an answer using the augmented input

```
┌──────────┐     ┌───────────────┐     ┌──────────────┐     ┌──────────────┐
│  Query   │────>│   Retriever   │────>│  Retrieved   │────>│  Generator   │
│          │     │  (find docs)  │     │  Documents   │     │  (answer)    │
└──────────┘     └───────────────┘     └──────────────┘     └──────────────┘
                        │                                           │
                        ▼                                           ▼
                 ┌──────────────┐                          ┌──────────────┐
                 │  Vector DB   │                          │   Answer     │
                 │  (FAISS,     │                          │  (grounded   │
                 │   Pinecone)  │                          │   in docs)   │
                 └──────────────┘                          └──────────────┘
```

### Why RAG Over Pure LLMs

| Issue with LLMs | How RAG Solves It |
|----------------|-------------------|
| Hallucination | Grounds answers in retrieved docs |
| Outdated knowledge | Retrieves from up-to-date sources |
| No source attribution | Can cite exact source documents |
| Domain-specific gaps | Custom knowledge base fills gaps |
| Cost of fine-tuning | No model retraining needed |

### Simple RAG Implementation

```python
# pip install langchain faiss-cpu sentence-transformers

from langchain.document_loaders import TextLoader, PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.embeddings import HuggingFaceEmbeddings
from langchain.vectorstores import FAISS
from langchain.chains import RetrievalQA
from langchain.llms import HuggingFacePipeline
from transformers import pipeline

# Step 1: Load and chunk documents
documents = [
    "The Python programming language was created by Guido van Rossum in 1991. "
    "It emphasizes code readability with significant whitespace.",
    "PyTorch is a deep learning framework developed by Meta AI. "
    "It uses dynamic computational graphs and is popular in research.",
    "TensorFlow was developed by Google Brain and released in 2015. "
    "It supports both eager and graph execution modes.",
    "Scikit-learn is a machine learning library built on NumPy and SciPy. "
    "It provides simple APIs for classification, regression, and clustering.",
]

# Step 2: Split into chunks (for longer documents)
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=200,       # Characters per chunk
    chunk_overlap=50,     # Overlap between chunks
    separators=["\n\n", "\n", ". ", " "]
)

# For this example, documents are already short
chunks = documents  # In practice: text_splitter.split_documents(loaded_docs)

# Step 3: Create embeddings and vector store
embeddings = HuggingFaceEmbeddings(
    model_name="sentence-transformers/all-MiniLM-L6-v2"
)
vectorstore = FAISS.from_texts(chunks, embeddings)

# Step 4: Retrieve relevant documents
query = "What is PyTorch and who created it?"
retrieved_docs = vectorstore.similarity_search(query, k=2)

print("Retrieved documents:")
for i, doc in enumerate(retrieved_docs):
    print(f"  {i+1}. {doc.page_content}")

# Step 5: Generate answer using retrieved context
# (Using a local model for demonstration)
context = "\n".join([doc.page_content for doc in retrieved_docs])
prompt = f"""Based on the following context, answer the question.

Context: {context}

Question: {query}

Answer:"""

generator = pipeline("text2text-generation", model="google/flan-t5-base")
answer = generator(prompt, max_length=100)
print(f"\nAnswer: {answer[0]['generated_text']}")
```

### RAG with LangChain (Production Pattern)

```python
from langchain.chains import RetrievalQA
from langchain.prompts import PromptTemplate

# Custom prompt template
prompt_template = """Use the following pieces of context to answer the question. 
If you don't know the answer from the context, say "I don't have enough information."

Context: {context}

Question: {question}

Helpful Answer:"""

PROMPT = PromptTemplate(
    template=prompt_template,
    input_variables=["context", "question"]
)

# Create retrieval chain
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,  # Your LLM (OpenAI, local model, etc.)
    chain_type="stuff",  # "stuff" = put all docs in one prompt
    retriever=vectorstore.as_retriever(
        search_type="similarity",
        search_kwargs={"k": 4}  # Retrieve top 4 documents
    ),
    chain_type_kwargs={"prompt": PROMPT},
    return_source_documents=True  # Return sources for citation
)

# Query
result = qa_chain({"query": "What is PyTorch?"})
print(f"Answer: {result['result']}")
print(f"Sources: {[doc.metadata for doc in result['source_documents']]}")
```

### Advanced RAG Techniques

```python
# 1. Hybrid Search (keyword + semantic)
from langchain.retrievers import EnsembleRetriever
from langchain.retrievers import BM25Retriever

# BM25 for keyword matching
bm25_retriever = BM25Retriever.from_texts(chunks)
bm25_retriever.k = 3

# Dense retrieval for semantic matching
dense_retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# Combine both
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, dense_retriever],
    weights=[0.4, 0.6]  # Weight semantic higher
)

# 2. Reranking retrieved documents
# pip install sentence-transformers
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank_documents(query, documents, top_k=3):
    """Rerank retrieved documents using a cross-encoder"""
    pairs = [(query, doc.page_content) for doc in documents]
    scores = reranker.predict(pairs)
    
    # Sort by relevance score
    ranked = sorted(zip(documents, scores), key=lambda x: x[1], reverse=True)
    return [doc for doc, score in ranked[:top_k]]

# 3. Query transformation
def expand_query(original_query):
    """Generate multiple query variations for better retrieval"""
    # Use LLM to generate query variations
    prompt = f"""Generate 3 different versions of this search query to improve retrieval:
    Original: {original_query}
    Variations:"""
    # ... generate with LLM
    return variations
```

---

## Building a Complete QA Pipeline

```python
"""
Complete QA Pipeline: Document Ingestion → Retrieval → Answer Generation
"""
import os
from pathlib import Path
from typing import List, Dict, Optional
from dataclasses import dataclass

@dataclass
class QAResult:
    answer: str
    confidence: float
    sources: List[str]
    context_used: str

class QAPipeline:
    def __init__(self, model_name="google/flan-t5-base", embedding_model="all-MiniLM-L6-v2"):
        from transformers import pipeline
        from sentence_transformers import SentenceTransformer
        import faiss
        import numpy as np
        
        # Initialize components
        self.generator = pipeline("text2text-generation", model=model_name)
        self.embedder = SentenceTransformer(embedding_model)
        self.documents = []
        self.embeddings = None
        self.index = None
    
    def add_documents(self, texts: List[str], chunk_size=500, overlap=100):
        """Ingest documents with chunking"""
        import numpy as np
        import faiss
        
        # Chunk texts
        chunks = []
        for text in texts:
            words = text.split()
            for i in range(0, len(words), chunk_size - overlap):
                chunk = " ".join(words[i:i + chunk_size])
                if len(chunk.strip()) > 50:  # Skip very short chunks
                    chunks.append(chunk)
        
        self.documents = chunks
        
        # Create embeddings
        self.embeddings = self.embedder.encode(chunks, show_progress_bar=True)
        
        # Build FAISS index
        dimension = self.embeddings.shape[1]
        self.index = faiss.IndexFlatIP(dimension)  # Inner product (cosine sim)
        
        # Normalize for cosine similarity
        faiss.normalize_L2(self.embeddings)
        self.index.add(self.embeddings)
        
        print(f"Indexed {len(chunks)} chunks from {len(texts)} documents")
    
    def retrieve(self, query: str, top_k: int = 3) -> List[str]:
        """Retrieve most relevant chunks"""
        import numpy as np
        import faiss
        
        query_embedding = self.embedder.encode([query])
        faiss.normalize_L2(query_embedding)
        
        scores, indices = self.index.search(query_embedding, top_k)
        
        results = []
        for idx, score in zip(indices[0], scores[0]):
            if idx < len(self.documents):
                results.append(self.documents[idx])
        
        return results
    
    def answer(self, question: str, top_k: int = 3) -> QAResult:
        """Full QA pipeline: retrieve + generate"""
        # Retrieve relevant context
        contexts = self.retrieve(question, top_k=top_k)
        combined_context = "\n\n".join(contexts)
        
        # Generate answer
        prompt = f"""Answer the question based on the context provided.
If the answer is not in the context, say "I cannot find the answer in the provided documents."

Context:
{combined_context}

Question: {question}

Answer:"""
        
        result = self.generator(prompt, max_length=200, do_sample=False)
        answer_text = result[0]["generated_text"]
        
        return QAResult(
            answer=answer_text,
            confidence=0.0,  # Would need model logits for real confidence
            sources=contexts[:2],
            context_used=combined_context[:500]
        )

# Usage
pipeline = QAPipeline()
pipeline.add_documents([
    "Python was created by Guido van Rossum and released in 1991...",
    "Machine learning is a subset of artificial intelligence...",
    # ... more documents
])

result = pipeline.answer("Who created Python?")
print(f"Answer: {result.answer}")
print(f"Sources: {result.sources[0][:100]}...")
```

---

## Evaluation Metrics

### QA Metrics

#### Exact Match (EM)

```python
def exact_match(prediction, ground_truth):
    """Binary: is the prediction exactly right?"""
    return prediction.strip().lower() == ground_truth.strip().lower()

# Strict! "1889" matches "1889" but "in 1889" does NOT match "1889"
```

#### F1 Score (Token-level)

```python
def compute_f1(prediction, ground_truth):
    """Token-level F1 between prediction and ground truth"""
    pred_tokens = prediction.lower().split()
    truth_tokens = ground_truth.lower().split()
    
    common = set(pred_tokens) & set(truth_tokens)
    
    if len(common) == 0:
        return 0.0
    
    precision = len(common) / len(pred_tokens)
    recall = len(common) / len(truth_tokens)
    f1 = 2 * precision * recall / (precision + recall)
    
    return f1

# Example
print(compute_f1("Gustave Eiffel", "Gustave Eiffel"))  # 1.0
print(compute_f1("built by Gustave Eiffel", "Gustave Eiffel"))  # 0.8
```

### Summarization Metrics

#### ROUGE (Recall-Oriented Understudy for Gisting Evaluation)

```python
import evaluate

rouge = evaluate.load("rouge")

predictions = ["The cat sat on the mat and looked out the window."]
references = ["The cat was sitting on the mat watching through the window."]

results = rouge.compute(predictions=predictions, references=references)
print(results)
# {
#   'rouge1': 0.72,  # Unigram overlap
#   'rouge2': 0.44,  # Bigram overlap  
#   'rougeL': 0.63,  # Longest Common Subsequence
#   'rougeLsum': 0.63 # LCS on summary level
# }
```

**ROUGE Variants Explained:**

| Metric | What it measures | Formula |
|--------|-----------------|---------|
| ROUGE-1 | Unigram overlap | $\frac{\text{matching unigrams}}{\text{unigrams in reference}}$ |
| ROUGE-2 | Bigram overlap | $\frac{\text{matching bigrams}}{\text{bigrams in reference}}$ |
| ROUGE-L | Longest common subsequence | Based on LCS length |

$$\text{ROUGE-N} = \frac{\sum_{s \in \text{ref}} \sum_{gram_n \in s} Count_{match}(gram_n)}{\sum_{s \in \text{ref}} \sum_{gram_n \in s} Count(gram_n)}$$

#### BERTScore (Semantic Similarity)

```python
# pip install bert-score
import evaluate

bertscore = evaluate.load("bertscore")

predictions = ["The weather is nice today"]
references = ["It's a beautiful day outside"]

results = bertscore.compute(
    predictions=predictions, 
    references=references, 
    lang="en"
)
print(f"Precision: {results['precision'][0]:.4f}")
print(f"Recall: {results['recall'][0]:.4f}")
print(f"F1: {results['f1'][0]:.4f}")
# Much better than ROUGE for paraphrases!
```

#### Faithfulness (for RAG/Abstractive systems)

```python
def check_faithfulness(summary, source):
    """
    Check if summary is faithful to source (no hallucinations).
    Uses NLI (Natural Language Inference) model.
    """
    from transformers import pipeline
    
    nli = pipeline("text-classification", model="facebook/bart-large-mnli")
    
    # Check each sentence in summary against source
    sentences = summary.split(". ")
    results = []
    
    for sent in sentences:
        result = nli(f"{source} [SEP] {sent}")
        # If classified as "entailment" → faithful
        # If "contradiction" → hallucination!
        results.append({
            "sentence": sent,
            "label": result[0]["label"],
            "score": result[0]["score"]
        })
    
    return results
```

---

## Common Mistakes

### 1. Not Handling "No Answer" Cases
```python
# WRONG - always returning an answer
answer = qa_pipeline(question=q, context=c)
return answer["answer"]  # Might be garbage if answer isn't in context!

# RIGHT - check confidence
answer = qa_pipeline(question=q, context=c)
if answer["score"] < 0.3:
    return "I don't have enough information to answer this question."
return answer["answer"]
```

### 2. Ignoring Context Window Limits
```python
# WRONG - stuffing 10,000 tokens into a 512-token model
inputs = tokenizer(very_long_context + question, truncation=True)
# The answer might be in the truncated part!

# RIGHT - sliding window or hierarchical approach
# Split long documents, process each chunk, aggregate results
```

### 3. Not Deduplicating Retrieved Chunks in RAG
```python
# WRONG - retrieved chunks may be near-duplicates
contexts = retriever.retrieve(query, top_k=10)
# Might get 5 very similar paragraphs from nearby document sections

# RIGHT - deduplicate by similarity threshold
def deduplicate_contexts(contexts, threshold=0.85):
    from sentence_transformers import util
    unique = [contexts[0]]
    for ctx in contexts[1:]:
        is_duplicate = False
        for existing in unique:
            sim = util.cos_sim(embed(ctx), embed(existing))
            if sim > threshold:
                is_duplicate = True
                break
        if not is_duplicate:
            unique.append(ctx)
    return unique
```

### 4. Evaluation with Wrong Metric
```python
# WRONG for summarization - using BLEU (designed for translation)
# WRONG for QA - using ROUGE (designed for summarization)

# RIGHT:
# QA → Exact Match + F1
# Summarization → ROUGE + BERTScore
# Generation quality → Human evaluation + faithfulness metrics
```

### 5. Not Handling Multi-Answer Questions
```python
# Some questions have multiple valid answers
# "Who founded Apple?" → "Steve Jobs" or "Steve Wozniak" or "Steve Jobs and Steve Wozniak"

# RIGHT - compare against all valid answers, take the max score
def evaluate_multi_answer(prediction, ground_truths):
    """Evaluate against multiple valid answers"""
    scores = [compute_f1(prediction, gt) for gt in ground_truths]
    return max(scores)
```

### 6. Chunk Size Too Large or Too Small in RAG
```python
# TOO SMALL (50 tokens) - loses context, fragments sentences
# TOO LARGE (2000 tokens) - too much noise, dilutes relevant info

# GOOD DEFAULTS:
# - General text: 300-500 tokens with 50-100 overlap
# - Technical docs: 500-800 tokens with 100-200 overlap
# - Code: 100-300 tokens with 50 overlap

# RULE: chunk_size should match your typical answer length
```

---

## Interview Questions

### Conceptual

**Q1: Explain the difference between extractive and abstractive QA.**
> Extractive QA identifies an exact span from the context as the answer — it highlights existing text. Abstractive QA generates new text that may paraphrase or synthesize information from the context. Extractive is simpler, more reliable (no hallucination), but limited to verbatim answers. Abstractive can provide more natural answers but risks generating incorrect information.

**Q2: How does BERT solve extractive QA?**
> BERT takes [CLS] + question + [SEP] + context + [SEP] as input. Two linear layers predict start and end logits for each token. The answer span is the substring between the highest-scoring start and end positions (with start ≤ end constraint). Training uses cross-entropy loss on the correct start and end positions.

**Q3: What is RAG and why is it important?**
> RAG (Retrieval-Augmented Generation) combines a retriever (finds relevant documents from a knowledge base) with a generator (produces answers using retrieved context). It's important because: (1) reduces hallucination by grounding answers in documents, (2) knowledge can be updated without retraining, (3) enables source citation, (4) works with proprietary/private data.

**Q4: How would you evaluate a summarization system?**
> Automatic metrics: ROUGE (overlap-based), BERTScore (semantic similarity), METEOR. Human evaluation: fluency, coherence, relevance, faithfulness/factual consistency. For production systems, also measure: compression ratio, coverage of key points, and absence of hallucinated facts. No single metric captures all aspects.

**Q5: Explain the difference between ROUGE-1, ROUGE-2, and ROUGE-L.**
> ROUGE-1 measures unigram overlap (individual word matching). ROUGE-2 measures bigram overlap (word pair matching, captures some phrase-level similarity). ROUGE-L measures the longest common subsequence, capturing sentence-level structure. ROUGE-2 is generally most correlated with human judgments for multi-sentence summaries.

### Practical

**Q6: Your RAG system sometimes returns hallucinated answers. How do you fix it?**
> 1. Add confidence thresholding — return "I don't know" for low-confidence answers
> 2. Implement faithfulness checking using NLI models
> 3. Improve retrieval quality (better embeddings, reranking, hybrid search)
> 4. Use smaller, more precise chunks
> 5. Add explicit instructions in the prompt: "Only use information from the context"
> 6. Post-processing: verify named entities and facts against retrieved docs

**Q7: How would you build a QA system for a company's internal documentation?**
> 1. Ingest all docs (PDFs, wikis, Confluence) using appropriate loaders
> 2. Chunk documents with domain-appropriate sizes and overlap
> 3. Create embeddings using a model fine-tuned on domain data
> 4. Store in a vector database (Pinecone, Weaviate, Qdrant)
> 5. Implement hybrid search (BM25 + semantic) for robust retrieval
> 6. Add reranking for precision
> 7. Use an instruction-tuned LLM as generator with guardrails
> 8. Add feedback loop for continuous improvement
> 9. Implement access controls matching the source documents

**Q8: How do you handle questions that need information from multiple documents?**
> Multi-hop QA approaches: (1) Iterative retrieval — answer first sub-question, use it to formulate next query, (2) Graph-based reasoning over document connections, (3) Large context windows that can fit multiple documents, (4) Decompose complex question into simpler sub-questions (query decomposition), (5) Chain-of-thought prompting to reason step by step.

---

## Quick Reference

### Model Selection for QA

| Task | Best Model | Notes |
|------|-----------|-------|
| Extractive QA (English) | `deepset/roberta-base-squad2` | Fast, handles unanswerable |
| Extractive QA (Multilingual) | `deepset/xlm-roberta-large-squad2` | 100+ languages |
| Abstractive QA | `google/flan-t5-large` | Good instruction following |
| Long-context QA | `allenai/longformer-base-4096` | Up to 4096 tokens |
| Open-domain QA | RAG (DPR + reader) | Scales to large KBs |

### Model Selection for Summarization

| Task | Best Model | Max Input |
|------|-----------|-----------|
| News summarization | `facebook/bart-large-cnn` | 1024 tokens |
| Extreme summarization | `google/pegasus-xsum` | 512 tokens |
| Long documents | `allenai/led-base-16384` | 16,384 tokens |
| Multi-document | `allenai/primera` | 4096 tokens |
| Instruction-based | `google/flan-t5-large` | 512 tokens |

### RAG Chunking Strategy

| Document Type | Chunk Size | Overlap | Separator |
|--------------|-----------|---------|-----------|
| General text | 300-500 | 50-100 | Paragraph/sentence |
| Legal docs | 500-800 | 100-200 | Section/paragraph |
| Code | 100-300 | 50 | Function/class |
| FAQs | 1 QA pair | 0 | QA boundary |
| Tables | Entire table | 0 | Table boundary |

### Key Formulas

| Metric | Formula | Range |
|--------|---------|-------|
| Exact Match | $EM = \mathbb{1}[pred = truth]$ | 0 or 1 |
| F1 (token) | $F1 = \frac{2 \cdot P \cdot R}{P + R}$ | [0, 1] |
| ROUGE-N | $\frac{\text{matching n-grams}}{\text{n-grams in reference}}$ | [0, 1] |
| BERTScore | Cosine similarity of contextual embeddings | [-1, 1] |

### Pipeline Architecture Patterns

```
Pattern 1: Simple QA
  Question → Model → Answer

Pattern 2: Extractive QA
  Question + Context → BERT → Start/End positions → Answer span

Pattern 3: RAG
  Question → Retriever → Top-K docs → Generator(question + docs) → Answer

Pattern 4: Multi-step RAG
  Question → Decompose → [Sub-Q1, Sub-Q2] → Retrieve each → 
  → Aggregate contexts → Generate final answer

Pattern 5: Conversational QA
  History + Question → Rewrite query → Retrieve → Generate → Update history
```

---

## Summary

| Component | Purpose | Key Tech |
|-----------|---------|----------|
| Extractive QA | Find answer span in text | BERT + start/end heads |
| Abstractive QA | Generate natural answer | T5, BART, GPT |
| Retrieval | Find relevant documents | Embeddings + Vector DB |
| RAG | Grounded generation | Retriever + Generator |
| Extractive Summarization | Select important sentences | TextRank, BERT |
| Abstractive Summarization | Generate concise summary | BART, Pegasus, T5 |
| Evaluation | Measure quality | EM, F1, ROUGE, BERTScore |
